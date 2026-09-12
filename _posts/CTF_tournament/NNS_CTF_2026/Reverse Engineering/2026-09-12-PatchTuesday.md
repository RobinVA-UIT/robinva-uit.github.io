---
title: NNS CTF 2026 - Patch Tuesday  # Tên bài viết sẽ hiện to đùng
date: 2026-09-12 23:14:00 +0700      # Thời gian đăng (Quan trọng: +0700 là giờ VN)
categories: [CTF, Tournaments, NNS CTF 2026, Reverse Engineering]         # Danh mục lớn, danh mục con
tags: [ctf, reverse engineering, x64dbg, xor]     # Tag để tìm kiếm (viết thường)
author: "RobinVA"
---

# Patch Tuesday

## Author: hoover

## Tags: beginner

## Artifact(s)

> [Link](https://github.com/RobinVA-UIT/robinva-uit.github.io/tree/main/_posts/CTF_tournament/NNS_CTF_2026/Reverse%20Engineering/Artifacts/Patch%20Tuesday)

Sha256sum: `a84d21cbc972ba8b125d4954d66e871e219a832bc5dc30724d299e58d10cdeaa`

## Description

New to reverse engineering? This beginner challenge is an introduction to
dynamic debugging and patching Windows programs.

The provided `x86-64` `.exe` promises a free flag, but it does not behave the
way we want. Open it in a debugger such as x64dbg or IDA Free, run it, and
step through the code that decides whether you receive the flag. Watch how
the program compares values and follows a conditional jump.

Try changing that jump while debugging, or patch the instruction in the
executable and run your modified file again. Your goal is to make the
program follow the path that produces the expected behavior.

## Tool(s)

- x64dbg

## Walkthrough

Looking at the main function, it is not hard to see:

- The `GetStdHandle` code to retrieve output and input handles (1st block);

- The 2nd block where it prints out something with `WriteFile` (actually, it is the string `"Press ENTER to get a free NNS{ flag: "`), and handles user's input with `ReadFile`

![1](</assets/img/CTF_tournament/NNS CTF 2026/Patch Tuesday/1.jpg>)

A strange call happens in the 3rd block. The value of `lpNumberOfBytesRead` in the previous `ReadFile` call is passed to `RCX` before executing a function at `0x140001160`

![2](</assets/img/CTF_tournament/NNS CTF 2026/Patch Tuesday/2.jpg>)

Inside the function, it compares `lpNumberOfBytesRead` to 0x1337 (4919 in Base 10). Of course, this will always be false since `nNumberOfBytesToRead` is hard-coded to 0x10 (16) before the `ReadFile` call, so `lpNumberOfBytesRead` cannot exceed this value.

![3](</assets/img/CTF_tournament/NNS CTF 2026/Patch Tuesday/3.jpg>)

![4](</assets/img/CTF_tournament/NNS CTF 2026/Patch Tuesday/4.jpg>)

Not sure the intention of the author here, though...

After that is the 4th block, where `eax` (or the whole `rax`) is set to 0 via a `test` command. This will definitely raise the Zero Flag (ZF) to 1, therefore triggers the `je` command right below, eventually moving us to `0x140001127`:

![5](</assets/img/CTF_tournament/NNS CTF 2026/Patch Tuesday/5.jpg>)

As expected, no flag will pop out at that address. A string indicating no flag and a call to exit process is where the address leads to.

![6](</assets/img/CTF_tournament/NNS CTF 2026/Patch Tuesday/6.jpg>)

However, what if ZF was 0 at the moment that `je` command run? If it was the case, we would come to 0x14000108D:

![7](</assets/img/CTF_tournament/NNS CTF 2026/Patch Tuesday/7.jpg>)

At that address, a XOR loop appears, posing a high chance of finding the flag.

![8](</assets/img/CTF_tournament/NNS CTF 2026/Patch Tuesday/8.jpg>)

## Solution

Since the original code will not point us to the flag when running it, we need to patch something.

The best method here is replacing the `je` command with NOP, so we can jump to the XOR logic:

![9](</assets/img/CTF_tournament/NNS CTF 2026/Patch Tuesday/9.jpg>)

Place a breakpoint at the end of the loop:

![10](</assets/img/CTF_tournament/NNS CTF 2026/Patch Tuesday/10.jpg>)

Watch dump at `0x140003000`. The XOR key is there:

![11](</assets/img/CTF_tournament/NNS CTF 2026/Patch Tuesday/11.jpg>)

Run the process, and the flag would appear if done correctly:

![12](</assets/img/CTF_tournament/NNS CTF 2026/Patch Tuesday/12.jpg>)

## Flag: `NNS{1_h0p3_you_p47ch3d_7h3_0pc0d3_dur1ng_run71m3_jnz_1s_much_b3773r_7h4n_jz}`
