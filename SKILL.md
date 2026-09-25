---
name: prompt2elf
description: Generates, modifies, explains, and verifies raw Linux x86-64 ELF executables from hexadecimal machine code without a compiler, assembler, linker, libc, or language runtime. Use for direct binary generation, syscall-only programs, ELF byte-layout work, hexwriter bootstrapping, instruction encoding, or changes to the bundled Hello World, Mandelbrot, and HTTP server examples.
compatibility: Requires Linux x86-64 and permission to read, write, and execute files. Uses the bundled bin/hexwriter.bin to materialize hexadecimal sources. Network programs may require permission to bind a user-approved local port.
metadata:
  author: Faustino Aguilar
  version: "1.1.0"
  target: linux-x86-64
  format: elf64
---

# Prompt2ELF

Generate final Linux x86-64 executable bytes directly from a program specification. Keep a readable hexadecimal source and its executable output synchronized.

## Use this skill when

Activate Prompt2ELF when the user asks to:

- Create a raw Linux x86-64 executable without a compiler or assembler
- Encode or decode ELF64 headers and x86-64 instructions
- Build a syscall-only command, renderer, server, or utility
- Modify one of the programs under `hex/` and `bin/`
- Rebuild binaries with `hexwriter.bin`
- Explain offsets, opcodes, registers, syscalls, or inline data
- Verify that hexadecimal sources reproduce committed executables
- Design a small BusyBox-style multicall binary

If the requested target is not Linux x86-64, state that the bundled profile does not apply. Obtain the architecture, operating system ABI, executable format, and endianness before generating bytes for another target.

## Invocation shorthand

Treat this invocation:

```text
/prompt2elf forge "PROGRAM DESCRIPTION"
```

as a request to design, encode, materialize, and verify a new raw executable from the quoted description. For example:

```text
/prompt2elf forge "hello, world"
```

The word `forge` is scoped under the Prompt2ELF skill. It is not a standalone executable and must not invoke Foundry, a Rust package, or any other installed `forge` command.

## Resolve the skill root

Treat the directory containing this `SKILL.md` as `SKILL_ROOT`.

Before generating or changing a binary:

1. Read `hex/README.md` from `SKILL_ROOT`.
2. Read the closest existing hexadecimal example.
3. Use paths under `SKILL_ROOT` for the bundled writer, sources, and outputs.
4. Do not assume the caller's current working directory is the skill directory.

## Required constraints

1. Do not use a compiler, assembler, linker, `objcopy`, or another source-to-binary tool.
2. Do not use Python, Node.js, Perl, Ruby, or another language runtime to emit executable bytes.
3. Use `bin/hexwriter.bin` to materialize `.hex` files unless the host interface can write arbitrary binary bytes directly.
4. Keep a canonical pure-hexadecimal source for every generated executable.
5. Keep source and executable changes in the same commit.
6. Do not insert comments, `0x` prefixes, labels, or prose into `.hex` files.
7. Never guess branch displacements, RIP-relative offsets, lengths, or ELF sizes. Calculate them explicitly.
8. Preserve little-endian encoding for ELF fields and x86 immediates. Use network byte order only where a protocol requires it.
9. Do not overwrite an existing output until its purpose and preservation requirements are understood.
10. Treat every generated executable as untrusted until it passes structural and behavioral validation.

Optional inspection commands such as `file`, `stat`, `cmp`, `readelf`, or `objdump` may validate an artifact. The generation process must not depend on them.

## Safety requirements

- Generate only software the user is authorized to create and run.
- Do not create credential theft, persistence, evasion, destructive payloads, exploit delivery, or unauthorized access capabilities.
- Default network listeners to loopback. Bind `0.0.0.0` only when the user explicitly requests external reachability.
- Ask before opening a public listener, replacing an existing executable, or performing an operation with external side effects.
- Run new binaries with minimal privileges in a sandbox, container, VM, or similarly restricted environment when available.
- Never embed credentials, tokens, private keys, or other secrets in executable data.

## Supported baseline

The bundled profile is:

```text
Architecture: x86-64
Endianness: little-endian
Operating system: Linux
Executable format: ELF64
Object type: ET_EXEC
Image base: 0x400000
Entry file offset: 0x78
Entry virtual address: 0x400078
Program headers: one PT_LOAD entry
Section headers: none
Dynamic interpreter: none
Runtime interface: Linux syscalls
```

The baseline file begins with:

| Range | Size | Purpose |
| --- | ---: | --- |
| `0x00` to `0x3f` | 64 bytes | ELF64 header |
| `0x40` to `0x77` | 56 bytes | `PT_LOAD` program header |
| `0x78` onward | Variable | Instructions and inline data |

Use a readable and executable load segment for immutable code and inline data. Use stack memory for temporary writable state in small programs. If a task needs writable static storage, design appropriate segment flags and mappings rather than silently making all code writable.

## Linux syscall ABI

Use the Linux x86-64 syscall convention:

```text
rax = syscall number
rdi = argument 1
rsi = argument 2
rdx = argument 3
r10 = argument 4
r8  = argument 5
r9  = argument 6
```

`syscall` returns a value in `rax` and clobbers `rcx` and `r11`.

Common syscall numbers used by this repository:

| Number | Name |
| ---: | --- |
| 0 | `read` |
| 1 | `write` |
| 2 | `open` |
| 3 | `close` |
| 9 | `mmap` |
| 41 | `socket` |
| 42 | `connect` |
| 43 | `accept` |
| 44 | `sendto` |
| 45 | `recvfrom` |
| 49 | `bind` |
| 50 | `listen` |
| 54 | `setsockopt` |
| 60 | `exit` |
| 91 | `fchmod` |
| 257 | `openat` |

