## Description

> Daddy told me about cool MD5 hash collision today.
> I wanna do something like that too!

This challenge demonstrates how a “hash collision” can be achieved through **integer arithmetic over raw binary input**, rather than cryptographic weakness.

---

## Initial Triage

### File Enumeration

```bash id="t1"
ls -la
```

```text id="t2"
-r-xr-sr-x   1 root col_pwn 15164 Mar 26  2025 col
-rw-r--r--   1 root root      589 Mar 26  2025 col.c
-r--r-----   1 root col_pwn    26 Apr  2  2025 flag
```

**Observations**

* Binary is setgid
* flag is protected
* source code is provided

This suggests a **logic / arithmetic challenge**, not memory corruption.

---

## Binary Characteristics

```bash id="t3"
file col
```

```text id="t4"
setgid ELF 32-bit LSB pie executable, Intel 80386, dynamically linked, not stripped
```

Key implications:

* 32-bit arithmetic
* pointer casting depends on architecture
* integer overflow wraps naturally

---

## Source Code Analysis

```c id="t5"
unsigned long hashcode = 0x21DD09EC;

unsigned long check_password(const char* p){
    int* ip = (int*)p;
    int i;
    int res = 0;

    for(i = 0; i < 5; i++){
        res += ip[i];
    }

    return res;
}
```

---

## Key Observations

* Input is cast from `char*` to `int*`
* Data is interpreted as **five 32-bit integers**
* Only the **sum of these integers matters**
* No cryptographic hash is used

This is a **linear checksum collision problem**, not a cryptographic hash collision.

---

## Validation Logic

The program checks:

```c id="t6"
sum(ip[0] to ip[4]) == 0x21DD09EC
```

Expanded:

```text id="t7"
(ip[0] + ip[1] + ip[2] + ip[3] + ip[4]) mod 2^32 == 0x21DD09EC
```

---

## Dynamic Confirmation (GDB)

At runtime:

* `argv[1]` is treated as raw bytes
* each iteration reads 4 bytes as one integer
* loop runs exactly 5 times

So input is split into:

```text id="t8"
[4 bytes][4 bytes][4 bytes][4 bytes][4 bytes]
```

---

## Integer Overflow Behavior

Because arithmetic is 32-bit:

* overflow wraps automatically
* no bounds checking exists

This allows arbitrary sum construction.

---

## Constructing the Payload

### Strategy

Pick four equal integers and solve for the fifth:

```text id="t9"
sum = hashcode
```

### Calculation

```python id="t10"
hashcode = 0x21DD09EC
base = 0x01010101

remaining = (hashcode - base * 4) & 0xffffffff
```

Result:

```text id="t11"
remaining = 0x1DD905E8
```

---

### Final Values

```text id="t12"
ip[0] = 0x01010101
ip[1] = 0x01010101
ip[2] = 0x01010101
ip[3] = 0x01010101
ip[4] = 0x1DD905E8
```

---

## Payload Execution

```bash id="t13"
./col "$(printf \
'\x01\x01\x01\x01\
\x01\x01\x01\x01\
\x01\x01\x01\x01\
\x01\x01\x01\x01\
\xE8\x05\xD9\x1D')"
```

---

## Result

```text id="t14"
Two_hash_collision_Nicely
```

---

## Takeaway

This challenge demonstrates that:

* “hashing” can be just arithmetic, not cryptography
* casting `char*` to `int*` changes data interpretation completely
* integer overflow is predictable and exploitable
* collisions can be constructed deterministically without brute force

No memory corruption was required - only **understanding how raw bytes are interpreted**.
