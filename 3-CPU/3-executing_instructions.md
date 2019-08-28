# Program Counter
* 程序计数器

>* 原文链接 https://blog.ryanlevick.com/DMG-01/public/book/
>* 译文出处 https://github.com/suhanyujie/emulate-game-boy-translation

# 寄存器数据指令
>Instructions on Register Data - Instruction Execution and Control Flow 译文

目前为止，我们知道了可以操作寄存器数据的指令了。但这些指令在 CPU 上如何执行呢？要理解这个，我们首先需要明白我们的指令存储在哪儿？

## Game ROM
目前我们知道了 Game Boy 拥有一个执行指令的 CPU 和内存。内存可以看做是一个非常大的 8 比特位数值的数组。

这个大数组的开头是 255 字节（从 0x0000 到 0x00FF），这些字节被硬编码到 Game Boy 的物理电路中。这 255 字节的内容就是告诉 Game Boy 如何启动的指令（例如，它已经准备好启动游戏）并且显示 [标志性的屏显](https://www.youtube.com/watch?v=ClJWTR_lCL4)。后面本书会详细说明这些指令所做的东西，但现在把它们看做一个指令集合，其中很多是我们在前一章学过的，其余的，我们将在接下来的章节中学习到。

当 Game Boy 用户插入了一个“游戏卡”时，CPU 执行完这 255 字节的指令后，该游戏卡的内容对 CPU 来讲也是可执行的。我们稍后将在书中讨论如屏显和图形数据等内容在内存中存储的位置。现在，我们只需要知道游戏卡中从 0x100 开始到 0x3FFF 的内存包含的内容。

所以我们的内存是简单的 8 位整形长数组（确切的说是其中的 0xFFFF 或 65,536）。这些数字中的每个都能被解码为 CPU 能“懂”的指令。但是 CPU 如何判断执行哪一个呢？

## 程序计数器
除了寄存器数据，我们的 CPU 还保存了一个 16 位的数值，叫做“程序计数器”（通常缩写为 PC），它告诉 Game Boy 当前正在执行哪条指令。这个 16 位数值能够寻址到内存中的 0xFFFF 数据。事实上，我们谈到内存数组时，我们通常不使用“索引”这个词，而是使用“地址”这个词。

我们添加一个程序计数器到 CPU 以及可以用于在 CPU 中寻址的内存。

```rust
struct CPU {
  registers: Registers,
  pc: u16,
  bus: MemoryBus,
}

struct MemoryBus {
  memory: [u8; 0xFFFF]
}

impl MemoryBus {
  fn read_byte(&self, address: u16) -> u8 {
    self.memory[address as usize]
  }
}
```

现在我们有了程序计数器，它可以告诉我们当前执行的指令在内存中的哪个地址。到本书的后面，我们不会再过多地讨论内存的内容或者内存中存储的数据。现在你应该将内存想象成一个大数组，并且我们可以从中读取数据。

现在我们需要给 CPU 添加方法，也就是 CPU 使用计数器从内存中读取指令并执行。

主要设置步骤如下:
    - 使用程序计数器在内存中读取指令字节
    - 将字节转换为 `Instruction` 枚举中的某个实例。
    - 如果转换成功，指令调用 `execute` 方法，否则就 panic，并返回下一个程序计数器
    - 把下一个程序计数器设定到 CPU 中

```rust
impl CPU {
  fn step(&mut self) {
    let mut instruction_byte = self.bus.read_byte(self.pc);

    let next_pc = if let Some(instruction) = Instruction::from_byte(instruction_byte) {
      self.execute(instruction)
    } else {
      panic!("Unkown instruction found for: 0x{:x}", instruction_byte);
    };

    self.pc = next_pc;
  }
}
```

因此，我们需要添加方法来完成上面提到的工作。我们需要更改执行方法来返回下一个程序计数器，我们还需要添加一个函数，该函数接收一个字节的参数并返回一个 `Instruction`。我们从后者开始。将指令字节解码为一个 `Instruction` 不难。指令是通过字节数值表示的唯一标识。例如，`A` 寄存器的逻辑或（`OR`）由字节 0x87 作为标识。要实现 `H` 寄存器的逻辑或（`OR`）该怎么做呢？结果是 0xB4。`SCF`（或 Set Carry Flag）指令由字节 0x37 作为标识。我们可以使用“[指令指南](https://blog.ryanlevick.com/DMG-01/public/book/appendix/instruction_guide/index.html)”来找出字节值对应哪个 `Instruction`。

```rust
impl Instruction {
  fn from_byte(byte: u8) -> Option<Instruction> {
    match byte {
      0x02 => Some(Instruction::INC(IncDecTarget::BC)),
      0x13 => Some(Instruction::INC(IncDecTarget::DE)),
      _ => /* TODO: 给其余的指令增加映射 */ None
    }
  }
}
```

现在我们改变一下执行方法，让它返回下一个程序计数器：

```rust
impl CPU {
  fn execute(&mut self, instruction: Instruction) -> u16 {
    match instruction {
      Instruction::ADD(target) => {
        match target {
          ArithmeticTarget::C => {
            let value = self.registers.c;
            let new_value = self.add(value);
            self.registers.a = new_value;
            self.pc.wrapping_add(1)
          }
          _ => { /* TODO: 支持更多的目标 */ self.pc }
        }
      }
      _ => { /* TODO: 支持更多的指令 */ self.pc }
    }
  }
}
```

现在我们有能力从内存中读取指令，该指令指向我们的计数器，解码该指令字节作为 `Instruction` 枚举的变体，执行指令，返回新的程序计数器，最后在 CPU 上设定新的程序计数器。以上这些，就是描述 Game Boy 中的指令是如何执行的！除了 ……

## 前缀指令
我们已经正确列出 Game Boy 中所能执行的一半的指令，以及这些指令是如何执行的。另一半指令的工作方式也是类似的，只是它们不是由单个字节标识的，而是先由“前缀字节”标识。这个前缀字节告诉 CPU：嘿！你读取的下一个指令字节不应该作为普通指令来执行，而应作为“前缀指令”来执行。

这个前缀字节是数值“0xCB”。因此，我们需要添加“先从内存中检查读取的字节是否为 0xCB”的逻辑。如果是，那么我们需要再读取一个字节，并将这个字节解释为“前缀指令”。例如，如果我们从内存中读取 0xCB，我们知道将会解码一个“前缀指令”。然后读取另一个字节。如果那个字节是 0xB4，这时候我们就不应该像通常情况下地将其解释为以 `H` 为目标的 `OR` 指令，而应该解释为 `RES` 指令，以 `H` 寄存器的第 6 位作为目标。同样，我们可以使用“[指令指南](https://blog.ryanlevick.com/DMG-01/public/book/appendix/instruction_guide/index.html)”来辅助我们了解给定字节的解码的映射。

我们来用代码实现！

```rust
impl CPU {
  fn step(&mut self) {
    let mut instruction_byte = self.bus.read_byte(self.pc);
    let prefixed = instruction_byte == 0xCB;
    if prefixed {
      instruction_byte = self.bus.read_byte(self.pc + 1);
    }

    let next_pc = if let Some(instruction) = Instruction::from_byte(instruction_byte, prefixed) {
      self.execute(instruction)
    } else {
      let description = format!("0x{}{:x}", if prefixed { "cb" } else { "" }, instruction_byte);
      panic!("Unkown instruction found for: {}", description)
    };

    self.pc = next_pc;
  }
}

impl Instruction {
  fn from_byte(byte: u8, prefixed: bool) -> Option<Instruction> {
    if prefixed {
      Instruction::from_byte_prefixed(byte)
    } else {
      Instruction::from_byte_not_prefixed(byte)
    }
  }

  fn from_byte_prefixed(byte: u8) -> Option<Instruction> {
    match byte {
      0x00 => Some(Instruction::RLC(PrefixTarget::B)),
      _ => /* TODO: 给其他指令增加映射 */ None
    }
  }

  fn from_byte_not_prefixed(byte: u8) -> Option<Instruction> {
    match byte {
      0x02 => Some(Instruction::INC(IncDecTarget::BC)),
      _ => /* TODO: 给其他指令增加映射 */ None
    }
  }
}
```

程序计数器所执行的每一次“步进”由指令的“宽度”决定 —— 即需要多少字节来描述整个指令。对于简单的指令，一个字节足矣 —— 唯一标识指令的字节。到目前为止，我们看到的所有指令都是 1 或 2 字节宽（前缀指令是两个字节 —— 前缀和指令标识 —— 而其他指令只有一个字节 —— 仅用于标识）。在后面，我们将看到其他类型的指令，其中有“操作数”或指令需要执行的数据。这些指令有时甚至可以达到 3 个字节宽。

然而，程序计数器不需要向前移动固定的步长。事实上，有一些指令以任意方式操纵程序计数器，有时会将程序计数器发送到离原先位置很远的某个位置。

## 跳转指令
计算机的真正力量是它们“做决定”的能力，也就是说，它们能够“做决定”。在给定的一个条件下做事情，在另一个条件下做另一件事。在硬件上来讲，这通常是通过“跳转”实现的，或者根据特定的条件改变程序中的位置（程序计数器指示的位置）。对于 Game Boy 的 CPU 来说，这些条件由标志寄存器指定。例如，有一个指令说“跳转”，如果其标志寄存器的零标志位为真，则设定程序计数器到某个位置。这给了游戏一种执行特定指令的方式，如果指令的结果导致设置特殊的标志，则改变了游戏代码的不同部分。我们列出了跳转的类型：

- JP：跳转到特定地址，取决于以下条件：零标志位为 true，零标志位为 false，进位标志位为 true， 进位标志位为 false，或者一直跳转。
- JR：根据上述的条件，相对于当前程序计数器跳转一定的步长。
- JPI：跳转到存储在 HI（寄存器）中的地址

你能在“[指令指南](https://blog.ryanlevick.com/DMG-01/public/book/appendix/instruction_guide/index.html)”找到这些跳转指令如何工作的详细资料

“跳转”的实现比较简单：

```rust
enum JumpTest {
  NotZero,
  Zero,
  NotCarry,
  Carry,
  Always
}
enum Instruction {
  JP(JumpTest),
}

impl CPU {
  fn execute(&mut self, instruction: Instruction) -> u16 {
    match instruction {
      Instruction::JP(test) => {
        let jump_condition = match test {
            JumpTest::NotZero => !self.registers.f.zero,
            JumpTest::NotCarry => !self.registers.f.carry,
            JumpTest::Zero => self.registers.f.zero,
            JumpTest::Carry => self.registers.f.carry,
            JumpTest::Always => true
        };
        self.jump(jump_condition)
      }
      _ => { /* TODO: support more instructions */ self.pc }
    }
  }

  fn jump(&self, should_jump: bool) -> u16 {
    if should_jump {
      // Gameboy 是小端字节序，所以读取 pc + 2 作为最重要的部分，并且 pc + 1 的位置作为不太重要的部分
      let least_significant_byte = self.bus.read_byte(self.pc + 1) as u16;
      let most_significant_byte = self.bus.read_byte(self.pc + 2) as u16;
      (most_significant_byte << 8) | least_significant_byte
    } else {
      // 如果我们不跳转，我们仍然需要将程序计数器向前移动 3 个字节，因为跳转指令的宽度为 3 字节（1个字节标识，2个字节跳转地址）
      self.pc.wrapping_add(3)
    }
  }
}
```

需要注意的是，我们跳转到的地址位于指令标识符后面的两个字节中。正如代码示例中的注释所解释的，Game Boy 是小端字节序，这意味着当数字大于 1 个字节的范围时，内存段中开头的部分是最不重要的部分，然后才是最重要的字节部分。

```other
+-------------+-------------- +--------------+
| Instruction | Least Signif- | Most Signif-
| Identifier  | icant Byte    | icant Byte   |
+-------------+-------------- +--------------+
```

我们现在成功执行了存储在内存中的指令！我们了解到了，当前指令由程序计数器跟踪。然后我们从内存中读取指令并执行，然后得到下一个程序计数器。有了这个，我们甚至能够添加一些新的指令，让游戏有条件地精确控制下一个程序计数器的位置。接下来，我们将进一步研究读写内存的指令。
