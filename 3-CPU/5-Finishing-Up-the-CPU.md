>* Finishing Up the CPU 译文 —— CPU 部分完成收尾
>* 原文链接：https://blog.ryanlevick.com/DMG-01/public/book/
>* 译文出处：https://github.com/suhanyujie/emulate-game-boy-translation
>* 译者：[suhanyujie](https://github.com/suhanyujie)
>* ps：水平有限，翻译不当之处，还请指正，谢谢！

## [完成 CPU](#finishing-up-the-cpu)

我们已经快要完成 CPU 的构建了。还有一些其他指令需要实现，其中很多我们暂时不会涉及，因为那些虽然与 Game Boy 紧密相关，但还没到讨论的时候。

## [Remaining Instructions](#remaining-instructions)

在本章中，我们会着重研究这两个指令：`NOP` 和 `HALT`

### [NOP](#nop)

`NOP` 可能是 Game Boy 中最简单的指令。它表示空操作，即它除了将程序计数器累加一次外，没有其他任何影响或作用。

### HALT

`HALT` 则比 `NOP` 复杂得多。当 Game Boy 运行时，它会不断地循环执行指令。`HALT` 指令使游戏能够中断 CPU 以执行其他更多指令。Game Boy 最终如何恢复执行指令会在本书的后面进行讨论，但现在，我们先让 Game Boy 的 CPU 暂停。之所以这样，“省电”是好处之一。如果游戏没有要执行的东西，它可以暂停 CPU，节省电量。

现在，我们通过向 CPU 结构体添加一个 `is_halted` 布尔值来实现 `HALT` 指令。在 `execute` 的一开始，我们可以检查 CPU 是否停止，如果是，则返回。`HALT` 指令就会将 `is_halted` 值设为 true。

### Where We Are

到目前为止，我们已经了解了 CPU 及其可以执行的很多指令。我们知道，这些指令在内存中，而内存只是一个由 8 位的数值组成的数组。CPU 读取这些字节，将它们解码为指令，并执行。有些指令是对 CPU 寄存器的值进行算术运算。有些指令会让 CPU 改变它的程序计数器，有效地进行跳跃。一些指令是从内存读取和写入，包括内存的特殊位置（我们称之为堆栈），它的行为类似于堆栈这种数据结构。最后，我们还学习了两个特殊的指令：`NOP` 什么都不做；`HALT` 中断 CPU 让它执行其他指令。

在本书的下一部分，我们会脱离这些 CPU 知识区，进行更深入的内存探索。
