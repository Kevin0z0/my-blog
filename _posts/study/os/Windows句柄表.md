---
title: 'Windows句柄表'
author: Kevin。
tags:
  - os
  - Windows
categories:
  - study
date: 2022-10-04 14:53:00
img: /images/article-banner/QQ截图20221005234010.jpg
typora-root-url: ..\..\..\
---

# Windows句柄表

## 句柄表介绍

句柄表分为全局句柄表和进程句柄表，Windows用句柄来管理进程中的对象引用，进程句柄表仅在一个进程范围内才有效，若将该进程中的一个句柄传递给另一个进程后，该句柄不再有效。而全局句柄表在操作系统全局都有效，在windbg中可用`dd pspcidtable`列出全局句柄表，其中`pspcidtable`指向`_handle_table` 

`_handle_table_entry` 按4KB来申请内存，一个结构体大小为8B，每申请一个新的页面来存放句柄项，句柄表的容量增加512个。

```c
struct _HANDLE_TABLE
{
    ULONG TableCode;                                 //指向句柄表的储存结构                   
    struct _EPROCESS* QuotaProcess;                  //句柄表的内存资源记录
    VOID* UniqueProcessId;                           //创建进程的ID，用于回调函数
    struct _EX_PUSH_LOCK HandleLock;                 //句柄表锁，仅在句柄表扩展时使用
    struct _LIST_ENTRY HandleTableList;              //所有句柄表形成一个链表
    struct _EX_PUSH_LOCK HandleContentionEvent;      //若在访问句柄时发生竞争，则在此推锁上等待
    struct _HANDLE_TRACE_DEBUG_INFO* DebugInfo;      //调试信息，仅当调试句柄时才有意义
    LONG ExtraInfoPages;                             //审计信息所占用的页面数量
    union 
    { 
        ULONG Flags;                                 //标志域
        UCHAR StrictFIFO:1;                          //
    }; 
    ULONG FirstFreeHandle;                           //空闲链表表头的句柄索引，用于FIFO类型的空闲链表
    ULONG LastFreeHandleEntry;                       //最近被释放的句柄索引，用于FIFO类型空闲链表
    ULONG NextHandleNeedingPool;                     //下一次句柄表扩展的起始句柄索引
    ULONG HandleCount;                               //正在使用的句柄表项数量
    ULONG HandleCountHighWatermark;                  //
};

struct _HANDLE_TABLE_ENTRY
{
    union
    {
        VOID* Object;                               //指向句柄所代表的对象
        ULONG ObAttributes;                         //最低三位有特别含义
        										//OBJ_HANDLE_ATTRIBUTES宏定义
        struct _HANDLE_TABLE_ENTRY_INFO* InfoTable; //各个句柄的第一个表项
        										//使用此成员指向第一张表
        ULONG Value;                                //
    };
    union
    {
        ULONG GrantedAccess;                        //
        struct
        {
            USHORT GrantedAccessIndex;              //
            USHORT CreatorBackTraceIndex;           //
        };
        LONG NextFreeTableEntry;                    //
    };
}; 
```

TableCode域是一个指针，指向句柄表的最高层表项页面，低2位为当前句柄表的层数，如果为0，说明句柄表只有一层，能容纳512个句柄，如果为1，则有2层，能容纳512 x 1024个句柄表，以此类推，最多不可超过2^24=16777216个句柄，而每个句柄表的第一项都有特殊作用，真正可供进程使用的是511个。

![不同层的句柄表结构](/images/image-20221004185123226.png)

windbg中`dq (TableCode & 0xfffffffc)`显示TableCode下的句柄表，里每一项都有8字节，前4字节代表句柄值`（dq 地址）`，后4字节代表对象，由于是8位对其，所以对象的低三位清零

## 获取进程句柄表

`!process 0 0`随意找一个PROCESS ID

`dt _eprocess 89954b28`根据PROCESS ID列出进程结构体

![找到0xc4位置的ObjectTable](/images/image-20221004204253602.png)

