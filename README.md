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
    - MAX693 RST signal (active-high) is open collector. This was not mentioned in the chip's datasheet, therefore no pull-up was designed into the board. ![#1](https://github.com/teknoman117/P80C550-EVN/issues/1)
    - MAX693 ~RST signal (active-low, wired-OR) does not propagate to RST (active-high). Pushing the reset button will reset only the peripherals and not the CPU. ![#2](https://github.com/teknoman117/P80C550-EVN/issues/2)

The Hardware
============

![Assembled Without Expansion Headers](/Assets/P80C550-EVN-no-expansion.webp)

## Processor

The CPU is a ROM-less CMOS 8051 microcontroller made by Signetics in the late 1980's. The only major differences from the original are that the Signetics part integrates an 8 bit analog to digital converter (with an 8 channel mux on Port 1), a programmable watchdog timer, and is built on a CMOS process. By nature of being ROM-less, the majority of the I/O ports must be used for the external bus interface (Port 2 = A8..15, Port 0 = AD0..7, Port 3.6 and 3.7 are the ~RD and ~WR signals). This is both good and bad, as we must use extra chips to get GPIO functions, but allows us to use far more than the 2K/4K/8K of ROM the 8051 could be ordered with and the 128 bytes of internal RAM. You can also use EEPROM or Flash for code storage, which is much nicer than needing to pick up an EPROM eraser.

## Memory and Memory Map

The memory for the board is provided via two 28 pin JEDEC compliant memory sockets supporting up to 32 KiB each. The left socket is for the ROM and the right socket is for the external data memory. Typically, an SRAM would be installed into the data socket, but this could also be a ROM or Flash device (such as AT29Cxx) if memory beyond the 128 byte internal RAM of the 8051 and 50 byte RTC/CMOS RAM is not required. Both sockets are mapped in the lower half of their respective address spaces (ROM in lower 32 KiB of CODE memory and the RAM in the lower 32 KiB of XDATA memory).

The onboard peripherals are mapped in the 8 KiB of address space following the XDATA memory as 1 KiB chunks. This may seem wasteful, but it allows using a single '138 3-to-8 decoder for peripheral chip selects (active high enable is A15, active low enables are A14 and A13. A12 -> A10 are then used for the select input).

| Code Address | Description |
| ------------ | ----------- |
| 0000h -> 7FFFh | 32 KiB ROM socket (by default - see section on [Expansion](#expansion)) |
| 8000h -> FFFFh | **unused** |

| Data Address | Description |
| ------------ | ----------- |
| 0000h -> 7FFFh | 32 KiB XDATA socket |
| 8000h | [Power Status Register](#power-status-register) |
| 8400h -> 8FFFh | **unused** |
| 9000h -> 903Fh | RTC |
| 9400h -> 9401h | UART / SDLC - Channel B |
| 9402h -> 9403h | UART / SDLC - Channel A |
| 9800h -> 9801h | I2C controller |
| 9C00h -> 9C01h | LCD |
| A000h -> FFFFh | **unused** |

## Interrupts

The P80C550 has two external interrupt lines which can either be falling edge triggered or low level triggered. There is an internal pullup resistor on the CPU and peripherals with open collector interrupt outputs can be wire-OR'd together on the same interrupt line.

| Interrupt | Devices |
| --------- | ------- |
| EX0 | Power Fault, RTC, I2C |
| EX1 | UART / SDLC controller |

## Power and Battery Saver

The board can either be powered directly via the 5V rail, or by a 7V to 18V terminal block, which is regulated to 5V by an AP62150WU-7 buck regulator. The input voltage range means the board can be safely powered by
- 5 to 12 AA(A) batteries
- 6 to 12 cell NiMH batteries
- 2 to 4 cell Li-ion or Li-Poly batteries

There is also a holder for an *optional* CR1216, CR1220, or CR1225 3V coin cell to power the RTC. If omitted, RTC and CMOS state will be lost on power-off. A CR1216 is good for at least 2 months (and counting).

A MAX693 microprocessor supervisor is used to manage the power-on reset delay, the RTC backup power, and a lower power fault signal to the CPU.

A voltage divider is provided via a potentiometer to set the low power threshold. A jumper ("PFI Input Select") is provided to select whether the source voltage for the divider is provided from the Vin terminal block or the 5V rail. The former is used in battery powered or other situations using Vin and the latter is used when the 5V rail is directly driven. A 5.1V zener diode protects from accidental overvoltage on the voltage reference inputs.

The MAX693 will report a low power fault when the reference input drops below **1.3V**. By default, this will trigger the EX0 interrupt on the P80C550. A single bit register ([Power Status Register](#power-status-register)) is provided to check the status of the power fault output of the MAX693 and to mask the interrupt it generates. It will also hold the CPU in a reset state if the 5V rail drops below 4.4V.

In Vin powered scenarios, the AP62150WU-7 buck regulator will shut down when the reference input drops below **1.2V** and completely cut power to the board. The voltage divider should be configured such that power is cut when the voltage drops to the minimum voltage of your battery, or the minimum voltage of the regulator (about 6.5V).

To determine the output voltage of the voltage divider at a specific input voltage given a target cutoff voltage, use the following formula:

 > Vout = Vin * (1.2V / Vcutoff);

Vin is the current input voltage and Vcutoff is the desired voltage for the regulator to shut off. Vout is the voltage you should tune the voltage divider to output at that input voltage. See the example configurations below. The emphasis on maximum voltages is to ensure you are aware that "fresh off the charger" batteries have a notably higher voltage than their nominal voltage (which they will drop to quickly), so be aware of this when tuning the cutoff reference.

For scenarios directly powering the 5V rail, the voltage divider should be configured to output 1.444V at 5V. This will trigger the lower power fault if the rail drops to 4.5V, which is the minimum safe CPU operating voltage.

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

## UART / SDLC

The AM85C30 "ESCC" can function as a Dual Asynchronous UART with all of the modem control signals, but without FIFOs. It can also operate as an SLDC or HLDC communications controller with FIFOs. Apparently these formed the basis of various networking architectures in the past that I know nothing about. The lack of FIFOs in asynchronous serial mode is a bummer, especially because these chips are good for up to 2 Mbit data rates. If a FIFO is a must, either rework the board for a 16C550 uart or use the Z85230 (still manufactured). It retains all of the SDLC/HDLC bits but adds a 4 byte TX FIFO and an 8 byte RX FIFO in asynchronous mode. The 10 MHz version is $15 on DigiKey.

## I2C

The PCF8584 is a simple I2C controller supporting 100 KHz SCL frequency in either master or slave mode. Not really anything of note here. I had to source this from eBay because the 5V version of the part went out of production years ago.

## Expansion

TODO

3 headers

## LCD

![Assembled With LCD](/Assets/P80C550-EVN-assembled.webp)

Any LCD module compatible with the 16 pin HD44780 header may be used. The mounting holes are spaced for the seemingly popular LCD2004A module that can be found pretty much everywhere. I used this module specifically (not an endorsement of this seller or any form of affiliate link) - [eBay](https://www.ebay.com/itm/171907112716).

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

Background
----------
![Breadboard Computer](/Assets/P80C550-EVN-breadboard.webp)

The design goal was for the major components to be sourced from the collection of random electronics I've accumulated over the years - hence a board built around archaic parts newly created in 2021.

The peripherals are straight out of a 1980's PC builder's kit. I think it was 2003 when I received the core parts of this board. I was a member of Chibots in Chicago as a kid and I picked them up during the regular "I'm not taking this back home with me" parts dump / swap meet at the end of club meetings. I ended up with a tubes of 8051 controllers, MAX693 microprocessor supervisors, AM85C30 Dual UARTs / SDLC / HDLC controllers, MC146818 RTC/RAMs (used on PC/AT), and some engineering sample RS232 to TTL level shifters (XC prefix Motorola parts). I have to imagine the person who gave them to me had to have built a board along these lines sometime in the past. Probably with these components. I bet they didn't have the luxury of dirt-cheap 4 layer PCBs though and open source EDA tools :).
