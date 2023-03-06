# Calling conventions[调用规约] and stack frames RISC-V [函数调用栈]

板块: 进程 线程

**Calling conventions and stack frames RISC-V**

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image1.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image1.png)

汇编代码并不是在内存上执行，而是在寄存器上执行，也就是说，当我们在做add，sub时，我们是对寄存器进行操作

我们通过load将数据存放在寄存器中，这里的**数据源**可以是来自内存，也可以来自另一个寄存器

之后我们在寄存器上执行一些操作。如果我们对操作的结果关心的话，我们会将操作的结果**store**在某个地方。这里的目的地可能是内存中的某个地址，也可能是另一个寄存器。这就是通常使用寄存器的方法。

寄存器是用来进行任何运算和数据读取的最快的方式，这就是为什么使用它们很重要，也是为什么我们更喜欢使用寄存器而不是内存

通常我们在谈到寄存器的时候，我们会用它们的ABI名字。不仅是因为这样描述更清晰和标准，同时也因为在写汇编代码的时候使用的也是ABI名字

RISC-V 64中所有寄存器都是64位大小，各种各样的数据类型都会被改造的可以放进这64bit中。比如说我们有一个32bit的整数，取决于整数是不是有符号的，会通过在前面补32个0或者1来使得这个整数变成64bit并存在这些寄存器中。

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image2.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image2.png)

32个**用户**寄存器，用户应用程序(汇编代码)可以使用全部的寄存器 并且使用寄存器的指令性能是最好的

a0到a7寄存器是用来作为**函数的参数**。

如果一个函数有超过8个参数，我们就需要用内存了。从这里也可以看出，当可以使用寄存器的时候，我们不会使用内存，我们只在不得不使用内存的场景才使用它。

表单中的第4列，Saver列，当我们在讨论寄存器的时候也非常重要。它有两个可能的值Caller，Callee。我经常混淆这两个值，因为它们只差一个字母。

一个简单的记住它们的方法是：

1. Caller Saved寄存器在函数调用的时候不会被自动保存(临时寄存器)；需要caller主动保存的寄存器
2. Callee Saved寄存器在函数调用的时候会被自动保存/保持；callee**会**保存的寄存器，callee结束后恢复寄存器，对caller来说是透明的

https://zhuanlan.zhihu.com/p/462483036

<aside>
💡 caller --> Not Preserved across func call
callee --> Preserved across func call

</aside>

这里的意思是，一个**Caller Saved寄存器可能被其他函数重写**。假设我们在函数a中调用函数b，任何被函数a使用的并且是Caller Saved寄存器，调用函数b可能重写这些寄存器。我认为一个比较好的例子就是Return address寄存器（注，保存的是函数返回的地址），你可以看到ra寄存器是Caller Saved，这一点很重要，它导致了当在函数a中调用函数b的时候，b会重写Return address。所以基本上来说，任何一个Caller Saved寄存器，作为调用方的函数要小心可能的数据可能的变化；任何一个Callee Saved寄存器，作为被调用方的函数要小心寄存器的值不会相应的变化。

# **Stack**

下面是一个非常简单的栈的结构图，其中每一个区域"块"都是一个Stack Frame，每执行一次函数调用就会产生一个Stack Frame。

一个stack由多个stack frame组成，**stack frame由函数调用自动产生**

每个**stack frame由 Return Addr, PrevFP, Saved Register, Local Var, ...组成**，每个stack frame不一定相同由于组成部分大小相异

但每个stack一定相同 大小为PGSIZE

函数栈帧 (Stack Frame)

函数调用经常是嵌套的，在同一时刻，栈中会有多个函数的信息。每个未完成运行的函数占用一个独立的连续区域，称作栈帧(Stack Frame)。栈帧存放着函数参数，局部变量及恢复前一栈帧所需要的数据等，**函数调用时入栈的顺序**为：

`实参N~1 → 主调函数返回地址 → 主调函数帧基指针EBP → 被调函数局部变量1~N`

