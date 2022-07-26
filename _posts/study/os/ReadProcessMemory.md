---
title: 'ReadProcessMemory调用过程'
author: Kevin。
tags:
  - os
  - Windows
categories:
  - study
date: 2022-07-02 20:44:00
img: /images/article-banner/QQ截图20220702234227.jpg
typora-root-url: ..\..\..\
---



# ReadProcessMemory调用过程

## 基本用法

根据进程读句柄取内存空间

```c
BOOL ReadProcessMemory(
    HANDLE hProcess,  //进程句柄
    LPCVOID lpBaseAddress, // 内存地址
    LPVOID lpBuffer, // 接收的内容，缓冲区指针
    DWORD nSize, // 读取的字节数，如果为0直接返回
    LPDWORD lpNumberOfBytesRead //指向传输到指定缓冲区的字节数的指针
							 //如果lpNumberOfBytesRead为空，则忽略该参数
)//  //成功返回1 失败返回0
```

## 函数分析

![ReadProcessMemory函数](/images/image-20220702204623967.png)

由反汇编可以看出，`ReadProcessMemory`函数直接调用了`NtReadVirtualMemory`，而`NtReadVirtualMemory`在ntdll.dll中，`ntdll`又与`ntoskrnl.exe`对应，在`ntoskrnl.exe`中找到`NtReadVirtualMemory`

### NtReadVirtualMemory

经过一系列初始化，函数调用了`MmCopyVirtualMemory`

```c
push    eax      // HandleInformation
lea     eax, [ebp+Object]
push    eax      // Object
push    dword ptr [ebp+AccessMode] // AccessMode
push    _PsProcessType  // ObjectType
push    10h      // DesiredAccess
push    [ebp+ProcessHandle] // Handle
call    _ObReferenceObjectByHandle@24 // 跟据进程句柄获取EPROCESS结构的指针
mov     [ebp+var_Eprocess], eax
test    eax, eax 
jnz     short loc_4B1592 //如果_ObReferenceObjectByHandle返回0 则调用_MmCopyVirtualMemory
lea     eax, [ebp+var_size]
//--------------------------------------传入参数-------------------------------------
push    eax      // BytesRead
push    dword ptr [ebp+AccessMode] // AccessMode
push    esi      // NumberOfBytesToRead
push    [ebp+Buffer]    // Buffer
push    [edi+_KTHREAD.ApcState.Process] // CurrentProcess
push    [ebp+BaseAddress] // BaseAddress
push    [ebp+Object]    // Process
//--------------------------------------传入参数-------------------------------------
call    _MmCopyVirtualMemory@28 // // 调用_MmCopyVirtualMemory
mov     [ebp+var_Eprocess], eax
mov     ecx, [ebp+Object] // Object
call    @ObfDereferenceObject@4 // ObfDereferenceObject(x)
```

### MmCopyVirtualMemory

```c
mov     edi, edi
push    ebp
mov     ebp, esp
cmp     [ebp+Length], 0 //如果要读取的长度为0则直接返回
jz      loc_526787
push    ebx
mov     ebx, [ebp+process]
mov     ecx, ebx
mov     eax, large fs:_KPCR.PrcbData.CurrentThread
cmp     ebx, [eax+_KTHREAD.ApcState.Process] 
jnz     short loc_4A22F8
mov     ecx, [ebp+CurrentProcess]//如果目的线程和当前线程相同，将目标线程指针替换成当前线程

loc_4A22F8:     
add     ecx, 80h //    RunRef
mov     [ebp+process], ecx
call    @ExAcquireRundownProtection@4
test    al, al
jz      loc_52678E
cmp     [ebp+Length], 1FFh
push    esi //保存esi环境
push    edi //保存edi环境
mov     edi, [ebp+arg_18]
ja      loc_4ADEF1  //如果要读取的长度大于1FF，则调用_MiDoMappedCopy

loc_4A2320:      
push    edi      // BytesRead
push    dword ptr [ebp+AccessMode]
push    [ebp+Length]    // Length
push    [ebp+Address]   // Address
push    [ebp+CurrentProcess] // TAEGET_PROCESS
push    [ebp+BaseAddress] // int
push    ebx      // PRKPROCESS
call    _MiDoPoolCopy@28 // // 否则调用_MiDoPoolCopy
mov     esi, eax

loc_4A2338:     
mov     ecx, [ebp+process] // RunRef
call    @ExReleaseRundownProtection@4 // ExReleaseRundownProtection(x)
pop     edi
mov     eax, esi
pop     esi

loc_4A2344:    
pop     ebx

loc_4A2345:     
pop     ebp
retn    1Ch
```

