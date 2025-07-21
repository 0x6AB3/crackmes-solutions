# Reverse Engineering Challenge: Behind The Scenes

This writeup details the process of solving the "Behind The Scenes" reverse engineering challenge, which involves uncovering a hidden password and flag by analyzing a Linux ELF executable.

## Tools Used

*   `file`: For basic file type identification.
*   `radare2` (r2): A powerful reverse engineering framework for disassembling and debugging.

## Initial Analysis

First, let's examine the binary's type and run it to understand its basic behavior.

```bash
[PG@CherryIce rev_behindthescenes]$ file behindthescenes
behindthescenes: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=e60ae4c886619b869178148afd12d0a5428bfe18, for GNU/Linux 3.2.0, not stripped
```
The binary is a 64-bit, not stripped ELF executable, which means symbol names are available, aiding analysis.

Running the program without arguments or with an incorrect one:
```bash
[PG@CherryIce rev_behindthescenes]$ ./behindthescenes
./challenge <password>
[PG@CherryIce rev_behindthescenes]$ ./behindthescenes test
# (No output, program likely exits or crashes silently)
```
The program expects a password as a command-line argument.

## Reverse Engineering with Radare2

We'll use `radare2` in debug mode to analyze the binary.

```bash
[PG@CherryIce rev_behindthescenes]$ r2 -d behindthescenes
```

### Initial Analysis and Hidden Code

After performing auto-analysis (`aaa` or `aaaa`), we can list functions (`afl`). We observe a `main` function and a peculiar `sym.segill_sigaction`.

Disassembling `main`:
```assembly
┌ 135: int main (int argc, char **argv);
│ ...
│           0x563a4791c2b8      488d056aff..   lea rax, [sym.segill_sigaction] ; 0x563a4791c229
│           0x563a4791c2bf      48898560ff..   mov qword [var_a0h], rax
│           0x563a4791c2c6      c745e80400..   mov dword [var_18h], 4
│           0x563a4791c2cd      488d8560ff..   lea rax, [var_a0h]
│           0x563a4791c2d4      ba00000000     mov edx, 0
│           0x563a4791c2d9      4889c6         mov rsi, rax
│           0x563a4791c2dc      bf04000000     mov edi, 4
│           0x563a4791c2e1      e8fafdffff     call sym.imp.sigaction  ; int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact)
└           0x563a4791c2e6      0f0b           ud2
```
The `main` function registers `sym.segill_sigaction` as the handler for `SIGILL` (signal 4) using `sigaction`. Immediately after, it executes a `ud2` (undefined instruction). A `ud2` instruction causes a `SIGILL`.

### Understanding `sym.segill_sigaction`

Disassembling `sym.segill_sigaction`:
```assembly
┌ 56: sym.segill_sigaction (int64_t arg1, int64_t arg2, int64_t arg3);
│ ...
│           0x563a4791c248      488b80a800..   mov rax, qword [rax + 0xa8]
│           0x563a4791c24f      488d5002       lea rdx, [rax + 2]
│           0x563a4791c253      488b45f8       mov rax, qword [var_8h]
│           0x563a4791c257      488990a800..   mov qword [rax + 0xa8], rdx
│ ...
```
This signal handler takes the `ucontext_t` structure (passed as `arg3` or `rdx`), accesses the saved instruction pointer (`rip`) at offset `0xa8` within it, adds `2` to `rip`, and writes the new value back. This means the program will resume execution 2 bytes *after* the `ud2` instruction.

Therefore, after the `ud2` at `0x563a4791c2e6`, execution will jump to `0x563a4791c2e6 + 2 = 0x563a4791c2e8`. This is the "behind the scenes" logic.

### Following the Execution Flow

