>* 原文链接 https://blog.ryanlevick.com/DMG-01/public/book/
>* 译文出处 https://github.com/suhanyujie/emulate-game-boy-translation

# CPU 寄存器
在前一章中，我们概述了 CPU 的一些职责。在本节中，我们只探讨其中之一：将数据保存到寄存器中。

## 概述
Game Boy 的 CPU 是专门为 Game Boy 定制的。该芯片非常类似于 [Intel 8080](https://en.wikipedia.org/wiki/Intel_8080)，而 Intel 8080 也非常类似于 [Zilog Z80](https://en.wikipedia.org/wiki/Zilog_Z80)。在 70 年代和 80 年代中，Intel 8080 和 Zilog Z80 常被用于许多不同的计算机设备，而 Game Boy 内部的芯片只是用于 Game Boy。关于 8080 和 Z80 的工作原理大部分也适用于 Game Boy 的芯片。在这里不会详细讨论它们之间的区别，但需要知道，虽然它们与 Game Boy 的芯片很类似，但也很不同。

## 寄存器
该 CPU 中包含了 8 个不同的“寄存器”。寄存器负责保存 CPU 在执行各种指令时可以操作的小块数据。Game Boy 的 CPU 是一个 8 位的 CPU，这意味着它的每个寄存器可以容纳 8 位（即 1 字节）。这里讲 8 个不同的寄存器标识为“a”，“b”，“c”，“d”，“e”，“f”，“g”，“h”，“l”。

我们通过指定寄存器的代码开始构建 CPU：

```rust
struct Registers {
  a: u8,
  b: u8,
  c: u8,
  d: u8,
  e: u8,
  f: u8,
  h: u8,
  l: u8,
}
```

我们设定寄存器的类型是 `u8`。它代表的是 8 位无符号整数。如果要了解数字如何存储在计算机中，请阅读[数字指南](https://blog.ryanlevick.com/DMG-01/public/book/cpu/appendix/numbers.html)。

虽然 CPU 只有 8 位的寄存器，但是有一些指令需要游戏同时（Rust 中表示为 `u16`—— 一个 16 位无符号整数）读写 16 位数据（也就是 2 字节）。因此，我们需要能够读写“虚拟”的 16 位寄存器。这些寄存器被引用为“af”（“a”和“f”组合）、“bc”（“b”和“c”组合）、“de”（“d”和“e”组合），还有“hl”（“h”和“l”的组合）。我们来实现一下“bc”吧：

```rust
struct Registers { a: u8, b: u8, c: u8, d: u8, e: u8, f: u8, h: u8, l: u8, }
impl Registers {
  fn get_bc(&self) -> u16 {
    (self.b as u16) << 8
    | self.c as u16
  }

  fn set_bc(&mut self, value: u16) {
    self.b = ((value & 0xFF00) >> 8) as u8;
    self.c = (value & 0xFF) as u8;
  }
}
```

这里我们看到第一个“位操作”实例，它可以使用四个位操作符：“>>”，“<<”，“&”以及“|”。如果你不熟悉这类操作符，可以查看[位操作符指南](https://blog.ryanlevick.com/DMG-01/public/book/cpu/appendix/bit_manipulation.html)。

为了读取“bc”寄存器，我们需要先将“b”寄存器视为 `u16`（实际上只是将一字节的 0 值添加到高数值位）。然后我们移动“b”寄存器的 8 个位置，让它占据高字节的位置。最后，我们将其与“c”寄存器中的值进行按位或操作。结果是一个双字节数，其中“b”的内容位于高字节位，“c”的内容位于低字节位。

## 标志寄存器
寄存器讲的差不多了，但是为了便于后面使用，我们可以改进一下寄存器。“f”寄存器是一种特殊的寄存器，称为“标志”寄存器。这个寄存器的低四位总是 0 值，在一些情况下，CPU 自动写入其中高四位。换句话说，CPU 进行状态“标识”。我们现在不讨论这个细节，只需知道其中的位置和对应的作用：
    * Bit 7: “零值”（zero）
    * Bit 6: “减法，差集”（subtraction）
    * Bit 5: “半进位”（half carry）
    * Bit 4: “进位”（carry）

下面是标志寄存器的示意图：

```other
   ┌-> Carry
 ┌-+> Subtraction
 | |
1111 0000
| |
└-+> Zero
  └-> Half Carry
```

因此，虽然我们可以仍然将“标志寄存器”建模为简单的 8 位数值（毕竟，真实情况也是如此），但对高 4 位（就是半个字节）设定特定的含义并让低 4 位（就是半个字节）保持为 0 值的建模更不容易出错。

因此，我们将创建一个 `FlagsRegister` 结构类型： 

```rust
struct FlagsRegister {
    zero: bool,
    subtract: bool,
    half_carry: bool,
    carry: bool
}
```

由于我们需要把这个寄存器看作为一个 8 位数值，所以我们可以利用标准库中的一些 trait 来简化这个过程：

```rust
struct FlagsRegister {
   zero: bool,
   subtract: bool,
   half_carry: bool,
   carry: bool
}
const ZERO_FLAG_BYTE_POSITION: u8 = 7;
const SUBTRACT_FLAG_BYTE_POSITION: u8 = 6;
const HALF_CARRY_FLAG_BYTE_POSITION: u8 = 5;
const CARRY_FLAG_BYTE_POSITION: u8 = 4;

impl std::convert::From<FlagsRegister> for u8  {
    fn from(flag: FlagsRegister) -> u8 {
        (if flag.zero       { 1 } else { 0 }) << ZERO_FLAG_BYTE_POSITION |
        (if flag.subtract   { 1 } else { 0 }) << SUBTRACT_FLAG_BYTE_POSITION |
        (if flag.half_carry { 1 } else { 0 }) << HALF_CARRY_FLAG_BYTE_POSITION |
        (if flag.carry      { 1 } else { 0 }) << CARRY_FLAG_BYTE_POSITION
    }
}

impl std::convert::From<u8> for FlagsRegister {
    fn from(byte: u8) -> Self {
        let zero = ((byte >> ZERO_FLAG_BYTE_POSITION) & 0b1) != 0;
        let subtract = ((byte >> SUBTRACT_FLAG_BYTE_POSITION) & 0b1) != 0;
        let half_carry = ((byte >> HALF_CARRY_FLAG_BYTE_POSITION) & 0b1) != 0;
        let carry = ((byte >> CARRY_FLAG_BYTE_POSITION) & 0b1) != 0;

        FlagsRegister {
            zero,
            subtract,
            half_carry,
            carry
        }
    }
}
```

`std::convert::From` trait 可以让我们轻松地将 FlagsRegister 从 `u8` 进行来回转换

既然我们有了特殊的 `FlagsRegister`，可以将 `Registers` 结构体中的“f”字段的 `u8` 进行替换。

就是这样！我们拥有寄存器所需的所有功能。接下来，我们研究寄存器的不同指令。
