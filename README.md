# Flight Controller (TakeOff)
# BLDC Flight Controller
<p align="center">
  <img src="https://github.com/user-attachments/assets/e346459f-61f0-44b9-9794-b8c8e9086e30" width="400" height="400"/>
  <img src="https://github.com/user-attachments/assets/02164593-6775-4292-92d6-750c4424bcbd" width="400" height="400"/>
</p>

An STM32F405-based Betaflight flight controller, designed in KiCad specifically for Takeoff. Designed a 4-layer, 38×38mm STM32F405RGT6-based flight controller PCB from schematic to fabrication-ready Gerbers for a 20 to 30 unit drone build program at Toronto Metropolitan University.

Current Progress: Ordered an initial test set, waiting for delivery from JLCPCB (08/24/2026)

This README follows the same block structure as the schematic sheet.

---

## Specifications

| | |
|---|---|
| MCU | STM32F405RGT6, LQFP-64 |
| Board size | 38 × 38 mm |
| Mounting | 30.5 × 30.5 mm |
| Stackup | 4 layer, F.Cu signal / In1.Cu GND / In2.Cu 3V3 / B.Cu signal |
| Finish | ENIG |
| Assembly | Single sided |
| Input | 3S LiPo |
| IMU | ICM-42605 (U5), SPI1 |
| Baro | DPS368 (U6), I2C1 |
| Blackbox | W25Q128JVS (U8), 16 MB, SPI3 |
| Motor outputs | 4, DShot capable |
| Receiver | UART1 on J3, pluggable |
| USB | USB-C, ESD protected |

---

## Microcontroller
<p align="center">
<img width="766" height="597" alt="image" src="https://github.com/user-attachments/assets/acd9029c-c7d1-4309-8750-9f86e93a33dd" />
</p>

### Decoupling and Filtering

C7 to C13 give one 100nF capacitor for each VBAT and VDD pin as per the datasheet, and C5 is required as it needs one 4.7µF capacitor regardless of the number of VDD pins. On the right side is the analog supply, where the 3.3V passes through the ferrite bead (FB1) which filters the extra mV of switching noise by having a high impedance at high frequency and becomes 3.3VA, which supplies power for the VDDA. C19 and C21 are from the datasheet's ADC area figure. The purpose of C19 and C21 is to take that switching noise towards ground through the two capacitors. The purpose of these capacitors is to supply charge when the MCU needs it suddenly, and it's super close to it rather than waiting to get it from the power source. The other function is to also manage the chip's current spikes from causing too much noise as mentioned earlier.

C1 and C3 (2.2µF) go on VCAP_1 and VCAP_2 to GND, not to +3.3V.

### NRST

The internal pull-up resistor is only 40kΩ, which is a high impedance. Adding a 10kΩ resistor (R10) makes it about 8k combined in parallel, allowing for lower impedance and therefore picking up less noise. C23 (0.1µF) to ground helps prevent resets from the noise of the circuit. This is a common RC reset pairing.

### BOOT0

BOOT0 holds low if the button (SW1) is not pressed, which means it boots from user firmware in flash. If the button is pressed, the pin now reads high at startup, which loads the DFU bootloader instead of flash, needed in case you need to recover through USB and reflash with new firmware. BOOT0 is only read at the instant of reset/power-up, so the button only matters at that moment, not afterward.

### Crystal

C = 2·(CL − C_stray) to find capacitance values.

Recommended specs for a flight controller: frequency tolerance ≤ ±20 ppm @ 25°C, temperature stability ≤ ±50 ppm over −20 to +85°C.

Y1, HCI0132M4-8000F18DZNGL, has CL = 18pF and C_stray = 4pF, giving C = 27pF (C17, C18).

This is what gives the STM chip a stable time reference, every cycle is run from this timing. Most MCUs work fine for simple tasks using their internal oscillator, but for applications where timing plays an important role, like PID loops, an external passive crystal is used.

### I2C Pull-ups

