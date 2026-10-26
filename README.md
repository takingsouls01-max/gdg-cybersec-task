# gdg-cybersec-task
Cybersecurity challenge solutions for GDG Tech Wildcard : Apple Pi3 and Hidden Recipe, with reproducible walkthroughs, scripts, and documented findings.
*Thinking...*


# Task 1 — Apple Pie Binary Analysis

**Category:** Cybersecurity / Binary Analysis  
**Technology:** Linux binary analysis tools  
Binary `apple_pie`  
**Flag:** `gdg{P1E_3xpl01ted_lol}`

---

## Problem Statement

The GDG Cybersecurity Team released a Linux binary as an internal test build. The binary uses modern security mechanisms and basic obfuscation.

The objective was to:

- Identify the file type and architecture.
- Verify the integrity of the downloaded binary.
- Analyze the binary's security protections.
- Understand the password-checking logic.
- Identify potential vulnerabilities.
- Recover the hidden flag.

---

## Provided Artifact

The binary was provided through Google Drive.

Expected SHA-256 hash:

```text
26ede057655358bec1ae71ba51391397986b342e9cbf3c3422a4eb845c12c4d9
```

The task description referred to the binary as `apple_pi3`, while the downloaded file was used as:

apple_pie


1. Download and Verify the Binary

After downloading the file, I placed it inside a working directory:


mkdir apple_pie_analysis
cd apple_pie_analysis


The binary was then renamed to:
text
apple_pie


Before analyzing or executing it, I verified its SHA-256 checksum:
sha256sum apple_pie

Expected output:
26ede057655358bec1ae71ba51391397986b342e9cbf3c3422a4eb845c12c4d9  apple_pie
# 2. Identify the File Type

The first analysis command was:


file apple_pie


The result identified the program as an ELF executable, for example:


ELF 32-bit LSB pie executable, Intel 80386, dynamically linked


This provided several important pieces of information:

- It is an **ELF** file, which is the standard executable format on Linux.
- It is a **32-bit** executable.
- It uses the **Intel 80386** architecture.
- It is a **PIE**, or Position Independent Executable.
- It is dynamically linked.

Because it was a 32-bit program, the disassembly used 32-bit registers such as:


eax
ebx
ecx
edx
esp
ebp


This was important later when examining the binary in GDB.

---

# 3. Check the File Permissions

Initially, the file might not have had executable permissions. I granted execution permission using:

chmod +x ./apple_pie

The permissions were checked with:


# 4. Run the Program Normally

The binary was executed with:

./apple_pie

The program displayed a password prompt:


=== The Elusive PIE ===
Enter password:

The binary also asked for another message after the password check.


# 5. Extract Readable Strings

The next step was to inspect readable strings embedded in the binary:


strings -a -t x apple_pie


The `-a` option searches the entire file, while `-t x` displays the hexadecimal file offset of each string.

Important strings included:


=== The Elusive PIE ===
Enter password:
Hmm... interesting.
Wrong password... but maybe there's still a way in.
Say something to the program:
You found the hidden treasure: %s
not_the_password


The string:


not_the_password


was a strong indication of the correct password.

The string:

```text
You found the hidden treasure: %s
```

suggested that a hidden code path or function existed that could print the flag.

The read-only data section could also be inspected with:

```bash
readelf -p .rodata apple_pie
```

---

# 6. Check Security Protections

The binary's security protections were checked with:

```bash
checksec --file=apple_pie
```

Depending on the installed version of `checksec`, the output showed protections such as:

- PIE enabled
- NX enabled
- Stack canary status
- RELRO status

The important protection for this challenge was PIE.

PIE means that the program's code is designed to run correctly even when loaded at a different memory address. This is normally combined with ASLR, which randomizes memory locations.

Because of PIE and ASLR, hardcoded runtime addresses cannot always be used directly. Instead, the program calculates addresses relative to a base register, such as `ebx`.

