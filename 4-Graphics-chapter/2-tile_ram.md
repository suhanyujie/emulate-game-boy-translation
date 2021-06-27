# Tile Ram
* 内存块/片

>* Tile Ram 译文
>* 原文链接：https://github.com/rylev/DMG-01/blob/master/book/src/graphics/tile_ram.md
>* 译文出处：https://github.com/suhanyujie/emulate-game-boy-translation
>* 译者：[suhanyujie](https://github.com/suhanyujie)

Before we can display background graphics to the screen, we have to have a good understanding of how background graphics actually work and where those graphics are stored in memory.
>在我们将背景图显示到显示屏上前，我们需要对背景图显示的工作原理进行比较清晰的了解。 

Game Boy games do not have direct control over what appears in the background. This is because the Game Boy is limited in how much it can store for graphics. The Game Boy has 0x1FFF (8191) bytes worth of storage for background graphics. Unlike more modern systems that employ a direct "frame buffer" (i.e., a long array of bytes where each byte or set of bytes describes how a corresponding pixel should be displayed on screen), the Game Boy uses a tiling system. This system allows the game to build 8 pixel by 8 pixel tiles and then place each tile on the screen at a certain index.
>Game boy 游戏机不能直接控制显示屏上的图像显示。这是因为 Game boy 的图形存储能力很有限。Game boy 有 0x1FFF (8191) 字节的存储空间用于显示。跟现如今很多系统使用的“帧缓冲区”（如，一个长字节数组，其中每个字节或一组字节对应的像素应该如何在屏幕上显示）有很大不同。Game boy 使用的是平铺系统。这个系统允许游戏构建 8x8 像素的区块，然后每个区块按一定的索引渲染到屏幕上。

待办事项:用图表显示两者之间的区别（译者注：原作者标注的，这部分暂时没有完善）

## 创建像素块
So before we look at how pixels on the screen are shown, we first have to see how games manipulate and store tiles.
>所以在看到屏幕上显示像素之前，我们首先看看游戏是如何操作和保存像素卡块的。

As we've seen before in our overview of the memory map of the Game Boy, tile data is stored between 0x8000 and 0x97FF (0x1800 or 6144 bytes worth of memory). This area actually contains two seperate tile sets. This allows the game to very quickly switch between two differnt graphic styles without having to switch the tiles out in the time between two screens. We'll explore how the game switches between the two tile sets a bit later.
>正如之前的 Game Boy 内存映射概述中所描述的，“平铺块”数据存储在 0x8000 和 0x97FF（0x1800 或 6144 字节的内存）之间的内存片段上。这个区域实际上包含两个独立的内存片段。这允许游戏非常快速地在两种不同风格的图形之间切换，而不必实时地分开两段内存。稍后，我们将探讨如何在这两个内存片段之间切换设置位。

For now, we'll be focusing on the first tile set in memory that resides at 0x8000 to 0x8FFF (for a total of 0x1000 or 4096 bytes worth of data). Each tile is encoded in 16 bytes (we'll talk about exactly what this encoding looks like below). So if we 0x1000 bytes worth of memory and each tile is encoded in 16 bytes, then we have 0x1000 / 0x10 or 0x100 (256) different tiles.
>现在，我们将关注内存中的第一片区域，它位于 0x8000 到 0x8FFF 之间（相当于总共 0x1000 大小或者 4096 字节的数据）。每个 tile 都被编码为 16 个字节（我们下面将讨论这种编码是什么样）。因此，如果我们有 0x1000 字节的内存，并且每个 tile 都被编码为 16 字节，那么我们就有 0x1000/0x10 即 0x100（256）个不同的内存块（tile）。

An observant reader might wonder why the first tile set takes up 0x1000 of the 0x1800 or two thirds worth of space alloted for tile memory. The truth is that the second tile set starts at 0x8800 and goes to 0x97FF. The chunk between 0x8800 and 0x8FFF is therefore shared by the two tile sets.
>细心的读者可能会想，为什么第一个像素块占用了 0x1800 中的 0x1000，或者分配给像素块三分之二的空间。事实上第二个像素块是从 0x8800 到 0x97FF。因此，0x8800 和 0x97FF 之间的区域被两个像素块共享的。

