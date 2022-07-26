---
title: 'WaitForSingleObject'
author: Kevin。
tags:
  - os
  - Windows
categories:
  - study
date: 2022-07-05 20:25:00
img: /images/article-banner/QQ截图20220726235121.jpg
typora-root-url: ..\..\..\
---

# WaitForSingleObject

由于进程间通信必须凭借某个已打开的对象才能发生，所以Windows调用`NtWaitForSingleObject`和`NtWaitForMultipleObjects`使线程在对象上等待，`NtWaitForMultipleObjects`是前者的扩展，使一个线程可以同时在多个对象上等待。线程可以同时等待多个目标，当所有对象全部满足条件时才完成的叫合取；其中任一对象满足条件就算完成的叫析取。等待发生于线程和对象之间，一个线程可以等待多个对象，而一个对象也可以被多个线程所等待。

等待的关系就由_KWAIT_BLOCK结构体承载

```c
typedef struct _KWAIT_BLOCK
{
     LIST_ENTRY WaitListEntry; //挂入等待目标的等待块队列，双链队列
     PKTHREAD Thread; //指向所关联的指针
     PVOID Object; //等待目标的指针
     PKWAIT_BLOCK NextWaitBlock; //线程的等待块队列
     WORD WaitKey;
     UCHAR WaitType;
     UCHAR SpareByte;
} KWAIT_BLOCK, *PKWAIT_BLOCK;
```

当进程都在睡眠等待时，会同挂KTHREAD结构体中的WaitListEntry挂入PRCB的等待队列WaitListHead，PRCB的这个队列会把所有处于等待睡眠状态的线程都串在一起。每个此对象都描述了哪个线程Thread域再等待哪个对象Object域，有几个对象就有几个wait_block，NextWaitBlock指向自己说明只有一个等待块。

0x008   WaitListHead  _LIST_ENTRY   代表这个对象可以被等待，此链表包含了所有正在等待该分发器对象的线程

以下结构体均含有此结构

```c
_KTIMER
_KEVENT
_KPROCESS
_KTHREAD
_FILE_OBJECT
_KSEMAPHORE
_KMUTANT
```

## 用法

三种同步方式：事件、信号量、互斥

```c
DWORD WaitForSingleObject(
    HANDLE hHandle, //指明一个内核对象的句柄  Event/Semaphore/Mutex
    DWORD dwMilliseconds //等待时间
); 
```

### Event

```c
#include <stdio.h>
#include <Windows.h>

HANDLE handle;
int data = 0;

VOID func(){
    WaitForSingleObject(handle, 5000);
    printf("test_%d\n", data++);
}

int main(){
    handle = CreateEvent(
        NULL, 
        TRUE, 
        FALSE, //FALSE先等待后执行，TRUE直接执行 
        "Event1"
    );
    CreateThread(NULL, 0, (LPTHREAD_START_ROUTINE)func, NULL, 0, NULL);
    getchar();
    printf("Hello World\n");
    return 0;
}

```

### Semaphore

信号量可以限制执行的线程数，通过`WaitForSingleObject(semaphore, 时间)`来限制等待执行时间或`INFINITE`无限等待

```c
HANDLE CreateSemaphore(
    LPSECURITY_ATTRIBUTES lpSemaphoreAttributes, // SD
    LONG lInitialCount, // 最开始可执行的事件数
    LONG lMaximumCount, // 最大可执行的事件数
    LPCT STRlpName// 事件名称
);
```

如果`lInitialCount > lMaximumCount` 或 `lInitialCount < 0` 或`lMaximumCount <= 0`，则此信号量无效

```c
#include <stdio.h>
#include <Windows.h>

HANDLE semaphore;
int data = 0;

VOID func(){
    WaitForSingleObject(semaphore, INFINITE);
    printf("test_%d\n", data++);
}

int main(){
    semaphore = CreateSemaphore(NULL, 2, 3, "Test1"); //最开始执行2个线程，最多执行3个
    for(int i = 0; i < 10; i++){
        CreateThread(NULL, 0, (LPTHREAD_START_ROUTINE)func, NULL, 0, NULL);
    }
    
    for(int j = 0; j < 10; j++){
        Sleep(1000);
        ReleaseSemaphore(semaphore, 3, NULL); //释放的信号量大于可最大可执行的事件量则释放无效
    }
    printf("Hello World\n");
    return 0;
}
```

### Mutex

互斥体，在Windows系统中，线程可以在等待函数中指定一个此线程已经拥有的互斥体，由于Windows的防死锁机制，这种做法不会阻止此线程的运行。

```c
HANDLE CreateMutex(
    LPSECURITY_ATTRIBUTESlpMutexAttributes, // 指向安全属性的指针
    BOOLbInitialOwner, // 初始化互斥对象的所有者
    LPCTSTRlpName // 指向互斥对象名的指针
);
```

