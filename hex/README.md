# Prompt2ELF hexadecimal sources

This directory contains the canonical hexadecimal source for every executable in `../bin/`.

Each source is plain hexadecimal text. `../bin/hexwriter.bin` reads the digits from standard input, combines each pair into one byte, creates the requested output file, and marks it executable.

No source file contains assembly syntax, directives, symbols, relocations, comments, or compiler metadata.

## Build commands

Run these commands from the repository root:

```bash
./bin/hexwriter.bin /tmp/hello.bin < hex/hello.hex
./bin/hexwriter.bin /tmp/hexwriter.bin < hex/hexwriter.hex
./bin/hexwriter.bin /tmp/mandelbrot.bin < hex/mandelbrot.hex
./bin/hexwriter.bin /tmp/server.bin < hex/server.hex
```

The writer creates or truncates the output path. Do not point it at a file that must be preserved.

## Source rules

Follow these rules when editing or generating a `.hex` file:

1. Use only hexadecimal digits and whitespace.
2. Keep an even number of hexadecimal digits.
3. Do not use `0x` prefixes.
4. Do not put comments in the source.
5. Recalculate every affected branch displacement and RIP-relative address.
6. Update `p_filesz` and `p_memsz` when the final file size changes.
7. Keep the ELF entry address synchronized with the first instruction.
8. Rebuild with `hexwriter.bin` and compare the result with the intended artifact.

Comments are especially unsafe because letters from `a` through `f` are valid hexadecimal digits. The writer ignores other non-hexadecimal characters, so a prose comment can silently add unwanted nibbles.

Line breaks have no binary meaning. They only group related fields and instructions for human review.

## Shared ELF64 layout

Every current example uses the same minimal Linux x86-64 ELF structure:

| File range | Size | Purpose |
| --- | ---: | --- |
| `0x00` to `0x3f` | 64 bytes | ELF64 executable header |
| `0x40` to `0x77` | 56 bytes | One `PT_LOAD` program header |
| `0x78` onward | Variable | Machine code and inline data |

The first eight source lines encode the 120-byte ELF and program header pair.

| Source line | File offset | Encoded fields |
| --- | --- | --- |
| 1 | `0x00` | ELF magic, 64-bit class, little-endian data, ELF version, System V ABI |
| 2 | `0x10` | `ET_EXEC`, x86-64 machine ID, ELF version, entry address `0x400078` |
| 3 | `0x20` | Program-header offset `0x40`, zero section-header offset |
| 4 | `0x30` | Flags and ELF table sizes, one program header, zero section headers |
| 5 | `0x40` | `PT_LOAD`, readable and executable flags, file offset zero |
| 6 | `0x50` | Virtual and physical base address `0x400000` |
| 7 | `0x60` | Total file size as `p_filesz` and `p_memsz` |
| 8 | `0x70` | Segment alignment `0x1000` |

The load segment maps the complete file at virtual address `0x400000`. The kernel starts execution at `0x400078`, which is file offset `0x78` inside that mapping.

There is no dynamic interpreter and no section table. The programs invoke Linux directly with `syscall`.

## Linux x86-64 syscall convention

The examples use the following register convention:

```text
rax = syscall number
rdi = argument 1
rsi = argument 2
rdx = argument 3
r10 = argument 4
r8  = argument 5
r9  = argument 6
```

The return value is placed in `rax`. Values from `-4095` through `-1` indicate kernel errors.

## `hello.hex`

### Purpose

Creates `bin/hello.bin`, a 167-byte executable that writes `Hello, world!` followed by a newline and exits with status zero.

`bin/hello-from-writer.bin` is a byte-for-byte copy produced by feeding this source to `hexwriter.bin`.

### File layout

| Range | Size | Contents |
| --- | ---: | --- |
| `0x00` to `0x77` | 120 bytes | ELF and program headers |
| `0x78` to `0x98` | 33 bytes | Machine instructions |
| `0x99` to `0xa6` | 14 bytes | UTF-8 and ASCII message |

Total size is `0xa7`, encoded as `a700000000000000` twice on line 7.

### Instruction map

