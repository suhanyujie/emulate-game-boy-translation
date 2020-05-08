# Instructions for Reading and Writting to Memory
* 读写内存的指令

>* 原文链接：https://blog.ryanlevick.com/DMG-01/public/book/
>* 译文出处：https://github.com/suhanyujie/emulate-game-boy-translation
>* 译者：[suhanyujie](https://github.com/suhanyujie)

# 寄存器数据指令
>Instructions for Reading and Writting to Memory 译文

现在我们已经了解指令是如何执行的，以及从内存中获取要读取的指令的基本知识，接下来我们要研究从内存的不同地方读写指令。

## 内存加载
首先，当我们谈论内存的读写时，我们通常使用“加载”这个词。我们把数据从一个地方加载到另一个地方 —— 例如，将寄存器 A 的内容加载到地址为 0xFF0A 的内存中，或者从地址为 0x0040 的内存中加载数据到寄存器 C 中。“加载”不一定是在寄存器和某块内存之间发生，它也可以发生在两个寄存器之间甚至内存的两个区域之间。

我们要查找的所有指令都称为 `LD` 指令。我们将使用 `LoadType` 的枚举类型来区分加载的类型。枚举用于描述我们正在使用的加载类型。

我们来看看 `LD` 指令的实现，它的 `LoadType` 是 `Byte`，它将“一个字节”的内容从一个地方加载到另一个地方。

```rust
fn write_byte(&self, addr: u16, byte: u8) {}
enum LoadByteTarget {
    A, B, C, D, E, H, L, HLI
}
enum LoadByteSource {
    A, B, C, D, E, H, L, D8, HLI
}
enum LoadType {
  Byte(LoadByteTarget, LoadByteSource),
}
enum Instruction {
  LD(LoadType),
}

impl CPU {
  fn execute(&mut self, instruction: Instruction) -> u16 {
    match instruction {
      Instruction::LD(load_type) => {
        match load_type {
          LoadType::Byte(target, source) => {
            let source_value = match source {
              LoadByteSource::A => self.registers.a,
              LoadByteSource::D8 => self.read_next_byte(),
              LoadByteSource::HLI => self.bus.read_byte(self.registers.get_hl()),
              _ => { panic!("TODO: implement other sources") }
            };
            match target {
              LoadByteTarget::A => self.registers.a = source_value,
              LoadByteTarget::HLI => self.bus.write_byte(self.registers.get_hl(), source_value),
              _ => { panic!("TODO: implement other targets") }
            };
            match source {
              LoadByteSource::D8  => self.pc.wrapping_add(2),
              _                   => self.pc.wrapping_add(1),
            }
          }
          _ => { panic!("TODO: implement other load types") }
        }
      }
      _ => { panic!("TODO: support more instructions") }
    }
  }
}
```

For loads with a register as a source, we simply read the register's value. If the source is a `D8` (meaning "direct 8 bit value"), the value is stored directly after the instruction, so we can simply call `read_next_byte` which reads the byte directly after the byte the program counter is currently pointing to. Lastly, if the source is `HLI` we use the value inside of the `HL` register as an address from which we read an 8 bit value from memory.
当把寄存器作为源时，加载时，我们只需读取寄存器的值。而如果源是 `D8`（意思是“直接 8 位值”），那么该值在调用指令后将直接存储起来，因此我们可以简单地调用 `read_next_byte`，它直接读取位于程序计数器当前所指向位置后的字节。最后，如果源是 `HLI`，我们使用 `HL` 寄存器作为地址，从其中的内存中读取 8 位数据值。

The target is merely the reverse of the source (except that we can't have `D8` as a target). If the target is a register, we write the source value into that register, and if the target is `HLI` we write to the address that is stored inside of the `HL` register.
目标仅仅是源的反向（除非我们不能将 `D8` 作为目标）。如果目标是一个寄存器，我们将来源中的值写入该寄存器，如果目标是 `HLI`，我们将写到存储在 `HL` 寄存器内的地址中。

使用 16 位寄存器 `BC`、`DE` 和 `HL` 来存储地址是非常常见的。

我们看看其他类型的加载：

* `Word`: just like the `Byte` type except with 16-bit values
* `Word`: 就像 `Byte` 类型一样，只是有 16 位的值
* `AFromIndirect`: load the A register with the contents from a value from a memory location whose address is stored in some location
* `AFromIndirect`: 从内存位置（地址存储在某个位置）加载包含值的内容的寄存器
* `IndirectFromA`: load a memory location whose address is stored in some location with the contents of the A register
* `IndirectFromA`: 加载一个内存位置，其地址与寄存器的内容一起存储在某个位置
* `AFromByteAddress`: Just like `AFromIndirect` except the memory address is some address in the very last byte of memory.
* `AFromByteAddress`: 就像 `AFromIndirect`，只不过内存地址是内存的最后一个字节的某个地址
* `ByteAddressFromA`: Just like `IndirectFromA` except the memory address is some address in the very last byte of memory.
* `ByteAddressFromA`: 类似于 `IndirectFromA`，只不过内存地址是内存的最后一个字节的某个地址。

有关这些说明的更详细内容，可以参考[说明指南](https://blog.ryanlevick.com/DMG-01/public/book/appendix/instruction_guide/index.html)。

这些指令用于写入和写入内存中的任意位置，但是有一组指令是专门处理栈的内存段。我们下面看看什么是栈，以及操作栈的指令有哪些。

## 栈
在查看 Game Boy 中被称为栈的内存区域之前，我们要更好的理解堆栈是什么。简单来讲，堆栈是一个简单的数据结构，你可以向其中添加值（例如将值”push”进去），然后把这些值取出（例如将值“pop”出来）。记住堆栈的关键点是出栈和入栈的顺序是相反的，例如，如果降三个项目“A”，“B”，“C”按顺序送入栈中，取出时的顺序则是“C”，“B”，“A”。

The Game Boy CPU has built in support for a stack like data structure in memory. This stack lives somewhere in memory (we'll talk about how it's location in memory is set in just a minute), and it holds on to 16 bit values. How is it built?
Game Boy 的 CPU 已经对内存中栈数据结构建立了支持。该堆栈在内存中的某个位置中（我们将讨论在一分钟内设置它的位置），并且它的值是 16 位的。那么是如何建立支持的呢？

首先 CPU 上有一个额外的 16 位寄存器，它指向栈的顶部。这个寄存器叫做 `SP` 或者堆栈指针，因为它指向堆栈顶部位置。我们先将这个寄存器加到 CPU 中：

```rust
struct CPU {
  registers: Registers,
  pc: u16,
  sp: u16,
  bus: MemoryBus,
}
```

现在我们有了栈指针，也就知道了栈所在的位置，但我们如何从这个栈中 push 和 pop 数据呢？

Game Boy 的 CPU 拥有这两个指令的功能。`PUSH` 会把 16 位寄存器的内容写入栈中，`POP` 会将数据从栈顶写入 16 位寄存器中。

Here's what's actually happening when a `PUSH` is performed:
下面是当你 `PUSH` 数据时所发生的的：

* 栈指针减 1。
* 将 16 位值中高位有效字节写到当前栈指针指向的内存中。
* 栈指针再次 _减_ 1.
* 将 16 位数据的低位字节写入到当前栈指针指向的内存中

注意，栈指针减 1 而不是加 1。这是因为栈地址在内存中是向下增长的。这一点很重要，因为栈的正常位置位于内存的末端。在后面的章节中，我们将看到 Game Boy 的引导 ROM 将栈指针设到内存的末尾。因此，当栈递增时，它会从内存的末端增长到内存的起始端。

实现一下 `PUSH`:

```rust
impl CPU {
  fn execute(&mut self, instruction: Instruction) -> u16 {
    match instruction {
      Instruction::PUSH(target) => {
        let value = match target {
          StackTarget::BC => self.registers.get_bc(),
          _ => { panic!("TODO: support more targets") }
        };
        self.push(value);
        self.pc.wrapping_add(1)
      }
      _ => { panic!("TODO: support more instructions") }
    }
  }

  fn push(&mut self, value: u16) {
    self.sp = self.sp.wrapping_sub(1);
    self.bus.write_byte(self.sp, ((value & 0xFF00) >> 8) as u8);

    self.sp = self.sp.wrapping_sub(1);
    self.bus.write_byte(self.sp, (value & 0xFF) as u8);
  }
}
```

我们现在可以将值 push 到栈中。下面是 `PUSH` 操作发生时的流程：

* 从栈指针指向的内存中的 16 位值中读取低有效位的字节。
* 栈指针 _加_ 1
* 从栈指针指向的内存中的 16 位值中读取高有效位的字节。
* 栈指针再次 _加_ 1
* 返回高有效位和低有效位组合在一起的值。

我们实现一下 `POP`：

```rust
impl CPU {
  fn execute(&mut self, instruction: Instruction) -> u16 {
    match instruction {
      Instruction::POP(target) => {
        let result = self.pop();
        match target {
            StackTarget::BC => self.registers.set_bc(result),
            _ => { panic!("TODO: support more targets") }
        };
        self.pc.wrapping_add(1)
      }
      _ => { panic!("TODO: support more instructions") }
    }
  }

  fn pop(&mut self) -> u16 {
    let lsb = self.bus.read_byte(self.sp) as u16;
    self.sp = self.sp.wrapping_add(1);

    let msb = self.bus.read_byte(self.sp) as u16;
    self.sp = self.sp.wrapping_add(1);

    (msb << 8) | lsb
  }
}
```

好了！我们现在有了可以使用的栈了。但是它到底用来做什么呢？一个内建的栈是用于创建一个“调用”栈，它可以让游戏“调用”函数并从中返回一些值。我们看看它是如何工作的。

## 函数调用
在大多数编程语言中，当你调用一个函数时，调用函数的状态会被保存在某个地方，以让对应的函数执行，然后当被调用函数返回时，被调用函数的状态会恢复。所以 Game Boy 也内置了对这种机制的支持，其中保存的状态就是调用函数时程序计数器的状态。这意味着“调用一个函数”中的函数还可以继续调用其他函数，当所有这些调用执行完成后，将会回到一开始调用函数时的地方。

这个功能是由两种类型的指令来处理的：`CALL` 和 `RET`（即“返回”）。其中，`CALL` 的工作方式是使用我们已知的 `PUSH` 和 `JP`（即 jump）指令。要执行 `CALL` 指令，我们必须执行以下操作：

* 将下一个程序计数器（例如没有跳转，我们就会得到该程序计数器）`PUSH` 到栈中
* `JP`（即“跳转”）到下一个内存字节指向的地址上（即函数）

就是这样！我们调用了函数。但是当我们在调用的函数中执行 `RET`（即返回）指令时会发生什么呢？

* 从栈中 `POP` 出下一个程序计数器，并跳转到它指向的位置。

这很简单！我们用代码实现它：

```rust
impl CPU {
  fn execute(&mut self, instruction: Instruction) -> u16 {
    match instruction {
      Instruction::CALL(test) => {
          let jump_condition = match test {
              JumpTest::NotZero => !self.registers.f.zero,
              _ => { panic!("TODO: support more conditions") }
          };
          self.call(jump_condition)
      }
      Instruction::RET(test) => {
          let jump_condition = match test {
              JumpTest::NotZero => !self.registers.f.zero,
              _ => { panic!("TODO: support more conditions") }
          };
          self.return_(jump_condition)
      }
      _ => { panic!("TODO: support more instructions") }
    }
  }

  fn call(&mut self, should_jump: bool) -> u16 {
    let next_pc = self.pc.wrapping_add(3);
    if should_jump {
      self.push(next_pc);
      self.read_next_word()
    } else {
      next_pc
    }
  }

  fn return_(&mut self, should_jump: bool) -> u16 {
    if should_jump {
      self.pop()
    } else {
      self.pc.wrapping_add(1)
    }
  }
}
```

现在我们可以很容易地调用函数并从中返回数据了。至此，我们已经完成绝大部分的 CPU 指令了！