Check syscall arguments and constants against the target Linux x86-64 ABI. Do not transfer numbers from another architecture.

## Generation workflow

### 1. Define behavior

Write down:

- Inputs and outputs
- Exit behavior
- Required syscalls
- Error paths
- Mutable state
- Inline data
- User-visible constraints

For a network program, also define address, port, protocol, backlog, connection lifecycle, and response framing.

### 2. Allocate registers and memory

Assign every persistent value to a register or stack slot. Account for registers clobbered by `syscall` and by each instruction sequence.

Prefer a simple stack frame for buffers and state. Confirm that buffer ranges do not overlap.

### 3. Build an offset ledger

Before emitting hexadecimal, create a temporary ledger with these columns:

```text
file offset | virtual address | length | bytes | operation | target
```

Record every instruction, label, data block, and header field.

Use these formulas:

```text
next_instruction = instruction_offset + instruction_length
relative_branch = target_offset - next_instruction
rip_displacement = data_offset - next_instruction
total_file_size = header_size + code_size + data_size
virtual_address = image_base + file_offset
```

Encode signed negative displacements in two's-complement little-endian form.

### 4. Encode instructions

For each instruction:

1. Select the exact opcode form.
2. Determine operand width.
3. Determine required REX prefix bits.
4. Encode ModR/M and SIB bytes when needed.
5. Encode immediates and displacements in little-endian order.
6. Add its exact length to the ledger.

Do not optimize until a clear version is correct. Compactness comes after correctness.

### 5. Finalize ELF metadata

Calculate and encode:

- `e_entry`
- `e_phoff`
- `e_ehsize`
- `e_phentsize`
- `e_phnum`
- `p_offset`
- `p_vaddr`
- `p_filesz`
- `p_memsz`
- `p_flags`
- `p_align`

For the baseline layout, `p_filesz` and `p_memsz` equal the complete file size. Update both whenever code or data length changes.

### 6. Resolve all references again

After final sizes are known, independently recalculate:

- Every short and near branch
- Every loop back edge
- Every RIP-relative data address
- Every embedded string length
- Every protocol content length
- Every file and memory size

A one-byte change can invalidate every later relative displacement.

### 7. Write pure hexadecimal source

Store the canonical source at:

```text
hex/NAME.hex
```

Line breaks may group headers, instructions, and data. The parser ignores whitespace.

### 8. Materialize the executable

From `SKILL_ROOT`, run:

```bash
./bin/hexwriter.bin bin/NAME.bin < hex/NAME.hex
```

For an initial test, write to a new temporary output path instead of replacing a committed binary.

### 9. Validate structure

Verify at minimum:

- Expected file size
- Executable permission
- ELF64 class
- x86-64 machine type
- Entry address
- Load-segment bounds and flags
- Absence of an unintended dynamic interpreter
- Source rebuild is byte-for-byte reproducible

Example reproduction check:

```bash
./bin/hexwriter.bin /tmp/NAME.bin < hex/NAME.hex
cmp -s bin/NAME.bin /tmp/NAME.bin
```

### 10. Validate behavior

Run the narrowest behavior test that proves the program works:

- Capture standard output and exit status for command-line tools
- Count dimensions for renderers
- Use a loopback client for servers
- Exercise success and expected failure paths
- Confirm that long-running processes can be stopped cleanly

Do not declare success based only on ELF recognition.

### 11. Document and report

When adding an example:

1. Add `hex/NAME.hex`.
2. Add `bin/NAME.bin`.
3. Document purpose, size, layout, control-flow groups, syscalls, and verification in `hex/README.md`.
4. Add user-facing execution instructions to `README.md`.
5. Report exact artifact size, target, behavior, and validation results.

## Modifying an existing example

1. Rebuild the current source into a temporary path.
2. Confirm it matches the committed binary.
3. Read its section in `hex/README.md`.
4. Update the offset ledger before changing bytes.
5. Modify the hexadecimal source.
6. Recalculate sizes and references.
7. Generate the new binary with `hexwriter.bin`.
8. Run structural and behavioral tests.
9. Update documentation when behavior, size, syscalls, or layout changes.

Never patch only `bin/NAME.bin`. The `.hex` source is canonical.

## Networking requirements

For raw socket programs:

- Use the correct network byte order for ports and addresses.
- Enable `SO_REUSEADDR` when repeatable local testing needs it.
- Use `MSG_NOSIGNAL` or another deliberate strategy for disconnected peers.
- Handle failed `socket`, `bind`, and `listen` calls.
- Close accepted client descriptors.
- Keep HTTP `Content-Length` synchronized with body bytes.
- State whether the server is sequential or concurrent.
- Default to `127.0.0.1` unless external binding was explicitly requested.

## Completion criteria

A Prompt2ELF task is complete only when:

- The hexadecimal source is present and valid.
- The executable was generated from that source.
- Header sizes and offsets are consistent.
- Every relative reference was recalculated.
- The output has executable permission.
- Structural validation passes.
- Behavioral validation passes.
- A fresh rebuild is byte-identical.
- Documentation reflects the final artifact.
- No compiler, assembler, linker, or language runtime participated in byte generation.

Use `hex/README.md` as the detailed reference for the bundled examples and `README.md` for installation and user-facing commands.