### MiDoMappedCopy

```c
push    0ACh
push    offset stru_41C3D0
call    __SEH_prolog
xor     edi, edi
mov     [ebp+Status], edi // STATUS_SUCCESS
mov     eax, [ebp+arg_BaseAddress]
mov     [ebp+CurrentAddress], eax
mov     eax, [ebp+Address]
mov     [ebp+TargetAddress], eax
mov     eax, 0E000h     // MI_MAPPED_COPY_PAGES * PAGE_SIZE
bufferSize = ecx
TotalSize = eax
mov     bufferSize, [ebp+Length]
cmp     bufferSize, TotalSize
ja      short loc_4ADD8A // 如果要读取的长度小于等于0xe000，则将要读取的长度设定为0xe000
mov     TotalSize, bufferSize

loc_4ADD8A:      
lea     ebx, [ebp+MemoryDescriptorList] // MDL是用来建立一块虚拟地址空间与物理页面之间的映射
mov     [ebp+MDL], ebx
mov     [ebp+remainingSize], bufferSize
mov     [ebp+CurrentSize], TotalSize
mov     [ebp+FailedInProbe], edi
mov     [ebp+BadAddress], edi // 0
mov     [ebp+HaveBadAddress], edi // 0

loc_4ADDA2:     
mov     eax, [ebp+remainingSize] // while(remainingSize > 0)
cmp     eax, edi
jbe     loc_4ADEDF      // return
cmp     eax, [ebp+CurrentSize]
jb      loc_50DFB6      // 如果RemainingSize < CurrentSize，则CurrentSize = RemainingSize

loc_4ADDB6:     
lea     eax, [ebp+ApcState]
push    eax      // ApcState
push    [ebp+PROCESS]   // PROCESS
call    _KeStackAttachProcess@8 // 将本进程附加到目标进程中
mov     [ebp+BaseAddress], edi
mov     [ebp+PagesLocked], edi
mov     [ebp+var_4C], edi
mov     [ebp+ms_exc.registration.TryLevel], edi
mov     eax, [ebp+arg_BaseAddress]
xor     esi, esi
inc     esi
cmp     [ebp+CurrentAddress], eax
True = esi
jnz     short loc_4ADE05 // 初始化MemoryDescriptorList
cmp     [ebp+AccessMode], 0
jz      short loc_4ADE05 // 初始化MemoryDescriptorList
mov     [ebp+FailedInProbe], True
mov     eax, [ebp+Length]
cmp     eax, edi
jz      short loc_4ADE02
mov     ecx, [ebp+arg_BaseAddress]
add     eax, ecx
cmp     eax, ecx
jb      loc_50DFBE
cmp     eax, _MmUserProbeAddress
ja      loc_50DFBE

loc_4ADE02:      
mov     [ebp+FailedInProbe], edi

loc_4ADE05:      
mov     [ebx+_MDL.Next], edi // 初始化MemoryDescriptorList
mov     eax, [ebp+CurrentAddress]
and     eax, 0FFFh
mov     ecx, [ebp+CurrentSize]
lea     edx, [eax+ecx+0FFFh]
shr     edx, 0Ch
lea     edx, [edx*4+1Ch]
mov     [ebx+_MDL.Size], dx
mov     [ebx+_MDL.MdlFlags], di
mov     edx, [ebp+CurrentAddress]
and     edx, 0FFFFF000h
mov     [ebx+_MDL.StartVa], edx
mov     [ebx+_MDL.ByteOffset], eax
mov     [ebx+_MDL.ByteCount], ecx
push    edi      // Operation
push    dword ptr [ebp+AccessMode] // AccessMode
push    ebx      // MemoryDescriptorList
call    _MmProbeAndLockPages@12 // MmProbeAndLockPages(x,x,x)
mov     [ebp+PagesLocked], True
push    20h      // Priority -> HighPagePriority
push    edi      // BugCheckOnFailure -> False
push    edi      // RequestedAddress -> NULL
push    esi      // CacheType -> MmCached
push    edi      // AccessMode -> KernelMode
push    ebx      // MemoryDescriptorList
call    _MmMapLockedPagesSpecifyCache@24 
mov     [ebp+BaseAddress], eax
cmp     eax, edi
jz      loc_526621

loc_4ADE61: 
lea     eax, [ebp+ApcState]
push    eax      // ApcState
call    _KeUnstackDetachProcess@4 // KeUnstackDetachProcess(x)
lea     eax, [ebp+ApcState]
push    eax      // ApcState
push    [ebp+CurrentProcess] // PROCESS
call    _KeStackAttachProcess@8 // KeStackAttachProcess(x,x)
mov     eax, [ebp+arg_BaseAddress]
cmp     [ebp+CurrentAddress], eax
jnz     short loc_4ADE96
cmp     [ebp+AccessMode], 0
jz      short loc_4ADE96
mov     [ebp+FailedInProbe], esi
push    esi      // Alignment
push    [ebp+Length]    // Length
push    [ebp+Address]   // Address
call    _ProbeForWrite@12 // ProbeForWrite(x,x,x)
mov     [ebp+FailedInProbe], edi

loc_4ADE96:      
      
mov     [ebp+var_4C], esi
mov     ecx, [ebp+CurrentSize]
mov     esi, [ebp+BaseAddress]
mov     edi, [ebp+TargetAddress]
mov     eax, ecx
shr     ecx, 2
rep movsd
mov     ecx, eax
and     ecx, 3
rep movsb
or      [ebp+ms_exc.registration.TryLevel], 0FFFFFFFFh
lea     eax, [ebp+ApcState]
push    eax      // ApcState
call    _KeUnstackDetachProcess@4 // KeUnstackDetachProcess(x)
push    ebx      // MemoryDescriptorList
push    [ebp+BaseAddress] // BaseAddress
call    _MmUnmapLockedPages@8 // MmUnmapLockedPages(x,x)
push    ebx      // MemoryDescriptorList
call    _MmUnlockPages@4 // MmUnlockPages(x)
mov     eax, [ebp+CurrentSize]
sub     [ebp+remainingSize], eax
add     [ebp+CurrentAddress], eax
add     [ebp+TargetAddress], eax
xor     edi, edi
jmp     loc_4ADDA2      // while(remainingSize > 0)
// ---------------------------------------------------------------------------

loc_4ADEDF:     
mov     eax, [ebp+BytesRead]
mov     ecx, [ebp+Length]
mov     [eax], ecx
xor     eax, eax

loc_4ADEE9:     
call    __SEH_epilog
retn    1Ch
```

