# Finishing Up the CPU
* CPU 部分完成收尾
>Finishing Up the CPU 译文

>* 原文链接：https://blog.ryanlevick.com/DMG-01/public/book/
>* 译文出处：https://github.com/suhanyujie/emulate-game-boy-translation
>* 译者：[suhanyujie](https://github.com/suhanyujie)

## 完成 CPU

我们的 CPU 部分已经快要完成了。还有许多指令需要实现，其中的很多在这一章没有涉及，因为它们与 Game Boy 的其他部分息息相关，只不过我们还未讨论到罢了。

## 剩下的指令

在这一章，我们看看这两个指令：`NOP` 和 `HALT`

### NOP 指令

`NOP` is perhaps the simplest of the Game Boy's instructions. It stands for no-operation and it effectively does nothing except advance the program counter by 1.

### HALT 指令

`HALT` is a big more complicated than `NOP`. When the Game Boy is running, it is constantly in a loop executing instructions. The `HALT` instruction gives the game the ability to stop the CPU from executing any more instructions. How the Game Boy eventually continues executing instructions will be discussed later in the book, but for now, we have the ability to stop the Game Boy dead in its tracks. Reasons a game might want to do this include saving battery. If the game doesn't have anything to do, it can halt the CPU and save a bit energy.

For now, we'll implement `HALT` by adding a `is_halted` boolean to the CPU. At the beginning of `execute` we can check if the CPU is halted. If it is, we simply return. The `HALT` instruction simply sets this field to true.

### Where We Are

So far we've learned about the CPU and the many different instructions that the CPU can execute. We learned that these instructions live in memory which is just a long array of 8-bit numbers. The CPU reads in these bytes, decodes them as instructions and executes them. Some instructions simply do different arithmetic operations on the contents of the CPU's registers. Some instructions can cause the CPU to change its program counters, effectively "jumping" it to a different place in the game code. Some instructions read from and write to memory including a special part of memory we call the stack which behaves like a stack data structure. Finally, we learned about two special instructions: `NOP` which does nothing and `HALT` which stops the CPU from executing more instructions.

In the next section of the book, we'll be leaving the comfort of the CPU and exploring memory more closely.