TODO: 画一张图

```
8000-87FF: First part of tile set #1
8800-8FFF: Second part of tile set #1
           First part of tile set #2
9000-97FF: Second part of tile set #2
```

So how are each of the tiles encoded? First, we need to understand how many different colors a pixel of the Game Boy can display. The Game Boy is capable of displaying 4 different colors: white, light gray, dark gray, and black. The minimal number of bits that we need to encode this information is 2 bits since two bits can encode 4 different numbers: 0b00, 0b01, 0b10, and 0b11.
>每一片像素是如何编码的呢？首先，我们需要了解 Game Boy 的一个像素可以显示多种不同的颜色。Game Boy 可以显示 4 种不同的颜色：白色、浅灰色、深灰色和黑色。因此，编码这些最少需要 2 位，因为 2 位最多可以编码 4 个不同的数值：0b00、0b01、0b10 和 0b11.

>* Learn More
>* The way the Game Boy hardware displays the 4 different colors is simply by emitting 4 different levels of white light. For "white" for instance the light is fully on, while for black the light is fully off. Light and dark gray are at 33% light and 66% light respectively. In fact, calling these colors white, gray and black isn't really true since the screen of the original Game Boy was green so the colors players see are actually shades of green.
>* Game Boy 硬件显示 4 种不同颜色的方式是通过发射 4 种不同级别的白光。如，“白色”，灯是完全亮着的，而对于“黑色”，灯是完全关闭的。浅色和深灰色分别是 66% 的亮度和 33% 的亮度。事实上，称这些颜色为白色、灰色和黑色不完全正确，因为最初的 Game Boy 的屏幕是绿色，所以玩家看到的颜色实际上是绿色的阴影。

The bit value to color mapping is as follows:
>对颜色映射的“位值”如下：

```
+------+------------+
| 0b11 | 白色      |
| 0b10 | 黑灰      |
| 0b01 | 浅灰      |
| 0b00 | 黑色      |
+------+------------+
```

So each pixel of our 8x8 pixel (i.e., 64 pixels in total) tile will take 2 bits to represent. That means we'll need 64 * 2 or 128 bits total to represent all the pixels. In terms of number of bytes that's 128 / 8 or 16 bytes total as we've said above.
>所以，我们的 8x8 像素（即总共 64 像素）贴图的每个像素将需要占用 2 位。这意味着我们需要 64*2 即 128 位来表示所有像素。换算到字节数是 128/8 即 16 字节，和前面提到的一致。

So this shouldn't be too hard to encode right? Just start from the top left most pixel and every two bits we encode that pixels value right? Unfortunately not. The actual encoding scheme is a little bit more complicated.
>这应该不难编码，对吧？从最左上角的像素开始，每隔两位我们对像素值进行编码？很遗憾，不是这样的，实际的编码方式稍微有点复杂。

Each row of a tile is 2 bytes worth of data (8 pixels with 2 bits per pixel equals 16 bits or 2 bytes). Instead of each pixels value coming one after the other, each pixel is split between the two bytes. So the first pixel is encoded with the left most (i.e., most significant bit a.k.a bit 7) of each byte.
>每个内存块是 2 字节大小（8 个像素，单个占用 2 位，一共是 16 位/ 2 字节）。每个像素值不是一个接一个地出现，而是由两个分隔开的字节组成。因此，第一个像素是用每个字节地最左边（最大的有效位值是 7）表示的。

For example, let's imagine that the first two bytes of our tile set memory were 0xB5 (0b10110101) and 0x65 (0b01100101). These two bytes together will encode the data for the first tile. Byte 1 contains the value of the upper (a.k.a most significant) bit and byte 2 contains the value of the lower (least significant) bit.
>例如，我们假设内存块的第一个两字节是 0xB5 (0b10110101) 和 0x65 (0b01100101)。这两个字节一起给第一个图编码数据。字节 1 包含最高位的值，字节 2 包含最低位的值。 

Let's take a look at how this looks. In the following diagram that colors are represented by 1 letter "B" for black, "D" for dark-gray, "L" for light-gray and "W" for white.
>我们看看具体是怎样的。在下面的图表中，颜色用一个字母表示，B 代表黑色，D 代表 深灰色，L 代表浅灰色，W 代表白色。