### MiDoPoolCopy

```c
TotalSize = edi

push    248h
push    offset stru_41D0D8
call    __SEH_prolog
mov     eax, large fs:_KPCR.PrcbData.CurrentThread
mov     eax, [ebp+SourceAddress]
mov     [ebp+CurrentAddress], eax
mov     eax, [ebp+TargetAddress]
mov     [ebp+CurrentTargetAddress], eax
mov     eax, 10000h     // MI_MAX_TRANSFER_SIZE = 64 * 1024
mov     TotalSize, eax
cmp     [ebp+BufferSize], eax // if (BufferSize <= MI_MAX_TRANSFER_SIZE) TotalSize = BufferSize//
ja      short loc_4AD3B7
mov     TotalSize, [ebp+BufferSize]

loc_4AD3B7:      
and     [ebp+HavePoolAddress], 0
mov     esi, 200h// MI_POOL_COPY_BYTES
cmp     [ebp+BufferSize], esi
ja      loc_5266C7      // ExAllocatePoolWithTag

loc_4AD3C9:      
lea     eax, [ebp+StackBuffer]
mov     [ebp+PoolAddress], eax

loc_4AD3D2:      
xor     eax, eax
mov     [ebp+Status], eax
mov     [ebp+_], eax
mov     ecx, [ebp+BufferSize]
mov     [ebp+RemainingSize], ecx
mov     ebx, TotalSize
mov     [ebp+FailedInProbe], eax // STATUS_SUCCESS

loc_4AD3E5:      
xor     eax, eax
cmp     [ebp+RemainingSize], eax
ja      loc_4AD496      // RemainingSize < CurrentSize
cmp     [ebp+HavePoolAddress], eax
jnz     loc_526779      // return

loc_4AD3F9:      
mov     eax, [ebp+ReturnSize]
mov     ecx, [ebp+BufferSize]
mov     [eax], ecx
xor     eax, eax

loc_4AD403:      
 // MiDoPoolCopy(x,x,x,x,x,x,x)+793ED↓j
call    __SEH_epilog
retn    1Ch
// ---------------------------------------------------------------------------

loc_4AD40B:     
 // MiDoPoolCopy(x,x,x,x,x,x,x)+13C↓j ...
mov     ecx, ebx // RtlCopyMemory -> memcpy
mov     esi, [ebp+CurrentAddress]
mov     edi, [ebp+PoolAddress]
mov     eax, ecx
shr     ecx, 2
rep movsd // 把ESI的内容拷贝到EDI中
 // 把数据读到临时空间，这个临时空间是在内核堆栈或内核的内存中，即高2G的位置
mov     ecx, eax
and     ecx, 3
rep movsb // 对齐
lea     eax, [ebp+ApcState]
push    eax      // ApcState
call    _KeUnstackDetachProcess@4 // KeUnstackDetachProcess(x)
lea     eax, [ebp+ApcState]
push    eax      // ApcState
push    [ebp+TAEGET_PROCESS] // PROCESS
call    _KeStackAttachProcess@8 // KeStackAttachProcess(x,x)
mov     eax, [ebp+SourceAddress]
cmp     [ebp+CurrentAddress], eax
jnz     loc_4E24CD
cmp     [ebp+PreviousMode], 0
jz      loc_4E24CD
xor     esi, esi
inc     esi
mov     [ebp+FailedInProbe], esi
push    esi      // Alignment
push    [ebp+BufferSize] // Length
push    [ebp+TargetAddress] // Address
call    _ProbeForWrite@12 // ProbeForWrite(x,x,x)
and     [ebp+FailedInProbe], 0

loc_4AD462:   
mov     [ebp+var_3C], esi
mov     ecx, ebx
mov     esi, [ebp+PoolAddress]
mov     edi, [ebp+CurrentTargetAddress]
mov     eax, ecx
shr     ecx, 2
rep movsd
mov     ecx, eax
and     ecx, 3
rep movsb
or      [ebp+Exception.registration.TryLevel], 0FFFFFFFFh
lea     eax, [ebp+ApcState]
push    eax      // ApcState
call    _KeUnstackDetachProcess@4 // KeUnstackDetachProcess(x)
sub     [ebp+RemainingSize], ebx
add     [ebp+CurrentAddress], ebx
add     [ebp+CurrentTargetAddress], ebx
jmp     loc_4AD3E5
// ---------------------------------------------------------------------------

loc_4AD496:      
cmp     [ebp+RemainingSize], ebx
jb      loc_5266E7      // CurrentSize = RemainingSize//

loc_4AD49F:    
0 = esi
lea     eax, [ebp+ApcState]
push    eax      // ApcState
push    [ebp+SourceProcess] // PROCESS
call    _KeStackAttachProcess@8 // KeStackAttachProcess(x,x)
xor     esi, esi
mov     [ebp+var_3C], 0
mov     [ebp+Exception.registration.TryLevel], 0
mov     ecx, [ebp+SourceAddress]
cmp     [ebp+CurrentAddress], ecx // (CurrentAddress == SourceAddress) && (PreviousMode != KernelMode)
jnz     loc_4AD40B      // RtlCopyMemory -> memcpy
cmp     [ebp+PreviousMode], 0 //  KernelMode
jz      loc_4AD40B      // RtlCopyMemory -> memcpy
mov     [ebp+FailedInProbe], 1
mov     eax, [ebp+BufferSize]
cmp     eax, 0
jz      short loc_4AD4ED
add     eax, ecx
cmp     eax, ecx
jb      loc_4E24D5
cmp     eax, _MmUserProbeAddress
ja      loc_4E24D5

loc_4AD4ED:     
mov     [ebp+FailedInProbe], esi
jmp     loc_4AD40B      // RtlCopyMemory -> memcpy

```
### KeStackAttachProcess

