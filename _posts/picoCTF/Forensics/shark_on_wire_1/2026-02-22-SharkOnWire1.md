---
title: picoCTF 2019 - shark on wire 1  # Tên bài viết sẽ hiện to đùng
date: 2026-02-22 22:30:00 +0700      # Thời gian đăng (Quan trọng: +0700 là giờ VN)
categories: [CTF, Forensics, picoCTF]         # Danh mục lớn, danh mục con
tags: [network forensics, wireshark, pcap, udp]     # Tag để tìm kiếm (viết thường)
author: "RobinVA"
---

# Write-up | picoCTF 2019 | shark on wire 1
## Description
>*We found this [packet capture](https://challenge-files.picoctf.net/c_fickle_tempest/134d2a2cf6ec5b7e757effc9b32977af7cc324b8e99a5ddb64737794a14dc18d/capture.pcap). Recover the flag.*

> Hint: *Try using a tool like Wireshark, What are streams?*

---

## Walkthrough

Let's open the provided file, which is `capture.pcap`, in Wireshark. Remember to set the packet presentation in chronological order:

![capture](/assets/img/picoCTF/Forensics/shark_on_wire_1/capture.pcap.png)

Surfing through some packets at the beginning, I started to notice that some UDP packets had plaintext like these:

![sus1](/assets/img/picoCTF/Forensics/shark_on_wire_1/sus1.png)

![sus2](/assets/img/picoCTF/Forensics/shark_on_wire_1/sus2.png)

So I decided to follow one UDP packet. 

>*But how to follow a packet?*

> Right-click the packet that you want to follow -> Click `Follow` -> Choose the stream that you want (in this case, click `UDP stream`).

Right-click the packet that you want to follow

![red](/assets/img/picoCTF/Forensics/shark_on_wire_1/red.png)

This was just a nonsense text. However, after I closed this window and check other UDP packets (type "udp" in the filter box), I noticed the flag was segmented in packets that had the text length of 1:

![seg1](/assets/img/picoCTF/Forensics/shark_on_wire_1/seg1.png)

![seg2](/assets/img/picoCTF/Forensics/shark_on_wire_1/seg2.png)

![seg3](/assets/img/picoCTF/Forensics/shark_on_wire_1/seg3.png)

![seg4](/assets/img/picoCTF/Forensics/shark_on_wire_1/seg4.png)

Combining these, we can get the word "pico". Also, if you take a look at "Malformed packet" labeled packets and follow them, it will show not only 1 "pico", but 4 "pico" words. It will not provide you more information other than this:

![red2](/assets/img/picoCTF/Forensics/shark_on_wire_1/red2.png)

The next step is to follow packets that has text length of 1. I did by writing this in the filter:

`udp.length == 9`

In UDP protocol, "Length" includes 8 Header bytes and X payload (data) bytes. Regarding our case, the text length is 1, so 8 + 1 = 9.

![length9](/assets/img/picoCTF/Forensics/shark_on_wire_1/length9.png)

Now we follow a random packet among these, and the flag will be revealed:

![flag](/assets/img/picoCTF/Forensics/shark_on_wire_1/flag.png)

## Flag
`picoCTF{StaT31355_636f6e6e}`

**DISCLAIMER:** The code at the end of the flag may vary between versions, which means it is due to change. The flag provided in this writeup may not valid in the future.

## Commands/Tools used

> | Commands/Tools | Purpose(s) |
> |----------------|------------|
> |  Wireshark    | A tool to analyse captured network packet (`.pcap` and `.pcapng` files)


## Key takeaways/Lessons learned

* **Protocol structure**: Understand the fact that UDP header accounts for 8 bytes, which helps us create custom filter (`udp.length == 9`).
* **Utilize Wireshark's filter**: This challenge demonstrates the importance of using filter effectively to reduce noise and isolate suspicious packets.
* **Data stream**: Know how to use `Follow` to collect discrete data in the transmission and connect them to form human-readdable data.
