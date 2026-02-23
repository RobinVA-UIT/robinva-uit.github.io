---
title: picoCTF 2019 - shark on wire 2
  # Tên bài viết sẽ hiện to đùng
date: 2026-02-23 12:20:00 +0700      # Thời gian đăng (Quan trọng: +0700 là giờ VN)
categories: [CTF, Forensics, picoCTF]         # Danh mục lớn, danh mục con
tags: [network forensics, wireshark, pcap, udp]     # Tag để tìm kiếm (viết thường)
author: "RobinVA"
---

# Write-up | picoCTF 2019 | shark on wire 2
## Description
>*We found this [packet capture](https://challenge-files.picoctf.net/c_fickle_tempest/f208f7c58d3703ed5fe0a78f707f011fcbb8c0b210b35247762c7ccacb49fe46/capture.pcap). Recover the flag.*

> Hint: (None)

---

**NOTE**: I recommend reading the write-up of the previous part of this challenge series [here](https://robinva-uit.github.io/posts/SharkOnWire1/).

---

## Initial steps

Let's open the provided `.pcap` file in Wireshark:

![initial](/assets/img/picoCTF/Forensics/shark_on_wire_2/initial.png)

Checking some of the first packets, I noticed a part of the flag was in these UDP packets, just like the [previous challenge](https://robinva-uit.github.io/posts/SharkOnWire1/):

![udp1](/assets/img/picoCTF/Forensics/shark_on_wire_2/udp1.png)

![udp2](/assets/img/picoCTF/Forensics/shark_on_wire_2/udp2.png)

![udp3](/assets/img/picoCTF/Forensics/shark_on_wire_2/udp3.png)

I randomly followed the UDP stream of one of these packets, but of course, it was a red herring:

![red](/assets/img/picoCTF/Forensics/shark_on_wire_2/red.png)

Next, I decided to analyse other UDP packets as well.

First, I found three packets sent from `10.0.0.2`, but to three different destination, that had random suspicious text:

> In the first packet, it had a nonsense plaintext:
![red2](/assets/img/picoCTF/Forensics/shark_on_wire_2/red2.png)


> In the other two packets, they only contained a letter "i". I suspected it was the letter "i" in "pico":
![red3](/assets/img/picoCTF/Forensics/shark_on_wire_2/red3.png)
![red4](/assets/img/picoCTF/Forensics/shark_on_wire_2/red4.png)

Following the first one:

![notaflag](/assets/img/picoCTF/Forensics/shark_on_wire_2/not_a_flag.png)

... the second one:

![picosth](/assets/img/picoCTF/Forensics/shark_on_wire_2/picosth.png)

... and the third one:

![fake_flag](/assets/img/picoCTF/Forensics/shark_on_wire_2/fake_flag.png)

They were all red herrings, so I continued exploring.

Then, I discovered No. 50 packet. It really looked like a flag, but with out underlines (_) and curly brackets ({}):

![looklike](/assets/img/picoCTF/Forensics/shark_on_wire_2/looklike.png)

Once again, I followed it:

![fakeagain](/assets/img/picoCTF/Forensics/shark_on_wire_2/fakeflagagain.png)

That is not what we want: another fake intel.

There were too much red herrings, so I only show the most outstanding ones.

Scrolling to nearly the bottom of the list, I saw No. 1104 packet had the text "start":

![start](/assets/img/picoCTF/Forensics/shark_on_wire_2/start.png)

I thought this was kind of a sign to let the receiver know that the data transmission began.

I also noticed that the source `10.0.0.66` sent data to the same destination `10.0.0.1`, but the source port changed frequently:

![hint](/assets/img/picoCTF/Forensics/shark_on_wire_2/hint.png)

I typed in the filter bar this filter:

`ip.src == 10.0.0.66`

...to narrow down the investigation to packets that were sent from `10.0.0.66`.

![100066](/assets/img/picoCTF/Forensics/shark_on_wire_2/100066.png)

Taking a closer look at the source port, you can see that the last 3 digits of each port could be some kinds of encoding. I believed this was ASCII code.

## Vulnerability analysis
### Potential vulnerabilities
* Network steganography via source port number.

## Solution paths

To extract each number and decode to human-readdable text, I wrote a Python script:

```python
#equals to "using namespace scapy.all"
from scapy.all import *

flag = ""

#rdpcap = read pcap
packets = rdpcap('capture.pcap')

#browse all packets
for packet in packets:
    #Each packet has different protocols
    #We just want to check if the packet contains UDP.
    if UDP in packet and packet[UDP].dport == 22:
        #Divide by 1000 to get the last three digits
        flag += chr(packet[UDP].sport % 1000)

#.format will replace the curly brackets {} by the data inside flag variable
print("Flag: {}".format(flag))
```

After running the script, I got the flag:

![flag](/assets/img/picoCTF/Forensics/shark_on_wire_2/flag.png)

## Flag
`picoCTF{p1LLf3r3d_data_v1a_st3g0}`


## Commands/Tools used

> | Commands/Tools | Purpose(s) |
> |----------------|------------|
> |  Wireshark    | A tool to analyse captured network packets (`.pcap` and `.pcapng` files).
> | Python | Autonomously extract information in `.pcap` and `.pcapng` thanks to `scapy` library.

## Key takeaways/Lessons learned

* **Network steganography**: Learn a new network steganography technique - Source port number.
* **Utilize Python**: Instead of mannually copy each port number to decode, we can use Python to automate the process via `scapy`.