在**X86中**，栈帧的边界由 栈帧基地址指针 EBP 和 栈指针 ESP 界定，EBP 指向当前栈帧底部(高地址)，在当前栈帧内位置固定；ESP指向当前栈帧顶部(低地址)，当程序执行时ESP会随着数据的入栈和出栈而移动。因此函数中对大部分数据的访问都基于EBP进行。函数调用栈的典型内存布局如下图所示：

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image4.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image4.png)

**在RISV-V中**

stack frame中return address 的 offset固定为fp-8

to prev fp 的 offset 固定为 fp-16

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image5.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image5.png)

**栈生长(allocate是从高地址到低地址)，但使用时从低地址到高地址**

每一次我们**调用一个函数**，函数都会为自己创建一个Stack Frame，并且**只给自己用**。**这里注意，每新调用一个函数，RA寄存器会被覆盖成调用者的地址**

函数通过移动Stack Pointer来完成Stack Frame的空间分配。

对于Stack来说，是从高地址开始向低地址使用。所以栈总是向下增长。当我们想要创建一个新的Stack Frame的时候，总是对当前的Stack Pointer做减法。一个函数的**Stack Frame**包含了保存的**寄存器**，**本地变量**，并且，如果函数的参数多于8个，额外的参数会出现在Stack中。所以Stack Frame大小并不总是一样，即使在这个图里面看起来是一样大的。不同的函数有不同数量的本地变量，不同的寄存器，所以Stack Frame的大小是不一样的。但是有关Stack Frame有两件事情是确定的：

**Return address总是会出现在Stack Frame的第一位**

**指向前一个Stack Frame的指针也会出现在栈中的固定位置**

有关Stack Frame中有两个重要的寄存器，第一个是**SP**（Stack Pointer），它指向Stack的**底部**并代表了**当前Stack Frame的位置**。第二个是**FP**（Frame Pointer），它指向**当前Stack Frame的顶部**。因为Return address和指向前一个Stack Frame的指针都在当前Stack Frame的固定位置，所以可以通过当前的FP寄存器寻址到这两个数据。

我们保存前一个Stack Frame的指针的原因是为了让我们能跳转回去。所以当前函数返回时，我们可以将前一个Frame Pointer存储到FP寄存器中。所以我们使用Frame Pointer来操纵我们的Stack Frames，并确保我们总是指向正确的函数。

**Stack Frame必须要被汇编代码创建**，所以是编译器生成了汇编代码，进而创建了Stack Frame。

所以通常，在汇编代码中，函数的最开始你们可以看到Function prologue，之后是函数的本体，最后是Epollgue。这就是一个汇编函数通常的样子。

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image6.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image6.png)

我们从汇编代码中来看一下这里的操作。

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image7.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image7.png)

在我们之前的sum_to函数中，只有函数主体，并没有Stack Frame的内容。它这里能正常工作的原因是它足够简单，并且它是一个leaf函数。leaf函数是指不调用别的函数的函数，它的特别之处在于它不用担心保存自己的Return address或者任何其他的Caller Saved寄存器，因为它不会调用别的函数。

而另一个函数sum_then_double就不是一个leaf函数了，这里你可以看到它调用了sum_to。

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image8.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image8.png)

所以在这个函数中，需要包含prologue。

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image9.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image9.png)

这里我们对Stack Pointer减16，这样我们**为新的Stack Frame创建了16字节的空间**。之后我们将Return address保存在Stack Pointer位置。

之后就是调用sum_to并对结果乘以2。最后是Epilogue，

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image10.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image10.png)

这里首先将Return address加载回ra寄存器，通过对Stack Pointer加16来删除刚刚创建的Stack Frame，最后ret从函数中退出。

教授提问：如果我们删除掉Prologue和Epilogue，然后只剩下函数主体会发生什么？

学生回答：sum_then_double将不知道它应该返回的Return address。所以调用sum_to的时候，Return address被覆盖了，最终sum_to函数不能返回到它原本的调用位置。

教授：是的，完全正确，我们可以看一下具体会发生什么。

先在修改过的sum_then_double设置断点，然后执行sum_then_double。

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image11.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image11.png)