```c
#include <stdio.h>
#include <Windows.h>

HANDLE mutex;

DWORD func1(){
    WaitForSingleObject(mutex, INFINITE);
    printf("ThreadFun\n");
    return 0;
}

int main(){
    mutex = CreateMutex(
        NULL, 
        TRUE, //TRUE等待信号，FALSE直接执行
        "mutex"
    );
    CreateThread(NULL, 0,(LPTHREAD_START_ROUTINE)func1, NULL, 0, NULL);
    Sleep(5000);
    ReleaseMutex(mutex);
    getchar();
    printf("Hello World\n");
    return 0;
}

```



## NtWaitForSingleObject

```c
push    2Ch
push    offset stru_40EE88
call    __SEH_prolog
mov     eax, large fs:_KPCR.PrcbData.CurrentThread
mov     al, [eax+_KTHREAD.PreviousMode]
mov     [ebp+PreviousMode], al
mov     esi, [ebp+Timeout]
xor     ebx, ebx
cmp     esi, ebx
jz      short loc_496FB5
cmp     al, bl          //KernelMode
jz      short loc_496FB5
mov     [ebp+ms_exc.registration.TryLevel], ebx //从用户空间复制参数TimeOut的值
mov     eax, _MmUserProbeAddress
cmp     esi, eax
jnb     loc_529100
mov     ecx, [esi]
mov     [ebp+var_3C], ecx
mov     eax, [esi+4]
loc_496FA2:       
mov     [ebp+var_38], eax
mov     [ebp+var_34], ecx
mov     [ebp+var_30], eax
lea     esi, [ebp+var_34]
mov     [ebp+Timeout], esi
or      [ebp+ms_exc.registration.TryLevel], 0FFFFFFFFh
loc_496FB5:      
push    ebx             //HandleInformation
lea     eax, [ebp+ObjectPointer]
push    eax             //Object
push    dword ptr [ebp+PreviousMode]
push    ebx             //ObjectType
push    100000h         //SYNCHRONIZE
push    [ebp+ObjectHandle] //Handle
call    _ObReferenceObjectByHandle@24 //根据句柄找到目标对象
mov     edi, eax
cmp     edi, ebx
jl      short loc_497007
mov     ecx, [ebp+ObjectPointer]
mov     eax, [ecx-10h]
mov     eax, [eax+48h]  //Object->Type->DefaultObject
cmp     eax, ebx        //字段DefaultObject的值是指针或位移
jl      short loc_496FE0 //该判断为宏定义#define IsPointerOffset(Ptr)((LONG_PTR)(Ptr)>=0)
add     eax, ecx        //Object指针
loc_496FE0:     
mov     [ebp+ms_exc.registration.TryLevel], 1
push    esi             //Timeout
push    dword ptr [ebp+Alertable] //Alertable，表示是否允许本次等待因用户空间APC而中断
push    dword ptr [ebp+PreviousMode] //WaitMode
push    6               //UserRequest
push    eax             //Object
call    _KeWaitForSingleObject@20
mov     edi, eax
mov     [ebp+var_2C], edi

loc_496FFB:               
or      [ebp+ms_exc.registration.TryLevel], 0FFFFFFFFh
mov     ecx, [ebp+ObjectPointer] //Object
call    @ObfDereferenceObject@4 //ObfDereferenceObject(x)

loc_497007:               
mov     eax, edi

loc_497009:
call    __SEH_epilog
retn    0Ch
```

实际的等待目标不一定是该句柄所代表的对象，实际上有些对象不可等待。可直接等待的对象的数据结构的第一个成分为DISPATCHER_HEADER结构，且该结构中的SignalState为1，而不可等待的对象则没有。

宏定义`#define IsPointerOffset(Ptr)((LONG_PTR)(Ptr)>=0)`中，LONG_PTR又是指针又是整数，IsPointerOffset的意思九四hi判断一个指针实际上是否为一个偏移量，如果Ptr>=0，那么最高位则为0，若小于0，则最高位为1。由于Windows中用户空间为0x0 - 0x7FFFFFFF，系统空间为0x80000000以上，所以凡是指向内核中的任意指针，最高位必为1，所以只要对象最高位为0，则目标对象本身就是可等待对象。

## KeWaitForSingleObject

