---
layout: post
title:  "SLAE x86 Exam - Assignment 5 - Part 3"
categories: certs slae 
---


## Assignment #5 - Analyzing shellcode
_Student ID: **PA-7854**_

In this third and last part, we will analize the next Metasploit's payload: *linux/x86/read_file*.

The code is available at [my github repo](https://github.com/0xKiewicz/SLAE).

#### linux/x86/read_file
We will generate the payload with `msfvenom` as follows:

```bash
# msfvenom -p linux/x86/read_file PATH=/etc/passwd --platform linux -a x86 -f raw -o /tmp/readfile-pwd.txt
No encoder or badchars specified, outputting raw payload
Payload size: 73 bytes
Saved as: /tmp/readfile-pwd.txt
```

Running it with no other parameters, FD defaults to 1 which is `stdout`, to read file and print content on screen. 

We dissect the generated file with `ndisasm`:
```nasm
# ndisasm -u /tmp/readfile-pwd.txt
00000000  EB36              jmp short 0x38
00000002  B805000000        mov eax,0x5
00000007  5B                pop ebx
00000008  31C9              xor ecx,ecx
0000000A  CD80              int 0x80
0000000C  89C3              mov ebx,eax
0000000E  B803000000        mov eax,0x3
00000013  89E7              mov edi,esp
00000015  89F9              mov ecx,edi
00000017  BA00100000        mov edx,0x1000
0000001C  CD80              int 0x80
0000001E  89C2              mov edx,eax
00000020  B804000000        mov eax,0x4
00000025  BB01000000        mov ebx,0x1
0000002A  CD80              int 0x80
0000002C  B801000000        mov eax,0x1
00000031  BB00000000        mov ebx,0x0
00000036  CD80              int 0x80
00000038  E8C5FFFFFF        call 0x2
0000003D  2F                das
0000003E  657463            gs jz 0xa4
00000041  2F                das
00000042  7061              jo 0xa5
00000044  7373              jnc 0xb9
00000046  7764              ja 0xac
00000048  00                db 0x00
```

Since there are a few syscalls, as we can see the multiple `int 0x80` instructions, I will analyze and explain it by blocks of `interrupts`.
```nasm
00000000  EB36              jmp short 0x38
00000002  B805000000        mov eax,0x5
00000007  5B                pop ebx
00000008  31C9              xor ecx,ecx
0000000A  CD80              int 0x80
```
First instruction jumps to address `0x38` which is a `call 0x2` instruction, so basically this is a technique called `JMP-CALL-POP`, since after a `call` instruction is executed the next instruction's address is saved, then we `pop` that saved value, in this case into EBX (third line, address 0x7).
Number `0x5` on second line, `mov eax, 0x5`, refers to the `open` syscall.

Do you ask what was saved in memory? Again, we convert from hex to string and we can discover it:
```bash
# echo 2f6574632f706173737764 | xxd -r -p
/etc/passwd
```

`/etc/passwd` was the value we set for FILE parameter, so it's the file being opened.

Let's move forward.
```nasm
0000000C  89C3              mov ebx,eax
0000000E  B803000000        mov eax,0x3
00000013  89E7              mov edi,esp
00000015  89F9              mov ecx,edi
00000017  BA00100000        mov edx,0x1000
0000001C  CD80              int 0x80
```

This block refers to the `read` syscall (0x3), as we can see from the EAX register. This function signature is as follows:
```c
ssize_t read(int fd, void *buf, size_t count);
```
As we know, first parameter is referred by EBX (0x5), second parameter by ECX, which is pointing to EDI that holds the file to be read (`/etc/passwd`), and third parameter by EDX which is the length, 4096 bytes (0x1000).

```nasm
0000001E  89C2              mov edx,eax
00000020  B804000000        mov eax,0x4
00000025  BB01000000        mov ebx,0x1
0000002A  CD80              int 0x80
```

This is the `write` syscall (0x4), which the signature is similar to the previous one:
```c
ssize_t write(int fd, const void *buf, size_t count);
```
It basically, writes the contents of the file to the screen as EBX has the value 0x1 which stands for `stdout`.

```nasm
0000002C  B801000000        mov eax,0x1
00000031  BB00000000        mov ebx,0x0
00000036  CD80              int 0x80
```

This code block is the `exit` syscall, to do it gracefully.


