# win git error init_cheap:VirtualAlloc pointer is null, Win32 error 487

标签（空格分隔）： git

---

在idea利用git进行代码更新时遇到的问题，google了一下，早StackOverflow找到解决办法，在此share一下

Error message

    E:\vipshop\storm-sql>git pull origin joeywen
      0 [main] us 0 init_cheap: VirtualAlloc pointer is null, Win32 error 487
    AllocationBase 0x0, BaseAddress 0x68570000, RegionSize 0x2F0000, State 0x10000
    C:\Program Files (x86)\Git\bin\sh.exe: *** Couldn't reserve space for cygwin's heap, Win32     error 0
    
原因分析：
    
>Cygwin uses persistent shared memory sections, which can on occasion become corrupted. The symptom of this is that some Cygwin programs begin to fail, but other applications are unaffected. Since these shared memory sections are persistent, often a reboot is needed to clear them out before the problem can be resolved.

解决办法：

>I had the same problem. I found solution here http://jakob.engbloms.se/archives/1403
>`c:\msysgit\bin>rebase.exe -b 0x50000000 msys-1.0.dll`
>For me solution was slightly different. It was
>`C:\Program Files (x86)\Git\bin>rebase.exe -b 0x50000000 msys-1.0.dll`
>Before you rebase dlls, you should make sure it is not in use:
>`tasklist /m msys-1.0.dll`
>If the rebase command fails with something like:
>>ReBaseImage (msys-1.0.dll) failed with last error = 6
>
>You will need to perform the following steps in order:
>>-1. Copy the dll to another directory
>>-2.Rebase the copy using the commands above
>>-3.Replace the original dll with the copy.

参考：[Git Extensions: Win32 error 487: Couldn't reserve space for cygwin's heap, Win32 error 0](http://stackoverflow.com/questions/18502999/git-extensions-win32-error-487-couldnt-reserve-space-for-cygwins-heap-win32)