```c
mov     edi, edi        //KeWaitForMutexObject
push    ebp
mov     ebp, esp
sub     esp, 14h
push    ebx
push    esi
push    edi
mov     eax, large fs:_KPCR.PrcbData.CurrentThread
mov     edx, [ebp+Timeout]
mov     ebx, [ebp+Object]
mov     esi, eax
cmp     [esi+_KTHREAD.WaitNext], 0 //判断有没有等待
mov     [ebp+var_Timeout], edx
lea     edi, [esi+_KTHREAD.WaitBlock] //取出第一个等待块
lea     eax, [esi+(_KTHREAD.WaitBlock.WaitListEntry.Flink+48h)] //取出第四个等待块，即定时器等待块 TIMER_WAIT_BLOCK
jnz     InitializeWaitSingle

loc_40541F:             
call    ds:__imp__KeRaiseIrqlToSynchLevel@0 //KeRaiseIrqlToSynchLevel()
xor     ecx, ecx
cmp     [ebp+Timeout], ecx
mov     [esi+_KTHREAD.WaitIrql], al //初始化等待块 InitializeWaitSingle
mov     [esi+_KTHREAD.WaitBlockList], edi
mov     [edi+_KWAIT_BLOCK.Object], ebx
mov     [edi+_KWAIT_BLOCK.WaitKey], cx
mov     [edi+_KWAIT_BLOCK.WaitType], WaitAny //只要一个有信号
mov     [esi+_KTHREAD.WaitStatus], ecx
jz      loc_40EAFF
lea     eax, [esi+(_KTHREAD.WaitBlock.WaitListEntry.Flink+48h)] //如果结束的时间不等于0则添加定时器
mov     [edi+_KTHREAD.MutantListHead.Flink], eax //Flink指链表下一项，Blink指链表前一项
mov     [eax+_KTHREAD.MutantListHead.Flink], edi
mov     [esi+_KTHREAD.Timer.Header.WaitListHead.Flink], eax
mov     [esi+_KTHREAD.Timer.Header.WaitListHead.Blink], eax

loc_40545E:        
mov     al, [ebp+Alertable] //设置线程等待参数
mov     dl, byte ptr [ebp+WaitReason]
mov     [esi+_KTHREAD.Alertable], al
mov     al, [ebp+WaitMode]
test    al, al
mov     [esi+_KTHREAD.WaitMode], al
mov     [esi+_KTHREAD.WaitReason], dl
mov     [esi+_KTHREAD.___u24.WaitListEntry.Flink], ecx //清空等待链表
jz      loc_405528
cmp     [esi+_KTHREAD.EnableStackSwap], 0 //判断是否开启栈交互
jz      loc_405528
cmp     [esi+_KTHREAD.Priority], 19h //判断优先级
mov     [ebp+Object], 1
jge     loc_405528

loc_40549C:      
mov     eax, ds:_KeTickCount.LowPart
mov     [esi+_KTHREAD.WaitTime], eax //设置等待时间
mov     eax, large fs:_KPCR.Prcb
lea     ecx, [eax+_KPRCB.LockQueue]
call    @KeAcquireQueuedSpinLockAtDpcLevel@4 //KeAcquireQueuedSpinLockAtDpcLevel(x)
mov     edx, [ebp+Timeout]

loc_4054B8:            
cmp     [esi+_KTHREAD.ApcState.KernelApcPending], 0 //判断有没有内核APC等待执行
jnz     loc_4415EA

loc_4054C2:             
cmp     [ebx+_DISPATCHER_HEADER.Type], MutantObject //判断是否为MutantObject对象
jnz     loc_402657      //判断有没有信号，如果没有信号
mov     eax, [ebx+_DISPATCHER_HEADER.SignalState]
test    eax, eax        //判断是否有信号
jg      loc_402616      //处理互斥体
cmp     esi, [ebx+_KMUTANT.OwnerThread] //判断是否为同一个线程
jz      loc_402616      //信号状态的最大值

loc_4054DF:            
cmp     [ebp+Alertable], 0 //判断线程是否可唤醒
jnz     loc_43DD24      //要唤醒的是用户还是内核
cmp     [ebp+WaitMode], 0
jz      short loc_4054F9 //如果时间为0，跳
cmp     [esi+_KTHREAD.ApcState.UserApcPending], 0 //判断有无用户APC等待执行
jnz     loc_4204DC

loc_4054F9:           
test    edx, edx        //如果时间为0，跳
jz      loc_40ABDD
mov     eax, [edx+4]
or      eax, [edx]
jnz     loc_40ED9A

loc_40550C:            
mov     edi, 102h       //STATUS_TIMEOUT

loc_405511:             
push    esi             //Thread
call    _KiAdjustQuantumThread@4 //KiAdjustQuantumThread(x)

loc_405517:             
mov     cl, [esi+_KTHREAD.WaitIrql]
call    @KiUnlockDispatcherDatabase@4 //KiUnlockDispatcherDatabase(x)
mov     eax, edi

loc_405521:            
pop     edi
pop     esi
pop     ebx
leave
retn    14h
//---------------------------------------------------------------------------

loc_405528:      
mov     [ebp+Object], ecx
jmp     loc_40549C
```

### InitializeWaitSingle

