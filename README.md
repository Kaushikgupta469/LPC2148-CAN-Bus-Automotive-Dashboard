# 🛰️ LPC2148 CAN Bus Automotive Dashboard

![MCU](https://img.shields.io/badge/MCU-LPC2148-brightgreen?style=flat-square)
![Architecture](https://img.shields.io/badge/Architecture-ARM7TDMI-blue?style=flat-square)
![Protocol](https://img.shields.io/badge/Protocol-CAN%20Bus-orange?style=flat-square)
![Language](https://img.shields.io/badge/Language-Embedded%20C-lightgrey?style=flat-square)
![HAL](https://img.shields.io/badge/HAL-None-red?style=flat-square)

A two-node CAN bus communication system simulating an automotive Electronic Control Unit (ECU) network using the **LPC2148 (ARM7TDMI)** microcontroller.

Node A reads physical inputs and transmits CAN frames, while Node B receives the frames through interrupts and controls an LCD display and GPIO-connected LEDs. The project implements peripheral configuration and CAN communication at the hardware-register level, without relying on HALs or high-level driver libraries.

## 📐 System Architecture

```text
┌─────────────────────────────────┐         ┌─────────────────────────────────┐
│             NODE A              │         │             NODE B              │
│        Transmitter ECU           │         │         Receiver ECU            │
│                                 │         │                                 │
│ EINT0 → Right Indicator         │         │ CAN2 RX ISR receives frames     │
│ EINT1 → Left Indicator          │         │ Decode frames by CAN ID         │
│ EINT2 → Headlight               │         │ LCD → Vehicle speed             │
│ Timer1 → ADC sampling flag      │         │ GPIO P0.17 → Left indicator     │
│                                 │         │ GPIO P0.18 → Headlight          │
│ Main loop polls event flags     │◄───────►│ GPIO P0.19 → Right indicator    │
│ CAN2 transmits data frames      │ CAN Bus │ LED blinking and display update │
│                                 │ 125 kbps│                                 │
└─────────────────────────────────┘         └─────────────────────────────────┘
          LPC2148 #1                                  LPC2148 #2
```

### Communication Overview

- **Node A — Transmitter:** Monitors external interrupts and periodically samples analog inputs.
- **CAN Bus:** Transfers indicator states, simulated vehicle speed, and engine temperature data at a configured bitrate of 125 kbps.
- **Node B — Receiver:** Receives CAN messages through an interrupt service routine, decodes message identifiers, and updates the LCD and GPIO outputs.
- **Hardware-level implementation:** Uses direct register access for CAN, ADC, timers, external interrupts, GPIO, and the Vectored Interrupt Controller (VIC).

## 📡 CAN Frame Map

| CAN ID | Signal | Data | Trigger | Receiver Action |
|---|---|---|---|---|
| `0x212` | Left indicator | `0x2` ON / `0x3` OFF | EINT1 | Controls left indicator on P0.17 |
| `0x213` | Headlight | `0x4` ON / `0x5` OFF | EINT2 | Controls headlight on P0.18 |
| `0x214` | Right indicator | `0x6` ON / `0x7` OFF | EINT0 | Controls right indicator on P0.19 |
| `0x215` | Vehicle speed | ADC channel 2 reading | Timer1 | Displays vehicle speed on the LCD |
| `0x216` | Engine temperature | ADC channel 1 reading | Timer1 | Reserved for temperature display |

### Vehicle Speed Calculation

The ADC reading is scaled to a simulated speed ranging from 0 to 100 km/h.

\[
\text{Speed} = \frac{\text{ADC Reading}}{1023.0} \times 100
\]

The receiver displays the calculated value in the following format:

```text
SPEED: XX Kmph
```

A potentiometer connected to ADC channel 2 simulates the vehicle speed input, while another potentiometer on ADC channel 1 simulates engine temperature.

> **Note:** These are simulated signals generated using potentiometers, not measurements from an actual vehicle.

## 🔧 Key Register Configuration

The project configures the LPC2148 peripherals through direct memory-mapped register access.

### 1. CAN2 Bit Timing

```c
/* CAN2 bit timing configuration */
C2BTR = 0x001C001D;
```

The intended CAN bitrate is 125 kbps with a 15 MHz peripheral clock. The actual bitrate should be verified against the LPC2148 CAN bit-timing register definitions and the clock configuration used by the firmware.

### 2. Acceptance Filter

Node A bypasses the acceptance filter to receive all CAN messages:

```c
AFMR = 0;
```

Node B configures the acceptance filter to receive the relevant CAN identifiers.

```c
SFF_sa = 0;
ENDofTable = 0xC;
SFF_GRP_sa = 0xC;
EFF_sa = 0xC;
EFF_GRP_sa = 0xC;

AFMR = 2;
```

The filter table is intended to restrict received traffic to the configured automotive dashboard message IDs. Verify the table entries and acceptance-filter mode against the LPC2148 user manual before relying on this configuration.

### 3. External Interrupts

EINT0, EINT1, and EINT2 are configured for falling-edge-triggered events.

```c
EXTMODE |= 0x7;
EXTPOLAR = 0;
```

Each interrupt service routine updates an event flag, allowing the main loop to handle the corresponding CAN transmission.

### 4. Timer1 Configuration

Timer1 generates periodic interrupts to schedule ADC sampling.

```c
T1PR  = pclk - 1;
T1MR0 = 1;
T1MCR = 3;
```

The intended prescaler produces a 1 µs timer tick when the peripheral clock is configured appropriately. The match value must be selected according to the required interval; a match value of `1` does not by itself produce a one-second interval.

### 5. ADC Configuration

The ADC is configured for analog input sampling, with the result extracted from the ADC data register.

```c
ADCR |= (1 << ch_num);
ADCR |= (1 << 24);

while ((ADDR & (1U << 31)) == 0);

result = (ADDR >> 6) & 0x3FF;
```

The selected channel, ADC clock divider, pin configuration, and conversion-start sequence must be configured correctly for the chosen LPC2148 ADC peripheral.

## ⚡ Interrupt Architecture

### Node A — Transmitter

| VIC Slot | Interrupt Handler | IRQ Source | Purpose |
|---|---|---|---|
| Slot 0 | `EINT1_Handler` | EINT1 | Sets the left indicator event flag |
| Slot 1 | `EINT0_Handler` | EINT0 | Sets the right indicator event flag |
| Slot 2 | `TIMER1_Handler` | Timer1 | Schedules periodic ADC sampling |
| Slot 3 | `EINT2_Handler` | EINT2 | Sets the headlight event flag |

### Node B — Receiver

| VIC Slot | Interrupt Handler | IRQ Source | Purpose |
|---|---|---|---|
| Slot 2 | `CAN2_RX_Handler` | CAN2 RX | Receives and processes incoming CAN frames |

The interrupt handlers follow the LPC21xx IRQ convention. Peripheral interrupt sources are acknowledged as required, and the VIC interrupt is completed by writing to `VICVectAddr`.

```c
VICVectAddr = 0;
```

**Design principle:** Keep interrupt handlers short and perform application-level processing in the main loop wherever practical.

## 📁 Project Structure

```text
LPC2148-CAN-Bus-Automotive-Dashboard/
│
├── NodeA/                         # Transmitter ECU
│   ├── main.c                     # Main loop and CAN TX dispatch
│   ├── can_drive.c                # CAN2 initialization and communication
│   ├── adc_driver.c               # ADC initialization and sampling
│   ├── timer_interrupt.c          # Timer1 and VIC configuration
│   ├── eint0.c                    # Right indicator interrupt
│   ├── eint1.c                    # Left indicator interrupt
│   ├── eint2.c                    # Headlight interrupt
│   ├── delay.c                    # Delay utilities
│   └── header.h                   # Shared declarations and definitions
│
├── NodeB/                         # Receiver ECU
│   ├── main.c                     # CAN RX processing and output control
│   ├── can2_rx.drive.c            # CAN2 initialization and filtering
│   ├── can_tx.c                   # CAN RX interrupt and VIC setup
│   ├── lcd_drive.c                # LCD control and custom glyphs
│   ├── delay.c                    # Delay utilities
│   └── header.h                   # Shared declarations and definitions
│
└── README.md
```

*The file names above reflect the intended project structure. Adjust them if the actual repository uses different names.*

## 🛠️ Hardware Requirements

| Component | Quantity | Purpose |
|---|---:|---|
| LPC2148 development board | 2 | Transmitter and receiver ECUs |
| CAN transceiver | 2 | Interfaces the MCU CAN controller with the CAN bus |
| 16×2 LCD | 1 | Displays simulated vehicle speed |
| Push buttons | 3 | Trigger indicator and headlight events |
| LEDs | 3 | Represent left indicator, headlight, and right indicator |
| 10 kΩ potentiometers | 2 | Simulate speed and engine temperature |
| 120 Ω resistors | 2 | CAN bus termination |
| Jumper wires | As required | Connect peripherals and power |
| CAN twisted-pair cable | As required | CANH and CANL communication |

### CAN Transceiver Compatibility

Select a transceiver compatible with the board's supply voltage and logic levels. For example, the MCP2551 is a 5 V transceiver, whereas the SN65HVD230 is designed for 3.3 V operation. Check the specific module's specifications before connecting it to the LPC2148.

### CAN Bus Wiring

Connect the two nodes as follows:

| Node A | Node B |
|---|---|
| CANH | CANH |
| CANL | CANL |
| GND | GND |

Install a 120 Ω termination resistor at each physical end of the bus. With two 120 Ω terminators connected correctly, the unpowered bus should measure approximately 60 Ω between CANH and CANL.

**Important:** Never connect the MCU's CAN controller pins directly to CANH and CANL. A suitable CAN transceiver is required.

## 🚀 Build and Flash

This project targets the **Keil µVision** toolchain or another compatible ARM7TDMI C development environment.

1. Install Keil µVision and the appropriate LPC2148 device support.
2. Open the Node A and Node B projects separately.
3. Add the relevant `.c` files and header files to each project.
4. Configure the target device as LPC2148.
5. Verify the crystal frequency, PLL configuration, peripheral clock, and CAN bit timing.
6. Build both projects and resolve any compilation or linker errors.
7. Flash Node A and Node B using a supported JTAG interface or ISP programmer, such as Flash Magic.
8. Connect both boards through their CAN transceivers, CANH, CANL, and a common ground.
9. Power both nodes and verify CAN communication.
10. Trigger the push buttons and rotate the potentiometers to test the indicator, headlight, and speed-display functionality.

### Expected Behaviour

- The LCD displays the simulated vehicle speed.
- The left indicator responds to CAN ID `0x212`.
- The headlight responds to CAN ID `0x213`.
- The right indicator responds to CAN ID `0x214`.
- ADC channel 1 data is transmitted for the simulated engine temperature.
- ADC channel 2 data is transmitted for the simulated vehicle speed.

Actual behaviour depends on the firmware implementation, pin assignments, clock configuration, and CAN acceptance-filter setup.

## 🧪 Testing and Validation

The following checks can be used to validate the system:

- [ ] Verify CANH/CANL wiring and bus termination.
- [ ] Confirm both nodes use the same CAN bitrate.
- [ ] Verify the CAN2 peripheral clock and bit-timing registers.
- [ ] Confirm external interrupts respond to button presses.
- [ ] Check that the main loop transmits the correct CAN identifiers.
- [ ] Confirm Node B receives frames through the CAN interrupt.
- [ ] Test the left indicator, headlight, and right indicator independently.
- [ ] Verify ADC readings change when the potentiometers are adjusted.
- [ ] Confirm the LCD displays the expected speed values.
- [ ] Verify the acceptance filter accepts the intended identifiers.

## 🎯 Project Objectives

- Implement two-node automotive ECU communication using CAN.
- Configure the LPC2148 CAN peripheral at the register level.
- Demonstrate interrupt-driven CAN frame reception.
- Integrate ADC sampling, external interrupts, and timer-based events.
- Interface an LCD and GPIO-controlled LEDs with the receiver ECU.
- Understand low-level embedded C programming and peripheral configuration.

## 🔮 Future Improvements

- Add engine temperature conversion and LCD display.
- Implement CAN communication timeouts and error handling.
- Add message counters and communication diagnostics.
- Improve indicator timing using timer interrupts instead of blocking delays.
- Add fault detection for disconnected nodes and invalid sensor readings.
- Integrate real automotive sensor data where suitable hardware is available.
- Extend the project with additional ECU signals and diagnostic messages.

## 👤 Author

**Kaushik Gupta** — Embedded and Software Engineer

GitHub: [@Kaushikgupta469](https://github.com/Kaushikgupta469)

## 📄 License

This project is intended to be distributed under the [GNU General Public License v3.0 (GPL-3.0)](https://www.gnu.org/licenses/gpl-3.0.html) .

---

*Built to explore bare-metal embedded systems, CAN bus communication, and automotive ECU architecture using the LPC2148 microcontroller.*
