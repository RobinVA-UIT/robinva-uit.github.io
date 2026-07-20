---
title: Win32API - CreateProcessA  # Tên bài viết sẽ hiện to đùng
date: 2026-07-20 22:18:00 +0700      # Thời gian đăng (Quan trọng: +0700 là giờ VN)
categories: [Technical blogs, Win32 API]         # Danh mục lớn, danh mục con
tags: [blog, win32, cs, cpp, c, pinvoke]     # Tag để tìm kiếm (viết thường)
author: "RobinVA"
---

# Win32 API - CreateProcessA

This is an analysis of `CreateProcessA`, one of the most used Win32 APIs. Most of the information below is from the [Microsoft Docs](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessa) of this API.

In this blog, I will code in C#, though C's version will come soon.

> "Creates a new process and its primary thread. The new process runs in the security context of the calling process."
> -- [Microsoft](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessa#:~:text=Creates%20a%20new%20process%20and%20its%20primary%20thread.%20The%20new%20process%20runs%20in%20the%20security%20context%20of%20the%20calling%20process.) --

There are safer versions of this API, such as `CreateProcessAsUserA` or `CreateProcessWithLogonW` functions.

---

## Syntax in C++

```cpp
BOOL CreateProcessA(
  [in, optional]      LPCSTR                lpApplicationName,
  [in, out, optional] LPSTR                 lpCommandLine,
  [in, optional]      LPSECURITY_ATTRIBUTES lpProcessAttributes,
  [in, optional]      LPSECURITY_ATTRIBUTES lpThreadAttributes,
  [in]                BOOL                  bInheritHandles,
  [in]                DWORD                 dwCreationFlags,
  [in, optional]      LPVOID                lpEnvironment,
  [in, optional]      LPCSTR                lpCurrentDirectory,
  [in]                LPSTARTUPINFOA        lpStartupInfo,
  [out]               LPPROCESS_INFORMATION lpProcessInformation
);
```

## Syntax in C#

In C#, P/Invoke technique is applied, and you have to declare all the structs.

This is the sample C# code. The functionality of this program is to call `notepad.exe`:

```cs
using System;
using System.Runtime.InteropServices;

class Win32
{
    //Import Kernel32.dll via P/Invoke
    [DllImport("kernel32.dll", CharSet = CharSet.Ansi, SetLastError = true)]
    public static extern bool CreateProcessA(
        string lpApplicationNName,
        string lpCommandLine,
        ref SECURITY_ATTRIBUTES lpProcessAttributes,
        ref SECURITY_ATTRIBUTES lpThreadAttributes,
        bool bInheritHandles,
        uint dwCreationFlags,
        IntPtr lpEnvironment,
        string lpCurrentDirectory,
        ref STARTUPINFO lpStartupInfo,

        out PROCESS_INFORMATION lpProcessInformation
    );

    [DllImport("kernel32.dll")]
    public static extern bool CloseHandle(
        IntPtr hObject //Handle
    );

}

[StructLayout(LayoutKind.Sequential, CharSet = CharSet.Ansi)]
public struct STARTUPINFO
{
    public uint cb; //size of the struct, in bytes
    public string lpReserved; //Reversed; must be NULL
    public string lpDesktop; //The name of the desktop (or both desktop and window station for this process)
    public string lpTitle; //Title name on the title bar of the console
    public uint dwX; // X offset of the upperleft corner    
    public uint dwY; //Y offset of the upperleft corner
    public uint dwXSize; //The width of the window
    public uint dwYSize; //The height of the window

    //===IF A NEW CONSOLE WINDOW IS CREATED IN A CONSOLE PROCESS=== 
    public uint dwXCountChars; //Screen buffer width in char columns (if dwFlags specifies STARTF_USECOUNTCHARS)
    public uint dwYCountChar; //Screen buffer height in char columns (if dwFlags specifies STARTF_USECOUNTCHARS)
    public uint dwFillAttribute; //Initial text and background colors (if dwFlags specifies STARTF_USEFILLATTRIBUTE)
                                 //<BACKGROUND/FOREGROUND>_<RED/GREEN/BLUE> 
                                 //===END===

    public uint dwFlags; //Determines whether certain STARTUPINFO members are used when creates a window
    public ushort wShowWindow; //Can be any of the values in nCmdShow parameter for ShowWindow func (STARTF_USESHOWWINDOW)
    public ushort cbReserved2; //Reserved for use by C Run-time (CRT); must be zero
    public IntPtr lpReserved2; //Reserved for use by CRT; must be NULL

    public IntPtr hStdInput;//Std input handle for process (STARTF_USESTDHANDLES), else, default is keyboard buffer
                            //Specifies a hotkey value (STARTF_USEHOTKEY) (not sure what this is)
    public IntPtr hStdOutput;//Std output handle for process (STARTF_USESTDHANDLES), else, def is console screen
    public IntPtr hStdError; //Std error handle (STARTF_USESTDHANDLES), else, console windows's buffer
}

[StructLayout(LayoutKind.Sequential)]
public struct PROCESS_INFORMATION
{
    public IntPtr hProcess; //Handle to the newly created process
    public IntPtr hThread; //Handle to the primary thread of the newly created process
    public uint dwProcessId; //PID
    public uint dwThreadId; //TID
}

[StructLayout(LayoutKind.Sequential)]
public struct SECURITY_ATTRIBUTES
{
    public uint nLength; //The size, in bytes, of this struct
    public IntPtr lpSecurityDescriptor; //ptr to a SECURITY_DESCRIPTOR struct that controls access to the obj
                                        //If NULL, the obj is assigned the def sec desc associated with the access token of
                                        //the calling process.
    public bool bInheritHandle; //If the new process inherits the handle when the new process is created
}

class Program
{
    static void Main()
    {
        STARTUPINFO si = new STARTUPINFO();

        si.cb = (uint)Marshal.SizeOf(si);

        PROCESS_INFORMATION pi = new PROCESS_INFORMATION();

        SECURITY_ATTRIBUTES pa = new SECURITY_ATTRIBUTES();
        pa.nLength = (uint)Marshal.SizeOf(pa);
        SECURITY_ATTRIBUTES ta = new SECURITY_ATTRIBUTES();
        ta.nLength = (uint)Marshal.SizeOf(ta);

        //Flag
        uint new_console_flag = 0x00000010; //You can change to whatever value you want, as long as it's in the offical flag list



        string targetApp = @"C:\Windows\System32\notepad.exe";

        bool success = Win32.CreateProcessA(
            targetApp,
            null,
            ref pa,
            ref ta,
            false,
            new_console_flag,
            IntPtr.Zero,
            null,
            ref si,
            out pi
        );


        if (success)
        {
            Console.WriteLine($"[+] Successful. PID: {pi.dwProcessId}. Process handle (IntPtr) = {pi.hProcess}");
            Win32.CloseHandle(pi.hProcess);
            Win32.CloseHandle(pi.hThread);
        }
        else
        {
            Console.WriteLine($"[+] Unsuccessful.");
        }
    }
}

```

---

### LPCSTR lpApplicationName

* LPCSTR: Long pointer to constant string of chars

This is the name of the module that you wish to run

It can specify the full path of the module or the partial name. If you decide to use partial name, remember to place the executable file that includes this function to the same place as the module, so Auto Specification can play its role.

In C#, to specify a path, you have to add '@' before the path, which is between the quotation marks. This is to prevent errors caused by misidentification between "\" in terms of path and escape character.

One more thing to be cautious: `CreateProcessA` is the "A"(ANSI) version, but `string` in C# is UTF-16, so if your path contains characters from Chinese, Japanese or Vietnamese, the function will not work as expected. In this case, the "W" version (`CreateProcessW`) is the better choice over this version. 

* Solution: Load the parameter `CharSet = CharSet.Ansi` when using DllImport. .NET will marshal UTF-16 to ANSI before loading to Win32 API.

`lpApplicationName` can be left as "NULL". If this is the case, Windows will take the first part that ends with white space in the path assigned to `lpCommandLine` as the module to run. For that reason, you cannot left `lpCommandLine` empty or "NULL" if `lpApplicationName` is NULL also.[*]

**e.g:** `@"C:\Program Files\My App\test.exe"` 

 Windows will take out the part `@"C:\Program"` first, then add ".exe" if necessary (in this case, it is added), then run. If `C:\Program.exe` does not exist, it will extend to `C:\Program Files\My.exe` and further.

If the programmer demands to run Bash file via this method, [set lpApplicationName to cmd.exe and set lpCommandLine to the following arguments: /c plus the name of the batch file.](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessa#:~:text=set%20lpApplicationName%20to%20cmd.exe%20and%20set%20lpCommandLine%20to%20the%20following%20arguments%3A%20/c%20plus%20the%20name%20of%20the%20batch%20file.)
* e.g: "/c C:\path\to\your_script.bat" 

#### Bonus: Exploit

In [*], I mention the case when `lpApplicationName` is NULL-assigned and how Windows deals with it. However, this mechanic also poses a critical vulnerability.

Exploiter can utilize this by creating a malicious executable and naming it something like `Program.exe`. If the programmer is not careful with the mechanic, this shellcode can be executed easily.

To address the problem, we must cover the path by adding a pair of quotation marks, e.g: `@"\"C:\Program Files\My App\test.exe\" arg1 arg2";`, with `arg1`and `arg2` is the passed arguments to the module.


### LPSTR lpCommandLine

* LPSTR: Long pointer to string of chars

["The command line to be executed."](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessa#:~:text=The%20command%20line%20to%20be%20executed.)

`lpCommandLine`'s maximum length is 32767 chars (including "\0"). When `lpApplicationName` equals NULL, this variable is limited to **MAX_PATH** characters.

> Why **MAX_PATH**?

**MAX_PATH** equals 260. This is because in a path:

* "C:\": 3 chars;
* Path to directories and the executable: 256 chars (maximum);
* "\0": 1 char.

CreateProcessW - The Unicode version of this API, can modify this variable, so it cannot be a pointer to read-only memory, such as constant variable or a literal string.

Just like `lpApplicationName`, `lpCommandLine` can be NULL, and if it is NULL, the function considers `lpApplicationName` as the command line. Therefore, both cannot be NULL at the same time.

If both of them are non-NULL, `lpApplicationName` acts as the module name to be executed, and `lpCommandline` specifies the command line (both are null-terminated).

In C, argv[0] is the module name, so passing only the file to be opened in the module is not valid. Instead, do this:

```c
CreateProcess(
    L"C:\\Windows\\System32\\notepad.exe",   
    L"notepad.exe C:\\Data\\document.txt",   
);
``` 

### LPSECURITY_ATTRIBUTES lpProcessAttributes

>["The SECURITY_ATTRIBUTES structure contains the security descriptor for an object and specifies whether the handle retrieved by specifying this structure is inheritable. This structure provides security settings for objects created by various functions, such as CreateFile, CreatePipe, CreateProcess, RegCreateKeyEx, or RegSaveKeyEx."](https://learn.microsoft.com/en-us/windows/win32/api/wtypesbase/ns-wtypesbase-security_attributes#:~:text=The%20SECURITY_ATTRIBUTES%20structure,or%20RegSaveKeyEx.)

`lpProcessAttributes` decides if child processes can inherit the returned handle to the new process object.

```cpp
typedef struct _SECURITY_ATTRIBUTES {
  DWORD  nLength;
  LPVOID lpSecurityDescriptor; //Who can access?
  BOOL   bInheritHandle; //Is inheritance enabled
} SECURITY_ATTRIBUTES, *PSECURITY_ATTRIBUTES, *LPSECURITY_ATTRIBUTES;
```

If `lpProcessAttributes` == NULL, the handle cannot be inherited. If `lpProcessAttributes` == NULL or `lpSecurityDescriptor` (a member inside `SECURITY_DESCRIPTOR`) == NULL, default security descriptor is assigned.


### LPSECURITY_ATTRIBUTES lpThreadAttributes

Functionality is basically the same as `lpProcessAttributes`, but the use is for threads. Remember, process's handle and threads' handle are independent of each other.


### BOOL bInheritHandles

A boolean member to determine if the new process can inherit inheritable handles from the calling process.


### DWORD dwCreationFlags

> ["The flags that control the priority class and the creation of the process. "](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessa#:~:text=The%20flags%20that%20control%20the%20priority%20class%20and%20the%20creation%20of%20the%20process.)

* See the available Process Creation Flag values [here](https://learn.microsoft.com/en-us/windows/desktop/ProcThread/process-creation-flags).

* See the list of Priority Class values [here](https://learn.microsoft.com/en-us/windows/desktop/api/processthreadsapi/nf-processthreadsapi-getpriorityclass)

Child process' priority class is also controlled by this member (of the calling process).

- If no priority class flags is specified, the assigned default value is **NORMAL_PRIORITY_CLASS**;

- If the calling process's `dwCreationFlags` is either **IDLE_PRIORITY_CLASS** or **BELOW_NORMAL_PRIORITY_CLASS**, the value of this belonging to child process defaults to the priority class of the calling process

- If `dwCreationFlags` == 0:

+ The error mode (A variable to decide if there's any pop-up box if a critical error occurs) and the parent's console are inherited by the child;

+ Only ANSI characters are supported in the Environment block.


### LPVOID lpEnvironment

> ["A pointer to the environment block for the new process. If this parameter is NULL, the new process uses the environment of the calling process."](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessa#:~:text=A%20pointer%20to%20the%20environment%20block%20for%20the%20new%20process.%20If%20this%20parameter%20is%20NULL%2C%20the%20new%20process%20uses%20the%20environment%20of%20the%20calling%20process.)

Each variable of the environment block is a null-terminated string, and the block itself is also null-terminated.

The format of the block is simple:

"Var1=Value1\0
Var2=Value2\0
Var3=Value3\0
...
VarN=ValueN\0\0"

- When "\0\0" (ANSI) or "\0\0\0\0" (Unicode) is reached, we know that it is the end of the block;

- The variable's name must not contain equal sign "=".

- `CreateProcessA` fails if the block's size exceeds 32,767 characters. 

- If `lpEnvironment` wishes to point to Unicode-character environment block, **CREATE_UNICODE_ENVIRONMENT** (0x00000400) must be included in `dwCreationFlags`.

You can read more about environment variables [here](https://learn.microsoft.com/en-us/windows/win32/procthread/environment-variables).


### LPCSTR lpCurrentDirectory

This variable contains the full path to the current directory for the process. UNC Paths - Paths used to access network resources, is also supported.

If this is assigned to NULL, it will inherit the same current drive and directory of the parent process.

`lpCurrentDirectory` can be effectively used when the child process needs to work around files that are not in parent's directory, or work in isolation from the calling process's directory.


### LPSTARTUPINFOA lpStartupInfo

"A pointer to a STARTUPINFO or STARTUPINFOEX structure"

Remember to state **EXTENDED_STARTUPINFO_PRESENT** when pointing to STARTUPINFOEX.

Regardless of type, handles created in these struct must be closed via `CloseHandle` when they are no longer needed.


### LPPROCESS_INFORMATION lpProcessInformation

`PROCESS_INFORMATION` contains information, including handle, primary thread, PID, TID of the newly created process.

Similarly, handles in `PROCESS_INFORMATION` must also be closed with `CloseHandle` if there is no future use.


### Return value

- 0 when the function fails;
- Non-zero value when the function runs successfully.

**NOTE**: The function returns before the process has finished initialization.

`CreateProcessA` only creates the shell for the process. In initialization stage, the process loads DLLs, initialize memory, etc. Basically, if the API return successful sign, it does not mean the process is ready to run yet.

To solve this, you can utilize an API named [`WaitForInputIdle`](https://learn.microsoft.com/en-us/windows/desktop/api/winuser/nf-winuser-waitforinputidle) to wait until the initialization is finished.


### Remarks

#### Problems with `ExitProcess`

To shutdown a process, using `ExitProcess` function is recommended. The reason is that compared to other methods, `ExitProcess` does notify approaching termination to all DLLs attached to the process. 

However, if a thread calls `ExitProcess`, it causes all other threads of the same process to terminate. These threads also cannot execute any additional code (most notably, the thread termination code of attached DLLs). 

The best practice for closing process is:

1. The main thread, or the faulty thread, signals every other threads to stop via a flag.

2. In every thread, a while loop is needed to check this flag for a definite amount of time. When flag == true, these threads clean up the local resources, then call `return` or [`ExitThread`](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitthread).

3. The signaling thread will load other threads' handles to  [`WaitForMultipleObjects`](https://learn.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitformultipleobjects). This API forces the signaling thread to wait until all specified objects return.

4. When `WaitForMultipleObjects` finish, the signaling thread will close its handle, free memory, etc. Eventually, it returns or calls `ExitProcess`.

#### Child's Environment block

The only case when a process can directly change another process's content of its environment block is a parent process to its child processes. This only happens when child processes are in process creation.

#### Current directory information of system drive

In Windows, there's a thing called "current directory" for each drive. In environment variables context, current directory of any system drive has the name "=<drive_name>:=<Path>"

* e.g:

- If you are at C:\Users, the current directory for C will be "=C:=C:\Users". Whenever you switch to other drives and come back to C drive, it will automatically bring you to the "current directory" of C.

- The last directory that you were at before switching would become the current directory of the drive.

When Process A, for example, calls Process B, and A does not intervene anything related to the environment block, Windows automatically copies all environment variables
of A to B. 

In contrast, when A intervenes in terms of environment variables, Windows does NOT copy the **Current directory** variables from A to B. For that reason, programmers have to copy all the variables by themselves, sort them in the alphabetic order, and paste in the block of B.

---

## Endings

I have tried my best to sum up `CreateProcess` (mainly `CreateProcessA`) as much as possible. If there's any fault in the blog, please drop a comment below. 

The lab related to `CreateProcess` will come soon. Stay tuned!