```
             Bit 位置
A            7 6 5 4 3 2 1 0
d          +-----------------+
d  0x8000  | 1 0 1 1 0 1 0 1 |
r          |-----------------|
e  0x8001  | 0 1 1 0 0 1 0 1 |
s          +-----------------+
s            D L W D B W B W
                 颜色
```

Since reading the tile data happens much more often than writing it, we can store the tile data internally in a more friendly way.
>因为读取块数据比写入要频繁很多，所以我们以一种更友好的方式在内部存储数据块。

Let's write some code to see what we need. First, we're going to create a new struct that will be responsible for all the graphics needs of the Game Boy. This loosely mimics the set up of actual hardware where the CPU knows nothing about graphics and all. There's no one chip responsible for graphics instead there is dedicated video RAM and the screen hardware. It would over complicate things if we tried to too closely mimic this set up. Instead we'll create the GPU or "Graphic Processing Unit" to model all of our video needs.
>我们先写一部分代码看看需要什么。首先，创建一个结构体，它负责 Game Boy 的所有图形需求。它大致模仿了实际硬件的设置，其中 CPU 不会管理图形和其他处理。没有一个芯片负责图形，而是有专用的视频 RAM 和屏幕硬件。如果我们试图充分地模仿这种硬件，就会增加复杂度。相反，我们会创建 GPU 或“图形处理单元”来模拟所有的图像需求。

For now, our GPU will hold on to video RAM and our tile set data. Our video ram is just a long array which holds on to raw byte values. The tile set will also be an array of tiles. A tile is simply an array of 8 rows where a row is an array of 8 TileValues.
>现在，我们的 GPU 将保存视频和图像数据。我们的视频内存只是一个保存原始字节值的长数组。图像集合也是一个数组。平铺一下就是一个 8 行的数组，其中每一行是包含 8 个值的数组。

```rust
const VRAM_BEGIN: usize = 0x8000;
const VRAM_END: usize = 0x9FFF;
const VRAM_SIZE: usize = VRAM_END - VRAM_BEGIN + 1;

#[derive(Copy,Clone)]
enum TilePixelValue {
    Zero,
    One,
    Two,
    Three,
}

type Tile = [[TilePixelValue; 8]; 8];
fn empty_tile() -> Tile {
    [[TilePixelValue::Zero; 8]; 8]
}

struct GPU{
    vram: [u8; VRAM_SIZE],
    tile_set: [Tile; 384],
}
```

Let's go back to our memory bus to redirect any of writes in memory to our video ram to go to the GPU:
>我们回到内存总线，将内存中的所有数据重定向到视频内存中，以便于 GPU 进行处理：

```rust
# const VRAM_BEGIN: usize = 0x8000;
# const VRAM_END: usize = 0x9FFF;
# struct GPU { }
# impl GPU { fn read_vram(&self,addr: usize) -> u8 { 0 }
             fn write_vram(&self, addr: usize, value: u8) {  } }
# struct MemoryBus { gpu: GPU }

impl MemoryBus {
    fn read_byte(&self, address: u16) -> u8 {
        let address = address as usize;
        match address {
            VRAM_BEGIN ... VRAM_END => {
                self.gpu.read_vram(address - VRAM_BEGIN)
            }
            _ => panic!("TODO: support other areas of memory")
        }
    }

    fn write_byte(&self, address: u16, value: u8) {
        let address = address as usize;
        match address {
            VRAM_BEGIN ... VRAM_END => {
                self.gpu.write_vram(address - VRAM_BEGIN, value)
            }
            _ => panic!("TODO: support other areas of memory")
        }
    }
}
```

Notice how from the MemoryBus we don't directly access the vram but instead go through two methods read_vram and write_vram. This is so we can easily cache our tile set in the tile_set field of our CPU. Let's take a look at how these are implemented.
>注意，在内存总线（MemoryBus）中，我们不直接访问 vram（视频内存），而是通过两个方法 read_vram 和 write_vram 读写。这样我们就可以轻松地将内存片设置缓存到 CPU 的 tile_set 字段中。我们看看具体实现：