---

# 7. Disassemble the Main Function

The main function was inspected using:

```bash
objdump -d -M intel --disassemble=main apple_pie
```

GDB could also be used:

```bash
gdb -q ./apple_pie
```

Inside GDB:

```gdb
set pagination off
disassemble main
```

The disassembly contained instructions similar to:

```asm
lea eax,[ebx-0x1fc8]
mov DWORD PTR [ebp-0xc],eax
```

The instruction:

```asm
lea eax,[ebx-0x1fc8]
```

calculates an address relative to the value stored in `ebx`.

This is typical of position-independent code. Rather than using one fixed absolute address, the program calculates the location of data relative to the PIE base.

The following instruction:

```asm
mov DWORD PTR [ebp-0xc],eax
```

stores the calculated address in a local stack variable.

This was useful because it showed where the program saved a pointer to the string data.

---

# 8. Why We Used a Breakpoint Near `main+41`

The breakpoint was placed near the instruction that stored the calculated address:

```gdb
break *main+41
```

The exact offset can vary depending on the binary and disassembly, so it was selected by examining the instructions around the address calculation.

The reason for stopping at this point was:

1. `ebx` had already been initialized.
2. The PIE-relative address calculation had already occurred.
3. The address had been saved at `[ebp-0xc]`.
4. The pointer could then be inspected while the program was running.

The program must be started before registers can be examined:

```gdb
run
```

Trying to inspect `$eax` before `run` produced an error similar to:

```text
The program is not being run.
```

Once GDB stopped at the breakpoint, the saved pointer was inspected with:

```gdb
x/s *(char **)($ebp-0xc)
```

This displayed:

```text
"not_the_password"
```

The register could also be inspected directly:

```gdb
info registers eax
x/s $eax
```

The important lesson was that GDB registers such as `$eax`, `$ebx`, and `$ebp` are only available after the process has started.

---

# 9. Understanding the Password Check

The password check was effectively equivalent to code similar to:

```c
if (strncmp(input, "not_the_password",
            strlen("not_the_password")) == 0) {
    puts("Hmm... interesting.");
} else {
    puts("Wrong password... but maybe there's still a way in.");
}
```

The important function was:

```c
strncmp()
```

`strncmp` compares two strings for a specified number of characters. It returns zero when the compared characters match.

In this case, the correct input was:

```text
not_the_password
```

The password check itself worked correctly.

The earlier failure was caused by entering:

```text
not_the__password
```

with two underscores.

Therefore, the issue was not that `strncmp` failed. The issue was that the supplied password did not exactly match the expected string.

---

# 10. Test the Second Input

After entering the correct password, the program asked:

```text
Say something to the program:
```

A format-string test was supplied:

```text
%p.%p.%p.%p.%p.%p
```

This could also be supplied non-interactively:

```bash
printf 'not_the_password\n%%p.%%p.%%p.%%p.%%p.%%p\n' | ./apple_pie
```

The program printed values similar to:

```text
0x80.0xefbde5a0.0x64e2926e.(nil).(nil).(nil)
```

This demonstrated that the input was being interpreted as a format string.

The program likely used code equivalent to:

```c
printf(user_input);
```

This is unsafe.

The secure version should have been:

```c
printf("%s", user_input);
```

When `%p` is passed directly as the format string, `printf` attempts to read pointer arguments from the stack. Since the program did not provide matching arguments, unrelated stack values were printed.

Eventually, the program accessed an invalid value and terminated with:

```text
Segmentation fault (core dumped)
```

The crash was evidence of a format-string vulnerability.

---

# 11. Locate the Hidden Function

The binary contained a function named:

```text
def_nothing_important
```

The function name suggested that it was intentionally disguised as an unimportant function.

It could be located with:

```bash
nm -C apple_pie | grep def_nothing
```

or:

```bash
objdump -t apple_pie | grep def_nothing
```

It could also be examined in GDB:

```gdb
info address def_nothing_important
```

The function was disassembled with:

```gdb
disassemble def_nothing_important
```

The earlier string:

```text
You found the hidden treasure: %s
```

suggested that this function was responsible for printing the hidden result.

---

# 12. Call the Hidden Function in GDB

The binary did not include enough debugging type information for GDB to determine the function's return type automatically.

The first attempt produced an error similar to:

```text
has unknown return type
```

To solve this, the function was explicitly cast as returning `void`:

```gdb
call (void) def_nothing_important()
```

A complete GDB session was:

```bash
gdb -q ./apple_pie
```

Inside GDB:

```gdb
set pagination off
break main
run
call (void) def_nothing_important()
```

If a breakpoint was needed after the PIE setup, the following form was used:

```gdb
break *main+41
run
call (void) def_nothing_important()
```

The exact breakpoint offset was selected from the disassembly after identifying the instructions responsible for initializing the PIE base and calculating the string address.

The hidden function then printed:

```text
You found the hidden treasure: gdg{P1E_3xpl01ted_lol}
```

---

# 13. Final Flag

```text
gdg{P1E_3xpl01ted_lol}
```

---

# 14. Complete Command Summary

The main commands used during the analysis were:

```bash
mkdir apple_pie_analysis
cd apple_pie_analysis
```

```bash
sha256sum apple_pie
```

```bash
file apple_pie
```

```bash
chmod +x ./apple_pie
```

```bash
ls -l apple_pie
```

```bash
./apple_pie
```

```bash
strings -a -t x apple_pie
```

```bash
readelf -p .rodata apple_pie
```

```bash
checksec --file=apple_pie
```

```bash
objdump -d -M intel --disassemble=main apple_pie
```

```bash
objdump -t apple_pie | grep def_nothing
```

```bash
nm -C apple_pie | grep def_nothing
```

```bash
printf 'not_the_password\n%%p.%%p.%%p.%%p.%%p.%%p\n' | ./apple_pie
```

```bash
gdb -q ./apple_pie
```

GDB commands:

```gdb
set pagination off
disassemble main
break main
run
info registers
x/s $eax
x/s *(char **)($ebp-0xc)
info address def_nothing_important
disassemble def_nothing_important
call (void) def_nothing_important()
```

---

# 15. Security Findings

## PIE

The binary was compiled as a Position Independent Executable.

This makes the runtime address of code and data change between executions when ASLR is enabled.

The program therefore used relative addressing through registers such as `ebx`.

## Format-String Vulnerability

The second input was interpreted directly as a format string.

Unsafe pattern:

```c
printf(user_input);
```

Safe pattern:

```c
printf("%s", user_input);
```

This allowed stack values to be leaked with `%p` and caused a segmentation fault when invalid values were used.

## Password Handling

The password was checked using `strncmp`.

The check itself was not bypassed. The correct password was recovered from the binary's readable strings:

```text
not_the_password
```

## Hidden Function

A hidden function named:

```text
def_nothing_important
```

was present in the binary and could be called through GDB.

It printed the final flag.

---

# 16. Conclusion

The binary was first identified as a 32-bit PIE ELF executable. Its checksum was verified, execution permission was granted, and readable strings were extracted.

The embedded string:

```text
not_the_password
```

revealed the correct password. The password check used `strncmp` and worked correctly when the password was entered with the correct spelling.

The second input contained a format-string vulnerability because user input was passed directly to `printf`. Supplying `%p` leaked stack values and eventually caused a segmentation fault.

Disassembly showed PIE-relative address calculations using `ebx`. A breakpoint was placed after the relevant address calculation so the runtime pointer could be inspected.

Finally, the hidden function `def_nothing_important` was called in GDB using an explicit `void` cast:

```gdb
call (void) def_nothing_important()
```

This revealed the final flag:

```text
gdg{P1E_3xpl01ted_lol}
```
```
