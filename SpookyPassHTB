# Reverse Engineering Challenge: SpookyPass

This tutorial outlines the process of reverse engineering a simple ELF binary named `pass` to find the correct password and ultimately, the hidden flag.

## Tools Used

*   `file`: For basic file type identification.
*   `radare2` (r2): A powerful reverse engineering framework for disassembling and debugging.

## Initial Analysis

First, let's examine the binary's type using the `file` command:

```bash
[PG @CherryIce rev_spookypass]$ file pass
pass: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=3008217772cc2426c643d69b80a96c715490dd91, for GNU/Linux 4.4.0, not stripped
```

This output tells us it's a 64-bit ELF executable, dynamically linked, and importantly, **not stripped**, which means symbol names (like `main`, `puts`, `strcmp`) are still present, making analysis easier.

Next, let's try running the program to understand its basic functionality:

```bash
[PG @CherryIce rev_spookypass]$ ./pass
Welcome to the SPOOKIEST party of the year.
Before we let you in, you'll need to give us the password: gsdfg
You're not a real ghost; clear off!
```

The program prompts for a password and rejects an incorrect one.

## Reverse Engineering with Radare2

We'll use `radare2` in debug mode to analyze the binary. The `-d` flag starts `r2` in debugger mode.

```bash
[PG @CherryIce rev_spookypass]$ r2 -d pass
```

Once inside `r2`, we need to analyze the binary to identify functions and code blocks. `aaa` performs an auto-analysis.

```r2
[0x7f1ad085a5c0]> aaa
INFO: Analyze all flags starting with sym. and entry0 (aa)
INFO: Analyze imports (af @@@i)
INFO: Analyze entrypoint (af @ entry0)
INFO: Analyze symbols (af @@@s)
INFO: Analyze all functions arguments/locals (afva @@@F)
INFO: Analyze function calls (aac)
INFO: Analyze len bytes of instructions for references (aar)
INFO: Finding and parsing C++ vtables (avrr)
INFO: Analyzing methods (af @@ method.*)
INFO: Recovering local variables (afva @@@F)
INFO: Skipping type matching analysis in debugger mode (aaft)
INFO: Propagate noreturn information (aanr)
INFO: Use -AA or aaaa to perform additional experimental analysis
```

Now, list all functions using `afl` and filter for known symbols to find `main`:

```r2
[0x7f1ad085a5c0]> afl | grep -v "fcn"
0x560b9c3b0030    1      6 sym.imp.puts
0x560b9c3b0040    1      6 sym.imp.__stack_chk_fail
0x560b9c3b0050    1      6 sym.imp.strchr
0x560b9c3b0060    1      6 sym.imp.printf
0x560b9c3b0070    1      6 sym.imp.fgets
0x560b9c3b0080    1      6 sym.imp.strcmp
0x560b9c3b0090    1     37 entry0
0x560b9c3b2fc0    1     32 reloc.__libc_start_main
0x560b9c3b02ec    1     13 sym._fini
0x560b9c3b0189   11    355 main
0x560b9c3b0000    3     27 sym._init
0x560b9c3b0180    5     60 entry.init0
0x560b9c3b0130    5     55 entry.fini0
0x560b9c3b2fe0    9   4166 reloc.__cxa_finalize
```

We can see `main` at `0x560b9c3b0189`. Let's seek to `main` and print its disassembly using `pdf` (print disassembly function):

```r2
[0x560b9c3b2fc0]> s main
[0x560b9c3b0189]> pdf
```

Scrolling through the disassembly of `main`, we look for interesting function calls, especially those related to input or string comparisons. We find a call to `sym.imp.strcmp`:

```assembly
           0x560b9c3b0243      488d15360e..   lea rdx, str.s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5 ; 0x560b9c3b1080 ; "s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5"
           0x560b9c3b024a      4889d6         mov rsi, rdx
           0x560b9c3b024d      4889c7         mov rdi, rax
           0x560b9c3b0250      e82bfeffff     call sym.imp.strcmp     ; int strcmp(const char *s1, const char *s2)
```

This snippet clearly shows that the program is comparing our input (`rdi`) with the string `s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5` (`rsi`). This is likely the correct password.

Further down, if the `strcmp` is successful (i.e., `eax` is 0), the program proceeds to print "Welcome inside!" and then constructs the flag:

```assembly
           0x560b9c3b0284      488d05d52d..   lea rax, obj.parts      ; 0x560b9c3b3060 ; U"HTB{un0bfu5c4t3d_5tr1ng5}"
```

The string `HTB{un0bfu5c4t3d_5tr1ng5}` is loaded into `rax` and then processed, indicating it's the flag.

## Solution

The password is `s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5`.

Let's run the binary with the discovered password:

```bash
[PG @CherryIce rev_spookypass]$ ./pass
Welcome to the SPOOKIEST party of the year.
Before we let you in, you'll need to give us the password: s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5
Welcome inside!
HTB{un0bfu5c4t3d_5tr1ng5}
```

## The Flag

The flag is: `HTB{un0bfu5c4t3d_5tr1ng5}`