I2C pull-ups (R5, R6 at 4.7k) pull SDA and SCL high, as all the devices on the bus can only pull the lines low. Without pull-up resistors, the lines would float at an undefined voltage and the bus wouldn't function. The resistors pull SDA and SCL up to 3.3V, making the bus high by default, and devices pull it low to communicate. Since I2C is meant to be shared, the pull-ups are placed once, not duplicated, as multiple would lower the resistance and increase current draw.

### Status LEDs

Status LEDs are the fastest way to check if your flight controller is running correctly. D3 (blue, LED_1) and D4 (green, LED_2) are wired as sinking outputs, each sitting between +3.3V and its GPIO pin (PB13, PB14), so the MCU drives the pin low to light it, matching Betaflight's typical active-low LED convention. D5 (orange) is tied directly across +3.3V to GND with no GPIO involved, so it's lit whenever the board has power, a simple, always-on "board is alive" indicator.

---

## Power Supply
<p align="center">
<img width="1147" height="621" alt="image" src="https://github.com/user-attachments/assets/42fd7963-410b-4298-aac0-c1052695987a" />
</p>

### Reverse Voltage and Overvoltage Protection

To stop current from flowing when voltage is reversed, a P-channel MOSFET (Q1) is used with the drain connected to battery, source to the board, and a gate connected to ground through R1 (100k) and to SYS_VIN through R20 (100k). It would float randomly without the ground-side resistor, as it gives it a defined reference to pull against. If the polarity is correct, the body diode conducts first by pulling the source to the battery, and as the gate-to-source voltage goes negative, the MOSFET turns on. In reverse polarity, the body diode is facing the wrong way, so the source doesn't get pulled up, and with no way to develop a negative gate-to-source voltage, there's no way to turn the MOSFET on.

The TVS diode (D1) protects against voltage spikes. It sits between SYS_VIN and ground, and if a transient spike pushes past the diode breakdown point around 20 to 22V, it conducts hard, directing the current toward ground and stopping the voltage before it can go higher, protecting the buck converter's max 30V limit.

### 3.3V LDO (General)

This LDO (U3) steps the 5V rail down to 3.3V for the MCU, flash, barometer, and status LEDs, everything on the board except the gyro. It was chosen over powering these components straight off the buck's 5V output because most digital logic on the MCU needs a stable 3.3V, and over powering off the battery directly, which would waste far more power as heat given the larger voltage drop. VIN and EN are tied together so the regulator is always on once 5V is present, with input and output decoupling caps (C14, C16) placed right at the pins per the datasheet.

### 3.3V LDO (IMU)

The gyro gets its own dedicated LDO (U4), separate from the general 3.3V rail, purely for noise isolation. The general LDO's rail carries switching noise from the MCU's digital logic and other peripherals, which would show up directly as noise on the gyro's supply if it shared that rail, and gyro noise translates into noisier sensor readings and heavier filtering in Betaflight. The LP5912 was chosen specifically for its low output noise, keeping the IMU's power supply as clean as possible. Current draw on this rail is negligible, so the part was selected on noise performance rather than current capacity.

### 5V 2A Buck Converter

U2 steps SYS_VIN (battery voltage, up to 12.6V on 3S) down to 5V to power the receiver and both 3.3V LDOs. A buck converter was used instead of a linear regulator because the voltage drop from battery to 5V is too large to handle efficiently with an LDO, the wasted power would show up as heat rather than usable output. L1 (3.3µH) and C15 (47µF effective) come from TI's recommended LC table for a 5V output at 1.2MHz switching frequency; the feedback divider (R2/R3) sets the output voltage via Vout = Vref × (1 + R2/R3). RT is tied to GND to select 1.2MHz switching, and SS (C8, 33nF) sets a ~5ms soft-start ramp on power-up.

EN (pin 2) is left floating rather than tied to SYS_VIN. The TPS62933's EN pin has a 6V absolute maximum, and SYS_VIN reaches 12.6V at full 3S, so tying them together would put the pin at over twice its rated limit. The datasheet confirms the part enables itself via an internal pull-up current source with EN left open, so floating EN is a documented mode, not a workaround.

### Battery Voltage Measurement

