# windows SEH分析

与linux不同，windows的函数调用栈中存储了不止栈底指针（saved ebp）以及返回地址、局部变量、canary这几样，windows在栈中存放了许多的私货，这其中就包括了seh。

![igor1_seh3_stack_layout](https://s3cunda.github.io/assets/post/igor1_seh3_stack_layout.gif)

### SEH

什么是SEH？全称就是Structure Exception Handler，也就是结构化异常处理。

那么这个SEH，是干什么的呢？

> `SEH`是`Windows`操作系统上 对 C/C++ 程序语言做的语法拓展，用于处理异常事件的程序控制结构。异常事件是指打断程序正常执行流程的不在期望之中的硬件、软件事件。硬件异常是`CPU`抛出的如 除0、数值溢出等；软件异常是操作系统与程序通过 `RaiseException`语句抛出的异常。`Windows`拓展了C语言的语法，用 `try-except`与 `try-finally` 语句来处理异常。异常处理程序可以释放已经获取的资源、显示出错信息与程序内部状态供调试、从错误中恢复、尝试重新执行出错的代码或者关闭程序等。一个 `__try` 语句不能既有 `__except`，又有 `__finally`。但 `try-except`与 `try-finally`语句可以嵌套使用。

简而言之言而总之，SEH就是为了处理一些异常而出现的，那么这个SEH存储在哪里呢？

在了解SEH存储在哪里之前，我们先了解两个概念：TEB和TIB

TEB是线程环境块。是操作系统为了保存每个现成的私有数据创建的。

TIB是线程信息块，是保存线程基本信息的数据结构。

TEB长这个样子：

```c
nt!_TEB
   +0x000 NtTib            : _NT_TIB
   +0x01c EnvironmentPointer : Ptr32 Void
   +0x020 ClientId         : _CLIENT_ID
   +0x028 ActiveRpcHandle  : Ptr32 Void
   +0x02c ThreadLocalStoragePointer : Ptr32 Void
   +0x030 ProcessEnvironmentBlock : Ptr32 _PEB
   +0x034 LastErrorValue   : Uint4B
   +0x038 CountOfOwnedCriticalSections : Uint4B
   +0x03c CsrClientThread  : Ptr32 Void
   +0x040 Win32ThreadInfo  : Ptr32 Void
   +0x044 User32Reserved   : [26] Uint4B
   +0x0ac UserReserved     : [5] Uint4B
   +0x0c0 WOW32Reserved    : Ptr32 Void
   +0x0c4 CurrentLocale    : Uint4B
   +0x0c8 FpSoftwareStatusRegister : Uint4B
   +0x0cc SystemReserved1  : [54] Ptr32 Void
   +0x1a4 ExceptionCode    : Int4B
   +0x1a8 ActivationContextStack : _ACTIVATION_CONTEXT_STACK
   +0x1bc SpareBytes1      : [24] UChar
   +0x1d4 GdiTebBatch      : _GDI_TEB_BATCH
   +0x6b4 RealClientId     : _CLIENT_ID
   +0x6bc GdiCachedProcessHandle : Ptr32 Void
   +0x6c0 GdiClientPID     : Uint4B
   +0x6c4 GdiClientTID     : Uint4B
   +0x6c8 GdiThreadLocalInfo : Ptr32 Void
   +0x6cc Win32ClientInfo  : [62] Uint4B
   +0x7c4 glDispatchTable  : [233] Ptr32 Void
   +0xb68 glReserved1      : [29] Uint4B
   +0xbdc glReserved2      : Ptr32 Void
   +0xbe0 glSectionInfo    : Ptr32 Void
   +0xbe4 glSection        : Ptr32 Void
   +0xbe8 glTable          : Ptr32 Void
   +0xbec glCurrentRC      : Ptr32 Void
   +0xbf0 glContext        : Ptr32 Void
   +0xbf4 LastStatusValue  : Uint4B
   +0xbf8 StaticUnicodeString : _UNICODE_STRING
   +0xc00 StaticUnicodeBuffer : [261] Uint2B
   +0xe0c DeallocationStack : Ptr32 Void
   +0xe10 TlsSlots         : [64] Ptr32 Void
   +0xf10 TlsLinks         : _LIST_ENTRY
   +0xf18 Vdm              : Ptr32 Void
   +0xf1c ReservedForNtRpc : Ptr32 Void
   +0xf20 DbgSsReserved    : [2] Ptr32 Void
   +0xf28 HardErrorsAreDisabled : Uint4B
   +0xf2c Instrumentation  : [16] Ptr32 Void
   +0xf6c WinSockData      : Ptr32 Void
   +0xf70 GdiBatchCount    : Uint4B
   +0xf74 InDbgPrint       : UChar
   +0xf75 FreeStackOnTermination : UChar
   +0xf76 HasFiberData     : UChar
   +0xf77 IdealProcessor   : UChar
   +0xf78 Spare3           : Uint4B
   +0xf7c ReservedForPerf  : Ptr32 Void
   +0xf80 ReservedForOle   : Ptr32 Void
   +0xf84 WaitingOnLoaderLock : Uint4B
   +0xf88 Wx86Thread       : _Wx86ThreadState
   +0xf94 TlsExpansionSlots : Ptr32 Ptr32 Void
   +0xf98 ImpersonationLocale : Uint4B
   +0xf9c IsImpersonating  : Uint4B
   +0xfa0 NlsCache         : Ptr32 Void
   +0xfa4 pShimData        : Ptr32 Void
   +0xfa8 HeapVirtualAffinity : Uint4B
   +0xfac CurrentTransactionHandle : Ptr32 Void
   +0xfb0 ActiveFrame      : Ptr32 _TEB_ACTIVE_FRAME
   +0xfb4 SafeThunkCall    : UChar
   +0xfb5 BooleanSpare     : [3] UChar
```

TIB长这个样子：

```c
// Code in https://source.winehq.org/source/include/winnt.h#2635

typedef struct _NT_TIB{
    struct _EXCEPTION_REGISTRATION_RECORD *Exceptionlist; // 指向当前线程的 SEH
    PVOID StackBase;    // 当前线程所使用的栈的栈底
    PVOID StackLimit;   // 当前线程所使用的栈的栈顶
    PVOID SubSystemTib; // 子系统
    union {
        PVOID FiberData;
        ULONG Version;
    };
    PVOID ArbitraryUserPointer;
    struct _NT_TIB *Self; //指向TIB结构自身
} NT_TIB;
```

可以看到，其中的_EXCEPTION_REGISTRATION_RECORD *Exceptionlist就是指向当前线程的SEH的指针。

那么这个_EXCEPTION_REGISTRATION_RECORD就是SEH的结构体,具体来说长这个样子

```c
//  Code in https://source.winehq.org/source/include/winnt.h#2623

typedef struct _EXCEPTION_REGISTRATION_RECORD{
    struct _EXCEPTION_REGISTRATION_RECORD *Next; // 指向下一个结构的指针
    PEXCEPTION_ROUTINE Handler; // 当前异常处理回调函数的地址
}EXCEPTION_REGISTRATION_RECORD;
```

可以看到，是一个很经典的单链表结构。

其中，TEB存放于fs段开头位置，fs[0]即为TIB，TIB第一个字段就保存了SEH链表的头部指针。而SEH链表中其他的节点存储在栈中。

![SEH链表](https://s3cunda.github.io/assets/post/SEH链表.png)

### 当异常发生后，程序都干了什么？

当程序发生异常后，工作流程如下：

> 1. 产生硬件异常通过 `IDT`调用异常处理例程， 产生软件异常通过 `API`的层层调用产地异常信息。而异常又由于发生位置不同，分为内核异常和用户态异常，二者最后都会靠 `kiDispathException`函数来进行异常分发；
> 2. 当内核产生异常时，程序处理流程进入到 `KiDispatchException` 函数，在该函数内备份当前线程 `R3` 的 `TrapFrame`（即栈帧的基址）。异常处理首先判断这是否是第一次异常，判断是否存在内核调试器，如果有内核调试器，则把当前的异常信息发送给内核调试器；如果没有内核调试器或者内核调试器没有处该异常 ， 则进入**步骤3**，调用 `RtlDispatchException`。
> 3. 内核异常进入 `RtlDispatchException` 函 数， 如果`RtlDispatchException` 函数没有处理该异常，那么将再次尝试将异常发送到内核调试器，如果此时内核调试器仍然不存在或者没有处理该异常，那么此时系统会直接蓝屏；
> 4. 如果是用户态异常则经过 `KiDispatchException`进行用户态异常分发和处理。如果是第一次分发异常，则调用 `DbgKForwardException`将异常分发到内核调试器；如果内核调试器不存在或没有处理异常，则尝试将异常分发给用户态调试器；如果异常被处理，则进入**步骤10**；如果用户态调试器不存在或未处理异常，则检测是否是第一次处理异常，如果是第一次处理异常则进入**第5步**中的异常数据准备；
> 5. 准备一个返回`ntdll!KiUserExceptionDispatcher` 函数的应用层调用栈，结束本次`KiDispatchException` 函数的运行，调用`KiServiceExit` 返回用户层。此时函数栈帧是`ntdll!KiUserExceptionDispatcher`的执行环境，用户态线程从执行 `ntdll!KiUserExceptionDispatcher` 开始执行。该函数调用 `ntdll!RtlDispatchException`进行异常的分发，进入**第 6 步**；
> 6. 通过 `RtlCallVectoredExceptionHandlers`遍历 `VEH`链表尝试查找异常处理函数；如果 `VEH`未处理异常。则从 `fs[0]`读取 `ExceptionList`并开始执行 `SEH` 函数处理，进入**步骤7**；
> 7. 如果`SEH`没有处理函数处理该异常，则检查用户是否通过`SetUnhandledExceptionFilter`函数注册过进程的异常处理函数，如果用户注册过异常处理函数，调用该异常处理函数，如果异常没有被成功处理或没有自定义的异常处理函数，则进入**步骤3**；
> 8. 如果最后仍没有处理该异常，便会主动调用 `NtRaiseException`将该异常重新跑出来，但是此时不是第一次分发，此时 `NtRaiseException`流程重新调用了 `ntdll!KiDispatchException`，并再次进入用户态异常的处理分支，进入**步骤9**；
> 9. 第二次进入用户态异常处理时，不会再尝试发送到内核调试器，也不会再进行异常分发，而是直接尝试发送到用户态体异常调试器，如果最后异常仍未被处理则进入**步骤11**；
> 10. 异常被处理，调用 `NtContine`，将之前保存的 `TrapFrame`还原，程序继续从异常处正常运行；
> 11. 异常不能被处理，系统调用 `ntdll!KiDispatchException` 调用 `ZeTerminateProcess`结束进程。

也就是说，异常发生后，大概顺序是：内核->调试器->VEH->SEH

这里我们不关注内核和调试器，这个VEH又是个啥呢？

```
Vectored Exception Handling was introduced in Windows XP.[7] Vectored Exception Handling is made available to Windows programmers using languages such as C++ and Visual Basic. VEH does not replace Structured Exception Handling (SEH), rather VEH and SEH coexist, with VEH handlers having priority over SEH handlers.[1][7] Compared with SEH, VEH works more like kernel-delivered Unix signals.[8](wikipedia)
```

主要大体内容就是说VEH与SEH共存，且VEH优先级比SEH高。

我们可以注册多个VEH，VEH之间通过双向链表链接，所以相对于SEH，VEH可以指定位置。同时VEH保存在堆中。

当异常发生的时候，系统将遍历VEH链表，尝试处理异常。

###  SEH工作原理

讲了那么多别的，最终我们还是要回到最主要的问题上，SEH是个啥，他怎么工作的呢？

在线程初始化的时候，会自动在栈中安装一个SEH结构体，作为默认异常处理，他的next就是0xFFFFFFFFF，而这个异常程序大家应该都很熟悉，就是windows程序崩溃时那个弹窗，打印出来出错函数地址。

如果程序中使用了try、excpt、assert来处理异常信息，那么编译器就会在栈中压入一个SEH结构体，同时插入链表中。

当出现异常的时候，操作系统会先中断程序，然后从TIB中取出第一个SEH结构体（也就是最近的SEH结构），使用其中的handler处理这个异常。

如果这个异常处理函数处理不了这个异常，那么就顺着next往上找别的异常处理函数，直到找到一个可以处理这个异常的函数或者到底部，也就是弹出错误窗口然后杀死线程。

通常处理完异常后，需要执行展开（Unwind）操作，该操作先通知目标结点前的各异常处理函数释放资源，然后将之前的SEH链全部删除。该操作通常由各高级语言Rtl模块来完成，Win32汇编操作时既可以不展开，也可以手工展开，还可以使用RtlUnwind函数展开。

### unwind

当一个函数注册一个SEH的时候，通常都会干这些事：

```
push    一堆附加数据
push   offset _Handler 
push   fs:[0]        //next
mov    fs:[0],esp		//make head -> new seh
```

当触发异常调用SEH机制时，每个异常函数都需要四个重要的参数：

1. pExcept：指向EXCEPTION_RECORD的结构体的指针，其中包含了异常相关信息，如地址、异常类型等。
2. pFrame：指向栈中的SEH结构体
3. pContext：指向context结构体，包含了所有寄存器状态信息。
4. pDispatch：不知道干嘛的

在执行处理函数前，系统会将上述参数压栈，然后调用异常处理函数。

异常处理函数结束时有两个返回值：0代表处理成功，返回原来程序发生异常的地方，继续执行。

1代表失败，那么就继续顺着SEH链表往后找可以处理这个异常的函数。

当系统找到了可以处理异常的函数后，系统会将已经遍历过的异常处理函数在调用一边，这个过程就是unwind操作。

其目的就是通知前面失败的SEH，系统已经处理完了异常了，要把你们都搞掉，清理现场打包走人，然后将前面失败的SEH从链表里面删除。

那么为什么需要unwind操作呢？

如果说程序通过层层的调用在SEH链表中找到了一个可以成功处理的handler，那么这时异常被处理成功返回，此时如果直接根据context恢复现场，会涉及到许多压栈操作，那么这些压栈操作就会破坏原来的SEH链表信息，fs[0]指向一个错误地址，程序将发生异常。

具体unwind做了什么呢？有兴趣的可以参考下这篇文章：https://blog.csdn.net/LPWSTR/article/details/78714486?spm=1001.2014.3001.5501

### safe SEH

既然SEH存储在栈上，那么我们可以通过栈溢出修改SEH handler函数指针为shellcode地址，然后触发异常，函数进入SEH handler，就可以执行shellcode了。

为了针对这一种攻击手法，就有了safe SEH保护措施，那么safe SEH都做了哪些检查呢？

1. 检查异常处理链是否存在于当前程序栈中，如果不是，就终止异常处理函数调用。
2. 检查异常处理函数指针是否指向栈中，如果指向，终止异常处理函数调用。
3. 前面两个都通过后，调用新的函数RtlIsValidHandler，对异常处理函数做一个有效性验证。

那么这个函数又做了哪些检查呢？

1. 判断程序设置IMAGE_DLLCHARACTERISTICS_NO_SEH标识。设置了，异常就忽略，函数返回校验失败。
2. 检测程序是否包含SEH表。如果包含，则将当前异常处理函数地址与该表进行匹配，匹配成功返回校验成功，否则失败。
3. 判断 程序是否设置ILonly标识。设置了，标识程序只包含.NET编译人中间语言，函数直接返回校验失败
4. 判断异常处理函数是否位于不可执行页（non-executable page）上。若位于，校验函数将检测DEP是否开启，如若系统未开启DEP则返回校验成功；否则程序抛出访问违例的异常

如果异常处理函数的地址没有包含在加载模块的内存空间。校验函数将直接执行DEP相关检测，函数将依次进行如下检验：

1. 判断异常处理函数是否位于不可执行页（non-executable page）上。若位于，校验函数将检测DEP是否开启，如若系统未开启DEP则返回校验成功；否则程序抛出访问违例的异常
2. 判断系统是否允许跳转到加载模块的内存空间外执行，如允许则返回校验成功；否则返回校验失败

其伪代码如下：

```c
BOOL RtlIsValidHandler(handler)
{
    if (handler is in image){    //在加载模块内存空间内
        if (image has the IMAGE_DLLCHARACTERISTICS_NO_SEH flag ser)
            return FALSE;
        if (image has a SafeSEH table)     //含有安全SEH表，说明程序启用SafeSEH
            if (handler found in the table)    // 异常处理函数地址出现在安全SEH表中
                return TRUE;
            else    // 异常处理函数未出现在安全SEH表中
                return FALSE;
        if (image is a .NET assembly with the ILonly flag set)     //只包含IL
            return FALSE;
    }
    if (handler is on a non-executable page){    // 跑到不可执行页上
        if (ExecuteDispatchEnable bit set in the process flags)    //DEP关闭
            return TRUE;
        else
            raise ACESS_VIOLATION; //抛出访问违例异常
    }
    if (handler is not in an image){    // 在加载模块内存之外，并且在可执行页上
        if (ImageDispatchEnable bit set in the process flags)    // 允许在加载模块内存空间外执行
            return TRUE;
        else
            return FALSE;
    }
    return TRUE;    //前面所有条件都满足就允许这个异常处理函数执行
}
```

那么，如果我们想绕过safe SEH来攻击SEH的话，如何绕过呢？

首先前两点很好解决，修复SEH的next的指针，然后不把shellcode指向栈上即可。

那么后面的RtlIsValidHandler函数怎么办？

我们从这个函数逻辑分析，看一下什么情况才能允许我们执行SEH处理函数：

1. 异常处理函数位于加载模块内存范围之外，DEP关闭
2. 异常处理函数位于加载模块内存范围之内，相应模块未启用SafeSEH(安全SEH表为空)，同时相应模块不是纯IL
3. 异常处理函数位于加载模块范围之内，相应模块启用SafeSEH（安全SEH表不为空），异常处理函数地址包含在安全SEH表中

其中的DEP就是类似于linux中的NX，即堆栈数据段不可执行。

第一种情况还是比较简单的，在模块外的地址空间写shellcode或者找一个跳板跳到shellcode即可。

第二种情况，可以利用未开启safe SEH的模块中找到一条跳转指令跳到shellcode。

第三种情况有两种方式，一是清空SEH表，欺骗系统未开启safeSEH，二是将我们的指令注册到SEH表中（难度比较大）。

除了以上三种方式，有更为简单的攻击手法：

1.不攻击SEH

2.如果SEH异常处理函数指向堆区域，及时安全校验发现SEH已经不可信，仍然会调用其已经被修改的异常处理函数，所以只需要将shellcode搞到堆即可绕过。

### SEHOP

世界上没有什么事情是套娃解决不了的，如果有，那就再加一层套娃。

针对于SEH攻击，SEHOP（SEH Overwrite Protection）横空出世。

SEHOP主要任务就是来检测SEH链表的完整性，在调用handler之前系统会先遍历链表，看一下最后一个节点是否为系统固定的最终处理函数，如果是，那么皆大欢喜；不是的话，那么不进行异常处理，程序退出。

```c
if (process_flags & 0x40 == 0)  // 如果没有SEH记录则不进行检测
{
    if (record != 0xFFFFFFFF)  // 开始检测
    {
        do
        {
            if (record < stack_bottom || record > stack_top) // SEH 记录必须位于栈中
                goto corruption;
            if ((char *)record + sizeof(EXCEPTION_REGISTRATION) > stack_top) // SEH 记录结构需完全在栈中
                goto corruption;
            if ((record & 3) != 0) // SEH记录必须4字节对齐
                goto corruption;
            handler = record->handler;
            if (handler >= stack_bottom && handler < stack_top) // 异常处理函数地址不能位于栈中
                goto corruption;
            record = record->next;
        } while (record != 0xFFFFFFFF); // 遍历S.E.H链
    }
    if ((TEB->word_at_offset_0xFCA & 0x200) != 0)
    {
        if (handler != &FinalExceptionHandler) // 核心检测，地球人都知道，不解释了
        goto corruption;
    }
}
```

所以相应的绕过方法就是伪造一个SEH链，修复SEH链完整性。

### SEH scopetable

scopetable指向了一个用于描述函数中所有__try代码块的数组。在SEH4中，scopetable是一个被加密过后的scopetable的地址（xor cookie）

filterfunc指向异常过滤函数（__except中的表达式），handlerfunc指向except代码块。

如果filterdunc是NULL，那么Handlerfunc就指向__finally代码块。

具体有多少个try，体现在trylevel中。

```c
struct _EH4_SCOPETABLE {
        DWORD GSCookieOffset;
        DWORD GSCookieXOROffset;
        DWORD EHCookieOffset;
        DWORD EHCookieXOROffset;
        _EH4_SCOPETABLE_RECORD ScopeRecord[1];
};

struct _EH4_SCOPETABLE_RECORD {
        DWORD EnclosingLevel;
        long (*FilterFunc)();
            union {
            void (*HandlerAddress)();
            void (*FinallyFunc)(); 
    };
};
```

![windows栈帧](https://s3cunda.github.io/assets/post/windows栈帧.jpg)

在函数开始时，回先保存上个函数的ebp，然后将try level、加密后的scope table、sehhandler、seh next、异常指针、esp指针以及gs压栈，gs就是类似于canary（security cookie xor ebp）的东西,。

scopetable加密的方式就是异或一下securitycookie。

针对 `__except_handler`函数，如果我们伪造一个 `scope table`，把里面的 `FilterFunc`或者 `FinallyFunc`改为 `system('cmd')`的地址，然后把这个伪造的 `scope table`通过溢出覆盖掉原 `scope table`，就能够`getshell`。

当然由于 栈中存储的 `scope table`地址是 `_EH4_SCOPETABLE_addr ^ _security_cookie`得来，所以我们也得知道 `__security_cookie`的实际值。同时覆盖时，也不可避免覆盖掉 `GS Cookie`，`next SEH` 和 `except_handler`，但也必须保证这三个值的正确性。

### 参考链接

1. https://a1ex.online/2020/10/15/Windows-Pwn学习/
2. https://bbs.pediy.com/thread-189297.htm
3. http://www.openrce.org/articles/full_view/21
4. https://blog.csdn.net/LPWSTR/article/details/78711887
5. https://blog.csdn.net/LPWSTR/article/details/78714486?spm=1001.2014.3001.5501
6. https://bbs.pediy.com/thread-173853.htm
7. https://blog.csdn.net/qq_18218335/article/details/70543671
8. https://www.hexblog.com/wp-content/uploads/2012/06/Recon-2012-Skochinsky-Compiler-Internals.pdf