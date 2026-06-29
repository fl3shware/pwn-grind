## Description

> Daddy, teach me how to use random value in programming!

This challenge demonstrates that rand() without a seed is deterministic. Its output can be predicted using static analysis or a debugger. The goal is to compute the correct XOR input to retrieve the flag.

---

## Code-Level Analysis

```c
int main(){
    unsigned int random;
    random = rand();
    unsigned int key = 0;
    scanf("%d", &key);

    if ((key ^ random) == 0xcafebabe){
        setregid(getegid(), getegid());
        system("/bin/cat flag");
    }
}
```

---

## Key Observations

* rand() is called without srand()
  default seed is 1
  output is deterministic
* win condition is
  key XOR random equals 0xcafebabe
* XOR is reversible
  key equals random XOR 0xcafebabe
* no memory corruption involved
  only logic exploitation

---

## Vulnerability Analysis

The vulnerability is the use of rand() without initialization using srand().

When srand() is not called, rand() uses a default seed which makes its output predictable.

This is a logic vulnerability, not a memory corruption bug.

---

## Exploit Development

### Step 1 Recover rand output

Using GDB or static analysis

0x6b8b4567

This is the fixed output of rand() with seed 1

---

### Step 2 Compute key

key equals 0x6b8b4567 XOR 0xcafebabe
key equals 0xa1263dd9
key equals 2708864985

---

## Execution

./random
2708864985

Output
Good
m0mmy_I_can_predict_rand0m_v4lue

---

## Result

m0mmy_I_can_predict_rand0m_v4lue

---

## Takeaway

* rand() without srand() is predictable
* XOR checks are reversible if one value is known
* debugging or static analysis is enough
* secure randomness should use /dev/urandom or getrandom