| Offset | Hexadecimal | Operation |
| --- | --- | --- |
| `0x78` | `b801000000` | Put syscall number 1, `write`, in `eax` |
| `0x7d` | `bf01000000` | Put file descriptor 1, standard output, in `edi` |
| `0x82` | `488d3510000000` | Load the RIP-relative message address into `rsi` |
| `0x89` | `ba0e000000` | Put message length 14 in `edx` |
| `0x8e` | `0f05` | Enter the kernel and perform `write` |
| `0x90` | `b83c000000` | Put syscall number 60, `exit`, in `eax` |
| `0x95` | `31ff` | Set exit status in `edi` to zero |
| `0x97` | `0f05` | Enter the kernel and terminate the process |
| `0x99` | `48656c6c6f2c20776f726c64210a` | Store `Hello, world!` and newline |

The message has no zero terminator because `write` receives its exact length.

### Verification

```bash
./bin/hexwriter.bin /tmp/hello.bin < hex/hello.hex
cmp -s bin/hello.bin /tmp/hello.bin
/tmp/hello.bin
```

## `hexwriter.hex`

### Purpose

Creates `bin/hexwriter.bin`, a 334-byte bootstrap utility that turns plain hexadecimal text into an executable file.

Usage:

```bash
./bin/hexwriter.bin OUTPUT_PATH < INPUT.hex
```

It accepts lowercase and uppercase hexadecimal digits. Whitespace and other non-hexadecimal bytes are skipped. Use only hexadecimal and whitespace to avoid ambiguous input.

### File layout

| Range | Size | Contents |
| --- | ---: | --- |
| `0x00` to `0x77` | 120 bytes | ELF and program headers |
| `0x78` to `0x14d` | 214 bytes | Argument handling, parser, file output, and exit paths |

Total size is `0x14e`, encoded as `4e01000000000000` twice on line 7.

### Control-flow groups

| Source lines | Behavior |
| --- | --- |
| 9 to 18 | Require an output argument, call `open`, reject an open failure, and save the file descriptor |
| 19 to 20 | Reserve one stack byte and clear the high-nibble state |
| 21 to 28 | Read one input character from standard input and stop at end of file |
| 29 to 40 | Convert `0-9`, `a-f`, or `A-F` into a value from 0 through 15 |
| 41 to 46 | Save the first nibble, shift it left four bits, and return to the read loop |
| 47 to 55 | Combine the second nibble, write one output byte, clear parser state, and loop |
| 56 to 59 | Call `fchmod` with mode `0755` |
| 60 to 65 | Close the output and exit with status zero |
| 66 to 68 | Exit with status one when no output path exists or `open` fails |

### Syscalls

| Number | Name | Use |
| ---: | --- | --- |
| 0 | `read` | Read one textual character from standard input |
| 1 | `write` | Write one decoded byte to the output |
| 2 | `open` | Create or truncate the output with flags `0x241` |
| 3 | `close` | Close the output descriptor |
| 60 | `exit` | Return success or failure |
| 91 | `fchmod` | Set executable mode `0755` |

### Self-hosting verification

```bash
./bin/hexwriter.bin /tmp/hexwriter-copy.bin < hex/hexwriter.hex
cmp -s bin/hexwriter.bin /tmp/hexwriter-copy.bin
```

A successful comparison proves that the writer can reproduce its own exact executable bytes.

## `mandelbrot.hex`

### Purpose

Creates `bin/mandelbrot.bin`, a 322-byte executable that computes and prints a 64 by 32 ASCII Mandelbrot set.

The image is calculated at runtime. It is not stored as pre-rendered text.

### File layout

| Range | Size | Contents |
| --- | ---: | --- |
| `0x00` to `0x77` | 120 bytes | ELF and program headers |
| `0x78` to `0x141` | 202 bytes | Fixed-point renderer and output loop |

Total size is `0x142`, encoded as `4201000000000000` twice on line 7.

### Numeric representation

The renderer uses signed Q10 fixed-point values:

```text
1024 = 1.0
4096 = 4.0
```

Coordinates are generated as:

```text
cx = -2048 + column * 48
cy =  -768 + row * 48
```

For each character cell, the program performs at most 32 iterations:

```text
zx2 = (zx * zx) >> 10
zy2 = (zy * zy) >> 10

escape when zx2 + zy2 > 4096

new_zy = ((zx * zy) >> 9) + cy
new_zx = zx2 - zy2 + cx
```

Shifting the cross product by 9 implements both multiplication by 2 and division by the Q10 scale.

### Control-flow groups

