P80C550-EVN (Rev 1.0)
====================

The P80C550-EVN is a small 8051 single board computer designed around the notion of sourcing the major components from the random collection of electronics I've accumulated over the years. It started as a breadboard computer I built after being inspired to resume my electronics hobby after watching Ben Eater's YouTube series. After carting the breadboard computer around enough for the wires to start randomly loosening, I figured I might as well make a proper PCB for the project.

Features
--------
- CPU: P80C550 @ 11.0592 MHz
- 32 KiB ROM
- 32 KiB SRAM
- Expansion Bus
- HD44780 LCD connector
- 8x 8 bit ADC channels (from CPU)
- AM85C30 Dual UART or SDLC controller
- PCF8584 I2C controller
- MC146818 RTC and RAM with battery backup
- Runs off 5V, or 7V to 18V
- Configurable battery saver with CPU interrupt

Revision History
----------------
Revision 1.0
  - First Board Produced
  - Design and/or Production Errors
    - MAX693 RST signal (active-high) is open collector. This was not mentioned in the chip's datasheet, therefore no pull-up was designed into the board. [#1](https://github.com/teknoman117/P80C550-EVN/issues/1)
    - MAX693 ~RST signal (active-low, wired-OR) does not propagate to RST (active-high). Pushing the reset button will reset only the peripherals and not the CPU. [#2](https://github.com/teknoman117/P80C550-EVN/issues/2)

The Hardware
============

![Assembled Without Expansion Headers](/Assets/P80C550-EVN-no-expansion.webp)

## Processor

The CPU is a ROM-less CMOS 8051 microcontroller made by Signetics in the late 1980's. The only major differences from the original are that the Signetics part integrates an 8 bit analog to digital converter (with an 8 channel mux on Port 1), a programmable watchdog timer, and is built on a CMOS process. By nature of being ROM-less, the majority of the I/O ports must be used for the external bus interface (Port 2 = A8..15, Port 0 = AD0..7, Port 3.6 and 3.7 are the ~RD and ~WR signals). This is both good and bad, as we must use extra chips to get GPIO functions, but allows us to use far more than the 2K/4K/8K of ROM the 8051 could be ordered with and the 128 bytes of internal RAM. You can also use EEPROM or Flash for code storage, which is much nicer than needing to pick up an EPROM eraser.

## Memory Map

The memory for the board is provided via two 28 pin JEDEC compliant memory sockets supporting up to 32 KiB each. The left socket is for the ROM and the right socket is for the external data memory. Typically, an SRAM would be installed into the data socket, but this could also be a ROM or Flash device (such as AT29Cxx) if memory beyond the 128 byte internal RAM and 50 byte RTC/CMOS RAM is not required. Both sockets are mapped in the lower half of their respective address spaces (ROM in lower 32 KiB of CODE memory and the RAM in the lower 32 KiB of XDATA memory).

The onboard peripherals are mapped in the 8 KiB of address space following the XDATA memory as 1 KiB chunks. This may seem wasteful, but it allows using a single '138 3-to-8 decoder for peripheral chip selects (active high enable is A15, active low enables are A14 and A13. A12 -> A10 are then used for the select input).

| Code Address   | Description |
| -------------- | ----------- |
| 0000h -> 7FFFh | 32 KiB ROM socket (by default - see [Expansion](#expansion)) |
| 8000h -> FFFFh | **unused**  |

| Data Address   | Description                                     |
| -------------- | ----------------------------------------------- |
| 0000h -> 7FFFh | 32 KiB XDATA socket                             |
| 8000h          | [Power Status Register](#power-status-register) |
| 8400h -> 8FFFh | **unused**                                      |
| 9000h -> 903Fh | [RTC](#rtc)                                     |
| 9400h -> 9403h | [UART / SDLC](#uart--sdlc)                      |
| 9800h -> 9801h | [I2C controller](#i2c)                          |
| 9C00h -> 9C01h | [LCD](#lcd)                                     |
| A000h -> FFFFh | **unused**                                      |

## Interrupts

The P80C550 has two external interrupt lines which can either be falling edge triggered or low level triggered. There is an internal pullup resistor on the CPU and peripherals with open collector interrupt outputs can be wire-OR'd together on the same interrupt line.

| Interrupt | Devices                |
| --------- | ---------------------- |
| EX0       | Power Fault, RTC, I2C  |
| EX1       | UART / SDLC            |

## Power and Battery Saver

The board can either be powered directly via the 5V rail, or by a 7V to 18V terminal block, which is regulated to 5V by an AP62150WU-7 buck regulator. The input voltage range means the board can be safely powered by
- 5 to 12 AA(A) batteries
- 6 to 12 cell NiMH batteries
- 2 to 4 cell Li-ion or Li-Poly batteries

There is also a holder for an *optional* CR1216, CR1220, or CR1225 3V coin cell to power the RTC. If omitted, RTC and CMOS state will be lost on power-off. A CR1216 is good for at least 2 months (and counting).

A MAX693 microprocessor supervisor is used to manage the power-on reset delay, the RTC backup power, a low power fault signal, and provide 5V rail undervoltage protection.

A potentiometer is provided to configure the low power threshold. A jumper ("PFI Input Select") is provided to select whether the source voltage is provided from the Vin terminal block or the 5V rail. The former is used in battery powered or other scenarios using the input regulator and the latter is used when the 5V rail is directly driven. A 5.1V zener diode protects from accidental overvoltage on the voltage reference inputs.

The MAX693 will report a low power fault when the reference input drops below **1.3V**. By default, this will trigger the EX0 interrupt on the P80C550. A single bit register ([Power Status Register](#power-status-register)) is provided to check the status of the power fault output of the MAX693 and to mask the interrupt it generates.

For scenarios directly powering the 5V rail, the voltage divider should be configured to output 1.444V at 5V. This will trigger the lower power fault if the rail drops to 4.5V, which is the minimum safe CPU operating voltage. The MAX693 will hold the CPU in a reset state if the 5V rail drops further to 4.4V.

In Vin powered scenarios, the AP62150WU-7 buck regulator will shut down when the reference input drops below **1.2V** and completely cut power to the board. The voltage divider should be configured such that power is cut when the voltage drops to the minimum voltage of your battery, or the minimum voltage of the regulator (about 6.5V).

To determine the output voltage of the voltage divider at a specific input voltage given a target cutoff voltage, use the following formula:

 > Vout = Vin * (1.2V / Vcutoff);

Vin is the current input voltage and Vcutoff is the desired voltage for the regulator to shut off. Vout is the voltage that the voltage divider should be configured to output at that input voltage. See the example configurations below. The emphasis on maximum voltages is to ensure you are aware that "fresh off the charger" batteries have a notably higher voltage than their nominal voltage (which they will drop to quickly), so be aware of this when tuning the cutoff reference.

### Example Voltage Divider Configurations
- Directly powering the 5V rail
  - PFI Input Select: 5V rail
  - Vcutoff = 4.5V (minimum **cpu** voltage)
  - Vout = **5V** * (1.3V / 4.5V) = 1.444V
    - Low Power Fault at 4.5V on 5V rail
    - CPU held in reset at 4.4V on 5V rail

- Li-Ion or Li-Poly Batteries (per cell: 4.2V maximum, 3.7V nominal, and 3.0V minimum)
  - PFI Input Select: Vin
  - 2 cell battery (8.4V maximum, 7.4V nominal, 6V minimum)
    - Vcutoff = 6.5V (minimum **regulator** voltage)
    - Vout = **8.4V** * (1.2V / 6.5V) = 1.550V
    - Vout = **7.4V** * (1.2V / 6.5V) = 1.366V
      - Low Power Fault at 7.04V on Vin (~12% charge)
      - Power cutoff at 6.5V on Vin (~7% charge)
  - 3 cell battery (12.6V maximum, 11.1V nominal, 9V minimum)
    - Vcutoff = 9V (minimum **battery** voltage)
    - Vout = **12.6V** * (1.2V / 9V) = 1.680V
    - Vout = **11.1V** * (1.2V / 9V) = 1.480V
      - Low Power Fault at 9.75V on Vin (~7% charge)
      - Power cutoff at 9V on Vin (~5% charge)
  - 4 cell battery (16.8V maximum, 14.8V nominal, 12V minimum)
    - Vcutoff = 12V (minimum **battery** voltage)
    - Vout = **16.8V** * (1.2V / 12V) = 1.680V
    - Vout = **14.8V** * (1.2V / 12V) = 1.480V
      - Low Power Fault at 13V on Vin (~7% charge)
      - Power cutoff at 12V on Vin (~5% charge)

- NiMH Batteries (per cell: 1.5V maximum, 1.2V nominal, and 1.0V minimum)
  - PFI Input Select: Vin
  - 7 cell battery (10.5V maximum, 8.4V nominal, 7.0V minimum)
    - Vcutoff = 7V (minimum **battery** voltage)
    - Vout = **10.5V** * (1.2 / 7V) = 1.800V
    - Vout = **8.4V** * (1.2 / 7V) = 1.440V
      - Low Power Fault at 7.58V on Vin (~5% charge)
      - Power cuttoff at 7V on Vin (~0% charge)

### Power Status Register
- Address: 8000h
- Read: Bit 7 is the ~PFO (Power Fault) line from the MAX 693 Supervisor
  - 0 = Low Voltage
  - 1 = Nominal Voltage
- Write: Bit 7 is the ~PFO (Power Fault) interrupt mask
  - 0 = Enable PFO interrupt
  - 1 = Disable PFO interrupt
- Bits 0 -> 6 undefined

## UART / SDLC

The AM85C30 "Enhanced Serial Communications Controller" can function as a UART or as an SDLC/HDLC controller. In UART mode, there is a 3 byte RX FIFO, but no TX FIFO. The SDLC/HDLC mode adds a 10 entry frame status FIFO, but the data FIFOs remain the same. The chip is intended to be used with a DMA engine, but this does not exist on the 8051 platform. If a deeper FIFO is required, there is a more modern (less old?) drop-in compatible version available called the Z85230, which appears to still in production. It adds a 4 byte TX FIFO and increases the RX FIFO to 8 bytes from 3 bytes.

| Data Address | Description         |
| ------------ | ------------------- |
| 9400h        | Channel B - Control |
| 9401h        | Channel B - Data    |
| 9402h        | Channel A - Control |
| 9403h        | Channel A - Data    |

## I2C

The PCF8584 is an I2C controller supporting a 100 KHz SCL frequency in either master or slave mode.

| Data Address | Description      |
| ------------ | ---------------- |
| 9800h        | Register Access  |
| 9801h        | Control          |

## RTC

The RTC is the classic MC146818 chip that was present in the PC/AT that defined the pinout and register layout that later parts such as the Dallas DS1285, DS12885, and DS1685 use. No special battery input, no integrated oscillator, etc. The MAX693 is used to avoid needing the diode setup specified in the manual. 

| Data Address   | Description                    |
| -------------- | ------------------------------ |
| 9000h          | Seconds                        |
| 9001h          | Seconds (Alarm)                |
| 9002h          | Minutes                        |
| 9003h          | Minutes (Alarm)                |
| 9004h          | Hours                          |
| 9005h          | Hours (Alarm)                  |
| 9006h          | Day of Week                    |
| 9007h          | Day of Month                   |
| 9008h          | Month                          |
| 9009h          | Year                           |
| 900Ah          | Register A                     |
| 900Bh          | Register B                     |
| 900Ch          | Register C                     |
| 900Dh          | Register D                     |
| 900Eh -> 903Fh | 50 bytes of Battery Backed RAM |

## LCD

![Assembled With LCD](/Assets/P80C550-EVN-assembled.webp)

Any LCD module compatible with the 16 pin HD44780 header may be used. The mounting holes are spaced for the popular LCD2004A module (20x4 characters) that can be found pretty much everywhere.

| Data Address | Description   |
| ------------ | ------------- |
| 9C00h        | LCD - Control |
| 9C01h        | LCD - Data    |

## PCB
| Front Side | Back Side |
| ---------- | --------- |
| ![Front](/Assets/pcb-front.webp) | ![Back](/Assets/pcb-back.webp) |

| Layout (Power Planes Not Shown) |
| ---------------------------------- |
| ![Layout](/Assets/pcb-layout.webp) |

The PCB is a 98 mm by 69 mm, 4-layer board (2 signal layers, one +5V plane, and one ground plane) following the design rules for the low cost JLCPCB 4-layer process. Using 4 layers made the routing far easier, as I didn't have to worry about getting power in and around all of the signal traces. At time time of writing, 5 boards using Leaded HASL and a green PCB material is $8 (USD). Switching to ENIG-RoHS increases the price to $23.40 (USD) for 5 boards. 10 boards is $29 (USD), as they seemingly charge a flat rate for switching to ENIG on small orders. I also opted for FR4 TG155 because it's more tolerant to hot air soldering.

To order, you can contact me at [hardware@nrlewis.dev](mailto:hardware@nrlewis.dev) and see if I have any laying around, or you can send the gerber files (attached to release tags) to your board house of choice. JLCPCB is my recommendation, as it seems that they have a lower 4-layer board tier available. PCBWay seems good as well, but their 4-layer process is a bit overkill for this board.

The smallest components on the PCB are 0603 passives. These are the smallest size I'm comfortable hand soldering. Call me crazy, but I assembled this whole board using a Weller WE1010NA soldering station with a (small point) chisel tip, flux core solder, a flux pen, some solder wick, and a pair of tweezers from an iFixit kit. Apply flux generously, hold the components onto the pads with tweezers, hold your breath, and apply solder.

## Expansion

All of the major bus signals are broken out to stackable headers to allow for stacked expansion boards.

### ROM Expansion

The ROM read signal (~CODE_RD) and chip select (~CODE_CS) are broken out to the address low-byte header. The ~CODE_CS pin has a pulldown to keep the ROM enabled when no expansion boards are installed. The main idea behind these signals is to enable the creation of a self-programming expansion. By default, the ROM socket occupies both the lower and upper 32 KiB of the code address space, as the chip select is always enabled and A15 is ignored. A self programming module would allow the onboard ROM be either relocated or disabled when switching to an external memory.

#### Address Low-Byte Header

| Pin | Description |
| --- | ----------- |
| 1   | GND         |
| 2   | +5V         |
| 3   | A0          |
| 4   | A1          |
| 5   | A2          |
| 6   | A3          |
| 7   | A4          |
| 8   | A5          |
| 9   | A6          |
| 10  | A7          |
| 11  | ~CODE_RD    |
| 12  | ~CODE_CS    |
| 13  | ~RST        |

#### Address High-Byte Header

| Pin | Description |
| --- | ----------- |
| 1   | A15         |
| 2   | A14         |
| 3   | A13         |
| 4   | A12         |
| 5   | A11         |
| 6   | A10         |
| 7   | A9          |
| 8   | A8          |
| 9   | ~WR         |
| 10  | ~RD         |
| 11  | +5V         |
| 12  | GND         |

#### LCD Header

| Pin | Description  |
| --- | ------------ |
| 1   | GND          |
| 2   | +5V          |
| 3   | LCD_VO       |
| 4   | A0           |
| 5   | ~WR          |
| 6   | LCD_E        |
| 7   | D0           |
| 8   | D1           |
| 9   | D2           |
| 10  | D3           |
| 11  | D4           |
| 12  | D5           |
| 13  | D6           |
| 14  | D7           |
| 15  | LCD_A (+5V)  |
| 16  | LCD_K        |

#### CPU Peripheral Header

| Pin | Description       |
| --- | ----------------- |
| 1   | UART_IEO          |
| 2   | CLK (11.0592 MHz) |
| 3   | P3.4 - T0         |
| 4   | P3.5 - T1         |
| 5   | P3.3 - ~INT1      |
| 6   | P3.2 - ~INT0      |
| 7   | P3.1 - TxD        |
| 8   | P3.0 - RxD        |
| 9   | +5V               |
| 10  | GND               |

#### ADC Header

| Pin | Description  |
| --- | ------------ |
| 1   | P1.0 (ADC 0) |
| 2   | P1.1 (ADC 1) |
| 3   | P1.2 (ADC 2) |
| 4   | P1.3 (ADC 3) |
| 5   | P1.4 (ADC 4) |
| 6   | P1.5 (ADC 5) |
| 7   | P1.6 (ADC 6) |
| 8   | P1.7 (ADC 7) |

#### UART Headers

| Pin | Description |
| --- | ----------- |
| 1   | DCD         |
| 2   | CTS         |
| 3   | RTS         |
| 4   | DTR         |
| 5   | TX          |
| 6   | TX Clock    |
| 7   | RX          |
| 8   | RX Clock    |
| 9   | SYNC        |
| 10  | REQ         |
| 11  | GND         |
| 12  | +5V         |

## Datasheets

[P80C550EBAA - DigChip](https://www.digchip.com/datasheets/download_datasheet.php?id=741162&part-number=P80C550EBAA)

[MAX693 - Maxim](https://datasheets.maximintegrated.com/en/ds/MAX690-MAX695.pdf)

[AM85C30 (1) - ChipDB](http://datasheets.chipdb.org/AMD/07513.pdf)

[AM85C30 (2) - Rockby](https://www.rockby.com.au/DSheets/26719.pdf)

[PCF8584 - NXP](https://www.nxp.com/docs/en/data-sheet/PCF8584.pdf)

[MC146818 - NXP](https://www.nxp.com/docs/en/data-sheet/MC146818.pdf)

[LCD2004A - Beta-eStore](https://www.beta-estore.com/download/rk/RK-10290_410.pdf)

## Background

![Breadboard Computer](/Assets/P80C550-EVN-breadboard.webp)

The design goal was for the major components to be sourced from the collection of random electronics I've accumulated over the years - hence a board built around archaic parts newly created in 2021.

The peripherals are straight out of a 1980's PC builder's kit. I think it was 2003 when I received the core parts of this board. I was a member of Chibots in Chicago as a kid and I picked them up during the regular "I'm not taking this back home with me" parts dump / swap meet at the end of club meetings. I ended up with a tubes of 8051 controllers, MAX693 microprocessor supervisors, AM85C30 Dual UARTs / SDLC / HDLC controllers, MC146818 RTC/RAMs (used on PC/AT), and some engineering sample RS232 to TTL level shifters (XC prefix Motorola parts). I have to imagine the person who gave them to me had to have built a board along these lines sometime in the past. Probably with these components. I bet they didn't have the luxury of dirt-cheap 4 layer PCBs though and open source EDA tools :).