Since the MCU's ADC can only safely read 0 to 3.3V, R8/R9 divide SYS_VIN down to a safe range before it reaches VBAT_SENSE. C22 (100nF) filters switching noise from the buck converter off the sense line, keeping the voltage reading stable rather than jittery. At full charge, the ADC should see about 1.1V.

---

## Sensors and Flash Memory
<p align="center">
<img width="1076" height="452" alt="image" src="https://github.com/user-attachments/assets/f7737ae2-4b39-4939-b638-ff7221152a87" />
</p>

### IMU

The gyro went through a few iterations before landing here. BMI270 was the original pick for its low cost, but Betaflight's own manufacturer guidelines explicitly discourage it for new designs, as its gyro is uncalibrated, which can cause angle drift. ICM-42688-P (Betaflight's current top recommendation) was considered but is significantly more expensive at genuine-part pricing, with cheaper "compatible" clone parts carrying real reliability risk. LSM6DSO was also considered as a near-free alternative, but multiple sources flag it for a higher noise floor and weaker track record specifically in flight-controller applications. ICM-20602 was chosen as the middle ground: it falls under Betaflight's "ICM2060X" table row (8kHz gyro sampling, 4kHz PID loop on F405 with bidirectional DShot), has a genuine multi-year track record in FPV flight controllers, and adds only a modest cost premium over BMI270.

That part later went out of stock, and the board now carries an ICM-42605 (U5) instead. This is not a footprint-compatible swap, the whole ICM-426xx family is LGA-14 (2.5 × 3.0mm), not LGA-16 (3 × 3mm) like the ICM-20602, so the footprint and layout around U5 were redone from scratch.

Decoupling follows the datasheet's typical operating circuit: VDD gets 0.1µF (C25) + 2.2µF (C26) in parallel, VDDIO gets 10nF (C27). FSYNC is tied to GND since Betaflight doesn't use frame-sync functionality. The RESV pins (2, 3, 7, 10, 11) are tied to GND per the datasheet requirement, worth checking these directly against the pin table and footprint rather than trusting the drawn symbol, since this footprint came from UltraLibrarian and that source has had pin-type errors before.

### Barometer

Chosen on Betaflight's own explicit recommendation, their documentation states plainly that the DPS310 chip is much better than the older BMP280, and the DPS368 is better again. It's worth sourcing the genuine Infineon part specifically, since Betaflight's docs also warn that DPS310/368 clones are common and tend to have poorer temperature compensation.

Runs on I2C1, shared with any future I2C peripherals on the board. SDO is tied to GND through R16 (10kΩ) to set the I2C address to 0x76, this was originally a direct-to-GND wire, but was changed to go through a resistor after ERC flagged a net conflict (SDO is an Output-type pin, and tying it directly to a net carrying a PWR_FLAG creates a driver conflict). Functionally identical, and arguably better practice since it limits current if the pin is ever accidentally driven. VDD and VDDIO both get 100nF decoupling (C30, C31) directly at the pins, and GND_1/GND_2 are tied together.

CSB_N is tied to +3.3V rather than left as a no-connect. Per the DPS368 datasheet, floating is legal in I2C mode because of the internal pull-up, but that pull-up is weak (60k–180k) and the part has a one-way trapdoor: an active low glitch on CSB switches it to SPI mode until the next power-on reset. Tying it high avoids that failure mode for free.

### 16MB Flash

Used for Betaflight's blackbox logging, recording flight data (gyro, motor output, RC input) to onboard storage for post-flight analysis and tuning. Runs on SPI3, dedicated to this chip alone. CS, CLK, DI, and DO connect to SPI3's CS, SCK, MOSI, and MISO respectively. WP# and HOLD#/RESET# are both tied directly to +3.3V, both pins double as data lines (IO2, IO3) in Quad SPI mode, but since this board only uses standard SPI, they're held high per the datasheet's requirement that both be pulled to VCC when unused, rather than left floating where they could cause read/write glitches. A single 100nF cap (C34) decouples VCC to GND, placed as close to the pins as possible per Winbond's recommendation.

