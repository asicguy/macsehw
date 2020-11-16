# "BBU" Apple Custom Silicon

The "BBU" (Bob Bailey Unit), as it is called on the Macintosh SE's
printed circuit board silkscreen, is a relatively complex Apple custom
silicon chip, compared to the other custom chips on the Macintosh SE's
Main Logic Board (MLB).  Despite its intimidating look as a chip with
a huge number of pins, its purpose can be summarized as follows.

* Take the master 16 MHz clock as input and divided it down to
  generate the 8 MHz, 3.7 MHz, and 2 MHz clock signals as output.

  Note that these are the specific frequencies used: 15.667200 MHz,
  7.8336 MHz, 3.672 MHz, and 1.9584 MHz.

  Also, note that this is the routing of the source 16 MHz clock
  signal.  (1) 16 MHz master clock crystal -> (2) GLU chip, logically
  ANDs master clock with GLU `*OE` to disable whole system clock under
  certain circumstances), (3) BBU chip.

* Provide a single address bus interface to ROM, RAM, and I/O devices,
  including simple digital I/O pins.  Namely, for the ROM, RAM, SCC,
  VIA, and IWM, it uses a simple method of checking which of the upper
  four address lines is set and then driving the corresponding chip
  select pins.

    * 0 (0x000000-0x3fffff): Select RAM.
    * 1 (0x400000-0x7fffff): Select ROM, SCSI, or boot-time RAM overlay.
    * 2 (0x800000-0xbfffff): Select SCC.
    * 3 (0xc00000-0xffffff): Select IWM or VIA.

  In particular, the zones are further subdivided as follows for the
  Macintosh Plus, according to MESS/MAME source code:

    * 0x000000 - 0x3fffff: RAM/ROM (switches based on overlay)
    * 0x400000 - 0x4fffff: ROM
    * 0x580000 - 0x5fffff: 5380 NCR/Symbios SCSI peripherals chip
    * 0x600000 - 0x6fffff: RAM, boot-time overlay only
    * 0x800000 - 0x9fffff: Zilog 8530 SCC (Serial Control Chip) Read
    * 0xa00000 - 0xbfffff: Zilog 8530 SCC (Serial Control Chip) Write
    * 0xc00000 - 0xdfffff: IWM (Integrated Woz Machine; floppy)
    * 0xe80000 - 0xefffff: Rockwell 6522 VIA
    * 0xf00000 - 0xffffef: ??? (the ROM appears to be accessing here)
    * 0xfffff0 - 0xffffff: Auto Vector

  The Macintosh SE goes a step further and adds invalid address guard
  zones around the SCC and IWM mappings:

    * 0x000000 - 0x3fffff: RAM/ROM (switches based on overlay)
    * 0x400000 - 0x4fffff: ROM
    * 0x580000 - 0x5fffff: 5380 NCR/Symbios SCSI peripherals chip
    * 0x600000 - 0x6fffff: RAM, boot-time overlay only
    * 0x900000 - 0x9fffff: Zilog 8530 SCC (Serial Control Chip) Read
    * 0xb00000 - 0xbfffff: Zilog 8530 SCC (Serial Control Chip) Write
    * 0xd00000 - 0xdfffff: IWM (Integrated Woz Machine; floppy)
    * 0xe80000 - 0xefffff: Rockwell 6522 VIA
    * 0xf00000 - 0xffffef: ??? (the ROM appears to be accessing here)
    * 0xfffff0 - 0xffffff: Auto Vector

  Note that all of these address ranges specified can be handled by
  only checking the upper 5 bits of the address (A19-A23), which route
  directly into the BBU from the main address bus.  Well, except for
  the last one, but our BBU doens't need to do any specially handling
  for the sake of that zone.

  TODO VERIFY: Does the Macintosh Plus and Macintosh SE really not
  expose the boot-time RAM address remapping, just the ROM address
  remapping?  This is what MESS/MAME source code seems to claim.