```c
xor     ecx, ecx
mov     [esi+_KTHREAD.WaitNext], 0
mov     [esi+_KTHREAD.WaitBlockList], edi //第一个等待块
and     [edi+_KWAIT_BLOCK.WaitKey], 0 //单等待，等待第一个，等待超时和唤醒
inc     ecx
mov     [edi+_KWAIT_BLOCK.Object], ebx //传进来的等待的对象
mov     [edi+_KWAIT_BLOCK.WaitType], cx //ECX = 1 WaitAny 只有一个有信号就执行
and     [esi+_KTHREAD.WaitStatus], 0
test    edx, edx
jnz     short loc_43EC3C //判断时间是否等于0，如果不等于0则添加定时器
mov     [edi+_KWAIT_BLOCK.NextWaitBlock], edi //如果没有时间，则指向自己

loc_43EBFE:             
mov     al, [ebp+Alertable]
and     [esi+_KTHREAD.___u24.WaitListEntry.Flink], 0
cmp     [ebp+WaitMode], 0 //判断等待的模式是用户还是内核，0内核，1用户
mov     [esi+_KTHREAD.Alertable], al //唤醒初始化
mov     al, [ebp+WaitMode]
mov     [esi+_KTHREAD.WaitMode], al
mov     al, byte ptr [ebp+WaitReason]
mov     [esi+_KTHREAD.WaitReason], al
jz      short loc_43EC50 //把等待参数设置为0
cmp     [esi+_KTHREAD.EnableStackSwap], 0
jz      short loc_43EC50 //把等待参数设置为0
cmp     [esi+_KTHREAD.Priority], 19h
jge     short loc_43EC50 //把等待参数设置为0
mov     [ebp+Object], ecx

loc_43EC2F:            
mov     eax, ds:_KeTickCount.LowPart
mov     [esi+_KTHREAD.WaitTime], eax //设置等待时间
jmp     loc_4054B8      //判断有没有内核APC等待执行
//---------------------------------------------------------------------------

loc_43EC3C:          
mov     [edi+_KWAIT_BLOCK.NextWaitBlock], eax //第四个等待块，时间
mov     [eax+_KWAIT_BLOCK.NextWaitBlock], edi //插入链表
mov     [esi+_KTHREAD.Timer.Header.WaitListHead.Flink], eax
mov     [esi+_KTHREAD.Timer.Header.WaitListHead.Blink], eax
jmp     short loc_43EBFE
//---------------------------------------------------------------------------

loc_43EC50:            
and     [ebp+Object], 0 //把等待参数设置为0
jmp     short loc_43EC2F
```

### loc_402657

处理MutantObject对象

```c
loc_402657:        
cmp     [ebx+_DISPATCHER_HEADER.SignalState], 0 // 判断有没有信号，如果没有信号
jle     loc_4054DF      // 判断线程是否可唤醒
mov     al, [ebx+_DISPATCHER_HEADER.Type] // 如果有信号
mov     cl, al
and     cl, GateObject
cmp     cl, EventSynchronizationObejct // 判断是否为事件对象
jz      loc_4109B0
cmp     al, SemaphereObject // 是否为信号量对象
jz      loc_4135E4

loc_402679:        
xor     edi, edi
jmp     loc_405511      // Thread
```

### loc_402616

处理互斥体

```c
loc_402616:           
cmp     eax, 80000000h  // 信号状态的最大值
jz      loc_44AB58      // 执行APC并派发异常状态
dec     [ebx+_KMUTANT.Header.SignalState] // 信号-1
jnz     short loc_40264F
movzx   eax, [ebx+_KMUTANT.ApcDisable]
sub     [esi+_KTHREAD.KernelApcDisable], eax
cmp     [ebx+_KMUTANT.Abandoned], 1 // 判断是否为废弃的互斥
mov     [ebx+_KMUTANT.OwnerThread], esi
jz      loc_442ABF

loc_40263D:     
mov     ecx, [esi+_KTHREAD.MutantListHead.Blink]
mov     edx, [ecx+_LIST_ENTRY.Flink]
lea     eax, [ebx+_KMUTANT.MutantListEntry]
mov     [eax+_LIST_ENTRY.Flink], edx
mov     [eax+_LIST_ENTRY.Blink], ecx
mov     [edx+_LIST_ENTRY.Blink], eax
mov     [ecx+_LIST_ENTRY.Flink], eax

loc_40264F:
mov     edi, [esi+_KTHREAD.WaitStatus]
jmp     loc_405511      // Thread
```

### loc_43DD24

处理可唤醒线程

```c
loc_43DD24:     
movsx   eax, [ebp+WaitMode] // 要唤醒的是用户还是内核
cmp     [eax+esi+_KTHREAD.Alerted], 0
jnz     loc_44AB80
cmp     [ebp+WaitMode], 0 // 判断是否为内核
jz      short loc_43DD44 // 内核是否可唤醒
lea     eax, [esi+(_KTHREAD.ApcState.ApcListHead.Flink+8)] // 判断用户APC是否为空
cmp     [eax], eax
jnz     loc_430975

loc_43DD44:
cmp     [esi+_KTHREAD.Alerted], 0 // 内核是否可唤醒
jz      loc_4054F9      // 如果时间为0，跳
jmp     loc_44AB72
```

## Semaphore

临界区：是一个访问共用资源，当有线程进入临界区段时，其他线程或是进程必须等待

信号量：简单来说就是同一时间可运行的线程数或进程数，通过P、V操作进行信号量的增减

### NtCreateSemaphore

创建一个信号量对象

