---
title: picoCTF 2022 - Operation Oni  # Tên bài viết sẽ hiện to đùng
date: 2026-02-17 15:11:00 +0700      # Thời gian đăng (Quan trọng: +0700 là giờ VN)
categories: [CTF, Forensics, picoCTF]         # Danh mục lớn, danh mục con
tags: [disk image, sleuthkit, ssh]     # Tag để tìm kiếm (viết thường)
author: "RobinVA"
---

# Write-up | picoCTF 2022 | Operation Oni
## Description
>*Download this disk image, find the key and log into the remote machine.
Note: if you are using the webshell, download and extract the disk image into /tmp not your home directory.
Additional details will be available after launching your challenge instance.*

> Hint: None

---

## Initial steps

We are provided a disk image file named `disk.img`.

I do `fdisk -l disk.img` (`-l` stands for "list) to see the partition list of the disk image.

![fdisk](/assets/img/picoCTF/Forensics/Operation_Oni/fdisk.png)

I extract these two partitions via `dd` (Data duplicator):

> First partition: `dd if=disk.img of=disk.img1 bs=512 skip=2048 count=204800`
>
> Second partition: `dd if=disk.img of=disk.img2 bs=512 skip=206848 count=264192`

* `if=` : Input file.
* `of=` : Output file.
* `bs=512` : Block size. In the picture above, we can see that each sector takes up 512 bytes.
* `skip=` : Skip to the start of the partition and start extracting from that point.
* `count=` : The number of sectors that need to be extracted.

Now we got two separate partitions to analyse. I execute `fls -r <img name>` (`-r` stands for "recursive) to see all files and folders inside each one.

In `disk.img1`, there's nothing special as usual:

![fls1](/assets/img/picoCTF/Forensics/Operation_Oni/fls1.png)

However, `disk.img2` contains the `root` directory, and I notice there's two interesting files in it:

![fls2](/assets/img/picoCTF/Forensics/Operation_Oni/fls2.png)

These are public and private keys of Ed25519 encryption, and seems like we need the private one to log in the remote machine.

**Bonus knowledge:** If you want to know whether a public key and a private key is from a same encryption session, do this:

1.  Use `ssh-keygen -l -f <keyfile>` for each key and export these to two separate `.txt` file.
2. Use `diff -u <private.txt> <public.txt>` (`-u` stands for (unified)) to confirm.

## Vulnerability analysis
### Potential vulnerabilities
* Encryption key exposure.

## Solution paths

We extract `id_ed25519` file from the second partition by executing:

`icat -i raw -f ext4 disk.img2 2345 > id_ed25519.txt`

* `-i raw` : Image type. `raw` makes `icat` copy bit by bit of the image.
* `-f ext4` : File system type. When using `fsstat`, we can confirm the type of this img is `ext4`:

![file_system_file](/assets/img/picoCTF/Forensics/Operation_Oni/file_system_file.png)

We do not use `-r` (recover) since this file was not deleted.

We connect to the remote machine by using the file we just got:

`ssh -i id_ed25519.txt -p <provided-port-number> ctf-player@saturn.picoctf.net`

After connected, use `ls` to check the existence of the flag and `cat` to read it:

![result](/assets/img/picoCTF/Forensics/Operation_Oni/flag.png)

## Flag
`picoCTF{k3y_5l3u7h_af277f77}`

**DISCLAIMER:** The code at the end of the flag may vary between versions, which means it is due to change. The flag provided in this writeup may not valid in the future.

## Commands/Tools used

> | Commands/Tools | Purpose(s) |
> |----------------|------------|
> |  `fdisk -l`    | Display the partition list of the disk image.
> | `dd`        | Copy a partition in a disk image and export it.
> | `fls -r` | Display all files and directories in a partition.
> | `icat` | Extract file from a partition.

## What did we learn?

* See all allocated space in disk image via `fdisk` and extract each partition out via `dd`.
* Connect to a remote machine without password via private key. 
