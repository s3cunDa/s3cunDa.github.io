# 控制流完整性

针对于漏洞利用，最终的效果和目的就是劫持控制流，控制目标程序做一些他本来做不了的事情。可以达到这一目的的方式有很多，比如ROP、劫持函数指针等等。而这些都来自于软件中一些漏洞，如缓冲区溢出、释放后利用等等。最初防御的方式就是头疼医头，脚疼医脚，哪里出现了漏洞比如缓冲区溢出，我们就检查一下内存边界，或者在边界处设置一个cookie（canary）。

或许是漏洞多的补不过来，之前的防御方式不能很好的完成防御计算机被破坏的工作，原本的防御方式经过几轮较量后衍生出了很多绕过方式，这些攻击手法就是现代漏洞利用技术的核心，比如ROP。如果攻击者通过层层阻挠，到达了执行ROP这一步，那么后续的路基本就畅通无阻了，因为之前并没有防御ROP的有效方式。

CFI即Control Flow Integrity控制流完整性就是指程序运行时控制流的合法性。这一步概念被提出来主要就是为了针对ROP的防御。可以将程序运行看作是一辆车在路上跑，开发者遵循的安全开发准则，比如说严格控制好边界等可以看作是司机在路上遵守交通规则；而之前的防御如canary等内存边界检查机制可以理解为马路边上的防护栅栏；而CFI验证可以看作是车内的安全气囊、安全带等装置。

那么这个CFI验证具体干什么呢不管一个程序有多复杂，他所能覆盖到的代码分枝路线以及行为虽然很多，但是不是无限的，他的活动范围总会有一个边界。如果一个攻击者通过程序中的漏洞控制了这个程序，那么攻击者肯定不会满足于程序本身给提供的代码分枝进行执行，总会超过这一边界，去执行一些程序中本来没有的逻辑。

CFI验证顾名思义，就是确保程序在预期的范围内执行。针对于这一思路，目前已经有很多的实现方式。

### windows cfg

cfg全称就是Control Flow Guard，即控制流保护。其主要思路就是在间接跳转前后插入一段代码，用于验证其有效性。

为什么是间接跳转呢？因为直接跳转写死在代码段，攻击者利用不了。

如何验证其有效性呢？在编译时会记录各个间接跳转函数的地址，生成一个白名单，在函数发生间接跳转时就会对照这一白名单,如果在白名单里面，皆大欢喜，不在的话那就抛出异常。

那么具体怎么做的呢？这里偷个懒，引用下其他前辈的文章：

https://xz.aliyun.com/t/2587

https://www.anquanke.com/post/id/85493

https://blog.csdn.net/cssxn/article/details/101285088

https://blog.csdn.net/stevegao_tencent/article/details/43486485?utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.essearch_pc_relevant&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.essearch_pc_relevant

大体来说，windows中的CFG主要防御的是一些虚表函数指针，如果想要攻击类似于IOFILE的vtable的话，开启了CFG后就完全把这条路堵死了。

然而CFG也有缺点，也就是绕过方法，比如他没有防御返回地址、SEH指针等，可以攻击这些没有被CFG防御的区域.同样，由于CFG依赖白名单，而这一白名单是在编译时生成的，他没有扩展性，所以一些临时生成代码比如JIT生成的代码就没有CFG保护。

### 发展

对于CFI验证，学术界提出了很多相应的解决办法。对于这种底层的验证方案，不光要考虑可行性和安全性，同时效率也是不可忽视的一个重要因素。各种专家学者提出了很多的方案，这篇文章中做了一些简要的介绍：

https://www.inforsec.org/wp/?p=495

### 控制流劫持的末日——CET

或许是厌倦了软件防护花里胡哨的算法以及效率的折衷，intel提出了一个似乎更佳完美的解决方案：CET。

这个CET全称是**Control-flow Enforcement Technology**，并不是大学英语等级考试的CET。