* Control the RAM and ROM switches to expose the ROM overlay at
  0x000000 and RAM at 0x600000 at startup.  Snoop the address bus to
  detect the first access to the standard ROM address mapping and
  disable the boot-time ROM overlay, until the next RESET.

  Also, apparently, according to MESS/MAME source code, the first
  write to the regular RAM address zone also disables the ROM overlay.

* Act as a DRAM controller.  Set the ROM/RAM control signals depending
  on the particular address requested, i.e. `*EN245`, `*ROMEN`,
  `*RAS`, `*CAS0L`, `*CAS0H`, `*CAS1L`, `*CAS1H`, `RAM R/*W`.
  `*PMCYC` is apparently used to totally disable DRAM row and column
  access strobes only during startup.  The F257 chips are used to
  select separate address portions for the DRAM row and column access
  strobes.  The LS245 chips are used to disable DRAM access during ROM
  access.

  DRAM is accessed by sending the row access strobe first, the column
  access strobe second.

  Also, note that the address multiplexing is configured as follows:

  Row access strobe: A1, A11, A12, A13, A14, A15, A16, RA7*, A18, RA9*.
  Column access strobe: A2, A3, A4, A5, A6, A7, A8, RA7*, A10, RA9*.

  RA7 and RA9 are controlled directly by the BBU rather than being
  wired through the F257 address multiplexers.  Now, the devil is in
  the details here because the exact control mechanism depends on how
  much RAM is installed.

  If only 64K RAM SIMMs are installed (which is an unsupported
  configuration), then RA7 is either A9 (row) or A10 (column).  Bus
  snooping must be used to copy RA8 over to RA7 on the column access
  strobe.  If there are two rows of DRAM SIMMs, A17 is used to
  determine which one to use.  It could be possible that one of the
  unused, unlabeled BBU pins controls this setting.  With the minimum
  configuration of one row of DRAM SIMMs, this means you can configure
  a Macintosh SE with only 128K of RAM.  Hilarious!

  If only 256K RAM SIMMs are installed, then RA7 is either A17 (row)
  or A9 (column).  RA9 is not used.  If there are two rows of DRAM
  SIMMs, A19 is used to determine which one to use.

  If 1MB RAM SIMMs are installed, then RA7 is either A17 (row) or A9
  (column).  RA9 is either A20 (row) or A19 (column).  If there are
  two rows of DRAM SIMMs, A21 is used to determine which one to use.

  Since RAM accesses are always on even bytes in 16-bit quantities, A0
  is implied to be zero and therefore also is not available on the
  CPU.

* So, wow.  Here's a list of all possible Macintosh SE RAM
  configurations.

  128K (double undocumented), 256K (double undocumented), 512K
  (undocumented), 1MB, 2MB, 4MB.

* Refresh the DRAM by periodically reading some arbitrary memory from
  every available row.  Unlike the Apple II, the contiguous
  organization of the screen, sound, and PWM disk speed buffers does
  not allow for these periodic functions to double as automatic DRAM
  refresh.  How does this need play together with the PDS card's
  ability to request priority access over `DTACK`?  Maybe the refresh
  circuitry still continues to function, but without driving DTACK for
  the duration that the PDS card requests driving the signal.

  However, one interesting trick is that the address multiplexers are
  configured to access alternating DRAM rows when reading consecutive
  addresses rather than all coming from a single DRAM row.  I am not
  sure of the motivation behind this, but it seems like it could have
  been extended so that reading consecutive memory addresses would
  provide automatic DRAM memory refresh, thus allowing the video
  circuitry to double in this role without providing the drawbacks of
  nonlinear video memory to software.

  Unfortunately, this scheme also complicates reusing the same DRAM
  row for performance improvements.

