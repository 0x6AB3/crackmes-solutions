FindThePassword1 - Xeeven Available at: https://crackmes.one/crackme/632cf67b33c5d4425e2cd501

How to run:
1) Using 7zip, extract the .7z file, repeat with the .tar file 
        (password=crackmes.one) (use 7z x <filename> to extract)
2) Enter the findthepassword1 directory after extracting all files
3) Run the file with ./findthepassword1 and follow the instructions

How to find password (static-analysis):
1) Open findthepassword1 with Radare2 (r2 <file>)
2) Analyse the binary (aaa)
3) List the functions (afl)
        Here we see entry0, this is where the program begins
        Note the memory address 0x08049001
4) Seek the entrypoint address (s 0x08049001)
        We can use 'pd n' to print n disassembly lines from this address
        pd 75 will show all the content we need for this context
5)