| Source lines | Behavior |
| --- | --- |
| 9 to 12 | Reserve stack space, install the character palette, and initialize the row counter |
| 13 to 15 | Calculate `cy` and reset the column counter |
| 16 to 20 | Calculate `cx` and reset `zx`, `zy`, and the iteration counter |
| 21 to 30 | Calculate squared terms and test the escape radius |
| 31 to 41 | Update the complex value and repeat up to 32 iterations |
| 42 to 46 | Select `@` for bounded points or a palette character for escaped points |
| 47 to 50 | Store one character and continue the column loop |
| 51 to 59 | Append newline, write a 65-byte row, and continue the row loop |
| 60 to 62 | Exit with status zero |

The eight-byte palette stored on the stack is:

```text
 .:-=+*#
```

A point that survives all iterations uses `@`.

### Verification

```bash
./bin/hexwriter.bin /tmp/mandelbrot.bin < hex/mandelbrot.hex
cmp -s bin/mandelbrot.bin /tmp/mandelbrot.bin
/tmp/mandelbrot.bin
```

The program emits 32 lines and 2,080 bytes, including one newline per row.

## `server.hex`

### Purpose

Creates `bin/server.bin`, a 449-byte HTTP/1.0 server that listens on `0.0.0.0:9000` and returns `Hello, world!` for every accepted request.

The server is sequential. It handles one client, closes that client, and returns to `accept`.

### File layout

| Range | Size | Contents |
| --- | ---: | --- |
| `0x00` to `0x77` | 120 bytes | ELF and program headers |
| `0x78` to `0x171` | 250 bytes | Socket setup, accept loop, request read, response send, and error exit |
| `0x172` to `0x1c0` | 79 bytes | HTTP headers and response body |

Total size is `0x1c1`, encoded as `c101000000000000` twice on line 7.

### Control-flow groups

| Source lines | Behavior |
| --- | --- |
| 9 to 17 | Reserve stack space, create an IPv4 stream socket, and save its descriptor |
| 18 to 25 | Enable `SO_REUSEADDR` with `setsockopt` |
| 26 to 30 | Build a 16-byte `sockaddr_in` on the stack |
| 31 to 37 | Bind the socket and branch to failure exit on error |
| 38 to 43 | Listen with backlog 16 and branch to failure exit on error |
| 44 to 51 | Accept a client and retry when `accept` is interrupted or fails |
| 52 to 58 | Read up to 480 request bytes and close clients that send no data |
| 59 to 66 | Send the fixed 79-byte response with `MSG_NOSIGNAL` |
| 67 to 70 | Close the client and return to the accept loop |
| 71 to 73 | Exit with status one after socket, bind, or listen failure |
| 74 to 78 | Store the complete HTTP response |

The port bytes in the stack structure are `23 28`. They are network byte order for hexadecimal `0x2328`, which is decimal port 9000.

### Syscalls

| Number | Name | Use |
| ---: | --- | --- |
| 0 | `read` | Consume the HTTP request |
| 3 | `close` | Close each client socket |
| 41 | `socket` | Create the listening socket |
| 43 | `accept` | Accept one client |
| 44 | `sendto` | Send the response with `MSG_NOSIGNAL` |
| 49 | `bind` | Bind `0.0.0.0:9000` |
| 50 | `listen` | Start listening with backlog 16 |
| 54 | `setsockopt` | Enable `SO_REUSEADDR` |
| 60 | `exit` | Terminate after a startup failure |

### Embedded response

```http
HTTP/1.0 200 OK
Content-Type: text/plain
Content-Length: 14

Hello, world!
```

The actual line endings in the binary are HTTP `CRLF` bytes, `0d0a`.

### Verification

```bash
./bin/hexwriter.bin /tmp/server.bin < hex/server.hex
cmp -s bin/server.bin /tmp/server.bin
/tmp/server.bin &
SERVER_PID=$!
curl http://127.0.0.1:9000/
kill "$SERVER_PID"
```

Binding to `0.0.0.0` exposes the service through every available network interface. Use a loopback address in new server programs unless the user explicitly requests external reachability.

## Adding another example

Use this sequence when adding a program:

1. Define the exact Linux x86-64 behavior and syscall boundary.
2. Create an offset ledger for headers, code, data, labels, and branch targets.
3. Encode the instructions and inline data manually.
4. Calculate the final file size.
5. Insert that size into both line-7 size fields.
6. Save pure hexadecimal text as `hex/NAME.hex`.
7. Generate `bin/NAME.bin` with `bin/hexwriter.bin`.
8. Verify ELF metadata, file size, permissions, behavior, and exit status.
9. Rebuild into a temporary path and compare both files byte for byte.
10. Document the new example here and in the repository README.

The hexadecimal source is canonical. Keep the executable and source synchronized in every commit.