* Scan the CRT by driving the primary digital control signals
  (`*VSYNC`, `*HSYNC`, `VIDOUT`).  Read directly from RAM buffers as
  required, and use `*DTACK` to prevent the CPU from accessing RAM at
  the same time.

* Generate the PWM signals for sound output and disk drive speed
  control.  Read directly from RAM buffers as required.  Note that the
  Macintosh SE deprecated the alternate sound buffer found in previous
  compact Macintoshes.

* Handle SCSI DMA transfers, if the mode is enabled by the ROM.
  However, as I understand it, the Macintosh SE ROM does not actually
  support SCSI DMA transfer, even though the hardware is capable of
  supporting it.

* Please note: For pin numbering, use this datasheet for reference
  with the schematic and the physical board layout.

  20201018/http://www.assmann-wsw.com/fileadmin/datasheets/ASS_0981_CO.pdf

There might be additional processing functions it may provide as a
convenience between the CPU and the various other hardware chips, but
chances are these processing functions are relatively simple.

Most of the I/O pins that are connected to the BBU are single-bit
digital I/O signals that are relatively easy to understand.  Reverse
engineering the Macintosh SE's firmware may be required to determine
how these pins are mapped into the CPU's address space, but once that
determination is made, providing a replica interface to most of the
connected hardware should be super-easy.

The following I/O chips are connected to the BBU:

* VIA interrupt controller

* IWM/SWIM floppy disk controller

* SCSI Controller

* Serial Communications Controller (SCC)

Other chips that are connected to the BBU are mainly interfaced via
only simple, single-pin interfaces.

----------

## More explanation on pin functions

* `RA0` - `RA9`: Address-multiplexed pins intended to connect directly
  to the address lines on the RAM SIMMs, outputs.

* `RA9` is only controlled when the `MBRAM` input is TRUE, i.e. +5V.
  This indicates that 1MB RAM SIMMs are being used.  Otherwise, it is
  kept at zero and high memory addresses are marked as bus errors.
  256K RAM SIMMs are used in this case.

* When `ROW2` input is TRUE, i.e. +5V, it indicates that both RAM SIMM
  rows are in use.  Otherwise, only the first row of RAM SIMM is used
  and high addresses are marked as bus errors.  And, `CAS1L` and
  `CAS1H` are not driven.

* If both `MBRAM` and `ROW2` are TRUE, i.e. +5V, it is also possible
  for the BBU to detect a 2.5MB RAM configuration and adjust bus
  errors flagging accordingly.

* `RDQ0` - `RDQ15` are bidirectional data signals, they are the
  primary means by which single-pin I/O devices and the like are
  mapped into the address space that can be directly accessed by the
  CPU, in conjunction with the address inputs.  Namely, the BBU reads
  and writes to RAM, and the CPU accesses that RAM on its own time.

* I do not know if the C8M and C3.7M clock signals are inputs or
  outputs.  The master clock crystal is C16M, 16 MHz, generated by the
  "FOX" crystal oscillator on the MLB.  If the clock frequency is
  divided down by the BBU, then these are outputs.  In any case, C16M
  is definitely an input.

  Please note: The "F" suffixes is a good hint saying that a signal is
  "filtered," which only happens after a signal has already been
  output from the primary device.  So, if you are connecting to an "F"
  signal, that means you're an input.  Otherwise, you're an output.

  Therefore, that's pretty strong evidence that the lower frequency
  clock signals are divided down by the BBU.

* All peripheral/device chip select/enable signals are output signals.

* MC68000 output signals directly connected to the BBU are BBU input
  signals.  Namely: `*LDS`, `*UDS`, `R/*W`, `*AS`.

* Output signals that connect to MC68000 inputs: `*DTACK`, `*BERR`,
  `*IPL0`, `*IPL1`, `*IPL2`, `*VPA`.  Or are the interrupt signals BBU
  inputs?