`dt 0xe21dc8e0 _HANDLE_TABLE` 列出句柄表

![句柄表结构体](/images/image-20221004204439091.png)

`dq 0xe250b000` 句柄对象

![image-20221004204500922](/images/image-20221004204500922.png)

## 根据进程PID获取句柄对象

这里选择`vmtoolsd`进程为例，`pid`为`0x258`，计算方式为`pid / 4 / 512 = 0x258 / 4 / 512 = 0 ...0x96`

![vmtoolsd进程](/images/image-20221004211102264.png)

重复以上的步骤，在最后加上`计算出来的结果 x 8`，低3位清零就是进程_eprocess对象地址

![获取句柄对象](/images/image-20221004211332323.png)



## PsLookupProcessByProcessId分析

### 主函数

```assembly
004AA687    mov     edi, edi
004AA689    push    ebp
004AA68A    mov     ebp, esp
004AA68C    push    ebx
004AA68D    push    esi
004AA68E    mov     eax, large fs:_KPCR.PrcbData.CurrentThread
004AA694    push    [ebp+ProcessId]
004AA697    mov     esi, eax
004AA699    dec     [esi+_KTHREAD.KernelApcDisable] ; 内核APC-1
004AA69F    push    _PspCidTable    ; 全局句柄表指针-->HandleTable
004AA6A5    call    _ExMapHandleToPointer@8 ; ExMapHandleToPointer(x,x)
004AA6AA    mov     ebx, eax        ; 返回指针
004AA6AC    test    ebx, ebx        ; 判断返回指针是否为空
004AA6AE    mov     [ebp+ProcessId], 0C000000Dh
004AA6B5    jz      short loc_4AA6E9 ; 内核APC加1
004AA6B7    push    edi
004AA6B8    mov     edi, [ebx]
004AA6BA    cmp     byte ptr [edi], ProcessObject ; 判断是否为进程对象
004AA6BD    jnz     short loc_4AA6DC
004AA6BF    cmp     [edi+_EPROCESS.GrantedAccess], 0
004AA6C6    jz      short loc_4AA6DC
004AA6C8    mov     ecx, edi
004AA6CA    call    @ObReferenceObjectSafe@4 ; ObReferenceObjectSafe(x)
004AA6CF    test    al, al
004AA6D1    jz      short loc_4AA6DC
004AA6D3    mov     eax, [ebp+Process]
004AA6D6    and     [ebp+ProcessId], 0
004AA6DA    mov     [eax], edi
004AA6DC    push    ebx             ; BugCheckParameter2
004AA6DD    push    _PspCidTable    
004AA6E3    call    _ExUnlockHandleTableEntry@8 ; 释放锁
004AA6E8    pop     edi
004AA6E9    inc     [esi+_KTHREAD.KernelApcDisable] ; 内核APC加1
004AA6EF    jnz     short loc_4AA6FC
004AA6F1    lea     eax, [esi+_KTHREAD.ApcState]
004AA6F4    cmp     [eax+_KAPC_STATE.ApcListHead.Flink], eax
004AA6F6    jnz     loc_52E742
004AA6FC    mov     eax, [ebp+ProcessId]
004AA6FF    pop     esi
004AA700    pop     ebx
004AA701    pop     ebp
004AA702    retn    8
```

### _ExMapHandleToPointer

```assembly
00497FDD     mov     edi, edi
00497FDF     push    ebp
00497FE0     mov     ebp, esp
00497FE2     push    edi
00497FE3     mov     edi, [ebp+handle]
00497FE6     test    di, 7FCh
00497FEB     jz      loc_4AECDF
00497FF1     push    ebx
00497FF2     mov     ebx, [ebp+HandleTable]
00497FF5     push    esi
00497FF6     push    edi
00497FF7     push    ebx
00497FF8     call    _ExpLookupHandleTableEntry@8 ; 进程句柄和全局句柄都会进这里
00497FFD     mov     esi, eax
00497FFF     test    esi, esi
00498001     jz      loc_505D68
00498007 loc_498007:                             
00498007     mov     edx, [esi]      ; 取出地址
00498009     test    dl, 1           ; 判断最后一位是否为1
0049800C     jz      loc_50F645
00498012     lea     ecx, [edx-1]    ; 获取句柄
00498015     mov     eax, edx
00498017     lock cmpxchg [esi], ecx ; 减一交换
0049801B     cmp     eax, edx
0049801D     jnz     loc_50F64D
00498023     mov     eax, esi
00498025 loc_498025:                         
00498025     pop     esi
00498026     pop     ebx
00498027 loc_498027:            
00498027     pop     edi
00498028     pop     ebp
00498029     retn    8
```

