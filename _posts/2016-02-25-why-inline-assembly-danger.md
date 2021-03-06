---
layout: post
title: 为什么内联汇编很危险
categories:
- 语言特性
---
内联汇编是在编译器层面上提供的一种编程支持，能够将低阶语言(汇编)内嵌在高阶语言(如C/C++)中，常见目的是为了进行程序热点的性能调优，如果编写得当可以让程序在某些热点上突破性能瓶颈，但是内联汇编也是一把双刃剑，使用不当会伤及己身，本文会围绕一个例子阐述原因。

一个例子
======
《程序员的自我修养》书中7.7.5节"运行时状态的演示程序"展示了一种"手动反射"的方法来调用没有头文件的动态库中的函数，主要原理是通过使用**内联汇编**手动压栈的方式，将编译期未知的函数参数信息推迟到运行时决定，下面是一个在Linux下运行的简化例子。

~~~c
#include <stdio.h>
void fun(int i) {
    printf("%d\n",i);
}
int main(void) {
    int a = 10;
    void (*myfun)()  = fun;
    asm volatile("push %0\n"::"r"(a));
    myfun();
    asm volatile("add %0,%%esp\n"::"r"(sizeof(a)));
    return 0;
}
~~~

这段代码做了这么几件事：

1. 将有参fun函数赋给无参函数指针myfun;
2. 用内联汇编将a压栈，造出有参fun函数的调用环境，注意此时改变了main函数帧的栈顶;
3. 调用myfun函数;
4. 还原main函数的堆栈，将栈顶移回调用myfun前的位置

<!--more-->

在分析这段代码能否生效之前我们先回顾一下函数调用的过程：

> 1. 依照从右到左的顺序将参数压栈;
> 2. 保存函数返回地址，也就是 .text 代码段中下一条指令的地址;
> 3. 保存调用帧的栈底地址;
> 4. 将esp, ebp寄存器指向新的栈帧;
> 5. 执行函数内指令;
> 6. 函数结束准备返回，将调用帧的栈底地址弹出到ebp寄存器;
> 7. 弹出函数返回地址恢复调用现场

入栈以后也就是下面这个样子

![Function Stack]({{ site.baseurl }}/images/function_stack_800w.png)

对照函数压栈过程，可以看出来上面那段代码中的内联汇编替代了步骤1，并对步骤7做了补充以恢复调用栈，可以认为是在一个无参无返回值函数的调用栈之上再使用内联汇编增加了一层参数栈，从而构建了一个有参函数调用栈。

目前看起来一切OK，理论上这种hack手法可以将函数定义推迟到运行期决定，并且实现很简单，不幸的是，这段代码很可能无法如预期的那样工作。

无法工作?
======
是的，在我的系统上一运行这段代码编译的程序就会给我一个华丽丽的Segmentation fault(core dumped)，我的编译和运行环境都是Ubuntu14.04.3 LTS，gcc版本4.8.4，而当我在另一台机器上(Red Hat 5.4, gcc 4.1.2)运行这个程序时却如预期的打印了a的值10。为了了解这部分差异，我们分别在两个环境下用编译器生成汇编汇编代码进行对比，gcc生成汇编的方法如下

> gcc -S main.c

截取两个版本的编译器生成的main函数汇编代码如下

![Assembly Compare]({{ site.baseurl }}/images/assembly_compare_1.png)

首先，两边都是采用预分配的方式在入栈后为main函数调用栈申请足够的空间,对应代码左28行和右31行；
然后就是重点，左右两边在访问内存时使用基址寻址的方式上存在差异，gcc4.1.2选择了ebp寄存器作为基址，而gcc4.8.4选择esp寄存器作为基址，抛开这点看，两边指令完全一样；
再然后就是我们插入的内联汇编，编译器在内联之前已经将变量放入了eax寄存器中，对应代码左31~35行和右34~38行；
最后就是调用myfun指针指向的fun函数了，需要先将栈内的myfun指针装入eax寄存器中再进行调用，对应代码左36~38和右39~41。

不管采用栈顶还是栈底地址作为基址，正确寻址有一个前提条件，就是基址不能在编译器不知道的情况下改变。首先看一下ebp保存的栈底地址，通常来说，在入栈之后出栈之前，栈底地址都是不会改变的，因为ebp是串联起整个线程调用堆栈链表的头指针，一旦出栈前ebp的值被修改，会导致出栈无法恢复调用现场，也就是所谓的堆栈被破坏，所以采用ebp作为基址是安全的。而esp保存的栈顶地址则不同，可能在栈中发生变化，按照通常的理解局部变量入栈通常是采用push的方式在扩充栈空间的同时将变量保存到栈顶，这样每当有局部变量和函数调用的地方就会有esp的变动，而现代编译器普遍优化做法是预先计算栈空间大小，一次申请完成，这样可以减少大量寄存器操作，在这种情况下esp保存的栈顶地址也不会在栈中发生变化。见下图上半。

![Assembly Compare]({{ site.baseurl }}/images/stack_compare_800w.png)

但是使用内联汇编以后情况就不一样了，内联汇编是直接嵌入到代码生成的汇编语言中的，编译器不会解析内联汇编来调整生成的汇编指令，也就是说如果内联汇编修改了调用栈，编译器依然会按照没有被修改的调用栈去生成汇编指令。所以在修改esp寄存器保存的栈顶地址后，接下来的所有基址寻址指令都会找到错误的内存，导致程序崩溃。

关于内联汇编的建议
======
无论如何，使用内联汇编都意味着你需要掌握比实现程序逻辑更多的信息，所以如非必要，尽量避免使用内联汇编。当你使用内联汇编的时候，考虑下面列出的内容，可以减少出错的机会。

1. 编写短小，功能单调的内联汇编；
2. 时刻谨记不要破坏堆栈，在一个内联汇编块中如果使用了push等指令修改了esp保存的栈顶地址，一定也要在同一个内联汇编块中恢复它；
3. 阅读所使用编译器的手册中内联汇编部分内容，并了解你使用的每一个汇编指令

(完)