```c
mov     edi, edi
push    ebp
mov     ebp, esp
push    esi
push    edi
mov     eax, large fs:_KPCR.PrcbData.CurrentThread
mov     esi, eax
mov     eax, large fs:_KPCR.PrcbData.DpcRoutineActive
test    eax, eax
jnz     loc_44A944
mov     edi, [ebp+PROCESS]
cmp     [esi+_KTHREAD.ApcState.Process], edi
jz      loc_41D0C8
xor     ecx, ecx
call    ds:__imp_@KeAcquireQueuedSpinLockRaiseToSynch@4 
cmp     [esi+_KTHREAD.ApcStateIndex], OriginalApcEnvironment
mov     byte ptr [ebp+PROCESS], al
jnz     loc_431CA8 //如果当前线程以及被附加上，则直接将ApcState传入_KiAttachProcess，否则传入SavedApcState
lea     eax, [esi+_KTHREAD.SavedApcState]
push    eax
push    [ebp+PROCESS]
push    edi
push    esi
call    _KiAttachProcess@16 // 把其他进程加载到本进程
mov     eax, [ebp+ApcState]
and     [eax+_KAPC_STATE.Process], 0

loc_41CFEE:      
pop     edi
pop     esi
pop     ebp
retn    8

loc_41D0C8:  
mov     eax, [ebp+ApcState]
mov     [eax+_KAPC_STATE.Process], 1
jmp     loc_41CFEE
```