FLASH_CS (R19, 10k pull-up to +3.3V) prevents the CS line from floating during MCU reset.

---

## Connectors and Solderpads
<p align="center">
<img width="702" height="449" alt="image" src="https://github.com/user-attachments/assets/925c1778-c1a0-4f29-9704-a973d6b19ccd" />
</p>

### ESC

Carries battery power and motor signals down to the 4-in-1 ESC sitting below the FC in the stack. Pin 1 (+BATT) taps SYS_VIN, which is pre-buck and close to raw battery voltage, since the ESC needs full battery voltage to drive the motors rather than the board's regulated 5V rail. Pins 3 through 6 carry the DShot motor signals (M1–M4, mapped to PB0/PB1/PA3/PA2), and pin 7 carries ESC_CURRENT, the ESC's current-sense output, filtered through C29 (10nF) before reaching PA1 on the MCU.

There's no dedicated telemetry pin here. Since Bluejay (flashed on the ESC) supports bidirectional DShot, RPM, voltage, and temperature telemetry ride back over the same motor signal wires instead of needing a separate line, so this connector's pinout (V, G, 1–4, C) matches the physical ESC exactly with nothing extra tacked on. The mounting pad ties to GND for mechanical and shielding purposes. Backup solder pads mirror every signal on this connector, giving a fallback wiring path if the connector itself ever gets damaged.

### Receiver

Connects the ELRS receiver to USART1. Pins 3 and 4 (UART1_TX_RX, UART1_RX_RX) are crossed to the MCU's PA9/PA10 so the FC's TX reaches the receiver's RX and vice versa. A receiver's TX and the FC's TX can't just be wired straight across, since two transmit pins have nothing to say to each other. Power comes from +5V (pin 2) rather than 3.3V, matching typical receiver voltage requirements. This project shares a pool of ELRS receivers across multiple team boards instead of dedicating one per board, so this header uses a pluggable connector rather than solder pads, letting a receiver move quickly between boards. Combined with an ELRS bind phrase set once across the shared receiver pool, swapping a receiver onto a different board needs no rebinding at all, just unplug and replug. Same backup test point pattern as the ESC connector, and the mounting pad ties to GND.

---

## USB-C
<p align="center">
<img width="1036" height="428" alt="image" src="https://github.com/user-attachments/assets/25c1741b-bab0-4407-9e5d-c319767c9024" />
</p>
Provides an alternate 5V power source and USB data path (via PA11/PA12 for DFU flashing) alongside battery power. Uses the USB2.0_16P symbol variant, which exposes only the pins relevant to USB2.0 speed (VBUS, GND, CC1, CC2, D+, D-) and leaves out the SuperSpeed pins found on full USB-C receptacles, since the F405's USB peripheral only runs full-speed anyway.

CC1/CC2 pull down through R14/R15 at 5.1k, not the 1k that was originally sourced. That earlier value would have broken USB enumeration outright, caught during BOM review before ordering.

### ESD Protection

Sits between the USB-C connector and the MCU's USB pins to clamp any electrostatic discharge before it reaches PA11/PA12. USB-C ports are physically exposed, so every time someone plugs in a cable or touches the connector shell, there's a real chance of a static discharge hitting those pins directly, and the MCU's USB transceiver isn't built to survive a multi-kilovolt spike on its own.

Pins 1 and 6 are both internally the same net (D-), and pins 3 and 4 are both internally the same net (D+). The chip's SOT-23-6L package breaks each protected line out to two physical pins on opposite sides, which lets the incoming trace from the connector land on one side and the outgoing trace to the MCU leave from the other, keeping both runs short. Short traces matter here because the chip's ESD clamping performance depends on parasitic inductance staying low between the connector and the MCU.

### VBUS Filtering

This is the second half of the OR-ing setup that lets USB and battery power coexist on the shared 5V rail without one feeding back into the other. FB2, a 220Ω@100MHz ferrite bead, sits between VBUS and the rest of this stage, acting as a high-impedance path at high frequency while staying low-impedance at DC. This filters USB switching noise off VBUS before it can couple onto the 5V rail, the same idea as the ferrite bead used on the VDDA supply for the MCU. C32 (4.7µF) and C33 (100nF) provide bulk and high-frequency decoupling right at this node, same bulk-plus-small-ceramic pattern used everywhere else on the board.

