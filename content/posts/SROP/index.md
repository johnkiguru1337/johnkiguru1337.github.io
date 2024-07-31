---
title: "Sigreturn Oriented Programming"
date: 2024-07-30
draft: false
summary: "Anything you want to know about SROP technique"
tags: ["SROP"]
---

## Definition

Sigreturn-oriented programming (SROP) is an exploit development technique used to execute code, this attack employs the same basic assumptions behind the return-oriented programming (ROP) technique. When a signal occurs, the kernel “pauses” the process’s execution in order to jump to a signal handler routine. In order to safely resume the execution after the handler, the context of that process is pushed/saved on the stack (registers, flags, instruction pointer, stack pointer etc). When the handler is finished, `sigreturn()` is called which will restore the context of the process by popping the values off of the stack. This is what is being exploited in this technique.

We will cover (probably not exhaustively):
 - the different ways that can be used to exploit a x64/x86 binary using the `SROP` method.
 - The different ways to set the eax register to `0xf`
 - Examples of custom `sigcontexts`
 - Locating a jump addresses using `gdb-peda`  

### The different ways to set the eax register to 0xf

#### > The trivial case: we have a `mov eax, 0xf` gagdet

the case where this gadget is present in the binary is the simplest to exploit, since it will allow us to place 0xf into the eax register in a single action

```bash
➜  srop ropper --file srop --search "mov eax, 0xf; ret"
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: mov eax, 0xf; ret

[INFO] File: srop
0x0000000000401134: mov eax, 0xf; ret;
```

With these two gadgets, building an exploit becomes very simple
Here is the structure of our exploit.

 - Padding until we reach the saved rip address of the `mov eax, 0xf ; ret` gadget ( `0x0000000000401134` ) address of the `syscall` ; ret gadget ( `0x0000000000401139` ) SigContext structure with the desired parameters

#### > Using the `pop eax; ret` gadget

This case is a “`variant`” of the previous one where it is still rather simple to put the value `0xf` in the `eax` register.

```bash
➜  srop ropper --file srop --search "pop"
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: pop

[INFO] File: srop
0x000000000040110d: pop eax; ret; 
```

Here is the structure of our exploit.

 - Padding until we reach the saved `rip` address of the `pop eax ; ret` gadget ( `0x0000000000401020` ) `0xf` (`sigreturn` `syscall` number) address of the `syscall` ; `ret` gadget ( `0x000000000040101b` ) `SigContext` structure with the desired parameters

#### > Use the read `syscall` to set the `eax` register to `0xf`

An interesting thing to know is that the read `syscall` records the number of bytes read into the eax register.There are two methods to set the value `0xf` in `eax` using the read `syscall`:

##### a) Using the `mov eax, 0x0` gadget

 - Padding until we reach the saved `rip` address of the `mov eax, 0x0; ret` gadget address of the `syscall ; ret` gadget. 
 - Then we send a 15 bytes (`0xf` -> 15 in decimal) string to the binary, which will allow us to place the value `0xf` in eax . And finally :
   - address of the `syscall`; ret gadget `SigContext` structure with the desired parameters
 
##### b) Using the `pop eax` gadget

 - Padding until we reach the saved `rip` address of the `pop eax; ret` gadget, send `0x0` (read `syscall` number) address of the `syscall`; retgadget
 - Then we send a 15 bytes string to the binary, which will allow us to place the value `0xf` in `eax` and finally :
address of the `syscall`; `ret` gadget `SigContext` structure with the desired parameters

### Examples of custom `sigcontexts`

Once you have figured out how to call the `sigreturn` `syscall`, you need to figure out how to get a shell through the context that will be restored from the stack.

#### a) If the binary contains the `/bin/sh` string

The idea is to call the `execve` function ( `syscall` `0x3b` -> 59 in decimal ) with the string `/bin/sh` as parameter which will give us a shell. The string `/bin/sh` can either be present in the binary or you can write it in a memory area whose you know the address.

```less
| — — — — — — — — —
| Register | value |
|— — — — — — — — — — — — — — — — — —
| rip | syscall instruction address |
| — — — — — — — — — — — — — — — — — —
| rax | 0x3b (execve syscall) |
|— — — — — — — — — — — — — — —
| rdi | address of /bin/sh |
|— — — — — — — — — — — — — -
| rsi | 0x0 (NULL) |
|— — — — — — — — — —
| rdx | 0x0 (NULL) |
|— — — — — — — — — —
```

#### b) Use `mprotect`

We use `mprotect` to make a memory area of our choice executable and writable to allow shellcode execution at that address. Then we shift the stack to that area so we can easily write data to it. We put in rsp the address containing the entry point of the program to ensure a normal controlflow. We can then arrange to redirect the program to the shellcode address, which will be executed despite the NX protection.

```less
|— — — — — — — — —
| Register | value |
| — — — — — — — — — — — — — — — -
| rax | 0xa (`mprotect` `syscall`) |
| — — — — — — — — — — — — — — —-
| rdi | shellcode address |
|— — — — — — — — — — — — — — — — —
| rsi | size (0x1000 for exemple) |
|— — — — — — — — — — — — — — — — —
| rdx | 0x7 -> mode (rwx) |
|— — — — — — — — — — — — — — —
| rsp | entrypoint (new stack) |
|— — — — — — — — — — — — — — — — — — — — — -
| rip | address of the `syscall`; ret gadget |
|— — — — — — — — — — — — — — — — — — — — —
```

### Refs

 - https://mutur4.github.io/posts/binary-exploitation/srop/
 - https://bananamafia.dev/post/srop/
 - https://ctf--wiki-org.translate.goog/pwn/linux/user-mode/stackoverflow/x86/advanced-rop/srop/?_x_tr_sl=auto&_x_tr_tl=en&_x_tr_hl=en-GB
 - https://trustie.medium.com/sick-rop-htb-pwn-challenge-9b1310d9a6b