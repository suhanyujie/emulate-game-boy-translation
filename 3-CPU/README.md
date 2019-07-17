# CPU
Game Boy 的 CPU 是一种定制的芯片，名为夏普 LR35902。这个芯片跟更加流行的 [Intel 8080](https://en.wikipedia.org/wiki/Intel_8080) 和 [Zilog Z80](https://en.wikipedia.org/wiki/Zilog_Z80) 是相似的。8080 系列在 70 年代和 80 年代被用在很多的计算机中，包括第一个个人计算机 [Altair 8800](https://en.wikipedia.org/wiki/Altair_8800)。Z80 也是一个非常受欢迎的芯片，用在许多家庭电子设备中，如 Sega（世嘉） 家用控制台、主机系统和 Sega Genesis/Mega 驱动器。

我们不会很详细的介绍 LR35902 和 Intel 8080、Z80 的不同之处，但大体上，我们学到的大部分这个芯片知识会适用于从去年开始流行的其它芯片

在接下来的几节中，我们要研究 LR35902 可以执行不同的 CPU 指令，和它如何从内存中读取指令、解析指令并更新内部状态、内存、各种不同的 I/O 设备。
