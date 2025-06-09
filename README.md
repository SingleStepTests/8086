## 8086 V1

This is a set of 8086 CPU tests produced by Daniel Balsom using the [ArduinoX86](https://github.com/dbalsom/arduinoX86) interface and the [MartyPC](https://github.com/dbalsom/martypc) emulator.

### Current version: 1.0.0

 - The ```v1``` directory contains the current version of the 8086 test suite.

### About the Tests

These tests are produced with an Intel P80C86A-2 CPU, running in Maximum Mode. Bus signals are provided via an Intel 8288 Bus Controller.

2,000 tests are provided per opcode.

This is far less than the 10,000 tests included in the [8088 Test Set](https://github.com/SingleStepTests/8088/). My experience using my own tests is that 10,000 tests was way more than required to identify any issues these tests have a chance of uncovering. Therefore, this test set is considered supplemental to the 8088 tests, intended for those who may be interested in validating or otherwise investigating the bus activity and prefetching behavior of the 8086 CPU specifically.

All tests assume a full 1MB of RAM is mapped to the processor and writable. Bear in mind that on the 8086, the address space wraps around at FFFFF.

No wait states are incurred during any of the tests. The interrupt and trap flags are not exercised. 

This test set exercises the 8086's processor instruction queue. All the instructions execute from as full a prefetch queue as possible (5 bytes for instructions starting at odd addresses, 6 bytes otherwise).

Registers state is randomized, however each register has a 2% chance of being set to 0.  Memory contents are also randomized, but each test has a 2% chance of having memory set to all 0x00 or 0xFF. The rationale for this is to attempt to exercise more edge cases and ensure test coverage of the zero flag.

### Using the Tests

The simplest way to run each test is to override your emulated CPU's normal reset vector (FFFF:0000) to the CS:IP of the test's initial state, then reset the CPU. 

#### Prefetched Tests

- If cycle-accurate verification of bus activity is desired, the provided initial queue contents should be set after the CPU reset routine flushes the instruction queue. This should trigger your emulator to suspend prefetching since the queue is full. In my emulator, I provide the CPU with a vector of bytes to install in the queue before I call ```cpu.reset()```. You could also pass your reset method a vector of bytes to install.

- You will need to add the length of the queue contents to your PC register (or adjust the reset vector IP) so that fetching begins at the correct address. 

- Once you read the first instruction byte out of the queue to begin test execution, prefetching should be resumed since there is now room in the queue. It takes two cycles to begin a fetch after reading from a full queue, therefore tests that specify an initial queue state will start with two 'Ti' cycle states.

- If you're not interested in per-cycle validation, you can ignore the queue fields entirely.

### Why Are Instructions Longer Than Expected?

Instruction cycles begin from the cycle in which the CPU's queue status lines indicate that an instruction "First Byte" has been fetched.

Instructions cycles end when the queue status lines indicate that the first byte of the next instruction has been read from the queue. If the instruction started fully prefetched and the next instruction byte was present in the queue at the end of the instruction, the instruction cycles should reflect the documented "best case" instruction timings. If the queue was empty at the end of execution, the extra cycles spent fetching the next instruction byte may lengthen the instruction from documented timings. There is no indication from the CPU when an instruction ends - only when a new one begins.

### Segment Override Prefixes

Random segment override prefixes have been prepended to a percentage of instructions, even if they may not do anything. This isn't completely useless - a few bugs have been found where segment overrides had an effect when they should not have.

### String Prefixes

String instructions may be randomly prepended by a REP, REPE, REPNE instruction prefix. In this event, CX is masked to 7 bits to produce reasonably sized tests (A string instruction with CX==65535 would be over a million cycles in execution). 

### Instruction Prefetching

All bytes fetched after the initial instruction bytes are set to 0x90 (144) (NOP). Therefore, the queue contents at the end of all tests will contain only NOPs, with a maximum of 3 (since one has been read out).

### Test Formats

The 8086 test suite comes in two formats, JSON and a simple chunked binary format I call `MOO` (Machine Opcode Operation).
If your language of choice lacks great options for parsing JSON (such as C) you may prefer the binary format tests.

You can find more information about the binary format in README.md in the /v1_binary directory. There is also an example MOO parser in C under the [/tools directory of the 8088 CPU Test repo](https://github.com/SingleStepTests/8088/tree/main/tools)

Information on the JSON format proceeds below, but also serves as a detailed reference that is useful for interpreting the binary tests as well.

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

If you are not interested in writing a cycle-accurate emulator, you can ignore this section.

The `cycles` list contains sub lists, each corresponding to a single CPU cycle. Each contains several fields. From left to right, the cycle fields are:  

 - Pin bitfield
 - Multiplexed bus
 - segment status
 - memory status
 - IO status
 - BHE (Byte high enable) status (active-low)
 - data bus
 - bus status
 - T-state
 - queue operation status
 - queue byte read

The first column is a bitfield representing certain chip pin states. 

 - Bit #0 of this field represents the ALE (Address Latch Enable) pin output, which in Maximum Mode is output by the i8288. This signal is asserted on T1 to instruct the PC's address latches to store the current address. This is necessary since the address and data lines of the 8086 are multiplexed, and a full, valid address is only on the bus while ALE is asserted.
 - Bit #1 of this field represents the INTR pin input. This is not currently exercised, but may be in future test releases.
 - Bit #2 of this field represents the NMI pin input. This is not currently exercised, but may be in future test releases.

The `Multiplexed bus` is the value of the 20 address/data/status lines on the current cycle, read directly from the CPU. It contains a valid address only on T1 when ALE is active. During other T states it contains status pins and data bus contents. 

The `segment status` indicates which segment is in use to calculate addresses by the CPU, using segment-offset addressing. This field represents the S3 and S4 status lines of the 8086. It may be ES, DS, CS, SS, or invalid ("--").

The `memory status` field represents outputs of the attached i8288 Bus Controller. From left to right, this field will contain any combination of the letters "RAW" or "---".  'R' represents an active MRDC status line, 'A' represents an active AMWC status line, and 'W' represents an active MWTC status line. These status lines are active-low. A memory read will occur on T3 or the last Tw t-state when MRDC is active. A memory write will occur on T3 or the last Tw t-state when AMWC is active. At this point, the value of the data bus field will be valid and will represent the byte read or written.

The `IO status` field represents outputs of the attached i8288 Bus Controller. From left to right, this field will contain any combination of the letters "RAW" or "---".  'R' represents an active IORC status line. 'A' represents an active AIOWC status line. 'W' represents an active IOWC status line. These status lines are active-low. An IO read will occur on T3 or the last Tw t-state when IORC is active. An IO write will occur on T3 or the last Tw t-state when AIOWC is active. At this point, the value of the data bus field will be valid and will represent the byte read or written.

#### BHE

The 8086 CPU assumes an odd/even memory organization, similar in design to the 286 in the IBM AT. An 8086-based system does not typically access memory at odd locations - instead only words are addressed, at even memory locations only, with even bytes of memory are mapped to the lower 8 bits of the data bus, and odd bytes mapped to the upper 8 bits. The A0 line is therefore omitted from directly addressing the memory chips. A combination of A0 and BHE are used to drive the motherboard logic that activates the appropriate bus lines and memory banks.

The `BHE status` indicates whether the high half of the 16-bit data bus is being driven. It is active-low (0). If a bus cycle begins at an even address with BHE active, it is a 16-bit transfer. If a bus cycle begins at an odd address with BHE active, it is an 8-bit transfer, where the high (odd) byte of the data bus is active.  If a bus cycle begins at an even address with BHE inactive, it is an 8-bit transfer where the low (even) byte of the data bus is active. It is important to handle this logic correctly - the inactive half of the bus, if any, should be masked/ignored when validating against the tests.

The `data bus` indicates the value of the last 16 bits of the multiplexed bus. It is typically only valid on T3. 

The `bus status` lines indicate the type of bus m-cycle currently in operation. Either INTA, IOR, IOW, MEMR, MEMW, HALT, CODE, or PASV.  These states represent the S0-S2 status lines of the 8086.

The `T-state` is the current T-state of the CPU. Since this state is not exposed by the CPU, it is calculated based on bus activity.

The `queue operation status` will contain either `F`, `S`, `E` or `-`. `F` indicates a "First Byte" of an instruction or instruction prefix has been read.  `S` indicates a "Subsequent" byte of an instruction has been read - either a modr/m, displacement, or operand. `E` indicates that the instruction queue has been Emptied/Flushed. All queue operation statuses reflect an operation that actually occurred on the previous cycle.  This field represents the QS0 and QS1 status lines of the 8086. 

When the queue operation status is not `-`, then the value of the `queue byte read` field is valid and represents the byte read from the queue. 

For more information on the 8086 and 8288 status lines, see their respective white papers.

### Undefined Instructions

Note that these tests include many undocumented/undefined opcodes and instruction forms. The 8086 has no concept of an invalid instruction, and will perform some task for any provided sequence of instruction bytes. Additionally, flags may be changed by documented instructions in ways that are officially undefined.

### Per-Instruction Notes

 - **0F**: `POP CS` is not a particularly useful instruction, and is omitted from the test set due to prefetching complications.
 - **8F**: The behavior of 8F with reg != 0 is undefined. If you can figure out the rules governing its behavior, please let us know.
 - **9B**: WAIT is not included in this test set.
 - **8C,8E**: These instructions are only defined for a reg value of 0-3, however only the first two bits are checked, so the test set contains random values for reg.
 - **8D,C4,C5**: 'r, r' forms of LEA, LES, LDS are undefined. These forms are not included in this test set due to disruption of the last calculated EA by the CPU set up routine.
 - **A4-A7,AA-AF**: CX is masked to 7 bits. This provides a reasonable test length, as the full 65535 value in CX with a REP prefix could result in over one million cycles.
 - **C6,C7**: Although the reg != 0 forms of these instructions are officially undefined, this field is ignored. Therefore, the test set contains random values for reg.
 - **D2,D3**: CL is masked to 6 bits. This shortens the possible test length, while still hopefully catching the case where CL is improperly masked to 5 bits (186+ behavior).
 - **E4,E5,EC,ED**: All forms of the IN instruction should return 0xFF or 0xFFFF on IO read.
 - **F0, F1**: The LOCK prefix is not exercised in this test set.
 - **F4**: HALT is not included in this test set.
 - **D4, F6.6, F6.7, F7.6, F7.7** - These instructions can generate a divide exception (more accurately, a Type-0 Interrupt). When this occurs, cycle traces continue until the first byte of the exception handler is fetched and read from the queue. The IVT entry for INT0 is set up to point to 1024 (0400h).
     - NOTE: On the 8086 specifically, the return address pushed to the stack on divide exception is the address of the next instruction. This differs from the behavior of later CPUs and generic Intel IA-32 emulators.
 - **F6.7, F7.7** - Presence of a REP prefix preceding IDIV will invert the sign of the quotient, therefore REP prefixes are prepended to 10% of IDIV tests. This was only recently discovered by [reenigne](https://www.reenigne.org/blog/)
 - **FE**: The forms with reg field 2-7 are undefined and are not included in this initial release.

### metadata.json

If you are not interested in emulating the undefined behavior of the 8086, you can use the included `metadata.json` file which lists which instructions are undocumented or undefined and provides masks for undefined flags.

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