read_vram is very simple as it actually just reads from the vram array:
>read_vram 非常简单，因为它实际上只是从 vram 数组中读数据：

```rust
# struct GPU { vram: Vec<u8> }
impl GPU {
  fn read_vram(&self, address: usize) -> u8 {
    self.vram[address]
  }
}
```

However, write_vram is much more involved. Let's take a look at the code and go line by line to see what's happening:
>然而，write_vram 要复杂得多。我们看看代码，逐行分析一下发生了什么：

```rust
# #[derive(Copy,Clone)]
# enum TilePixelValue { Three, Two, One, Zero }
# struct GPU { vram: Vec<u8>, tile_set: [[[TilePixelValue; 8]; 8]; 384]  }
impl GPU {
    fn write_vram(&mut self, index: usize, value: u8) {
        self.vram[index] = value;
        // If our index is greater than 0x1800, we're not writing to the tile set storage
        // so we can just return.
        // 如果索引大于 0x1800，我们将不写入内存块，直接进行返回。
        if index >= 0x1800 { return }

        // Tiles rows are encoded in two bytes with the first byte always
        // on an even address. Bitwise ANDing the address with 0xffe
        // gives us the address of the first byte.
        // For example: `12 & 0xFFFE == 12` and `13 & 0xFFFE == 12`
        // tile 行用两个字节编码，第一个字节总是在偶数地址上。与 0xffe 进行按位和，得到第一个字节的地址。如 `12 & 0xFFFE == 12` and `13 & 0xFFFE == 12`
        let normalized_index = index & 0xFFFE;

        // First we need to get the two bytes that encode the tile row.
        // 首先，我们需要获得用于“平铺行”编码的两个字节。
        let byte1 = self.vram[normalized_index];
        let byte2 = self.vram[normalized_index + 1];

        // A tiles is 8 rows tall. Since each row is encoded with two bytes a tile
        // is therefore 16 bytes in total.
        // 一个 tile 有 8 行的高度。每一行都是用两个字节编码的，所以一个平铺总是 16 个字节。
        let tile_index = index / 16;
        // Every two bytes is a new row
        // 每两个字节组成一个新行
        let row_index = (index % 16) / 2;

        // Now we're going to loop 8 times to get the 8 pixels that make up a given row.
        // 现在，我们要循环 8 次，得到一整行 8 个像素。
        for pixel_index in 0..8 {
            // To determine a pixel's value we must first find the corresponding bit that encodes
            // that pixels value:
            // 1111_1111
            // 0123 4567
            //
            // As you can see the bit that corresponds to the nth pixel is the bit in the nth
            // position *from the left*. Bits are normally indexed from the right.
            //
            // To find the first pixel (a.k.a pixel 0) we find the left most bit (a.k.a bit 7). For
            // the second pixel (a.k.a pixel 1) we first the second most left bit (a.k.a bit 6) and
            // so on.
            //
            // We then create a mask with a 1 at that position and 0s everywhere else.
            //
            // Bitwise ANDing this mask with our bytes will leave that particular bit with its
            // original value and every other bit with a 0.
            let mask = 1 << (7 - pixel_index);
            let lsb = byte1 & mask;
            let msb = byte2 & mask;

            // If the masked values are not 0 the masked bit must be 1. If they are 0, the masked
            // bit must be 0.
            //
            // Finally we can tell which of the four tile values the pixel is. For example, if the least
            // significant byte's bit is 1 and the most significant byte's bit is also 1, then we
            // have tile value `Three`.
            let value = match (lsb != 0, msb != 0) {
                (true, true) => TilePixelValue::Three,
                (false, true) => TilePixelValue::Two,
                (true, false) => TilePixelValue::One,
                (false, false) => TilePixelValue::Zero,
            };

            self.tile_set[tile_index][row_index][pixel_index] = value;
        }

    }
}
```

TODO: other tile set TODO: tile map

We now have a cache of our tile ram so that not only do we have the information directly in VRAM but we also have it in a more accessible format.
>我们现在有了一个缓存块，这样我们不仅可以直接在 VRAM 中获得信息，而且还可以以更方便的格式获取信息。

Next we'll get into the details of rendering.
>接下来我们将讨论渲染的细节部分。