D6 (a Schottky diode) is what actually does the OR-ing. It sits between VBUS_FILTERING and the 5V rail, oriented so current can only flow from USB toward 5V, never backward. This matters for two reasons. If the board is running on battery power and someone plugs in a USB cable that isn't providing power, the diode stops the buck converter's 5V output from flowing backward out the USB port. And if USB is providing power while the battery is also connected, the diode lets USB supply the 5V rail without the buck converter's output fighting it on the same node. A Schottky diode specifically was chosen over a regular silicon diode for its lower forward voltage drop (typically 0.2 to 0.3V versus 0.6 to 0.7V), which wastes less voltage headroom on a rail that's already just 5V.

This does not mean connect a laptop directly to the FC while it's powered by battery. This does not provide full galvanic isolation.

---

## Misc.
<p align="center">
<img width="319" height="440" alt="image" src="https://github.com/user-attachments/assets/8eac6da1-1e25-48ef-9e62-a1308a0c3558" />
</p>
### External Buzzer

Betaflight's beeper output for low-battery warnings and the "lost model" finder function. BUZZER_PIN (PA0) drives the gate of Q2 through R18 (100Ω), a gate resistor that damps the switching edge and suppresses ringing from parasitic inductance. R17 (100kΩ) pulls the gate to GND by default, so the MOSFET stays off during MCU power-up or reset, before firmware has configured PA0 as an output. An unconfigured GPIO floats, and a floating gate could otherwise pick up noise and turn the buzzer on unexpectedly. BZ1's positive lead ties to +5V permanently, and its negative lead ties to Q2's drain. When BUZZER_PIN drives high, the gate voltage rises enough to turn the MOSFET on, completing the path from +5V through the buzzer to GND. When it drives low, R17 pulls the gate back down and the buzzer goes silent. Q2 acts purely as a low-side switch here, its gate never carries the buzzer's actual current, it only controls whether the drain-to-source path is open or closed.

### Debug Test Points

Bare probe points for SWDIO, SWCLK, NRST, 3.3V, and 5V, giving a fallback way to program or debug the board with an ST-Link if the USB-C connector ever fails or DFU mode isn't available. These are deliberately left as plain test points rather than a soldered header, since SWD access here is a backup path, not the primary programming method (that's USB DFU), and a header would cost board space and BOM for something used only occasionally. A pogo-pin probe or spring-loaded clip (Tag-Connect style) is the expected way to access these during actual debugging sessions.

---

## Net and Pin Map

| Function | Pin | Net |
|---|---|---|
| Motor 1–4 | PB0, PB1, PA3, PA2 | MOTOR1–4 |
| Gyro SPI1 | PA5, PA6, PA7, PC4, PC5 | SPI1_SCK, SPI1_MISO, SPI1_MOSI, CS_IMU, IMU_EXT1 |
| Flash SPI3 | PC10, PC11, PC12, PB3 | SPI3_SCK, SPI3_MISO, SPI3_MOSI, FLASH_CS |
| Baro I2C1 | PB6, PB7 | I2C_SCL, I2C_SDA |
| Receiver UART1 | PA9, PA10 | UART1_TX_RX, UART1_RX_RX |
| USB | PA11, PA12 | USBC_MCU−, USBC_MCU+ |
| Battery sense | PC3 | VBAT_SENSE |
| ESC current | PA1 | ESC_CURRENT |
| Buzzer | PA0 | BUZZER_PIN |
| Status LEDs | PB13, PB14 | LED_1, LED_2 |
| SWD | PA13, PA14 | SWDIO, SWCLK |
| Crystal | PH0, PH1 | HSE_IN, HSE_OUT |

---


## Tools

KiCad for schematic and PCB layout. STM32CubeMX for pin mapping, clocks, and DMA. JLCPCB and LCSC for fabrication and sourcing, with jlcsearch.tscircuit.com for part lookup.

---

