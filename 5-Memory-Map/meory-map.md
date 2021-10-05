>* Memory Map 译文（内存映射）
>* 原文链接：https://rylev.github.io/DMG-01/public/book/memory_map.html
>* 译文出处：https://github.com/suhanyujie/emulate-game-boy-translation
>* 译者：[suhanyujie](https://github.com/suhanyujie)
>* ps：水平有限，翻译不当之处，还请指正，谢谢！

# 内存映射

目前为止，我们可以把 Game Boy 的内存视为一个入口为 0xFFFF 的长数组，虽然这样简化的抽象有利于我们使用 CPU （CPU 对内存一无所知），但实际上，要复杂很多。内存的作用是特定的。实际上这些“内存”并不是有 RAM 芯片支持，而是直接与 Game Boy 屏幕和音频设备相关联。下面，我们对内存映射及其高级抽象的不同部分进行讨论。我们会从内存地址的最开始的部分进行，一直到最顶部。这一章，我们应该作为本书的其他部分的起点，如果你不太明白它们的原理，不用担心，我们会在后续进行更详细的讨论。

## 0x0000 - 0x00FF: 引导 ROM

当 Game Boy 首次启动时，最底部的 256 字节的内存是用于引导 ROM（boot ROM）。我们之前已经提到过关于引导 ROM - 它是如何负责引导 Game Boy，并使其运行一个游戏以及播放[标志性的屏闪画面](https://www.youtube.com/watch?v=ClJWTR_lCL4)。在本书的后面，我们会仔细检查引导 ROM。

## 0x0000 - 0x3FFF: Game ROM Bank 0

一旦 Game Boy 完成了引导，就会去除引导 ROM 对应的内存映射，因此它不再可用。这时，从引导 ROM 的内存位置开始，一直到地址 0x3FFF 的区域将会被游戏卡中的游戏代码所占用。在这段内存中，有两个区域需要注意：

* 0x0000 - 0x00FF —— 曾被引导 ROM 占用的这块区域，现在正保存着“中断表”（Interrupt Table）。我们会在后面讨论中断，但现在，最好把它们看作是高级编程中的“事件”。当特定的事件发生时，硬件会自动在中断表中查找如何处理该事件。

* 0x0100 - 0x014F - This area is known as the cartridge header area. It contains data about the cartridge that was loaded into Game Boy including its name, the size of cartridge ROM and even the nintendo logo. We'll talk about more about the contents of the cartridge when we talk about the boot ROM since it directly references this header area. If you want to really dive into specifics checkout [the cartridge header guide](https://rylev.github.io/DMG-01/public/book/appendix/cartridge_header.html)
>* 0x0100 - 0x014F —— 这个区域是常见的数据盒头部区域。它存储盒装游戏的数据，包括名称、盒装游戏的 ROM，甚至任天堂的 logo。当我们讨论引导 ROM 时，我们先具体了解一下数据盒内容，因为它直接引用了数据盒头部。如果你想深入了解细节，可以查看[数据盒头部指南](https://rylev.github.io/DMG-01/public/book/appendix/cartridge_header.html)

在介绍了这些特殊的区域后，接下来是普通的游戏代码。这个区域被称为区块 0（Bank 0）的原因将在下一节解释。

## 0x4000 - 0x7FFF: Game ROM Bank N
游戏 ROM 可以很大 —— 比可用内存所能容纳的还要大的多。为了解决这个问题，Game Boy 允许 ROM 进行“bank 切换”（bank switching）。ROM 是一个简单的光盘，游戏可以在运行时切换 0x4000 - 0x7FFF 之间的区域。第一个 bank，bank 0，如果它和内存大小相同，就无法切换。只有 0x4000 和 0x7FFF 之间的内存区域能够正常切换。在本书的后面，我们会详细讨论它如何工作。

## 0x8000 - 0x97FF: 内存块
This area of memory contains data about the graphics that can be displayed to the screen. The Game Boy uses a tiling system for grapics meaning that a game doesn't control the specific pixels that get drawn to the screen, at least not directly. Instead, the game creates a set of tiles which are square chunks of pixels. It can then place these tiles on the screen. So instead of saying "draw pixel 438 light green", it first says "create a tile with these pixels values" and then later "place that tile I made earlier in positon 5". The placement of tiles at certain positions happens in the next chunk of memory...
>这块带有图像数据的内存区域可以显示到屏幕上。Game Boy 使用了块平铺系统，这意味着游戏不能控制绘制到屏幕上的特定像素，至少不能直接控制。相反，游戏创造了一组像素组成的块。“游戏机”会将块绘制到屏幕上。所以说它不能“绘制 438 像素的浅绿色”，只能说它会通过一些像素来创建块，然后再将其放在 5 号位置。然后在内存的下一个特定的内存区域继续摆放。。。

## 0x9800 - 0x9FFF: 背景地图
As we described above, the Game Boy uses a tiling system for graphics. In memory 0x8000 to 0x97FF the game creates different tiles. These tiles however don't show up on screen. That happens in this section of memory where the game can map tiles to sections of the screen.
>正如之前所描述的，Game Boy 使用了块平铺系统。在内存 0x8000 到 0x97FF 的位置，游戏会创建不同的块。然而，这些方块并不会显示到屏幕上。而是发生在内存中，然后游戏机会将它们映射到屏幕的各个位置。

## 0xA000 - 0xBFFF: Cartridge RAM
Cartridges (being physical devices) sometimes had extra RAM on them. This gave games even more memory to work with. If the cartridge had this extra RAM the Game Boy automatically mapped the RAM into this area of memory.
>墨盒（Cartridges）（一个物理设备）有时会有额外的 RAM。这样会给游戏提供更多的内存空间。如果 cartridge 有额外的 RAM，Game Boy 会将 RAM 映射到内存区域。

## 0xC000 - 0xDFFF: 工作 RAM（Working RAM）
这是 Game Boy 允许在游戏时使用的 RAM。我们可以将这部分 RAM 视为一个普通的数组，游戏可以在其中读写字节，只有内存的这部分是适用的。

## 0xE000 - 0xFDFF: Echo RAM
这部分内存直接是工作 RAM 区域的镜像内存 —— 这意味着如果你写入数据到第一个“工作 RAM”地址（0xC000），相同的值将会出现在 Echo RAM 的第一个位置（0xE000）。任天堂不鼓励开发者使用这一内存区域，所以我们先忽略它。

## 0xFE00 - 0xFE9F: OAM（对象属性内存）
这个内存区域包含了图像精灵的描述。我们上面提到的块主要用于背景和关卡，而非角色、敌人或用户互动的对象。这些角色、敌人或用户互动的对象的实体叫做“精灵”，有额外的作用。描述它们的属性数据就存储在这里。

## 0xFEA0 - 0xFEFF: 预留
这个区域完全没有映射：从中读取只返回 0，而向其中写入则什么也不做。

## 0xFF00 - 0xFF7F: I/O 寄存器
这是内存密集的区域之一。实际上每个字节都有特殊的含义。屏幕和声音系统都使用它来确定对应的设置。后面会经常讨论这个。

## 0xFF80 - 0xFFFE: 高位内存区域
This area is also just normal RAM but is used a lot because some of the `LD` instructions we've already seen can easily target this area in memory. This area is also sometimes used for the stack, but the top of working RAM is also used for this purpose.
>这个区域只是普通的 RAM，但却经常被使用，因为我们之前已经看到一些 `LD` 指令可以很容易地命中这个区域。这个区域有时也用于栈的使用，“工作 RAM”的顶部同理也是用于这个目的。

## 0xFFFF: 启用的中断寄存器
内存的最后一个字节有特殊的含义，用于处理中断事件。我们将在书的后面详细讨论。

既然我们已经知道了所有内存区域的用途，我们就可以开始更详细的研究了。我们将关注第一个 RAM 区域和背景地图以及一些 I/O 寄存器，这将有助于我们最后在屏幕上显示图形！
