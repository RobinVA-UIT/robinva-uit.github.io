---
title: ArgvQuote - When passing argument is more complicated than you may assume  # Tên bài viết sẽ hiện to đùng
date: 2026-07-22 21:47:00 +0700      # Thời gian đăng (Quan trọng: +0700 là giờ VN)
categories: [Technical blogs, Win32 API]         # Danh mục lớn, danh mục con
tags: [blog, win32, cpp, c]     # Tag để tìm kiếm (viết thường)
author: "RobinVA"
---


# ArgvQuote - When passing argument is more complicated than you may assume

When I researched the ways to exploit `CreateProcess` API, I encountered a post that posed a problem of arguments convention in Windows Programming. That post was difficult to 
read as a newbie in this field, but later, I found it very intriguing. I decided to give it a try by writing the whole `ArgvQuote`- a func that resolves the problem - from scratch. 

Now I will dig in how every thing work behind the func.

---

## The problem

If you want to call a function, let's say you want to read a file name `poet.txt` in `notepad.exe`, you have to specify the (path) file name in the `lpCommandLine` variable 
in `CreateProcess` as well as the name of the module you want to run. In this case, it will be `notepad.exe poet.txt`.

* NOTE: You can left `lpCommandLine` empty if you already stated `lpApplicationName`. Nevertheless, we can just assume that we have to pass value to `lpCommandLine`.

Sounds simple, right? Now, if you want to pass in an argument that has whitespace inside it, the instinct will probably tell you that you need the a couple of quotation marks to 
wrap the argument. Something like this: `notepad.exe "the poet.txt"`. At this point, Windows, or specifically `CommandLineToArgvW` API, is not confused yet. It still considers `the poet.txt` as a single argument.

What if the argument itself contains quotation marks? The case here is `"funny" poet.txt`. You may think we can address it with `notepad.exe ""funny" poet.txt"`. However, 
Windows is not as smart as you think. In other words, Windows does not work as we - human beings - think.

From the argument string ""funny" poet.txt", Windows API will process it like this:

1. `"`

The `"` is stripped, indicating that everything behind this mark is a single argument until another quote mark appears.

2. `"`

Now we meet another quote mark, so it is also stripped away. Right behind this mark does not have any whitespace, so the current argument has not ended yet.

3. `funny"`

`funny` itself is a normal string, we add it to the temp result: `funny`. Behind is a quote mark, and Windows API will strip it also. Just like what I mentioned, until another quote mark, any char behind is included in the current arg.

4. ` poet.txt"`

We can see the end of this part is the quote mark, so ` poet.txt` is appended.

> The final result is `funny poet.txt`.

So how can we manipulate Windows API not mistaking our quotation mark as starting/ending indicator?

In terms of Windows Programming, if you only write `"` without the escaping character '\', it is considered the quotation mark for beginning and ending of the argument, not the quotation mark serves 
as a part of the file name itself.

* **NOTE**: The escaping char here is specifically for how the Windows API (via CommandLineToArgvW) parses the command line string, which is completely separate from how the C/C++ compiler resolves escape sequences in source code. In Windows API, backslash `\` can only escape quotation mark, not `\n`, `\t` or `\v`.

This issue goes further when you pass in the path to the file. Suppose you want Notepad to read the `.txt` file with the path `C:\Users\"Secret" Workspace of Mine\"funny" poet.txt`.
 Here are the arguments:

1. First arg

-  `C:\Users\"Secret"`

The first quote mark is escaped, so it is passed into the result. The last one is not, so it is stripped. Temp result: `C:\Users\"Secret`

- ` Workspace of Mine\"funny" `

The escaping char appears right before the first quote mark, therefore  twit is included. Pay attention to the last mark: not escaped, and a whitespace stands behind. This mark the end of the first arg.

Temp result: `"C:\Users\"Secret WorkSpace of Mine"Funny`

3. Second arg

- `poet.txt`

No quotation mark wrapper, so the rest is our second arg.

Final result: `"C:\Users\"Secret WorkSpace of Mine"Funny poet.txt`.

What started as a single file path is now split into two completely corrupted arguments. This is a solid proof proving that arguments handling in Windows API is extremely miserable, especially if you are not a meticulous person - and why a function like `ArgvQuote` is important.

---

## Solution

To deal with the problem, we need a function that formats the argument string into a Windows-accepted string. 

### Parameter

1. `const std::string& Argument` and `std::string& CommandLine`: Of course we need these 'cause we are pushing the argument string to the Command Line.

2. `bool Force`: Force the func to add quote marks wrapper

You would like to `Force` if:

- Your argument string is empty `""`: Windows automatically skips this argument, which means it is not passed to the CommandLine. Sometimes you do not want to pass any arg, right?