### KiAttachProcess

只要操作进程内存就会调用KiAttachProcess函数，hook KiAttachProcess函数即可知道哪个线程中的进程进来的

```c
mov     edi, edi
push    ebp
mov     ebp, esp
push    ebx
push    esi
mov     esi, [ebp+Thread]
push    edi
push    [ebp+SaveApcState]
mov     edi, [ebp+Process]
inc     [edi+_EPROCESS.Pcb.StackCount] // 当前进程内存中线程的栈计数+1
lea     ebx, [esi+_KTHREAD.ApcState] // ApcListHead[KernelMode]
push    ebx
call    _KiMoveApcState@8 // SaveApcState = ApcState，备份ApcState
mov     [ebx+_LIST_ENTRY.Blink], ebx
mov     [ebx+_LIST_ENTRY.Flink], ebx
lea     eax, [esi+(_KTHREAD.ApcState.ApcListHead.Flink+8)] // ApcListHead[UserMode]  8 = sizeof(LIST_ENTRY) * 1
mov     [eax+_LIST_ENTRY.Blink], eax
mov     [eax+_LIST_ENTRY.Flink], eax
lea     eax, [esi+_KTHREAD.SavedApcState]
cmp     [ebp+SaveApcState], eax
mov     [esi+_KTHREAD.ApcState.Process], edi // 更改为要读取的进程
mov     [esi+_KTHREAD.ApcState.KernelApcInProgress], 0
mov     [esi+_KTHREAD.ApcState.KernelApcPending], 0
mov     [esi+_KTHREAD.ApcState.UserApcPending], 0
jnz     short loc_41A519 // 判断进程是否在内存中
mov     [esi+_KTHREAD.ApcStatePointer], eax // 14C SavedApcState 替换指针的位置
mov     [esi+(_KTHREAD.ApcStatePointer+4)], ebx // 34 ApcState
mov     [esi+_KTHREAD.ApcStateIndex], AttechedApcEnvironment // 改成挂靠的APC

loc_41A519:      // CODE XREF: KiAttachProcess(x,x,x,x)+43↑j
cmp     [edi+_KPROCESS.State], ProcessInMemory
jnz     loc_44A8A2      // 判断是否在内存中
lea     esi, [edi+40h]

loc_41A526:      // CODE XREF: KiAttachProcess(x,x,x,x)+303D7↓j
mov     eax, [esi+_LIST_ENTRY.Flink]
cmp     eax, esi
jnz     loc_44A87F      // 判断线程的等待链表是否为空
mov     eax, [ebp+SaveApcState]
push    [eax+_KAPC_STATE.Process]
push    edi
call    _KiSwapProcess@8 // KiSwapProcess(x,x)
mov     cl, [ebp+ApcLock]
call    @KiUnlockDispatcherDatabase@4 // KiUnlockDispatcherDatabase(x)

loc_41A544:      // CODE XREF: 0044A89D↓j
 // KiAttachProcess(x,x,x,x)+30463↓j
pop     edi
pop     esi
pop     ebx
pop     ebp
retn    10h
```