```c
push    1Ch
push    offset stru_41D338
call    __SEH_prolog
mov     eax, large fs:_KPCR.PrcbData.CurrentThread
mov     al, [eax+_KTHREAD.PreviousMode]
mov     [ebp+PreviousMode], al
xor     ebx, ebx
cmp     al, bl
jz      loc_4A3AA2      // 判断上一层调用是否为0环
mov     [ebp+ms_exc.registration.TryLevel], ebx
mov     esi, [ebp+SemaphoreHandle]
mov     eax, _MmUserProbeAddress
cmp     esi, eax
jnb     loc_53BB06

loc_4A3A1C:      
mov     eax, [esi]      // 测试可读可写
mov     [esi], eax
or      [ebp+ms_exc.registration.TryLevel], 0FFFFFFFFh

loc_4A3A24: 
//参数检查 InitialCount > MaximumCount 或 InitialCount < 0 或 MaximumCount <= 0
cmp     [ebp+MaximumCount], ebx // 判断最大的信号量是否大于0
jle     loc_4A3AB1      // 小于等于0则 return STATUS_INVALID_PARAMETER 无效的参数
mov     edi, [ebp+InitialCount]
cmp     edi, ebx
jl      short loc_4A3AB1 // return STATUS_INVALID_PARAMETER 无效的参数
cmp     edi, [ebp+MaximumCount]
jg      short loc_4A3AB1 // 如果初始化参数大于最大参数，return STATUS_INVALID_PARAMETER 无效的参数
lea     eax, [ebp+Semaphore]
push    eax       
push    ebx       
push    ebx       
push    14h       
push    ebx       
push    dword ptr [ebp+PreviousMode] // BackTraceHash
push    [ebp+ObjectAttributes]
push    _ExSemaphoreObjectType
push    dword ptr [ebp+PreviousMode] // PreviousMode
call    _ObCreateObject@36 // 以类型对象ExSemaphoreObjectType为样板创建信号量对象结构
mov     [ebp+var_28], eax
cmp     eax, ebx
jl      short loc_4A3A97 // 判断创建有没有成功
push    [ebp+MaximumCount] // Limit
push    edi             // Count
push    [ebp+Semaphore] // Semaphore
call    _KeInitializeSemaphore@12 // 初始化信号结构体成员
lea     eax, [ebp+Handle]
push    eax             // Handle
push    ebx             // NewObject
push    ebx             // ObjectPointerBias
push    [ebp+DesiredAccess] // DesiredAccess
push    ebx             // PassedAccessState
push    [ebp+Semaphore] // Object
call    _ObInsertObject@24 // 插入信号量对象
mov     [ebp+var_28], eax // 返回信号量句柄
cmp     eax, ebx
jl      short loc_4A3A97 // 判断有没有插入成功
cmp     [ebp+PreviousMode], bl
jz      short loc_4A3AAA
mov     [ebp+ms_exc.registration.TryLevel], 1
mov     eax, [ebp+Handle]
mov     [esi], eax

loc_4A3A93:            
or      [ebp+ms_exc.registration.TryLevel], 0FFFFFFFFh

loc_4A3A97:       
mov     eax, [ebp+var_28]

loc_4A3A9A:
call    __SEH_epilog
retn    14h
; ---------------------------------------------------------------------------

loc_4A3AA2:           
mov     esi, [ebp+SemaphoreHandle]
jmp     loc_4A3A24
; ---------------------------------------------------------------------------

loc_4A3AAA:    
mov     eax, [ebp+Handle]
mov     [esi], eax
jmp     short loc_4A3A97
; ---------------------------------------------------------------------------

loc_4A3AB1:
mov     eax, 0C000000Dh // STATUS_INVALID_PARAMETER 无效的参数
jmp     short loc_4A3A9A
```

#### KeInitializeSemaphore

初始化Semaphore对象

```c
Semaphore       = dword ptr  8
Count           = dword ptr  0Ch
Limit           = dword ptr  10h

mov     edi, edi
push    ebp
mov     ebp, esp
mov     eax, [ebp+Semaphore]
mov     ecx, [ebp+Count]
mov     [eax+_KSEMAPHORE.Header.SignalState], ecx
lea     ecx, [eax+_KSEMAPHORE.Header.WaitListHead]
mov     [eax+_KSEMAPHORE.Header.Type], SemaphereObject
mov     [eax+_KSEMAPHORE.Header.Size], 5 //size以32位LONG为单位，所以为sizeof(KSEMAPHORE)/sizeof(ULONG) = 5
mov     [ecx+_LIST_ENTRY.Blink], ecx
mov     [ecx+_LIST_ENTRY.Flink], ecx
mov     ecx, [ebp+Limit]
mov     [eax+_KSEMAPHORE.Limit], ecx // 信号最大的上限值
pop     ebp
retn    0Ch
```

### NtReleaseSemaphore

释放任意个信号量，也就是V操作