- You simply want to ensure everything is wrapped by quotation marks. Mayber you want to escape bad cases as much as possible, who knows how crazy an argument string can be?


### Prerequisite

The module name (path) and the arg string must be separate by a whitespace. We need to push back a whitespace in `CommandLine` first.

What to do: Append the whitespace before doing anything.

```cpp
CommandLine.push_back(' ');
```

### First case

Suppose the Arg string does not contain whitespace, special chars [\t, \n, \v] (in case you want to pass arg string in source code, not from Console), and not forced.

What to do: Append the whole string.

```cpp
if (!Force && !Argument.empty() &&
        Argument.find_first_of(" \t\n\v\"") == size_t(-1)) {
        CommandLine.append(Argument);
    }
```

### Second case

This is the case when the arg string does not satisfy all conditions from the first case. Regardless of what case it is, we still wrap the arg string with double quotation marks anyway.

```cpp
CommandLine.push_back('"')
```

Now we need to iterate the whole arg string. The goal here is to count how many consecutive backslashes before reaching the end of the string or meeting any other chars.

```cpp
        int cnt = Argument.size();

        for (size_t i = 0; i < cnt; ++i) {
            int backslashesCount = 0;  // Count consecutive backslashes

            while (i != cnt && Argument[i] == '\\') {
                ++backslashesCount;
                ++i;
            }

        //=====Sub-cases=====

        } //Concluding step
```

#### Sub-case 1: Reach the end

It means the end of the string is either nothing or backslash(es).

In this case, we need to wrap quote marks, right? The neat part here is that if you pass the same number of backslashes to CommandLine just like the Arg string, it will escape the closing quote mark. As the result, your arg is messed up.

What to do: Escape all backslashes, making it the part of the arg.

```cpp
if (i == cnt) {
    CommandLine.append(backslashesCount * 2, '\\'); //Pass double the amount so that two backslashes can escape into one real backslash
}
```

#### Sub-case 2: The next char is a genuine quote mark

We need to escape this quote mark because it is a part of the arg string.

Furthermore, we also need the backslashes to be escaped also. Imagine if the arg string contains something like this:

`C:\Users\"Real.txt"`

The `\"` in the middle will become a genuine `"`, causing the arg to be `C:\Users"Real.txt`.

What to do: Escape all backslashes and the quote mark.

```cpp
else if (Argument[i] == '\"') {
    CommandLine.append(backslashesCount * 2 + 1, '\\');
    CommandLine.push_back('\"');
}
```

#### Sub-case 3

Now the backslash does not intervene any quote marks. We pass the same amount of backslash(es) to the CommandLine.

```cpp
else {
    CommandLine.append(backslashesCount, '\\')
    CommandLine.push_back(Argument[i]);
}
```

#### Concluding step

We add the last quote mark to wrap the arg.

```cpp
CommandLine.push_back('"');
```

---

## Code

```cpp
#include <iostream>
#include <string>

void ArgvQuote(const std::string& Argument, std::string& CommandLine,
               bool Force) {
    CommandLine.push_back(' ');
    // If (not Force) and (Argument is not Empty) and (Argument does not contain
    // space, tab, newline, quotationmark): Pass the Argument to the CommandLine
    // directly
    //
    if (!Force && !Argument.empty() &&
        Argument.find_first_of(" \t\n\v\"") == size_t(-1)) {
        CommandLine.append(Argument);
    }

    else {
        CommandLine.push_back('"');  // Push back the first quotation mark

        int cnt = Argument.size();

        for (size_t i = 0; i < cnt; ++i) {
            int backslashesCount = 0;  // Count consecutive backslashes

            while (i != cnt && Argument[i] == '\\') {
                ++backslashesCount;
                ++i;
            }

            // i == cnt means the last char of Argument is backslash
            // Solution: Append double the amount of backslash in Argument
            if (i == cnt) {
                CommandLine.append(backslashesCount * 2, '\\');
            }

            // If Arg[i] == '\"' ==> Need to escape the backslashes as well as
            // the quotation mark right behind the backslashes
            //  The number of bonus backslashes = The num of orginal backslashes +
            //  1 (for the mark)
            else if (Argument[i] == '\"') {
                CommandLine.append(backslashesCount * 2 + 1, '\\');
                CommandLine.push_back('\"');
            }

            // If not satisfy anything above, it means the backslashes are at the
            // middle of the Argument Add the same number of backslashes
            // NOTE: Do not misunderstand with how C/C++ resolves backslashes with a char
            // at next position. This is .exe file, C/C++ compiler has no work
            // here.
            else {
                CommandLine.append(backslashesCount, '\\');
                CommandLine.push_back(Argument[i]);
            }
        }

        CommandLine.push_back('"');
    }
}
```
