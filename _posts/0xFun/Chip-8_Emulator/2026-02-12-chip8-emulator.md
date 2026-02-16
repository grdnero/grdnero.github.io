---
title: "0xFun CTF: CHIP-8 Emulator Backdoor"
date: 2026-02-12 12:00:00 +0200
tags: [reverse, pwn, writeup]
categories: [0xFun, chip8Emulator]
---

## Challenge Overview

> *"Ever wondered how emulators tick under the hood? I built one --- the simplest of all, a CHIP-8 emulator. Alongside it, I've dropped 100+ games and programs for you to play... or are they really only for playing? Somewhere deep in this virtual silicon, a flaw hides. Uncover it, and in just quad cycles, the flag is yours. Miss it, and you'll be stuck endlessly."*

This challenge provides a custom-built CHIP-8 emulator binary and a collection of ROMs. The goal is to identify a hidden flaw in the emulator implementation and exploit it to retrieve the flag.

---

## Phase 1: Initial Analysis

I began by inspecting the provided assets. The environment consists of a 64-bit Linux executable and a directory of standard CHIP-8 .ch8 files.

![File Analysis](/assets/img/0xFun/Chip-8_Emulator/files.png)

**Key Finding:** The binary is **not stripped**. Preserved symbols mean we can directly observe class methods and logic flow in our decompiler, such as Cpu::fetch() and Cpu::execute().

### Program Behavior

Running the emulator with the -l 4 (DEBUG) flag allows us to trace every instruction as it executes in real-time. This is useful for seeing how the emulator handles standard ROMs before attempting to trigger the backdoor.

![Debug Output](/assets/img/0xFun/Chip-8_Emulator/program.png)

---

## Phase 2: Reverse Engineering with IDA

Navigating to Emulator::run, we find the standard instruction cycle: **Fetch, Decode, and Execute**.

![Instruction Cycle](/assets/img/0xFun/Chip-8_Emulator/ic.png)

### The Logic Flaw

Within Cpu::execute, the emulator dispatches instructions based on the high nibble (the first 4 bits of the opcode). The vulnerability is hidden deep within the 0xF instruction handler (Cpu::decode_F_instruction).

![Execute Switch](/assets/img/0xFun/Chip-8_Emulator/execute.png)

Inside the 0xF handler, I found a non-standard branch that checks the low byte of the opcode:

![Backdoor Branch](/assets/img/0xFun/Chip-8_Emulator/bd.png)

The logic extracts the low byte and compares it to **255 (0xFF)**. If it matches, it calls Cpu::superChipRendrer. In a standard CHIP-8 specification, 0xFF is not a valid operation for the F series, identifying it as a hidden backdoor.

---

## Phase 3: The Quad Cycle Exploit

The "quad cycles" hint from the description implies the backdoor must be triggered exactly four times. Since CHIP-8 opcodes are 2 bytes long, I crafted a minimal ROM containing four F0 FF instructions.

```bash
# F0 (High Byte) + FF (Low Byte) repeated 4 times
printf "\xF0\xFF\xF0\xFF\xF0\xFF\xF0\xFF" > exploit.ch8
```

```bash 
./chip8Emulator -r exploit.ch8
```

```bash
cat flag.txt 
```

![Backdoor Branch](/assets/img/0xFun/Chip-8_Emulator/final.png)