```c
push    20h
push    offset stru_41AA68
call    __SEH_prolog
mov     eax, large fs:_KPCR.PrcbData.CurrentThread
mov     al, [eax+_KTHREAD.PreviousMode]
mov     [ebp+AccessMode], al
mov     edi, [ebp+PreviousCount]
xor     ebx, ebx
cmp     edi, ebx
jz      short loc_4AE91C // 等于0，不需要上次增加信号的值
cmp     al, bl
jz      short loc_4AE91C // 判断是否为内核模式
mov     [ebp+ms_exc.registration.TryLevel], ebx
mov     eax, _MmUserProbeAddress
cmp     edi, eax
jnb     loc_53BB9A

loc_4AE914: 
mov     eax, [edi]      // 判断可读可写
mov     [edi], eax
or      [ebp+ms_exc.registration.TryLevel], 0FFFFFFFFh

loc_4AE91C:      
        // NtReleaseSemaphore(x,x,x)+26↑j
cmp     [ebp+ReleaseCount], ebx
jle     loc_53BBA1      // 如果添加的信号个数小于等于0，报异常，参数无效
push    ebx             // HandleInformation
lea     eax, [ebp+Object]
push    eax             // 信号对象
push    dword ptr [ebp+AccessMode] // AccessMode
push    _ExSemaphoreObjectType // ObjectType
push    2               // DesiredAccess
push    [ebp+SemaphoreHandle] // Handle
call    _ObReferenceObjectByHandle@24 // 把句柄转换成信号量对象结构体
mov     [ebp+var_Semaphore], eax
cmp     eax, ebx
jl      short loc_4AE98C // 判断有没有转换成功
mov     [ebp+var_20], ebx
mov     [ebp+ms_exc.registration.TryLevel], 1
push    ebx             // Wait
push    [ebp+ReleaseCount] // Adjustment
push    _ExpSemaphoreBoost // Increment
push    [ebp+Object]    // Semaphore
call    _KeReleaseSemaphore@16 // 释放信号的具体操作
mov     esi, eax
mov     [ebp+var_20], esi
or      [ebp+ms_exc.registration.TryLevel], 0FFFFFFFFh

loc_4AE969:      
mov     ecx, [ebp+Object] // Object
call    @ObfDereferenceObject@4 // 引用计数减1
cmp     [ebp+var_Semaphore], ebx
jl      short loc_4AE98C
cmp     edi, ebx
jz      short loc_4AE98C
cmp     [ebp+AccessMode], bl // 判断模式是否为0环
jz      short loc_4AE997
mov     [ebp+ms_exc.registration.TryLevel], 2
mov     [edi], esi

loc_4AE988:  
or      [ebp+ms_exc.registration.TryLevel], 0FFFFFFFFh

loc_4AE98C:     
mov     eax, [ebp+var_Semaphore]

loc_4AE98F: 
call    __SEH_epilog
retn    0Ch
; ---------------------------------------------------------------------------

loc_4AE997:
mov     [edi], esi
jmp     short loc_4AE98C
```

#### KeReleaseSemaphore

```c
mov     edi, edi
push    ebp
mov     ebp, esp
push    ecx
push    ebx
push    esi
push    edi             // 保存环境，提升堆栈
xor     ecx, ecx
call    ds:__imp_@KeAcquireQueuedSpinLockRaiseToSynch@4 
mov     esi, [ebp+Semaphore]
mov     ebx, [esi+_KSEMAPHORE.Header.SignalState] // 获取初始化信号量
mov     cl, al          // 保存锁的状态
mov     eax, [ebp+ReleaseCount] // 取出增加的信号量个数
lea     edi, [eax+ebx]  // EAX = 增加的信号量
        // EBX = 初始化的信号量
cmp     edi, [esi+_KSEMAPHORE.Limit] // 判断当前初始化的信号量+增加的信号量个数 大于Limit
mov     [ebp+var_Lock], cl
jg      loc_44ACFD      // 释放锁，返回异常
cmp     edi, ebx        // 判断信号量增加以后，有没有溢出
jl      loc_44ACFD      // return STATUS_INVALID_PARAMETER

loc_4120D1:         
test    ebx, ebx        // 判断当前信号量是是否为0
mov     [esi+_KSEMAPHORE.Header.SignalState], edi // 增加了
jnz     short loc_4120E9
lea     eax, [esi+_KSEMAPHORE.Header.WaitListHead]
cmp     [eax+_LIST_ENTRY.Flink], eax // 判断等待链表是否为空
jz      short loc_4120E9 // 如果为空，跳走，切换线程
mov     edx, [ebp+Increment]
mov     ecx, esi
call    @KiWaitTest@8   // 没有信号，但是等待链表不为空

loc_4120E9: 
mov     cl, [ebp+Wait]
test    cl, cl
jnz     loc_44AD11
mov     cl, [ebp+var_Lock]
call    @KiUnlockDispatcherDatabase@4 // KiUnlockDispatcherDatabase(x)

loc_4120FC:  
pop     edi
pop     esi
mov     eax, ebx
pop     ebx
leave
retn    10h
```

## Mutant

Mutant互斥门，可以又用户空间程序通过系统调用创建和使用的称为Mutant；而Mutex则为内核自产自用，不向用户开放

突变体对象：互斥体概念的具体表现，突变体对象不仅有信号状态，如果当前为无信号状态，则一直被某个线程占有。只有所有者线程才能够释放一个突变体对象，否则引发异常

```c
//KMUTANT结构体
typedef struct _KMUTANT {
  DISPATCHER_HEADER Header;
  LIST_ENTRY        MutantListEntry//链表节点
  struct _KTHREAD   *OwnerThread//当前正拥有该突变体对象的线程
  union {
    UCHAR MutantFlags;
    struct {
      UCHAR Abandoned : 1;
      UCHAR Spare1 : 7;
    } DUMMYSTRUCTNAME;
  } DUMMYUNIONNAME;
  UCHAR             ApcDisable//该突变表对象是否已被弃用
} KMUTANT, *PKMUTANT, *PRKMUTANT, KMUTEX, *PKMUTEX, *PRKMUTEX;

```

