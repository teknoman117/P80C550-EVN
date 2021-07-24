P80C550-EVN
===========

The P80C550-EVN is a small single board computer built around the Signetics P80C550EBAA ROM-less CMOS 8051 microcontroller.

![Assembled With LCD](/Assets/P80C550-EVN-assembled.webp)
![Assembled Without Expansion Headers](/Assets/P80C550-EVN-no-expansion.webp)

Features
--------
- CPU: P80C550EBAA @ 11.0592 MHz
- 32 KiB ROM (AT28C256)
- 32 KiB SRAM (HM62256)
- AM85C30 Dual USART
  - Replace with a Z85230 if a FIFO is desired
- PCF8584 I2C controller
- MC146818 RTC and RAM (32.768 KHz oscillator)
  - MAX 693 Supervisor
  - 3V CR1216 or CR1220 coin cell for RTC backup
- HD44780 or compatible Parallel LCD Port
  - Mounting holes are spaced for the LCD2004A module
- Expansion Headers
  - Bus signals
    - A0 -> A15, D0 -> D7
    - ~WR, ~RD, ~CODE_RD
    - ~INT0, ~INT1
    - ~CODE_CS (used to disable onboard ROM)
  - Unused CPU peripherals
    - UART
    - Counter Inputs
  - Power
    - +5V, GND
- Runs off 5V, or 6V to 20V
  - 5V rail can be directly driven
  - Power Input takes 6V to 20V
    - AP62150WU-7 switching regulator
  - Configurable cutoff/battery saver point
    - Potentiometer Voltage Divider (source from +5V or Vin)
      - 5.1V zener diode to protect from overvoltage on PFO sense input
    - Ref <= 1.3V triggers Power Fault interrupt (on ~INT0)
    - Ref <= 1.2V triggers shutdown

About
-----
The design goal was for the major components to be sourced from the collection of random electronics I've accumulated over the years - hence a board built around archaic parts newly created in 2021.

The CPU is a ROM-less CMOS 8051 microcontroller made by Signetics in the late 1980's. The only major differences between the original and this one is that the Signetics part integrates an 8 bit analog to digital converter (with an 8 channel mux on Port 1), a programmable watchdog timer, and is built on a CMOS process. By nature of being ROM-less, the majority of the I/O ports must be used for the external bus interface (Port 2 = A8..15, Port 0 = AD0..7, Port 3.6 and 3.7 are the ~RD and ~WR signals). This is both good and bad, as we must use extra chips to get GPIO functions, but allows us to use far more than the 2K/4K/8K of ROM the 8051 could be ordered with and the 128 bytes of internal RAM. You can also use EEPROM or Flash for code storage, which is much nicer than needing to pick up an EPROM eraser.

The PCB is a 4 layer board (2 signal layers, 1 power plane, 1 ground plane) designed around the JLCPCB guidelines. This lets 10 units of the board fall into the $8 tier. I ordered it with a black silkscreen and higher quality PCB material so it ended up being 5 for $15. This made routing the signals far simplier since I didn't have to worry about routing power in and around the signals.

The peripherals are straight out of a 1980's PC builder's kit. I think it was 2003 when I received the core parts of this board. I was a member of Chibots in Chicago as a kid and I picked them up during the regular "I'm not taking this back home with me" parts dump / swap meet at the end of club meetings. I ended up with a tubes of 8051 controllers, MAX693 microprocessor supervisors, AM85C30 Dual UARTs / SDLC / HDLC controllers, MC146818 RTC/RAMs (used on PC/AT), and some engineering sample RS232 to TTL level shifters (XC prefix Motorola parts). I have to imagine the person who gave them to me had to have built a board along these lines sometime in the past. Probably with these components. I bet they didn't have the luxury of dirt-cheap 4 layer PCBs though and open source EDA tools :).

The main I/O peripherals are the AM85C30 "SCC" (Serial Communications Controller) and the PCF8584 I2C controller. The AM85C30 can function as a Dual Asynchronous UART with all of the modem control signals, but without FIFOs. It can also operate as an SLDC or HLDC communications controller with FIFOs. Apparently these formed the basis of various networking architectures in the past that I know nothing about. The lack of FIFOs in asynchronous serial mode is a bummer, especially because these chips are good for up to 2 Mbit data rates. If a FIFO is a must, either rework the board for a 16C550 uart or use the Z85230 (still manufactured). It retains all of the SDLC/HDLC bits but adds a 4 byte TX FIFO and an 8 byte RX FIFO in asynchronous mode. The 10 MHz version is $15 on DigiKey.

The PCF8584 is a simple I2C controller supporting 100 KHz SCL frequency in either master or slave mode. Not really anything of note here. I had to source this from eBay because the 5V version of the part went out of production years ago.

Memory Map
----------
Code Memory
- 0x0000 -> 0x7FFF: 32 KiB ROM (28 pin JEDEC @ 5V)
- 0x8000 -> 0xFFFF: (repeat of ROM)

Data Memory
- 0x0000 -> 0x7FFF: 32 KiB RAM (28 pin JEDEC @ 5V)
- 0x8000:           Status Register
- 0x8400 -> 0x8FFF: (unused)
- 0x9000 -> 0x903F: RTC
- 0x9400 -> 0x9403: USART
- 0x9800 -> 0x9801: I2C Controller
- 0x9C00 -> 0x9C01: HD44780 Compatible LCD 
- 0xA000 -> 0xFFFF: (unused)

Status Register Notes
---------------------
- Read: Bit 7 is the ~PFO (Power Fault) line from the MAX 693 Supervisor
  - 0 = Low Voltage
  - 1 = Nominal Voltage
- Write: Bit 7 is the ~PFO (Power Fault) interrupt mask 
  - 0 = Enable PFO interrupt
  - 1 = Disable PFO interrupt

Revision History
----------------
- Revision 1.0
  - First Board Produced
  - Errors
    - MAX693 RST signal (active-high) is open collector. This was not mentioned in the chip's datasheet, therefore no pull-up was designed into the board. ![#1](https://github.com/teknoman117/P80C550-EVN/issues/1)
    - MAX693 ~RST signal (active-low, wired-OR) does not propagate to RST (active-high). Pushing the reset button will reset only the peripherals and not the CPU. ![#2](https://github.com/teknoman117/P80C550-EVN/issues/2)

Background
----------
![Breadboard Computer](/Assets/P80C550-EVN-breadboard.webp)