我们可以看到现在的ra寄存器是0x80006392，它指向demo2函数，也就是sum_then_double的调用函数。之后我们执行代码，调用了sum_to。

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image12.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image12.png)

我们可以看到ra寄存器的值被sum_to重写成了0x800065f4，指向sum_then_double，这也合理，符合我们的预期。我们在函数sum_then_double中调用了sum_to，那么sum_to就应该要返回到sum_then_double。

之后执行代码直到sum_then_double返回。如前面那位同学说的，因为没有恢复sum_then_double自己的Return address，现在的Return address仍然是sum_to对应的值，现在我们就会进入到一个无限循环中。

我认为这是一个很好的例子用来展示为什么跟踪Caller和Callee寄存器是重要的。

学生提问，为什在最开始要对sp寄存器减16？

TA：是为了Stack Frame创建空间。减16相当于内存地址向前移16，这样对于我们自己的Stack Frame就有了空间，我们可以在那个空间存数据。我们并不想覆盖原来在Stack Pointer位置的数据。

学生提问：为什么不减4呢？

TA：我认为我们不需要减16那么多，但是4个也太少了，你至少需要减8，因为接下来要存的ra寄存器是64bit（8字节）。这里的习惯是用16字节，因为我们要存Return address和指向上一个Stack Frame的地址，只不过我们这里没有存指向上一个Stack Frame的地址。如果你看kernel.asm，你可以发现16个字节通常就是编译器的给的值。

接下来我们来看一些C代码。

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image13.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image13.png)

demo4函数里面调用了dummymain函数。我们在dummymain函数中设置一个断点，

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image14.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image14.png)

现在我们在dummymain函数中。如果我们在gdb中输入info frame，可以看到有关当前Stack Frame许多有用的信息。

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image15.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image15.png)

Stack level 0，表明这是调用栈的最底层

pc，当前的程序计数器

saved pc，demo4的位置，表明当前函数要返回的位置

source language c，表明这是C代码

Arglist at，表明参数的起始地址。当前的参数都在寄存器中，可以看到argc=3，argv是一个地址

如果输入backtrace（简写bt）可以看到从当前调用栈开始的所有Stack Frame。

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image16.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image16.png)

如果对某一个Stack Frame感兴趣，可以先定位到那个frame再输入info frame，假设对syscall的Stack Frame感兴趣。

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image17.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image17.png)

在这个Stack Frame中有更多的信息，有一堆的Saved Registers，有一些本地变量等等。这些信息对于调试代码来说超级重要。

学生提问：为什么有的时候编译器会优化掉argc或者argv？这个以前发生过。

TA：这意味着编译器发现了一种更有效的方法，不使用这些变量，而是通过寄存器来完成所有的操作。如果一个变量不是百分百必要的话，这种优化还是很有常见的。我们并没有给你编译器的控制能力，但是在你们的日常使用中，你可以尝试设置编译器的optimization flag为0，不过就算这样，编译器也会做某些程度的优化。

**GDB**

info frame： 可以看到有关当前Stack Frame许多有用的信息。

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image15.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image15.png)

Stack level 0，表明这是调用栈的最底层

pc，当前的程序计数器

saved pc，demo4的位置，表明当前函数要返回的位置

source language c，表明这是C代码

Arglist at，表明参数的起始地址。当前的参数都在寄存器中，可以看到argc=3，argv是一个地址

如果输入backtrace（简写bt）可以看到从当前调用栈开始的所有Stack Frame。

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image16.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image16.png)

如果对某一个Stack Frame感兴趣，可以先定位到那个frame再输入info frame，假设对syscall的Stack Frame感兴趣。

![Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image17.png](Calling%20conventions%5B%E8%B0%83%E7%94%A8%E8%A7%84%E7%BA%A6%5D%20and%20stack%20frames%20RISC-V%20%20b569b734423c4260941d64a35c6bc453/image17.png)

在这个Stack Frame中有更多的信息，有一堆的Saved Registers，有一些本地变量等等。这些信息对于调试代码来说超级重要。