### KeInitializeMutant

初始化一个KMUTANT对象，并指定当前线程是否为它的所有者

```c
mov     edi, edi
push    ebp
mov     ebp, esp
xor     eax, eax
push    esi
mov     esi, [ebp+Mutant]
inc     eax             // EAX = 1
cmp     [ebp+InitialOwner], al
mov     [esi+_KMUTANT.Header.Type], MutantObject
mov     [esi+_KMUTANT.Header.Size], 8
jz      short loc_423834 // 如果为True，说明没有信号，跳走
    
loc_423834:  
    push    ebx
    push    edi
    mov     eax, large fs:_KPCR.PrcbData.CurrentThread
    and     [esi+_KMUTANT.Header.SignalState], 0
    mov     edi, eax
    xor     ecx, ecx
    mov     [esi+_KMUTANT.OwnerThread], edi // 把当前线程放入互斥体中
    call    ds:__imp_@KeAcquireQueuedSpinLockRaiseToSynch@4
    mov     edi, [edi+_KTHREAD.MutantListHead.Blink]
    mov     ebx, [edi+_LIST_ENTRY.Flink]
    lea     edx, [esi+_KMUTANT.MutantListEntry]
    mov     [edx+_LIST_ENTRY.Flink], ebx
    mov     [edx+_LIST_ENTRY.Blink], edi
    mov     [ebx+_LIST_ENTRY.Blink], edx
    mov     cl, al
    mov     [edi+_LIST_ENTRY.Flink], edx
    call    @KiUnlockDispatcherDatabase@4 // 无论有无信号均切换线程，防止死锁
    pop     edi
    pop     ebx
    jmp     loc_4237DB

and     [esi+_KMUTANT.OwnerThread], 0 // 把互斥体里面的线程清0
mov     [esi+_KMUTANT.Header.SignalState], eax // 如果为False，默认有信号

loc_4237DB:             // CODE XREF: KeInitializeMutant(x,x)+AC↓j
lea     eax, [esi+_KMUTANT.Header.WaitListHead]
mov     [eax+_LIST_ENTRY.Blink], eax // 初始化等待链表
mov     [eax+_LIST_ENTRY.Flink], eax
mov     [esi+_KMUTANT.Abandoned], FALSE // 设置互斥体对象没有被废弃
mov     [esi+_KMUTANT.ApcDisable], 0 // 内核APC是否已开启
pop     esi
pop     ebp
retn    8
```

### KeReleaseMutant

KeInitializeMutant反向操作（V），并防止死锁

```c
mov     edi, edi
push    ebp
mov     ebp, esp
push    ebx
push    esi
push    edi
xor     ecx, ecx
call    ds:__imp_@KeAcquireQueuedSpinLockRaiseToSynch@4 
mov     esi, [ebp+Mutant]
mov     bl, al
mov     eax, [esi+_KMUTANT.Header.SignalState]
mov     [ebp+Mutant], eax
mov     eax, large fs:_KPCR.PrcbData.CurrentThread
cmp     [ebp+Abandoned], 0
mov     edi, eax
jnz     loc_423808      // Abandon = TRUE， 强迫释放对象
cmp     [esi+_KMUTANT.OwnerThread], edi // 是否为当前线程中的互斥体
jnz     loc_4425C8

loc_402B7A:             // CODE XREF: .text:004425E6↓j
inc     [esi+_KMUTANT.Header.SignalState] // 给信号+1

loc_402B7D:           
cmp     [esi+_KMUTANT.Header.SignalState], 1
jnz     short loc_402BB3 // 判断信号是否为1
cmp     [ebp+Mutant], 0
jg      short loc_402BA4 // 判断原来的信号是否大于0
mov     eax, [esi+_KMUTANT.MutantListEntry.Flink] // 移除链表第一个元素
mov     ecx, [esi+_KMUTANT.MutantListEntry.Blink]
mov     [ecx+_LIST_ENTRY.Flink], eax
mov     [eax+_LIST_ENTRY.Blink], ecx
movzx   eax, [esi+_KMUTANT.ApcDisable]
add     [edi+_KTHREAD.KernelApcDisable], eax
jz      loc_40E9F9      // 如果ApcDisable和KernelApcDisable为空，但是链表不为空，把内核APC置1

loc_402BA4:            
and     [esi+_KMUTANT.OwnerThread], 0 // 把互斥体的线程清0
lea     eax, [esi+_KMUTANT.Header.WaitListHead]
cmp     [eax+_LIST_ENTRY.Flink], eax // 比较互斥体链表是否为空
jnz     loc_422997      // 不为空则调用KiWaitTest

loc_402BB3:    
mov     al, [ebp+Wait]
test    al, al
jnz     loc_44AC38
mov     cl, bl
call    @KiUnlockDispatcherDatabase@4 // 调用线程，防止死锁

loc_402BC5:   
mov     eax, [ebp+Mutant]
pop     edi
pop     esi
pop     ebx
pop     ebp
retn    10h
```

