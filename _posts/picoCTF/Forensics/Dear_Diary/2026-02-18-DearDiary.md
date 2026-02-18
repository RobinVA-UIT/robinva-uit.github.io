---
title: picoCTF 2024 - Dear Diary  # Tên bài viết sẽ hiện to đùng
date: 2026-02-18 10:36:00 +0700      # Thời gian đăng (Quan trọng: +0700 là giờ VN)
categories: [CTF, Forensics, picoCTF]         # Danh mục lớn, danh mục con
tags: [disk image, sleuthkit, hxd]     # Tag để tìm kiếm (viết thường)
author: "RobinVA"
---

# Write-up | picoCTF 2024 | Dear Diary
## Description
>*If you can find the flag on this disk image, we can close the case for good!
Download the disk image [here](https://artifacts.picoctf.net/c_titan/63/disk.flag.img.gz).*

> Hint: *If you're observing binary data raw in the terminal you may be misled about the contents of a block.*

---

## Initial steps

### Approach 1

As usual, I did `fdisk -l` to see the partition table of the disk image:

![fdisk](/assets/img/picoCTF/Forensics/Dear_Diary/fdisk.png)

I extracted each partition thanks to `dd` like what I did in the [previous challenge](https://robinva-uit.github.io/posts/OperationOni/#:~:text=I%20extract%20these%20two%20partitions%20via%20dd%20(Data%20duplicator)%3A).

After that, I proceeded to analyze the swap space for sensitive strings or artifacts. 

Since computer can move memory pages from "inactive" processes to swap instead of RAM, this partition usually contains volatile data remnants. This includes plain-text passwords, snippets of opened documents, or fragments of executed commands that were previously stored in memory.

To do so, I executed `strings disk.flag.img2 | grep "pico"` and `strings disk.flag.img2 | grep "pico"`, respectively:

![strings_img2](/assets/img/picoCTF/Forensics/Dear_Diary/strings_img2.png)

It seemed like the plaintext was probably segmented, and there's a lot of red herrings, so it is kind of a waste of time to find the flag here.

Next, we move to the first partition. I used `fls -r disk.flag.img1` and saw nothing special:

![fls_1](/assets/img/picoCTF/Forensics/Dear_Diary/fls_1.png)

Let's come to the last one by using `fls -r disk.flag.img2`. After scrolling down to last lines, a lot of suspicious files appeared in the `root` directory:

![root_terminal](/assets/img/picoCTF/Forensics/Dear_Diary/root_terminal.png)

We need to extract all of these by using `icat` ([example](https://robinva-uit.github.io/posts/OperationOni/#:~:text=icat%20%2Di%20raw%20%2Df%20ext4%20disk.img2%202345%20%3E%20id_ed25519.txt)).

Checking `.ash_history`, seemed like the hacker run `force-wait.sh`:

![ash_history](/assets/img/picoCTF/Forensics/Dear_Diary/ash_history.png)

In `force-wait.sh`, there was only a scot-free command:

![force-wait](/assets/img/picoCTF/Forensics/Dear_Diary/force-wait.png)

Both `innocuous-file.txt` and `its-all-in-the-name` were empty:

![innocuous](/assets/img/picoCTF/Forensics/Dear_Diary/innocuous.png)

![its-all-in-the-name](/assets/img/picoCTF/Forensics/Dear_Diary/its-all-in-the-name.png)

### Approach 2

In this approach, I utilized Autopsy instead of running commands.

Click `Data Sources` -> `disk.flag.img_1 Host` -> `disk.flag.img` -> `vol4` (this is `vol3` in the previous approach, because `vol1` in this case is not allocated) -> `root` -> `secret-secrets`.

![autopsy](/assets/img/picoCTF/Forensics/Dear_Diary/autopsy.png)

You can also see the same files like Approach 1.

---

Based on the filename `its-all-in-the-name`, I thought I needed to do some kinds of search involving the name `innocuous-file`.

## Vulnerability analysis
### Potential vulnerabilities
* Data Remanence
* Insecure Data Disposal

## Solution paths

I used HxD, opened `disk_flag_img3` and searched the keyword "innocuous-file". Scanning through each result, I noticed the flag was scattered:

![flag1](/assets/img/picoCTF/Forensics/Dear_Diary/flag1.png)

![flag2](/assets/img/picoCTF/Forensics/Dear_Diary/flag2.png)

![flag3](/assets/img/picoCTF/Forensics/Dear_Diary/flag3.png)

By copying each segment, we will get the flag.

## Flag
`picoCTF{1_533_n4m35_80d24b30}`

**DISCLAIMER:** The code at the end of the flag may vary between versions, which means it is due to change. The flag provided in this writeup may not valid in the future.

## Commands/Tools used

> | Commands/Tools | Purpose(s) |
> |----------------|------------|
> |  `fdisk -l`    | Display the partition list of the disk image.
> | `dd`        | Copy a partition in a disk image and export it.
> | `fls -r` | Display all files and directories in a partition.
> | `icat` | Extract file from a partition.
> | Autopsy | Basically Sleuthkit but has GUI.
> | HxD | HEX code reader (and editor also).

## What did we learn?

* Understand why our disk still has data of files that we deleted and how we can utilize it.
* Valuable data can be scattered throughout HEX code.
