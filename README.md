<p align="center">
  <img src="https://raw.githubusercontent.com/faustinoaq/prompt2elf/refs/heads/main/assets/prompt2elf-logo.svg" alt="Prompt2ELF elf wizard casting a command into executable bytes" width="1000">
</p>

# Prompt2ELF

**Can a coding agent go straight from a prompt to an executable, without writing source code first?**

Most coding agents stop at source code and leave the rest to a compiler. Prompt2ELF gives the agent the instructions and tiny bootstrap it needs to produce Linux x86-64 programs as final machine-code bytes. The agent lays out the ELF file, encodes the instructions, and calls the kernel directly. A 334-byte helper named `hexwriter.bin` turns the resulting hexadecimal text into an executable.

```text
/prompt2elf forge "hello, world"
```

That request produces a 167-byte Hello World executable. The same process produced a 322-byte Mandelbrot renderer and a 449-byte HTTP server, without a compiler or assembler.

The point is not to replace compilers for normal software development. Prompt2ELF is for learning how executables really work, building unusually small self-contained programs, and exploring a direct interface between human intent and machine code.

## See it work

Requirements: Linux on x86-64.

```bash
git clone https://github.com/faustinoaq/prompt2elf.git
cd prompt2elf
./bin/hello.bin
```

Output:

```text
Hello, world!
```

Render a Mandelbrot set:

```bash
./bin/mandelbrot.bin
```

Run the HTTP server:

```bash
./bin/server.bin &
SERVER_PID=$!
curl http://127.0.0.1:9000/
kill "$SERVER_PID"
```

The server binds to `0.0.0.0:9000`, so it is reachable through every available network interface while running.

## Included programs

| Program | Size | What it does |
| --- | ---: | --- |
| `hello.bin` | 167 bytes | Writes `Hello, world!` and exits |
| `hexwriter.bin` | 334 bytes | Converts hexadecimal text into an executable file |
| `mandelbrot.bin` | 322 bytes | Computes a 64 by 32 ASCII Mandelbrot set |
| `server.bin` | 449 bytes | Serves `Hello, world!` over HTTP on port 9000 |

Every executable has a matching source under `hex/`. The source is plain hexadecimal, not assembly.

## Use it as an Agent Skill

The repository root is a standard Agent Skills package. Clone or copy the complete repository into a skill directory recognized by your coding agent, keeping the directory name `prompt2elf`.

| Agent | Project skill location |
| --- | --- |
| Claude Code | `.claude/skills/prompt2elf/` |
| Devin | `.agents/skills/prompt2elf/` |
| GitHub Copilot | `.github/skills/prompt2elf/` |
| OpenCode | `.agents/skills/prompt2elf/` |

Then invoke it directly:

```text
/prompt2elf forge "print the current process ID"
```

Or describe the task naturally:

```text
Use Prompt2ELF to create a loopback-only HTTP status server.
```

`forge` is an action inside the Prompt2ELF skill. It does not invoke Foundry or another installed command named `forge`.

## How it works

1. The coding agent reads `SKILL.md` for the target ABI, encoding workflow, and safety rules.
2. It designs the ELF layout and writes the complete executable as hexadecimal text.
3. `hexwriter.bin` decodes that text, writes the bytes, and marks the output executable.
4. The rebuilt file is compared byte for byte and tested as a native Linux process.

Rebuild Hello World yourself:

```bash
./bin/hexwriter.bin /tmp/hello.bin < hex/hello.hex
/tmp/hello.bin
```

The writer can also reproduce itself:

```bash
./bin/hexwriter.bin /tmp/hexwriter-copy.bin < hex/hexwriter.hex
cmp -s ./bin/hexwriter.bin /tmp/hexwriter-copy.bin
```

## What zero-toolchain means

Reproducing these binaries does not require a compiler, assembler, linker, libc, package manager, or language runtime. Prompt2ELF still depends on an x86-64 processor, the Linux syscall ABI, the kernel's ELF loader, and a filesystem. The machine has not disappeared. The conventional source-to-binary toolchain has.

## Limits

- The bundled profile supports only Linux x86-64.
- Instruction offsets and ELF sizes must be exact, so small changes can require several recalculations.
- The examples favor size and clarity over complete production error handling.
- Generated native code should be inspected and run with minimal privileges.
- For ordinary application development, use a compiler.

## Documentation

- [`SKILL.md`](SKILL.md) contains the coding-agent workflow and constraints.
- [`hex/README.md`](hex/README.md) explains every bundled executable at the byte level.
- [`hex/`](hex/) contains the canonical hexadecimal sources.
- [`bin/`](bin/) contains the matching runnable artifacts.

The longer-term direction is a small multicall executable, similar in spirit to BusyBox, generated directly from prompts and Linux syscalls.