### _ExpLookupHandleTableEntry

计算句柄表地址

```assembly
004954C9     mov     edi, edi
004954CB     push    ebp
004954CC     mov     ebp, esp
004954CE     and     [ebp+handle], 0FFFFFFFCh ; 低三位清零
004954D2     mov     eax, [ebp+handle]
004954D5     mov     ecx, [ebp+HandleTable]
004954D8     mov     edx, [ebp+handle]
004954DB     shr     eax, 2          ; handle / 4
004954DE     cmp     edx, [ecx+_HANDLE_TABLE.NextHandleNeedingPool] ; 下一个句柄的起始索引
004954E1     jnb     loc_49CCBC      ; 如果大于等于，句柄清0，退出
004954E7     push    esi
004954E8     mov     esi, [ecx+_HANDLE_TABLE.TableCode]
004954EA     mov     ecx, esi
004954EC     and     ecx, 3          ; 低两位，代表句柄表的层数
004954EF     and     esi, 0FFFFFFFCh ; 清空低两位
004954F2     sub     ecx, 0          ; 判断是不是第一层
004954F5     jnz     loc_497E86      ; 第二层
004954FB     lea     eax, [esi+eax*8] ; *(dword*)(TableCode低两位清0 + (PID/4) * 8) (PID / 4 = eax)
004954FE loc_4954FE:                            
004954FE     pop     esi
004954FF loc_4954FF:  
004954FF     pop     ebp
00495500     retn    8
```

### loc_497E86

第二层算法

```assembly
00497E86 loc_497E86:                           
00497E86     dec     ecx             ; 第二层减去1
00497E87     mov     ecx, eax        ; eax = handle / 4
00497E89     jnz     loc_53A3F2      ; 第三层 (根据dec ecx计算得出的flags跳转)
00497E8F     shr     ecx, 9          ; ecx / 512
00497E92     mov     ecx, [esi+ecx*4]
00497E95 loc_497E95:
00497E95     and     eax, 1FFh       ; 第0项做特殊用途，(handle / 4) & 512 计算页目录索引
00497E9A     lea     eax, [ecx+eax*8]
00497E9D     jmp     loc_4954FE
```

### loc_53A3F2

第三层算法

```assembly
0053A3F2     shr     ecx, 13h        ; 1024 * 512 = 2^0x13
0053A3F5     mov     edx, ecx
0053A3F7     mov     ecx, [esi + ecx * 4] ; TableCode 低两位清零 + ecx * 4 求出
0053A3FA     shl     edx, 13h        ; 填充为0
0053A3FD     sub     eax, edx        ; 求出目录2的索引
0053A3FF     mov     edx, eax
0053A401     shr     edx, 9
0053A404     mov     ecx, [ecx+edx*4] ; 求出目录2
0053A407     jmp     loc_497E95
```

### 计算句柄表步骤

index = handle >> 2

目录1 = TableCode + index * 8



index2 = index >> 9

目录2 = 目录1 + 目录2索引 * 4

Object = 目录2 + (index2 & 0x1FF) * 8



index3 = index >> 13

目录3 = TableCode + index3 * 4 + ((index - (index3 << 13)) >> 9) * 4 + 0x1FF * 8

## 包含全局句柄表的函数

PsLookupProcessThreadByCid

PsLookupProcessByProcessId

mmGetSystemRoutineAddress

ExpLookupHandleTableEntry







