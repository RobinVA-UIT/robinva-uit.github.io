---
title: NNS CTF 2026 - Open Secret  # Tên bài viết sẽ hiện to đùng
date: 2026-09-07 20:53:00 +0700      # Thời gian đăng (Quan trọng: +0700 là giờ VN)
categories: [CTF, Reverse Engineering]         # Danh mục lớn, danh mục con
tags: [ctf, reverse engineering, ltrace, strace]     # Tag để tìm kiếm (viết thường)
author: "RobinVA"
---

# Open Secret

## Author: hoover

## Tags: beginner

## Artifact(s)

> [Here](https://github.com/RobinVA-UIT/robinva-uit.github.io/tree/main/_posts/CTF_tournament/NNS_CTF_2026/Reverse%20Engineering/Artifacts/Open%20Secret)

Sha256sum: d01d287a2bb96e7810ce8c682209d12bd8ffdcab6fafdcdba9acd175e2c8a2d5

## Description:

New to reverse engineering? This beginner challenge is an introduction to
tracing the system calls a Linux program makes.

The provided x86-64 ELF wants a license file before it will do anything,
but it will not tell you which one. The path is not written down in the
binary either and running strings on it only gets you no license. The
program builds the path in memory first, and only then asks the kernel to
open it.

This one talks to the kernel directly, so a library call tracer has
nothing to show you. Run the program under a system call tracer such as
strace instead, watch the call that opens the file, and read the path
out of its arguments. Your goal is to create the file it is looking for
and run the program again.

## Tools

- `strings`

- `strace`

- `ltrace`

## Walkthrough

I did some basic analysis on the executable:

```bash
❯ file open-secret
open-secret: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, BuildID[sha1]=00736964b5cf872c1b75f1b925843fb968d433fc, not stripped

❯ strings open-secret
no license
HOME=
GCC: (GNU) 16.2.1 20260810
main.c
path
flag
locked
home
full
__bss_start
_edata
_end
.symtab
.strtab
.shstrtab
.note.gnu.build-id
.text
.rodata
.eh_frame
.note.gnu.property
.data
.bss
.comment
```

The file is an ELF executable, and `strings` result does not return anything related to the flag.

Based on the challenge description, we need to monitor the system call (syscall) via a tracer like `strace`.

```bash
❯ strace "./open-secret"
execve("./open-secret", ["./open-secret"], 0x7ffd9ecee830 /* 76 vars */) = 0
openat(AT_FDCWD, "/home/admin/.config/nns/key", O_RDONLY) = 3
close(3)                                = 0
write(1, "NNS{7h3_p47h_w4s_h1dd3n_bu7_s7r4"..., 49NNS{7h3_p47h_w4s_h1dd3n_bu7_s7r4c3_s4w_7h3_0p3n}
) = 49
exit(0)                                 = ?
+++ exited with 0 +++
```

### Bonus

I did try the file with `ltrace`, and it also returns the flag. I suppose this is the author's fault.

```bash
❯ ltrace "./open-secret"
Couldn't find .dynsym or .dynstr in "/proc/25412/exe"
NNS{7h3_p47h_w4s_h1dd3n_bu7_s7r4c3_s4w_7h3_0p3n}
``

## Flag: NNS{7h3_p47h_w4s_h1dd3n_bu7_s7r4c3_s4w_7h3_0p3n}