研究者们似乎将分支跳转分成了两类，第一类是向前跳转，即call、jmp类型指令，第二类是后向跳转，也就是ret型指令。

那么众所周知，劫持控制流就是控制RIP指针，而RIP指针只能通过上述的两类指令进行修改，所以控制流劫持的攻击手段也都是针对于这些指令做文章。蛇打七寸，intel的CET防护措施似乎正好将剑戳进了控制流劫持的心窝。

#### 奇怪的指令——endbr64

起初并没有刻意的去参阅有关资料，而是在新版本的编译器中发现了一个奇怪的指令：endbr64，于是乎google一下，属实吓得不轻。

intel在硬件层面实现了对控制流完整性的相应检查防御措施，而这个奇怪的指令endbr64就是其中之一。

这个endbr64指令在旧版本的cpu中会被当作NOP指令，而在新的cpu中其实也是个空操作指令，但是会被当作一个标志，用于监控间接跳转，他会出现在函数的开头位置。

具体来说，就是当发生间接跳转时，cpu会从IDLE状态转换为WAITING状态，在WAITING状态的cpu运行的下一条指令必须为endbr64，如果不是的话，那么直接抛出一个异常，是的话CPU就转为IDLE状态继续执行。

> The ENDBRANCH (see Section 73 for details) is a new instruction that is used to mark valid jump target addresses of indirect calls and jumps in the program. This instruction opcode is selected to be one that is a NOP on legacy machines such that programs compiled with ENDBRANCH new instruction continue to function on old machines without the CET enforcement. On processors that support CET the ENDBRANCH is still a NOP and is primarily used as a marker instruction by the processor pipeline to detect control flow violations. The CPU implements a state machine that tracks indirect jmp and call instructions. When one of these instructions is seen, the state machine moves from IDLE to WAIT_FOR_ENDBRANCH state. In WAIT_FOR_ENDBRANCH state the next instruction in the program stream must be an ENDBRANCH. If an ENDBRANCH is not seen the processor causes a control protection exception (#CP), else the state machine moves back to IDLE state.

```
IF EndbranchEnabled(CPL) & EFER.LMA = 1 & CS.L = 1
  IF CPL = 3
  THEN
    IA32_U_CET.TRACKER = IDLE
    IA32_U_CET.SUPPRESS = 0
  ELSE
    IA32_S_CET.TRACKER = IDLE
    IA32_S_CET.SUPPRESS = 0
  FI
FI;
```

#### ROP的落幕 —— shadow stack

针对于ROP攻击，intel的CET策略是采用一个影子栈，专门用来记录返回地址等信息。

具体工作原理就是：

当运行call指令时，会同时向用户栈和影子栈压入返回地址。而当运行ret指令时，会讲用户栈弹出的返回地址与影子栈中弹出的返回地址做一个比较，若不相同则抛出异常。

那么这个影子栈存储在哪里呢？intel专门为这个影子栈策略提供了相应的寄存器和指令，分别为SSP（shadow stack pointer）和影子栈操作指令：

```
INCSSP – increment SSP (i.e. to unwind shadow stack)
RDSSP – read SSP into general purpose register
SAVEPREVSSP/RSTORSSP – save/restore shadow stack (i.e. thread switching)
```

具体的指令有哪些，这里我就没有细究，有兴趣可以翻阅intel文档。

https://binpwn.com/papers/control-flow-enforcement-technology-preview.pdf

### 结语

从最初简单的栈溢出执行shellcode到ROP，再到堆溢出利用，花式劫持虚函数指针，轰轰烈烈持续了几十年的内存破坏漏洞似乎在最近可预见的未来要到一个尾声了。似乎后CET时代的黑客们只能投机取巧攻击一些老旧的未被CET保护的设备，防御的成本越来越低，而攻击的成本则越来越高。而漏洞的攻防战还没结束，测信道、逻辑漏洞等等目前还是没有一个统一有效的保护措施，学无止境，学吧。