Let's examine the code starting from `0x563a4791c2e8`:
```assembly
            0x563a4791c2e8      83bd5cffff..   cmp dword [rbp - 0xa4], 2
        ┌─< 0x563a4791c2ef      741a           je 0x563a4791c30b
        │   0x563a4791c2f1      0f0b           ud2
        │   0x563a4791c2f3      488d3d0a0d..   lea rdi, str.._challenge__password_ ; 0x563a4791d004 ; "./challenge <password>"
        │   0x563a4791c2fa      e8d1fdffff     call sym.imp.puts       ; int puts(const char *s)
        │   0x563a4791c2ff      0f0b           ud2
        │   0x563a4791c301      b801000000     mov eax, 1
       ┌──< 0x563a4791c306      e92e010000     jmp 0x563a4791c439
       │└─> 0x563a4791c30b      0f0b           ud2
       │    0x563a4791c30d      488b8550ff..   mov rax, qword [rbp - 0xb0]
       │    0x563a4791c314      4883c008       add rax, 8
       │    0x563a4791c318      488b00         mov rax, qword [rax]
       │    0x563a4791c31b      4889c7         mov rdi, rax
       │    063a4791c31e      e8cdfdffff     call sym.imp.strlen     ; size_t strlen(const char *s)
       │    0x563a4791c323      4883f80c       cmp rax, 0xc            ; 12
       │┌─< 0x563a4791c327      0f8505010000   jne 0x563a4791c432
```

1.  **Argument Count Check:**
    *   `cmp dword [rbp - 0xa4], 2`: Compares `argc` (number of arguments) with 2.
    *   `je 0x563a4791c30b`: If `argc` is 2 (i.e., `./challenge <password>`), it jumps to `0x563a4791c30b`.
    *   Otherwise, it prints the usage message `./challenge <password>` and exits.

2.  **Password Length Check:**
    *   If `argc` is 2, execution continues at `0x563a4791c30b` (after another `ud2` skip).
    *   The program retrieves `argv[1]` (the provided password).
    *   `call sym.imp.strlen`: Calculates the length of the password.
    *   `cmp rax, 0xc`: Compares the length with `0xc` (decimal 12).
    *   `jne 0x563a4791c432`: If the length is not 12, it jumps to an exit path.

3.  **Segmented Password Comparisons (`strncmp`)**
    If the password length is 12, the program performs a series of `strncmp` calls, each comparing a 3-character segment of the input with a hardcoded string.

    *   **Segment 1 (offset 0, length 3):**
        ```assembly
        ...
        lea rsi, [0x563a4791d01b] ; "Itz"
        mov edx, 3
        call sym.imp.strncmp
        ...
        ```
        Compares the first 3 characters with "Itz".

    *   **Segment 2 (offset 3, length 3):**
        ```assembly
        ...
        add rax, 3  ; Moves pointer 3 bytes into input
        lea rsi, [0x563a4791d01f] ; "_0n"
        mov edx, 3
        call sym.imp.strncmp
        ...
        ```
        Compares characters 3-5 with "_0n".

    *   **Segment 3 (offset 6, length 3):**
        ```assembly
        ...
        add rax, 6  ; Moves pointer 6 bytes into input
        lea rsi, [0x563a4791d023] ; "Ly_"
        mov edx, 3
        call sym.imp.strncmp
        ...
        ```
        Compares characters 6-8 with "Ly_".

    *   **Segment 4 (offset 9, length 3):**
        ```assembly
        ...
        add rax, 9  ; Moves pointer 9 bytes into input
        lea rsi, [0x563a4791d027] ; "UD2"
        mov edx, 3
        call sym.imp.strncmp
        ...
        ```
        Compares characters 9-11 with "UD2".

4.  **Flag Printing:**
    If all `strncmp` calls return 0 (meaning the segments match), the program proceeds to print the flag:
    ```assembly
    ...
    lea rdi, str.__HTB_s_n  ; 0x563a4791d02b ; "> HTB{%s}
"
    mov rsi, rax            ; Your input password
    call sym.imp.printf
    ...
    ```
    The program uses `printf` to format the flag with your correct password.

## Solution

By concatenating the segments from the `strncmp` calls, we reconstruct the password:
`Itz` + `_0n` + `Ly_` + `UD2` = `Itz_0nLy_UD2`

Let's test it:
```bash
[PG @CherryIce rev_behindthescenes]$ ./behindthescenes Itz_0nLy_UD2
> HTB{Itz_0nLy_UD2}
```

## The Flag

The flag is: `HTB{Itz_0nLy_UD2}`
