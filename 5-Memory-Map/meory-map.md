>* Memory Map 译文
>* 原文链接：https://rylev.github.io/DMG-01/public/book/memory_map.html
>* 译文出处：https://github.com/suhanyujie/emulate-game-boy-translation
>* 译者：[suhanyujie](https://github.com/suhanyujie)
>* ps：水平有限，翻译不当之处，还请指正，谢谢！

# Memory Map

Up until now we've been treating the Game Boy's memory as one long array with 0xFFFF entries in it. While this was a helpful simplification for our work on the CPU (which doesn't know anything more about memory than that), in reality things are bit more complex. Sections of memory are actually used for very specific purposes. In fact there are parts of "memory" that are not actually backed by a RAM chip but instead are directly tied to things like the Game Boy screen or the Game Boy's audio device. Below we'll be "mapping" out memory and talking at a high level about what the different sections of the Game Boy's memory map do. We'll start at the very beginning of memory with address 0x0000 and work our way all the up to the top. This chapter should serve as the jumping off point for the rest of the book, so if you don't understand exactly how something works don't worry, we'll be going in much more detail later on.
>目前为止，我们可以把 Game Boy 的内存视为一个入口为 0xFFFF 的长数组，虽然这样简化的抽象有利于我们使用 CPU （CPU 对内存一无所知），但实际上，要复杂很多。内存的作用是特定的。实际上这些“内存”并不是有 RAM 芯片支持，而是直接与 Game Boy 屏幕和音频设备相关联。下面，我们对内存映射及其高级抽象的不同部分进行讨论。我们会从内存地址的最开始的部分进行，一直到最顶部。这一章，我们应该会从本书的其他部分开始，如果你不太明白它们的原理，不用担心，我们会在后续进行更详细的讨论。

## 0x0000 - 0x00FF: Boot ROM

When the Game Boy first boot's the very bottom 256 bytes of memory is occuppied with the boot ROM. We've talked a little bit about the boot ROM before - how it's responsible for bootstrapping the Game Boy to be able to run a game as well as for playing the [iconic splash screen](https://www.youtube.com/watch?v=ClJWTR_lCL4) on boot. Later in the book, we'll be examining the boot ROM very closely.
>当  Game Boy 首次启动时，最底部的 256 字节的内存是用于引导 boot 的 ROM。我们之前已经提到过关于启动 ROM - 它是如何负责引导 Game Boy，并使其运行一个游戏以及展示[标志性的屏闪画面](https://www.youtube.com/watch?v=ClJWTR_lCL4)。在本书的后面，我们会仔细检查引导 ROM。

## 0x0000 - 0x3FFF: Game ROM Bank 0

Once the Game Boy is done booting, it unmaps the boot ROM from memory so it is no longer available. From this point the area once occupied by the boot ROM all the way up to address 0x3FFF is occupied by game code loaded in from the cartridge. Inside of this memory are two areas worth noting:
>一旦 Game Boy 完成了引导，就会去除 ROM 对应的内存映射，因此 ROM 不再可用。从这一点开始，从启动 ROM 开始，一直到地址 0x3FFF 的区域将会被游戏卡中的游戏代码所占用。在这段内存中，有两个值的区域需要注意：

* 0x0000 - 0x00FF - the area once occupied by the Boot ROM now holds memory called the "Interrupt Table". We'll be talking at length in the future about interrupt's, but for now the best way to think about them is just like "events" in higher level programming. When specific things happen the hardware automatically looks inside the interrupt table at specific locations for how to handle those events.
>* 0x0000 - 0x00FF —— 曾被引导 ROM 占用的这块区域，现在正保存着“中断表”。我们会在后面讨论中断，但现在，最好把它们看作是高级编程中的“事件”。当特定的事件发生时，硬件会自动在中断表中查找如何处理该事件。

* 0x0100 - 0x014F - This area is known as the cartridge header area. It contains data about the cartridge that was loaded into Game Boy including its name, the size of cartridge ROM and even the nintendo logo. We'll talk about more about the contents of the cartridge when we talk about the boot ROM since it directly references this header area. If you want to really dive into specifics checkout [the cartridge header guide](https://rylev.github.io/DMG-01/public/book/appendix/cartridge_header.html)
>* 0x0100 - 0x014F —— 这个区域是常见的数据盒头部区域。它存储盒装游戏的数据，包括名称、盒装游戏的 ROM，甚至任天堂的 logo。当我们讨论引导 ROM 时，我们先具体了解一下数据盒内容，因为它直接引用了数据盒头部区域。如果你想深入了解细节，可以查看[数据盒头部指南](https://rylev.github.io/DMG-01/public/book/appendix/cartridge_header.html)

After these special areas is plain game code. The reason this area is refered to as Bank 0 is explained in the next section.
>在介绍了这些特殊的区域后，接下来是简单的游戏代码。这个区域被称为 区块 0（Bank 0）的原因将在下一节解释。

## 0x4000 - 0x7FFF: Game ROM Bank N
Game ROMs can be quite large - much larger than what can fit into the memory area available. To handle this, the Game Boy allows for ROM "bank switching". A ROM bank is simply a chunk of the cartirdge ROM. The game can switch in these chunks at run time into the area between 0x4000 - 0x7FFF. The first bank, bank 0, is always the same memory and cannot be switched out. Only the area of memory between 0x4000 and 0x7FFF is capable of being switched out. Later in the book we'll go into specifics of how this works.
>游戏 ROM 可以很大 —— 比可用内存所能容纳的还要大的多。为了解决这个问题，Game Boy 允许 ROM 进行“bank 切换”（bank switching）。ROM 是一个简单的光盘，游戏可以在运行时切换 0x4000 - 0x7FFF 之间的区域。第一个 bank，bank 0，如果它和内存大小相同，就无法切换。只有 0x4000 和 0x7FFF 之间的内存区域能够正常切换。在本书的后面，我们会详细讨论它如何工作。

## 0x8000 - 0x97FF: Tile RAM
This area of memory contains data about the graphics that can be displayed to the screen. The Game Boy uses a tiling system for grapics meaning that a game doesn't control the specific pixels that get drawn to the screen, at least not directly. Instead, the game creates a set of tiles which are square chunks of pixels. It can then place these tiles on the screen. So instead of saying "draw pixel 438 light green", it first says "create a tile with these pixels values" and then later "place that tile I made earlier in positon 5". The placement of tiles at certain positions happens in the next chunk of memory...
>这块带有图像数据的内存区域可以显示到屏幕上。Game Boy 使用了平铺像素系统，这意味着游戏不能控制绘制到屏幕上的特定像素块，至少不能直接控制。相反，游戏创造了一组像素组成的方块。游戏机会将像素块绘制到屏幕上。所以说它不能“绘制 438 像素的浅绿色”，只能说它会通过像素值来创建一些像素方块，然后再将其放在 5 号位置。然后在内存的下一个特定的内存区域继续摆放。。。

## 0x9800 - 0x9FFF: Background Map
As we described above, the Game Boy uses a tiling system for graphics. In memory 0x8000 to 0x97FF the game creates different tiles. These tiles however don't show up on screen. That happens in this section of memory where the game can map tiles to sections of the screen.
>正如之前所描述的，Game Boy 使用了图像块平铺系统。在内存 0x8000 到 0x97FF 的位置，游戏会创建不通的图像块。然而，这些方块并不会显示到屏幕上。而是发生在内存中，然后游戏机会将它们映射到屏幕的各个地方。

## 0xA000 - 0xBFFF: Cartridge RAM
Cartridges (being physical devices) sometimes had extra RAM on them. This gave games even more memory to work with. If the cartridge had this extra RAM the Game Boy automatically mapped the RAM into this area of memory.
>墨盒（Cartridges）（一个物理设备）有时会有额外的 RAM。这样会给游戏提供更多的内存。如果 cartridge 有额外的RAM，Game Boy 会将 RAM 映射到内存区域。

## 0xC000 - 0xDFFF: Working RAM
This is the RAM that the Game Boy allows a game to use. Our idea of RAM really just being a plain old array where the game could read and write bytes to really only applies to this section of memory.
>这是 Game Boy 允许游戏使用的 RAM。我们可以将这部分 RAM 视为一个普通的数组，游戏可以在其中读写字节，只有这部分是适用的。

## 0xE000 - 0xFDFF: Echo RAM
This section of memory directly mirrors the working RAM section - meaning if you write into the first address of working RAM (0xC000), the same value will appear in the first spot of echo RAM (0xE000). Nintendo actively discouraged developers from using this area of memory and as such we can just pretend it doesn't exist.
>这部分内存直接映射工作 RAM 区域 —— 这意味着如果你写入数据到第一个工作 RAM 地址（0xC000），相同的值将会出现在 Echo RAM 的第一个位置（0xE000）。任天堂不鼓励开发者使用这一内存区域，所以我们先忽略它。

## 0xFE00 - 0xFE9F: OAM (Object Atribute Memory)
This area of memory contains the description of graphical sprites. The tiles we talked about above were used for backgrounds and levels but not for characters, enemies or objects the user interacted with. These entities, known as sprites, have extra capabilties. The description for how they should look lives here.
>这个内存区域包含了图像精灵的描述。我们上面提到的块主要用于背景和关卡，而非角色、敌人或用户互动的对象。这些实体叫做“精灵”，有额外的作用。描述它们的属性数据就存储在这里。

## 0xFEA0 - 0xFEFF: Unused
This area is completely unmapped: reading from it just returns 0s and writing to it does nothing.
>这个区域完全没有映射：从中读取只返回 0，而向其中写入则什么也不做。

## 0xFF00 - 0xFF7F: I/O Registers
This is one of the most dense areas of memory. Practically every byte has a special meaning. It's used by both the screen and the sound system to determine different settings. We'll be talking a lot about this in the future.
>这是内存密集的区域之一。实际上每个字节都有特殊的含义。屏幕和声音系统都使用它来确定对应的设置。后面会经常讨论这个。

## 0xFF80 - 0xFFFE: High RAM Area
This area is also just normal RAM but is used a lot because some of the `LD` instructions we've already seen can easily target this area in memory. This area is also sometimes used for the stack, but the top of working RAM is also used for this purpose.
>这个区域只是普通的 RAM，但却经常被使用，因为我们之前已经看到一些 `LD` 指令可以很容易地命中这部分区域。这个区域有时也用于栈，工作 RAM 的顶部同理也是用于这个目的。

## 0xFFFF: Interrupt Enabled Register
The very last byte of memory has special meaning used for handling interrupt events. We'll be looking at this closely later in the book.
>内存的最后一个字节有特殊的含义，用于处理中断事件。我们将在书的后面详细讨论。

Now that we have an idea of what all the areas of memory are used for we can start diving more in detail. The first area we'll be looking at is at tile RAM and the background map along with a few of the I/O registers that will help us finally get some graphics on a screen!
>既然我们已经知道了所有内存区域的用途，我们就可以开始更详细的研究了。我们将看到的第一个 RAM 区域和背景地图以及一些 I/O 寄存器，这将有助于我们最后在屏幕上显示图形！
