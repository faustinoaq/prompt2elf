---
name: prompt2elf
description: Generates, modifies, explains, and verifies raw Linux x86-64 ELF executables from hexadecimal machine code without a compiler, assembler, linker, libc, or language runtime. Use for direct binary generation, syscall-only programs, ELF byte-layout work, hexwriter bootstrapping, instruction encoding, or changes to the bundled examples.
compatibility: Requires Linux x86-64 and permission to read, write, and execute files. Uses the bundled bin/hexwriter.bin to materialize hexadecimal sources.
metadata:
  author: Faustino Aguilar
  version: "1.0.0"
  target: linux-x86-64
  format: elf64
---

# Prompt2ELF project adapter

The canonical portable skill is located at `../../../SKILL.md`, relative to this file.

1. Read `../../../SKILL.md` in full.
2. Treat the repository root containing that file as `SKILL_ROOT`.
3. Follow every workflow, constraint, safety requirement, and completion criterion in the canonical skill.
4. Read `../../../hex/README.md` before generating or modifying executable bytes.

Do not use this adapter as a substitute for the canonical instructions.
