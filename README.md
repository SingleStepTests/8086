## 8086 V1

This is a set of 8086 CPU tests produced by Daniel Balsom using the [ArduinoX86](https://github.com/dbalsom/arduinoX86) interface and the [MartyPC](https://github.com/dbalsom/martypc) emulator.

### Current version: 1.0.1

 - The `v1` directory contains the current version of the 8086 test suite.

### About the Tests

These tests are produced with an `Intel P80C86A-2` CPU, running in Maximum Mode. Bus signals are provided via an `Intel 8288` Bus Controller.

2,000 tests are provided per opcode.

This is far less than the 10,000 tests included in the [8088 Test Set](https://github.com/SingleStepTests/8088/). My experience using my own tests is that 10,000 tests was way more than required to identify any issues these tests have a chance of uncovering. Therefore, this test set is considered supplemental to the 8088 tests, intended for those who may be interested in validating or otherwise investigating the bus activity and prefetching behavior of the 8086 CPU specifically.

All tests assume a full 1MB of RAM is mapped to the processor and writable. Bear in mind that on the 8086, the address space wraps around at `0xFFFFF`.

No wait states are incurred during any of the tests. The interrupt and trap flags are not exercised. 

This test set exercises the 8086's processor instruction queue. All the instructions execute from as full a prefetch queue as possible (5 bytes for instructions starting at odd addresses, 6 bytes otherwise).

Registers state is randomized, however each register has a 2% chance of being set to 0.  Memory contents are also randomized, but each test has a 2% chance of having memory set to all `0x00` or `0xFF`. The rationale for this is to attempt to exercise more edge cases and ensure test coverage of the zero flag.

### Using the Tests

The simplest way to run each test is to override your emulated CPU's normal reset vector (`FFFF:0000`) to the `CS:IP` of the test's initial state, then reset the CPU. 

#### Prefetched Tests

- If cycle-accurate verification of bus activity is desired, the provided initial queue contents should be set after the CPU reset routine flushes the instruction queue. This should trigger your emulator to suspend prefetching since the queue is full. In my emulator, I provide the CPU with a vector of bytes to install in the queue before I call ```cpu.reset()```. You could also pass your reset method a vector of bytes to install.

- You will need to add the length of the queue contents to your `PC` register (or adjust the reset vector `IP`) so that fetching begins at the correct address. 

- Once you read the first instruction byte out of the queue to begin test execution, prefetching should be resumed since there is now room in the queue. It takes two cycles to begin a fetch after reading from a full queue, therefore tests that specify an initial queue state will start with two `Ti` cycle states.

- If you're not interested in per-cycle validation, you can ignore the queue fields and not strictly validate code fetch bus transfers.

### Why Are Instructions Longer Than Expected?

Instruction cycles begin from the cycle in which the CPU's queue status lines indicate that an instruction "First Byte" has been fetched.

Instructions cycles end when the queue status lines indicate that the first byte of the next instruction has been read from the queue. If the instruction started fully prefetched and the next instruction byte was present in the queue at the end of the instruction, the instruction cycles should reflect the documented "best case" instruction timings. If the queue was empty at the end of execution, the extra cycles spent fetching the next instruction byte may lengthen the instruction from documented timings. There is no indication from the CPU when an instruction ends - only when a new one begins.

### Segment Override Prefixes

Random segment override prefixes have been prepended to a percentage of instructions, even if they may not do anything. This isn't completely useless - a few bugs have been found where segment overrides had an effect when they should not have.

### String Prefixes

String instructions may be randomly prepended by a `REP`, `REPE`, `REPNE` instruction prefix. In this event, `CX` is masked to 7 bits to produce reasonably sized tests (A string instruction with `CX==65535` would be over a million cycles in execution). 

### Instruction Prefetching

All bytes fetched after the initial instruction bytes are set to 0x90 (144) (`NOP`). Therefore, the queue contents at the end of all tests will contain only `NOP`s, with a maximum of 5 (since one has been read out).

### Test Formats

The 8086 test suite comes in two formats, JSON and a simple chunked binary format I call `MOO` (Machine Opcode Operation).
If your language of choice lacks great options for parsing JSON (such as C) you may prefer the binary format tests.

You can find more information about the binary format at the [MOO repository here](https://github.com/dbalsom/moo). 
This repository contains a Rust library crate for working with `MOO` files as well as code for parsing `MOO` files in C, C++ and Python.

Information on the JSON format proceeds below.

Sample test:
```json
{
    "name": "add cl, ah",
    "bytes": [0, 225],
    "initial": {
        "regs": {
            "ax": 13212,
            "bx": 45284,
            "cx": 47784,
            "dx": 43524,
            "cs": 59545,
            "ss": 61254,
            "ds": 3186,
            "es": 55910,
            "sp": 63905,
            "bp": 14753,
            "si": 42814,
            "di": 0,
            "ip": 22673,
            "flags": 64663
        },
        "ram": [
            [975393, 0],
            [975394, 225],
            [975395, 144],
            [975396, 144],
            [975397, 144]
        ],
        "queue": [0, 225, 144, 144, 144]
    },
    "final": {
        "regs": {
            "cx": 47835,
            "ip": 22675,
            "flags": 62598
        },
        "ram": [
            [975393, 0],
            [975394, 225],
            [975395, 144],
            [975396, 144],
            [975397, 144]
        ],
        "queue": [144, 144]
    },
    "cycles": [
        [0, 39987, "--", "---", "---", 0, 0, "PASV", "Ti", "F", 0],
        [0, 39987, "--", "---", "---", 0, 0, "PASV", "Ti", "S", 225],
        [1, 975398, "--", "---", "---", 0, 0, "CODE", "T1", "-", 0]
    ],
    "hash": "3cef51a56be51f983826f6611a489dfbc35dfdf5",
    "idx": 0
},
```
- `name`: A user-readable disassembly of the instruction.
- `bytes`: The raw bytes that make up the instruction.
- `initial`: The register, memory and instruction queue state before instruction execution.
- `final`: Changes to registers and memory, and the state of the instruction queue after instruction execution.
    - Registers and memory locations that are unchanged from the initial state are not included in the final state.
    - The entire value of `flags` is provided if any flag has changed.
- `hash`: A SHA1 hash of the test JSON. It should uniquely identify any test in the suite.
- `idx`: The numerical index of the test within the test file.

### Cycle Format

Even if you are not interested in writing a cycle-accurate emulator, you may be interested in parsing the CPU cycle states provided in the `cycles` list, so that you can validate you are peforming the same number of bus cycles, and in the same order.

One example of an emulator bug that cannot be uncovered by simply validating the initial and final memory and register states is the case where your instruction writes additional bytes that it **shouldn't**, such as doing an extra iteration of a string instruction. You can catch this by comparing the bus cycles your emulator performs to the bus cycles captured from the CPU.

The `cycles` list contains sub lists, each corresponding to a single CPU cycle. Each contains several fields. From left to right, the cycle fields are:  

 - Pin bitfield
 - Multiplexed bus
 - segment status
 - memory status
 - IO status
 - `BHE` (Byte high enable) status (active-low)
 - data bus
 - bus status
 - T-state
 - queue operation status
 - queue byte read

The first column is a bitfield representing certain chip pin states. 

 - Bit #0 of this field represents the `ALE` (Address Latch Enable) pin output, which in Maximum Mode is output by the i8288. This signal is asserted on `T1` to instruct the PC's address latches to store the current address. This is necessary since the address and data lines of the 8086 are multiplexed, and a full, valid address is only on the bus while `ALE` is asserted. Thus, the second column represents the value of the address latch, and not the address bus itself (which may not be valid in a given cycle).
 - Bit #1 of this field represents the `INTR` pin input. This is not currently exercised, but may be in future test releases.
 - Bit #2 of this field represents the `NMI` pin input. This is not currently exercised, but may be in future test releases.

The **multiplexed bus** is the value of the 20 address/data/status lines on the current cycle, read directly from the CPU. It contains a valid address only on T1 when ALE is active. During other T states it contains status pins and data bus contents. 

The **segment status** indicates which segment is in use to calculate addresses by the CPU, using segment-offset addressing. This field represents the `S3` and `S4` status lines of the 8086. It may be `ES`, `DS`, `CS`, `SS`, or invalid (`--`).

The **memory status** field represents outputs of the attached i8288 Bus Controller. From left to right, this field will contain `RAW` or `---`.  `R` represents the `MRDC` status line, A represents the `AMWC` status line, and `W` represents the `MWTC` status line. These status lines are active-low. A memory read will occur on `T3` or the last `Tw` t-state when `MRDC` is active. A memory write will occur on `T3` or the last `Tw` t-state when `AMWC` is active. At this point, the value of the data bus field will be valid and will represent the byte read or written.

The **IO status** field represents outputs of the attached i8288 Bus Controller. From left to right, this field will contain `RAW` or `---`.  `R` represents the `IORC` status line. `A` represents the `AIOWC` status line. `W` represents the `IOWC` status line. These status lines are active-low. An IO read will occur on `T3` or the last `Tw` t-state when `IORC` is active. An IO write will occur on `T3` or the last `Tw` t-state when `AIOWC` is active. At this point, the value of the data bus field will be valid and will represent the byte read or written.

#### BHE

The 8086 CPU assumes an odd/even memory organization, similar in design to the 286 in the IBM AT. An 8086-based system does not typically access memory at odd locations - instead only words are addressed, at even memory locations only, with even bytes of memory are mapped to the lower 8 bits of the data bus, and odd bytes mapped to the upper 8 bits. The A0 line is therefore omitted from directly addressing the memory chips. A combination of A0 and BHE are used to drive the motherboard logic that activates the appropriate bus lines and memory banks.

The **BHE status** indicates whether the high half of the 16-bit data bus is being driven. It is active-low (0). If a bus cycle begins at an even address with BHE active, it is a 16-bit transfer. If a bus cycle begins at an odd address with BHE active, it is an 8-bit transfer, where the high (odd) byte of the data bus is active.  If a bus cycle begins at an even address with BHE inactive, it is an 8-bit transfer where the low (even) byte of the data bus is active. It is important to handle this logic correctly - the inactive half of the bus, if any, should be masked/ignored when validating against the tests.

The **data bus** indicates the value of the last 16 bits of the multiplexed bus. It is typically only valid on T3. 

The **bus status** lines indicate the type of bus m-cycle currently in operation. Either INTA, IOR, IOW, MEMR, MEMW, HALT, CODE, or PASV.  These states represent the S0-S2 status lines of the 8086.

The **T-state** is the current T-state of the CPU. Since this state is not exposed by the CPU, it is calculated based on bus activity.

The **queue operation status** will contain either `F`, `S`, `E` or `-`. `F` indicates a "First Byte" of an instruction or instruction prefix has been read.  `S` indicates a "Subsequent" byte of an instruction has been read - either a modr/m, displacement, or operand. `E` indicates that the instruction queue has been Emptied/Flushed. All queue operation statuses reflect an operation that actually occurred on the previous cycle.  This field represents the QS0 and QS1 status lines of the 8086. 

When the queue operation status is not `-`, then the value of the **queue byte read** field is valid and represents the byte read from the queue. 

For more information on the 8086 and 8288 status lines, see their respective white papers.

### Undefined Instructions

Note that these tests include many undocumented/undefined opcodes and instruction forms. The 8086 has no concept of an invalid instruction, and will perform some task for any provided sequence of instruction bytes. Additionally, flags may be changed by documented instructions in ways that are officially undefined.

### Per-Instruction Notes


| Opcode(s)                      | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
|--------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **0F**                         | `POP CS` is currently omitted from the test set.<br/>`POP CS` will replace the value of `CS` without flushing the prefetch queue, which will likely cause unintended opcodes to execute unless the queue was emptied by jumping to this instruction.                                                                                                                                                                                                                                                            |
| **8F**                         | **8F** with ModR/M reg != 0 is undefined and has unusual behavior under test conditions, so this form is not included in tests.                                                                                                                                                                                                                                                                                                                                                                                 | 
| **9B**                         | The `WAIT` instruction is not included.                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| **8C,8E**                      | These instructions are only defined for a ModR/M reg value of 0-3, however only the first two bits are checked, so the test set contains random values for reg                                                                                                                                                                                                                                                                                                                                                  |
| **8D,C4,C5**                   | The `reg, reg` forms of `LEA`, `LES`, and `LDS` are undefined. These forms are not included in this test set due to disruption of the last calculated effective address by the CPU set up routine which would otherwise be exposed.                                                                                                                                                                                                                                                                             |
| **A4-A7,AA-AF**                | For `REP`-prefixed string instructions, `CX` is masked to 7 bits. This provides a reasonable test length, as the full 65535 value in `CX` with a `REP` prefix could result in over one million cycles.                                                                                                                                                                                                                                                                                                          |
| **C6,C7**                      | Although the ModR/M reg != 0 forms of these instructions are officially undefined, this field is ignored. Therefore, the test set contains random values for reg.                                                                                                                                                                                                                                                                                                                                               |
| **D2,D3**                      | The `OF` flag is undefined in the case that `CL > 1`. Using the flag mask from `metadata.json` will not validate correctness when `CL==1` or `0`.<br/>Later Intel CPUs limited the number of shifts possible by masking `CL` to 5 bits, however the 8088 does not mask the value of `CL` so shifts of up to 255 are possible.<br/>`CL` is masked to 6 bits in the test suite to shorten the possible test length while still hopefully catching the case where `CL` may be improperly masked to 5 bits.            |
| **E4,E5,EC,ED**                | All reads of IO port addresses should return 0xFF and incur no wait states.                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| **F0, F1**                     | The LOCK prefix is not exercised in this test set                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| **F4**                         | HALT is not included in the test set.                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| **D4, F6.6, F6.7, F7.6, F7.7** | These instructions can generate a divide exception (more accurately, a Type-0 Interrupt).<br/>When this occurs, cycle traces continue until the first byte of the exception handler is fetched and read from the queue. The IVT entry for INT0 is set up to point to 1024 (`0x0400`)<br/>**NOTE:** On the 8088 specifically, the return address pushed to the stack on divide exception is the address of the next instruction. This differs from the behavior of later CPUs and generic Intel IA-32 emulators. |
| **F6.5, F7.5**                 | Presence of a REP prefix preceding IMUL will invert the sign of the product. This is not currently tested.                                                                                                                                                                                                                                                                                                                                                                                                      |
| **F6.7, F7.7**                 | Presence of a REP prefix preceding IDIV will invert the sign of the quotient, therefore REP prefixes are prepended to 10% of IDIV tests. This was only recently discovered by reenigne                                                                                                                                                                                                                                                                                                                          |
| **FE** | Forms with ModR/M reg field of 2-7 are undefined and are not included in the main test set.                                                                                                                                                                                                                                                                                                                                                                                                                     |                                                                                                                                                                                                                                                                                                                                                                                                                          |
| **FF** | Forms with a ModR/M reg field of 2-7 and a register operand are undefined and not included in the main test set.                        

### Tricky Bits and Advice

If you are validating memory operations for both addresses and values, you might struggle a bit with `IDIV` exceptions. During the call to the Type-0 interrupt "exception" handler, the flags register will be written to the stack - containing several undefined flags. You will need some strategy to mark when flags are on the stack so you can mask them, or simply disable validation of memory operation values when you detect exceptions occur. One good way to detect an exception is if a test performs four successive reads from addresses `0x00000`-`0x00003`.

Challenge: Actually implement the undefined flags for division! Hint: You'll need to look at the microcode...

### metadata.json

If you are not interested in emulating the undefined behavior of the 8086 (or not ready to yet), you can use the included `metadata.json` file which lists which instructions are undocumented or undefined and provides masks for undefined flags.

```json
{
    "url": "https://github.com/SingleStepTests/8086/",
    "version": "1.0.0",
    "syntax_version": 2,
    "cpu": "8086",
    "cpu_detail": "Intel P80C86A-2 782003 J20",
    "generator": "arduino808X",
    "date": "2025",
    "opcodes": {
        "00": {
            "status": "normal"
        },
        "01": {
            "status": "normal"
        },
        "02": {
            "status": "normal"
        },
        "03": {
            "status": "normal"
        },
        ...
```

In metadata.json, opcodes are listed as object keys under the `opcodes` field, each key being the opcode hexadecimal string representation padded to two digits. Each opcode entry has a `status` field which may be `normal`, `prefix`, `alias`, `undocumented`, `undefined`, or `fpu`.

An opcode marked `prefix` is an instruction prefix. These opcodes will not have individual tests.
An opcode marked `alias` is simply an alias for another instruction. These exist because the mask that determines which microcode address maps to which opcode is not always perfectly specific. 
An opcode marked `undocumented` has well-defined and potentially useful behavior, such as SETMO and SETMOC. 
An opcode marked `undefined` likely has unusual or unpredictable behavior of limited usefulness.
An opcode marked `fpu` is an FPU instruction (ESC opcode).

If present, the `flags` field indicates which flags are undefined after the instruction has executed. A flag is either a letter from the pattern `odiszapc` indicating it is undefined, or a period, indicating it is defined. The `flags-mask` field is a 16 bit value that can be applied with an AND to the flags register after instruction execution to clear any flags left undefined.

An opcode may have a `reg` field which will be an object of opcode extensions/register specifiers represented by single digit string keys - this is the "reg" field of the modrm byte.  Certain opcodes may be defined or undefined depending on their register specifier or opcode extension. Therefore, each entry within this `reg` object will have the same fields as a top-level opcode object. 

