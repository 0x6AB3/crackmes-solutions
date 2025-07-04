# Crackme Writeup: FindThePassword1 - Xeeven

**Available at:** [https://crackmes.one/crackme/632cf67b33c5d4425e2cd501](https://crackmes.one/crackme/632cf67b33c5d4425e2cd501)

---

## How to Run

1.  **Extract the archives:**
    Using 7zip, extract the `.7z` file, then repeat with the `.tar` file. The password for the archives is `crackmes.one`.

    ```bash
    7z x 632cf67b33c5d4425e2cd501.zip
    7z x findthepassword1.tar.7z
    7z x findthepassword1.tar
    ```

2.  **Navigate to the directory:**
    Enter the `findthepassword1` directory after extracting all files.

    ```bash
    cd findthepassword1/
    ```

3.  **Execute the program:**
    Run the executable and follow its instructions.

    ```bash
    ./findthepassword1
    ```

---

## How to Find the Password (Dynamic Analysis Walkthrough)

This section details a dynamic analysis approach using Radare2 to uncover the correct password.

1.  **Open with Radare2 in debug mode:**

    ```bash
    r2 -d ./findthepassword1
    ```

2.  **Analyze the binary:**
    This command performs an auto-analysis, finding functions, strings, and other useful information.

    ```
    aaa
    ```
    *(Example output might show `entry0` at `0x08049001`)*

3.  **List functions:**
    Identify the program's entry point. `entry0` is typically where the program begins execution. Note its memory address.

    ```
    afl
    ```
    *(Example output might show `entry0` at `0x08049001`)*

4.  **Seek to the entrypoint:**
    Move Radare2's current seek address to the program's entry point.

    ```
    s 0x08049001
    ```

5.  **Print disassembly:**
    Display a sufficient number of disassembly lines to view the main runtime logic. `pd 75` should cover the relevant section for this challenge.

    ```
    pd 75
    ```

6.  **Identify the password comparison:**
    Look for instructions that compare strings. The `repe cmpsb` instruction is a strong indicator of a string comparison.

    ```assembly
    0x0804909a      f3a6           repe cmpsb byte [esi], byte es:[edi]
    ```
    This instruction compares two strings in memory, pointed to by `ESI` and `EDI`.

7.  **Set a breakpoint and continue:**
    Set a breakpoint at the comparison instruction and continue execution until the breakpoint is hit.

    ```
    db 0x0804909a # Creates a breakpoint at the comparison
    dc           # Continues running until the breakpoint is reached
    ```
    After you enter a password into the running program, the breakpoint will be triggered.

8.  **Examine strings in registers:**
    At the breakpoint, the `ESI` and `EDI` registers will hold pointers to the strings being compared.
    *   `ps @ edi`: Shows the string you entered (your password).
    *   `ps @ esi`: Shows the correct password that the program is comparing against.

    ```
    ps @ edi
    ps @ esi
    ```
    **Correct password identified:** `8675309`

    Using this password will allow you to see the success message.

---

## How to Patch (Walkthrough)

This section demonstrates how to patch the binary to bypass the password check, allowing any input to grant access.

1.  **Open with Radare2 in write mode:**
    Ensure you open the binary with write permissions.

    ```bash
    r2 -w ./findthepassword1
    ```

2.  **Analyze and find entry point:**
    Perform auto-analysis and list functions to find the entry point address (e.g., `0x08049001`).

    ```
    aaa
    afl
    ```

3.  **Seek to the entrypoint:**

    ```
    s 0x08049001
    ```

4.  **Find the conditional jump:**
    Print disassembly (`pd 75`) to locate the conditional jump instruction that occurs after the password check. This instruction determines whether the program proceeds to the success or failure path.

    ```assembly
    0x0804909c      e319           jecxz 0x80490b7
    ```
    This `jecxz` instruction (Jump if ECX is Zero) normally jumps to `0x80490b7` (which leads to the congratulation message) *only if* the `ECX` register is zero.

5.  **Seek to the jump instruction:**

    ```
    s 0x0804909c
    ```

6.  **Change to an unconditional jump:**
    Modify the `jecxz` instruction to an unconditional `jmp` to the success path (`0x80490b7`). This ensures the program always jumps to the congratulation message, regardless of the password entered.

    ```
    wa jmp 0x80490b7
    ```
    *(Note: If `wa` fails due to instruction size differences, you might need to use `wx` with the calculated hex bytes for the `jmp` instruction, e.g., `wx e916000000 @ 0x0804909c` if `0x16` is the correct relative offset.)*

7.  **Save changes:**
    Write the modified binary to disk.

    ```
    wf
    ```

8.  **Exit Radare2:**

    ```
    q
    ```

9.  **Run the patched program:**
    Execute the patched binary and input any password. It should now display the success message.

    ```bash
    ./findthepassword1
    ```

---

![image](https://github.com/user-attachments/assets/97261c51-7a8b-4a10-a44c-71460cf35a4e)
