---
title: NNS CTF 2026 - No Strings Attached  # Tên bài viết sẽ hiện to đùng
date: 2026-09-07 08:13:00 +0700      # Thời gian đăng (Quan trọng: +0700 là giờ VN)
categories: [CTF, Tournaments, NNS CTF 2026, Reverse Engineering]         # Danh mục lớn, danh mục con
tags: [ctf, reverse engineering, ltrace]     # Tag để tìm kiếm (viết thường)
author: "RobinVA"
---

# No Strings Attached

## Author: hoover
## Tags: beginner

## Artifact(s)

> [Here](https://github.com/RobinVA-UIT/robinva-uit.github.io/tree/main/_posts/CTF_tournament/NNS_CTF_2026/Reverse%20Engineering/Artifacts/No%20Strings%20Attached)

> Sha256sum: `083cdaded1ad5c1b0818d89286f98bb9685f4d90654ded79fde5a9b2d8fb0c7d`

## Description:

New to reverse engineering? This beginner challenge is an introduction to
tracing the library calls a Linux program makes.

The provided x86-64 ELF asks you to guess a passphrase. The passphrase
is the flag, but it is not written down anywhere in the file. Running
strings on the binary only gets you guess:, correct and rejected.
The program builds the real passphrase in memory first, and only then
hands it to a function in the C standard library to compare against yours.

Run the program under a library call tracer such as ltrace and watch the
comparison. Your goal is to read the passphrase out of the arguments the
program passes to it.

---

## Tool(s)

- `strings`

- `ltrace`

## Walkthrough

We're given a ELF executable file:

```bash
󰡯 no-strings-attached
❯ file no-strings-attached
no-strings-attached: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=7d8306bb906fec01cafc9e3079d6ffbdb6e00e35, for GNU/Linux 4.4.0, not stripped
```

Using `strings` on the executable, it seems like the program would ask the user to enter something:

```bash
❯ file no-strings-attached
no-strings-attached: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=7d8306bb906fec01cafc9e3079d6ffbdb6e00e35, for GNU/Linux 4.4.0, not stripped
❯ strings no-strings-attached
5/lib64/ld-linux-x86-64.so.2
read
__libc_start_main
strcmp
write
libc.so.6
GLIBC_2.2.5
GLIBC_2.34
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
PTE1
guess: #prompt to enter something
correct
rejected
;*3$"
GCC: (GNU) 16.2.1 20260810
crt1.o
__abi_tag
crtbegin.o
deregister_tm_clones
__do_global_dtors_aux
completed.0
__do_global_dtors_aux_fini_array_entry
frame_dummy
__frame_dummy_init_array_entry
main.c
secret
prompt
accepted
rejected
input
crtend.o
__FRAME_END__
_DYNAMIC
__GNU_EH_FRAME_HDR
_GLOBAL_OFFSET_TABLE_
__libc_start_main@GLIBC_2.34
_ITM_deregisterTMCloneTable
write@GLIBC_2.2.5
_edata
_fini
read@GLIBC_2.2.5
__data_start
strcmp@GLIBC_2.2.5
__gmon_start__
__dso_handle
_IO_stdin_used
_end
_dl_relocate_static_pie
__bss_start
main
_dl_relocate_static_pie_ifunc
__TMC_END__
_ITM_registerTMCloneTable
_init
.symtab
.strtab
.shstrtab
.note.gnu.build-id
.interp
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rela.dyn
.rela.plt
.init
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.sframe
.note.gnu.property
.note.ABI-tag
.init_array
.fini_array
.dynamic
.got
.got.plt
.data
.bss
.comment
```

Let's run this once to see what would happen:

```bash
❯ ./no-strings-attached
guess: dsdad
rejected
```

Based on the challenge description, we should use `ltrace` or similar tools to track library calls. Remember to cover your file parameter with double quotes

```bash
❯ ltrace "./no-strings-attached"
write(1, "guess: ", 7guess: )                                                          = 7
read(0d
, "d\n", 127)                                                             = 2
strcmp("d", "NNS{n0_str1ngs_1n_7h3_b1n4ry_bu7"...)                              = 22
write(1, "rejected\n", 9rejected
)                                                       = 9
+++ exited (status 2) +++
```

The flag appeared. But we need to see in full length. I looked the guide to use the command up on Google, then a [writeup](https://ctftime.org/writeup/24852) of a similar challenge popped up. The author used "-s" parameter to specify return length.

```bash
❯ ltrace -s 150 "./no-strings-attached"
write(1, "guess: ", 7guess: )                                                          = 7
read(0dad
, "dad\n", 127)                                                           = 4
strcmp("dad", "NNS{n0_str1ngs_1n_7h3_b1n4ry_bu7_ltr4c3_s4w_7h3_c0mp4r3}")       = 22
write(1, "rejected\n", 9rejected
)                                                       = 9
+++ exited (status 2) +++
```

## Flag: `NNS{n0_str1ngs_1n_7h3_b1n4ry_bu7_ltr4c3_s4w_7h3_c0mp4r3}`
