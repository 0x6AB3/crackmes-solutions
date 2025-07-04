# Crackme Writeup: Yekong's Vending Machine

**Challenge Source:** (Please provide the correct link to the challenge, as `sudo pacman -S usbutils` is a command, not a URL.)

---

## Challenge Overview

This writeup details the process of bypassing the coin mechanism in the "Yekong's Vending Machine" program to acquire the flag. The program simulates a vending machine where items can be purchased with coins. The objective is to obtain 100 coins to "buy" the flag, which is otherwise unobtainable through normal program interaction.

---

## How to Run the Program

1.  **Acquire the executable:**
    Download the challenge file from the provided link.
    *(Assuming the file is named `vending_machine` after extraction)*

2.  **Execute the program:**
    Launch the vending machine program from your terminal.

    ```bash
    ./vending_machine
    ```
    Keep this terminal open, as we will interact with it during the analysis.

---

## Dynamic Analysis Walkthrough: Acquiring the Flag

This section outlines a dynamic analysis approach using `scanmem` to manipulate the program's memory and obtain the required coins.

### 1. Identify the Process ID (PID)

To interact with the running vending machine program, we first need its Process ID (PID). Open a **second terminal** and use `pgrep` or `ps aux` to find the PID of the `vending_machine` process.

```bash
pgrep vending_machine
# Example output: 125542
```
*(Note: The PID will vary each time the program is run.)*

### 2. Attach `scanmem` to the Process

Launch `scanmem` in the second terminal, attaching it to the `vending_machine` process using the PID you just found. `sudo` is required as `scanmem` needs elevated privileges to access another process's memory.

```bash
sudo scanmem <PID>
# Example: sudo scanmem 125542
```
You will see a `scanmem` prompt, e.g., `scanmem (125542)>`.

### 3. Initial Memory Scan for Coin Value

The program starts with 10 coins. We will use this initial value to begin our search in memory. In the `scanmem` terminal, enter `10` and press Enter.

```
scanmem (125542)> 10
```
`scanmem` will scan the entire memory space of the `vending_machine` process for all locations containing the value `10`. At this stage, it's normal to find a large number of results, as the value `10` can appear in many unrelated memory locations.

### 4. Narrowing Down the Search (Iterative Refinement)

To pinpoint the exact memory address storing our coin count, we need to change the value in the vending machine program and then re-scan in `scanmem`. This iterative process helps `scanmem` filter out irrelevant memory locations.

*   **Change Value in Vending Machine:** In the **first terminal** (where the vending machine is running), purchase a cheap item like a "Coca Cola" or "Pepsi" (costing 5 coins). This will change your coin balance from 10 to 5.

*   **Re-scan in `scanmem`:** In the **second terminal** (the `scanmem` prompt), enter the new coin value, `5`, and press Enter.

    ```
    scanmem (125542)> 5
    ```
    `scanmem` will now filter its previous results, keeping only those memory addresses that *previously held `10` and now hold `5`*. This significantly reduces the number of potential candidates. In many simple programs, this step might narrow it down to just one or a few results.

### 5. Manipulate Coin Count

Once `scanmem` has successfully narrowed down the results to a single memory address (or a very small, manageable set), we can directly modify the coin count. Our goal is to acquire 100 coins to buy the flag.

*   **Set Coins to 100:** In the `scanmem` terminal, use the `set` command to change the value at the identified memory address to `100`.

    ```
    scanmem (125542)> set 100
    ```
    If there was only one result, `scanmem` will apply the change. If there were multiple, it might prompt you to confirm or specify which address to modify.

### 6. Obtain the Flag

Return to the **first terminal** where the vending machine program is running. You should observe that your coin balance has instantly updated to `100`. Now, you can select option `3` to "Buy Flag".

```
3. Buy Flag (100 coins)
```
Upon successful purchase, the program will reveal the flag:

```
flag{v3nd1ng_m4ch1n3_1s_4w3s0m3}
```

### 7. Clean Up

After obtaining the flag, you can exit both terminals by pressing `CTRL + C`.

---

**Educational Insights:**

*   **Dynamic Analysis:** This walkthrough demonstrates a fundamental technique in dynamic analysis, where a program's behavior and state are examined while it is running. This contrasts with static analysis, which examines code without execution.
*   **Memory Manipulation:** Tools like `scanmem` allow direct interaction with a process's memory. This is crucial for understanding how programs store and manage data, and for identifying vulnerabilities or bypasses.
*   **Iterative Search:** The process of repeatedly changing a value and re-scanning is a common strategy in memory analysis. It leverages the fact that the target value's memory address will consistently reflect the changes, while other addresses will not.
*   **Process Isolation:** The need for `sudo` highlights the operating system's security mechanisms that prevent one process from arbitrarily modifying the memory of another without explicit permission.