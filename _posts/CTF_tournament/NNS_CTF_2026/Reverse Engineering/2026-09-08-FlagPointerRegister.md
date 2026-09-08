---
title: NNS CTF 2026 - Flag Pointer Register  # Tên bài viết sẽ hiện to đùng
date: 2026-09-08 22:19:00 +0700      # Thời gian đăng (Quan trọng: +0700 là giờ VN)
categories: [CTF, Tournaments, NNS CTF 2026, Reverse Engineering]         # Danh mục lớn, danh mục con
tags: [ctf, reverse engineering, x64dbg, xor]     # Tag để tìm kiếm (viết thường)
author: "RobinVA"
---

# Flag Pointer Register

## Author: hoover

## Tags: beginner

## Artifact(s)

> [Here](https://github.com/RobinVA-UIT/robinva-uit.github.io/tree/main/_posts/CTF_tournament/NNS_CTF_2026/Reverse%20Engineering/Artifacts/Flag%20Pointer%20Register)

Sha256sum: `1b269f57c6642d12b20b24a68d98b5c3612e2d97307efd9fd019ffab0ba05032`

## Description

New to reverse engineering? This beginner challenge is an introduction to
the Windows x64 calling convention and how arguments are passed in
registers.

The provided `x86-64` `.exe` decodes the flag successfully, yet it still
prints the wrong message. Somewhere between the decoder and WriteFile,
a perfectly good pointer ends up in the wrong place.

Open it in a debugger such as x64dbg or IDA Free and follow the final
function calls. Watch the value returned in `RAX`, then inspect how `RCX`,
`RDX`, `R8`, and `R9` are prepared before the output call.

Put the correct pointer into the register where `WriteFile` expects its
buffer. Repair it live in the debugger and step through the program again.


## Tool(s)

- x64dbg

## Walkthrough

Open the provided file in x64dbg, then press F9 to jump to the entry point. The main function would look like this:

![1](/assets/img/CTF_tournament/NNS CTF 2026/Flag Pointer Register/1.jpg)

### STD_INPUT and STD_OUTPUT

Here is the part that the program retrieves STD_INPUT and STD_OUTPUT handles for getting input and printing:

![2](/assets/img/CTF_tournament/NNS CTF 2026/Flag Pointer Register/2.jpg)

> [List of `nStdHandle` values](https://learn.microsoft.com/en-us/windows/console/getstdhandle#:~:text=Parameters-,nStdHandle,-%5Bin%5D%0AThe)

| Value | Corresponding HEX value |
|-------|-------------------------|
| STD_INPUT | `0xFFFFFFF6` |
| STD_OUTPUT | `0xFFFFFFF5` |

- [140003040] = STD_OUTPUT

- `RSI` = STD_INPUT

### Print the prompt

Next, a string is printed out on the terminal with these code:

![3](/assets/img/CTF_tournament/NNS CTF 2026/Flag Pointer Register/3.jpg)

The [syntax of `WriteFile` function (API)](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-writefile#:~:text=see%20WriteFileEx.-,Syntax,-C%2B%2B) is:

```cpp
BOOL WriteFile(
  [in]                HANDLE       hFile,
  [in]                LPCVOID      lpBuffer,
  [in]                DWORD        nNumberOfBytesToWrite,
  [out, optional]     LPDWORD      lpNumberOfBytesWritten,
  [in, out, optional] LPOVERLAPPED lpOverlapped
);
```

Based on the [x64 calling convention of Windows](https://learn.microsoft.com/en-us/cpp/build/x64-calling-convention?view=msvc-170#:~:text=Integer%20valued%20arguments,the%20register%20necessary.), the parameter-register pairs of the API are:

| Parameter | Register |
|-----------|----------|
| `hFile` | `RCX` |
| `lpBuffer` | `RDX` |
| `nNumberOfBytesToWrite` | `R8` |
| `lpNumberOfBytesWritten` | `R9` |

The code above indicates the value of each parameter:

---

- `hFile` -> `RCX` = [140003040] = STD_OUTPUT

- `lpBuffer` -> `RDX` = [140002010] = "Press ENTER to receive the flag.\r\n"

![4](/assets/img/CTF_tournament/NNS CTF 2026/Flag Pointer Register/4.jpg)

"0D0A" is "\r\n"

- `nNumberOfBytesToWrite` -> `R8` = `R8D` (last 32-bit of `R8`) = 0x22 = 34

Note that if you write to `R8D` or `R8B`, the other untouched part of `R8` is erased. This is applied to all registers, not only `R8`.

- `lpNumberOfBytesWritten` -> `R9` = `RDI` = [140003048]

This is a blank space.

---

`WriteFile` function call is passed to `RBX`, then it is executed, which prints out the string in `lpBuffer`.

### Read input

![5](/assets/img/CTF_tournament/NNS CTF 2026/Flag Pointer Register/5.jpg)

[`ReadFile`'s syntax](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-readfile#:~:text=see%20ReadFileEx.-,Syntax,-C%2B%2B):

```cpp
BOOL ReadFile(
  [in]                HANDLE       hFile,
  [out]               LPVOID       lpBuffer,
  [in]                DWORD        nNumberOfBytesToRead,
  [out, optional]     LPDWORD      lpNumberOfBytesRead,
  [in, out, optional] LPOVERLAPPED lpOverlapped
);
```

---

- `hFile` -> `RCX` = `RSI` = STD_INPUT

- `lpBuffer` -> `RDX` = [14000304C]

This is a blank space.

- `nNumberOfBytesToRead` -> `R8` = `R8D` = 8

- `lpNumberOfBytesRead` -> `R9` = `RDI` = [140003048]

This is a blank space.

---

It just waits for the user to press the ENTER button.

### `flag-pointer-register` function

Right next to the input reading part is the primary logic of the program:

![7](/assets/img/CTF_tournament/NNS CTF 2026/Flag Pointer Register/7.jpg)

Have a look at the function:

![8](/assets/img/CTF_tournament/NNS CTF 2026/Flag Pointer Register/8.jpg)

In order to explain the motif of this function better, I will analyze it a little bit nonlinearly. 

#### First part - The setup

- `xor eax, eax`

This command means `eax` = 0. Since the program modifies `eax`, the upper half of `rax` is set to 0.

=> `RAX` = 0

- `lea rcx, qword ptr ds:[140002000]` and `lea rdx, qword ptr ds:[140003000]`

While something such as `mov rcx, qword ptr ds[140002000]` will set the value of `rcx` to whatever value stored in the address of 0x140002000, the `lea` command just sets the first operator to the value inside the square brackets "[]".

=> `RCX` = 0x140002000, `RDX` = 0x140003000

- The `for` loop

This piece of code at the end is a solid proof to prove that this function is a `for` loop:

![9](/assets/img/CTF_tournament/NNS CTF 2026/Flag Pointer Register/9.jpg)

It increases `RAX` by 1, then compare it to 0x39, which is 57 in decimal. If these two are not equal, the `RIP` will return to 0x140001010

![10](/assets/img/CTF_tournament/NNS CTF 2026/Flag Pointer Register/10.jpg)

... Else, `RAX`'s value will become 0x140003000 and the function concludes.

At 0x140003000, an ordinary `jump` command resides.

![11](/assets/img/CTF_tournament/NNS CTF 2026/Flag Pointer Register/11.jpg)

From the information provided, we can basically rewrite this part in pseudocode:

```c
//RAX = i
for(int i = 0; i<57, ++i)
{
	//Logic...

	
}

return 0x140003000;
```

#### Second part - Where interesting things happen

- `mov r8d, eax`

Set the latter half of `R8` to the value of `EAX`, which is the `i` counter. Regarding the rule, the upper half of `R8` must be 0, too.

=> `R8` = i

- `and r8d, 7`

Perform an AND calculation on `r8d` and 7, then set the result to `r8d`. The program wants to ensure that `r8d` will always be in the range between 0 and 7 (integer).

- `movzx r8d, byte ptr ds:[r8+rcx]`

Move the value holded in [r8+rcx] to r8d, but it onlys take a byte (at the lower half).

Because `r8d` (`r8`)'s value is only at between 0 and 7, the range to pay attention is between `[RCX]` (0x140002000) and `[RCX + 7]` (0x140002007).

I did highlight (in bold) that part in the dump section:

![12](/assets/img/CTF_tournament/NNS CTF 2026/Flag Pointer Register/12.jpg)

- `xor byte ptr ds:[rax+rdx], r8b`

After that, `r8b` (which is the value we get from `movzx`) is XORed to the lower-half-byte value in [rax+rdx] (a.k.a [i + 0x140003000]). For this reason, our focus is on the range of [0x140003000:0x140003038]:

![13](/assets/img/CTF_tournament/NNS CTF 2026/Flag Pointer Register/13.jpg) 

The result will be saved right in [rax+rdx]. I assume when we finishes the loop, the flag will completely manifest from 0x140003000.


## Solutions and code

### Solution 1

Extract the XOR key from address 0x140002000 and 0x140003000, then write a program to perform XOR. You can choose whatever language you wish.

```cpp
#include <iostream>
#include <string>

std::string XOR(std::string& k1, std::string& k2) {
    std::string res = "";

    for (size_t i = 0; i < 57; ++i) {
        std::string xor1 = k1.substr(i * 2, 2);
        std::string xor2 = k2.substr((i * 2) & 14, 2);

        res += static_cast<char>(std::stoi(xor1, nullptr, 16) ^
                                 std::stoi(xor2, nullptr, 16));
    }

    return res;
}

int main() {
    std::string key1 =
        "7F3CFA379122FD045946CD13D47EB604571E9D2BBC74F06C6E00CD34BC66B56A5F459A"
        "28BC21B504061A9A139464B535562DCB398570B6294C";

    std::string key2 = "3172A94CE316855B";

    std::cout << XOR(key1, key2);

    return 0;
}

```

Result:

```bash
❯ ./program_rewrite
NNS{r4x_h4d_7h3_fl4g_bu7_rdx_p01n73d_70_7h3_wr0ng_buff3r}⏎    
```

### Solution 2

Place a break point at 0x140001020, where `i` counter is added. Choose that line and press F2. If the address is marked in red, then it's successful.

![14](/assets/img/CTF_tournament/NNS CTF 2026/Flag Pointer Register/14.jpg)

Click on the Dump section. Then, Ctrl + G, enter 0x140003000 and press OK to jump there. The XOR key in that address will gradually changed to the flag when a XOR is performed.

![15](/assets/img/CTF_tournament/NNS CTF 2026/Flag Pointer Register/15.jpg)

Press F9 to jump to the entry point, and press F9 one more time to let the prompt appear on the terminal. Press ENTER or whatever you want.

![16](/assets/img/CTF_tournament/NNS CTF 2026/Flag Pointer Register/16.jpg)

Spam F9 until `RAX` reaches 0x38

![17](/assets/img/CTF_tournament/NNS CTF 2026/Flag Pointer Register/17.jpg)

Result:

![18](/assets/img/CTF_tournament/NNS CTF 2026/Flag Pointer Register/18.jpg)

## Flag: `NNS{r4x_h4d_7h3_fl4g_bu7_rdx_p01n73d_70_7h3_wr0ng_buff3r}`