* Is `*RES` an input only?  I would assume so, assuming there is
  another, dedicated circuit to control hard board resets.  Note that
  the PDS bus connector ties the `*HALT` and `*RES` signals together.
  So, if the MC68000 CPU executes a `RESET` instruction, that won't
  just reset all peripheral devices, but it will also reset the CPU
  itself.

  Note that the BBU needs a RESET input pin for its own sake since it
  includes sequential logic to scan the CRT and sound buffers.

* `C2M` is an output signal, it primarily controls the address
  multiplexers to select either the row address (zero) or column
  address (one).  Connecting directly to a simple 2 MHz clock could be
  adequate, or a more tailored method may be used for higher
  performance and lower memory access time.

* `*PMCYC` is an output signal.  Its primary conceptual purpose is to
  define "whose turn" it is to access DRAM, the CPU or the BBU?  This
  could be as simple as a 1 MHz clock, since the CPU always takes a
  multiple of 4 clock cycles at 8 MHz to access DRAM.  The symbol is
  probably short for Processor Memory CYCle.  It only connects to the
  PDS slot and the F257 chips.

----------

Peripheral device signals, input or output?

* `*EXT.DTK` is an input signal, for PDS use.  If this pin is pulled
  low, then the system expansion card is responsible for generating
  the `*DTACK` signal to indicate to the CPU that the data transfer is
  complete.  The BBU puts the signal in a high-impedance state.

  20200808/https://web.archive.org/web/20190909060927/http://www.ccadams.org/se/pinouts.html

  However, please note that the BBU can still access DRAM on its
  regular turn to do so.  The primary purpose of a PDS card driving
  `*DTACK`, of course, is for I/O-mapped I/O where the memory in
  question is hosted on the PDS card, not in the system's main DRAM.
  If the BBU were to start the process of accessing DRAM on behalf of
  the CPU, it will cancel it upon detection of this signal.

  In the Macintosh Classic schematic, this pin is connected to a
  pull-up resistor since the Classic doesn't support PDS expansion
  cards.

* `*EAREN` is very likely an output signal (also for PDS use), for the
  reason it is not indicated in the Bomarc Macintosh Classic
  schematics, i.e. it could be disconnected entirely.  Also, note that
  we don't need to actually implement this signal because it is marked
  as "reserved" in the PDS slot documentation.

* Output signals: `*SCCRD`, `*PWM`?, `*DACK`, `SND`,
  `*VSYNC`, `*HSYNC`, `VIDOUT`.

* Output SELECT signals: `IWM`, `*SCCEN`. `VIA.CS1`.

* Input signals: `VIDPG2`, `SNDRES`, `SCSIDRQ`, `VIAIRQ`.

----------

There is still more to learn/investigate relating to unspecified
signals.

----------

## Ideas for Enhancements

* Auto-detect jumper settings for RAM by testing for address
  wrap-around.  Unfortunately, because the CPU `HALT` and `RESET` are
  wired together, this probably requires circuit board changes to
  function successfully.

* Allow the use of even faster DRAM by speeding up the BBU's internal
  timing.  The internal clock would be driven by an internal
  oscillator as a phase-locked-loop on the 16 MHz external clock.  The
  CPU could then be able to always access DRAM with zero wait states.

* Add Memory Manager Unit (MMU) functionality to implement virtual
  memory.  In order to be compatible with the original MC68000, all
  the logic to handle page faults would need to be built into the BBU
  itself and the CPU is simply instructed to wait additional cycles by
  holding the `*DTACK` signal deasserted.

* Implement bank switching to allow access to more than 4 MB of RAM
  without requiring a CPU that is capable of virtual memory.  The
  original MC68000 CPU in particular does not allow for
  exception-handling that repeats execution of a faulted instruction,
  hence it is not capable of a straightforward implementation of
  virtual memory that uses page faults.

* It could be possible to choose a FPGA with a sufficient quantity of
  SRAM (or even DRAM) stacked within it such that the physical DRAM
  sticks are not needed.
