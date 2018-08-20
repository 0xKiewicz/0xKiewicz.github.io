---
layout: post
title:  "SLAE x86 Exam - Assignment 5 - Part 1"
categories: certs slae 
---


## Assignment #5 - Analyzing shellcode
_Student ID: **PA-7854**_

The assignment indications are as follows:
* Take up at least 3 shellcode samples from Metasploit for linux/x86
* Use GDB/Ndisasm/Libemu to dissect them


The code is available at [my github repo](https://github.com/0xKiewicz/SLAE)

I will shortly describe what are we going to do. First, we need to choose 3 different payloads, so I have chosen these three:
* `linux/x86/exec`
* `linux/x86/chmod`
* `linux/x86/read_file`

We will generate payloads with `msfvenom` which is the newer tool that replaces `msfpayload` and `msfencode`, both deprecated. Let's start with the first one:

#### linux/x86/exec

I generate the shellcode as follows:
```bash
# msfvenom -p linux/x86/exec CMD=whoami -f raw --platform linux -a x86 -o /tmp/exec-whoami.txt
No encoder or badchars specified, outputting raw payload
Payload size: 42 bytes
Saved as: /tmp/exec-whoami.txt
```

We dissect the generated file with `ndisasm`:
```bash
# ndisasm -u /tmp/exec-whoami.txt
00000000  6A0B              push byte +0xb
00000002  58                pop eax
00000003  99                cdq
00000004  52                push edx
00000005  66682D63          push word 0x632d
00000009  89E7              mov edi,esp
0000000B  682F736800        push dword 0x68732f
00000010  682F62696E        push dword 0x6e69622f
00000015  89E3              mov ebx,esp
00000017  52                push edx
00000018  E807000000        call 0x24
0000001D  7768              ja 0x87
0000001F  6F                outsd
00000020  61                popa
00000021  6D                insd
00000022  6900575389E1      imul eax,[eax],dword 0xe1895357
00000028  CD80              int 0x80
```

Let's analyze it line by line
```nasm
00000000  6A0B              push byte +0xb
```
We push the value 0xb (11 in decimal) which corresponds to the syscall `execve`.

```nasm
00000002  58                pop eax
```
Here we pop out that value and assign it to EAX.

```nasm
00000003  99                cdq
```
_Convert Double Quad_. This is like a `xor edx, edx`, it copies the sign bit (+ or -, bit 31) of the value in EAX into every bit of the EDX register.

If you remember from previous assignments, the way `execve` function parameters are:
```c
int execve(const char *filename, char *const argv[], char *const envp[]);
```
Which we usually push a sequence like:
`[command][NULL][*address of command][NULL]`, let's see how this is being implemented.

```nasm
00000004  52                push edx
```
Save the value of EDX (which are NULL bytes), string delimiter, in stack.

```nasm
00000005  66682D63          push word 0x632d
```
Save the value '-c' in stack, which stands for _command_ (see below).

```nasm
00000009  89E7              mov edi,esp
```
Save the pointer which ESP holds (-c) in EDI.

```nasm
0000000B  682F736800        push dword 0x68732f
00000010  682F62696E        push dword 0x6e69622f
```
These two lines save in stack the string `hs/nib/` (remember _little endian_?) which is the primary command we will execute.

```nasm
00000015  89E3              mov ebx,esp
```
Save the pointer which ESP holds (/bin/sh) in EBX, first parameter of `execve` function.

```nasm
00000017  52                push edx
```
Save the value of EDX (which are NULL bytes), string delimiter, in stack.


```nasm
00000018  E807000000        call 0x24
```
Call function located at address 0x24. 
This is a technique so that the next instruction's address is saved in stack (`push edi` in this case). Next 4 lines are ambiguous.
Call address location, which is 0x018, +0x4 (holds 4 bytes) + 0x1 should be the address holding the next instruction, which starts right below at 0x1f.
We can confirm this with `xxd`:

```bash
# echo 77686f616d6900 | xxd -r -p
whoami
```
That's the command previously entered in `msfvenom`. Let's keep analyzing, by skipping the first 36 characters, until the NULL byte after `whoami` command, so that `ndisasm` shows the instructions correctly:

```bash
# ndisasm -u -e 36 /tmp/exec-whoami.txt 
00000000  57                push edi
00000001  53                push ebx
00000002  89E1              mov ecx,esp
00000004  CD80              int 0x80
```

```nasm
00000000  57                push edi
```
EDI holds the pointer to ESP (-c) so we push that to stack.

```nasm
00000001  53                push ebx
```
EBX holds the primary command, `/bin/sh`.

```nasm
00000002  89E1              mov ecx,esp
```
This sets ECX to the list of arguments (argv).

```nasm
00000004  CD80              int 0x80
```
This instruction should be familiar, makes an interrupt and executes the syscall.

Basically the syscall contains: *execve("/bin/sh, ["bin/sh, "-c", "whoami"], \0)*
 