## KeSetEvent

```c
typedef enum _EVENT_TYPE{
    NotificationEvent, // 代表通知，是广播式的，通知公众某个事件已发生（Header中的SignalState为1）
    SynchronizationEvent // 用于同步，相当于初值为0，最大值为1的信号量，只有先释放才能继续执行
} EVENT_TYPE;
```

KeSetEvent相当于V操作，把事件对象的SignalState设置为1

```c
mov     edi, edi
push    ebp
mov     ebp, esp
push    ebx
push    esi
push    edi
xor     ecx, ecx
call    ds:__imp_@KeAcquireQueuedSpinLockRaiseToSynch@4 // KeAcquireQueuedSpinLockRaiseToSynch(x)
mov     ecx, [ebp+Event]
mov     edi, [ecx+_KEVENT.Header.SignalState]
mov     bl, al
lea     eax, [ecx+_KEVENT.Header.WaitListHead] // 有无信号
mov     esi, [eax+_LIST_ENTRY.Flink]
cmp     esi, eax        // 判断等待链表是否为空
jnz     short loc_40B0BC
mov     [ecx+_KEVENT.Header.SignalState], 1 // 如果为空，设置信号状态为1

loc_40B0A1:          
mov     cl, [ebp+Wait]
test    cl, cl
jnz     loc_445468
    
loc_445468:                            
    mov     eax, large fs:_KPCR.PrcbData.CurrentThread
    mov     [eax+_KTHREAD.WaitNext], cl
    mov     [eax+_KTHREAD.WaitIrql], bl
    jmp     loc_40B0B3
    
mov     cl, bl
call    @KiUnlockDispatcherDatabase@4 // 线程切换

loc_40B0B3:        
mov     eax, edi
pop     edi
pop     esi
pop     ebx
pop     ebp
retn    0Ch

 // ---------------------------------------------------------------------------
    
loc_40B0BC:          
xor     eax, eax
inc     eax             // EAX = 1
cmp     [ecx+_KEVENT.Header.Type], EventNotificationObject // 判断是否为事件通知类型
jnz     loc_40EDE1      // 处理事件同步
    
loc_40EDE1:
    cmp     [esi+_KWAIT_BLOCK.WaitType], ax // AX = 1 判断是不是任意等待信号
    jnz     loc_40B0C8      // 本身有信号则跳走
    movzx   edx, [esi+_KWAIT_BLOCK.WaitKey]
    mov     ecx, [esi+_KWAIT_BLOCK.Thread]
    push    0
    push    [ebp+Increment]
    call    @KiUnwaitThread@16 // 把线程从等待块中摘掉，设置线程为就绪状态
    jmp     loc_40B0A1

loc_40B0C8:            
test    edi, edi        // 本身有信号则跳走
jnz     short loc_40B0A1 // 不等于0说明有信号，跳转到执行代码
mov     edx, [ebp+Increment]
mov     [ecx+_KEVENT.Header.SignalState], eax
call    @KiWaitTest@8   // KiWaitTest(x,x)
jmp     short loc_40B0A1
```

## KiAcquireSpinLock

自旋锁就是在有一个线程已经进入临界区的时候，多核情况下的别的线程是一直在循环判断进入临界区的条件。没有线程切换，消耗更加的小，但要注意不要出现死锁。 并发访问不高的情况下可以使用自旋锁，如果是多个线程同时访问，会造成CPU的性能的浪费。

![lock前缀的x86指令](/images/image-20220705223256244.png)

```c
lock bts dword ptr [ecx], 0 ; 把数据存到CF里面，置1
jb      short loc_40B450 ; 判断CF是否为1
retn
; ---------------------------------------------------------------------------

loc_40B450: 
test    dword ptr [ecx], 1 ; 判断有没有设置成功
jz      short @KiAcquireSpinLock@4 

pause
jmp     short loc_40B450 ; 判断有没有设置成功
```

## Windbg获取进程的_kthread

`!process 0 0` 找到一个PROCESS ID，这里选择`vmtoolsd.exe`，因为可以看到较多的线程

`dt _kprocess [PROCESS]`列出`_KPROCESS`结构体，找到+50的`ThreadListHead`

![_KPROCESS结构体](/images/image-20220726220305238.png)

`ThreadListHead`的本质就是`_LIST_ENTRY`双向链表，多列几个可以发现Flink就是下一项，Blink是上一项，而第一项的上一项则是`ThreadListHead`地址，最后一项的下一项也是`ThreadListHead`的地址，即`_kprocess+0x50`

![List_Entry](/images/image-20220726220418701.png)

![链表最后一项的下一项和ThreadListEntry地址比较](/images/image-20220726220445127.png)

由`_kthread`结构体可以得知，在0x1b0处同样有一个`ThreadListEntry`，而此`ThreadListEntry`便是上面得出的地址，由此可以计算出`_kthread`的地址为`[_LIST_ENTRY] - 0x1b0`

![_kthread](/images/image-20220726221430972.png)

同样也可查看以下WaitBlockList和WaitListEntry

![WaitBlockList和WaitListEntry](/images/image-20220726224527424.png)



