# pin
Collection of pin tools


## General information

Unless stated otherwise, all pin tools were created with Visual Studio 2012 Express, and written in C++
You can download Visual Studio Express 2012 for Windows Desktop here: https://www.microsoft.com/en-us/download/details.aspx?id=34673

To download & install the corresponding version of Pin, check this link:
https://software.intel.com/en-us/articles/pintool-downloads


## Creating a new Pin project folder for use with Visual Studio

This repository contains a simple python script `createpintool.py` that allows you to create a new Visual Studio project folder, based on the "MyPinTool" example folder.
You can find the script in the `win32` folder in this repository.

Place the script into the source/tools folder inside your pin folder structure.
Run the script from that working folder, and specify the name of the new project.

Example:
```
C:\pin\vc11\source\tools>python createpintool.py MyNewProjectName
[+] Creating new PIN project MyNewProjectName
    - Copying clean project
[+] Updating project files
    - Processing makefile -> makefile
    - Processing makefile.rules -> makefile.rules
    - Processing MyPinTool.cpp -> MyNewProjectName.cpp
    - Processing MyPinTool.vcxproj -> MyNewProjectName.vcxproj
    - Processing MyPinTool.vcxproj.filters -> MyNewProjectName.vcxproj.filters
[+] Done
```


## Notes

1. For Visual Studio C++ 2012, you need vc11, not vc12!
2. You may have to disable SAFESEH.
3. All pintools must be considered "proof of concept" and come without any warranty whatsoever.  The pintools have only been tested in a very small number of situations and may not work on your system. I do, however, encourage everyone to improve the code, performance, stability and/or submit Pull Requests to add interesting functionality.
4. I have compiled all pin tools in "Debug" mode.
5. If your pin tool doesn't run (i.e. if it terminates almost immediately after launching it), launch pin with the `-xyzzy -mesgon log_win` options, to activate some verbose logging.  A file `pintool.log` will be created in case of C++ errors.
6. The Corelan_HeapLog pin tool doesn't always end up launching the process. Especially with apps that spawn a lot of threads (IE, etc), I've been seeing weird errors in pintool.log. (Something like `E:Unexpected memory deallocation request of aligned memory`. Not sure what to do with it or how to fix it. Pin bug ?).
7. Remove *.log files before running the Corelan_HeapLog pin tool again. Also, make sure any relevant previous processes are killed before launching a new pin instance.
8. Pin may continue to write output to the log file(s) even after the instrumented process looks like it's gone.  Wait for all processes to properly finish & terminate before accessing log files.
9. I wrote this pintool in just a few days, after spending about a week studying C++ basics from Bjarne Stroustrup's books.  I am fully aware my code can be optimized etc. I will update/improve as I learn how to write better code.

## Available pintools in this repository

### 1. Corelan_HeapLog
This pintool allows you to log all calls to RtlAllocateHeap, RtlReAllocateHeap, VirtualAlloc and RtlFreeHeap.<br>
Output is written to `corelan_heaplog.log`, unless you have specified the `-splitfiles 1` option. This option will tell the pin tool to store output into `corelan_heaplog_<pid>.log` files instead (Fresh file for every process)
By default, output will be appended to `corelan_heaplog.log`. (In other words, make sure to put the file aside if you're instrumenting a different app)<br>
Exceptions are written to `corelan_heaplog_exception.log`. (One file, exceptions are appended to this file)

You can specify a couple of command line options: <br>
`-logalloc <value>`    : enable or disable logging allocations by setting value to 1 or 0<br>
`-logfree <value>`     : enable or disable logging free operations by setting value to 1 or 0<br>
`-timestamp <value>`   : enable or disable showing timestamp of heap operation by setting value to 1 or 0<br>
`-splitfiles <value>`  : enable or disable splitting output files into files that contain reference to the PID. Set value to 1 or 0<br>
`-silent <value>`      : enable or disable writing allocs and frees to output file(s). Set value to 1 or 0<br>
`-bufferoutput <value>`: enable or disable buffering output to memory before writing to disk. Set value to 1 or 0<br>
Both log settings are enabled by default.<br>
Timestamp is disabled by default (as it may slow down the process a tiny little bit). <br>
The splitfiles option is disabled by default.<br>
The silent option is disabled by default. Enabling this option will speed up the process (as the cost of writing entries to file will be gone).  Of course, this only makes sense if you're only interested in seeing the exception context.<br>
The bufferoutput option is enabled by default.<br>
If you are logging alloc and free operations, then this pintool will attempt to detect double free situations.<br>

The pintool *should* be capable of instrumenting child processes, provided that you have specified the `-follow-execv` pin command line option.

Example:<br>
```C:\pin\vc11>pin -follow-execv -t c:\pin\vc11\source\tools\Corelan_HeapLog\Debug\Corelan_HeapLog.dll -timestamp 1 -logfree 0 -splitfiles 0 -- "c:\Program Files (x86)\Internet Explorer\iexplore.exe" http://127.0.0.1:8080/blah.html```

Example output of corelan_heaplog_exception.log:<br>
```
PID 3000 | Exception context:
EIP: 0x61EDE1CB (C:\Windows\SysWOW64\mshtml.dll)
EAX: 0x160AAFA8 rtlallocateheap(0x144) rtlfreeheap(0x144) rtlallocateheap(0x58) rtlfreeheap(0x58) 
EBX: 0x13346F30 rtlallocateheap(0xd0) 
ECX: 0x00000052 
EDX: 0x00000000 
EBP: 0x0B79D6C4 
ESP: 0x0B79D670 
ESI: 0x00000000 
EDI: 0x160AAFA8 rtlallocateheap(0x144) rtlfreeheap(0x144) rtlallocateheap(0x58) rtlfreeheap(0x58) 

Closing exception log file for PID 3000
############## EOF
```
If a register contains a value that belongs to a heap chunk, Corelan_HeapLog will show all heap related operations (alloc & free) for that chunk in chronological order. <br>
This should make it easier to detect Use After Free cases (i.e. use of a chunk that has been freed. See example output, EAX & EDI)<br>
Hint: if you're interested in finding UAF, don't forget to enable full page heap mode (+hpa) first for the process that you're going to instrument.