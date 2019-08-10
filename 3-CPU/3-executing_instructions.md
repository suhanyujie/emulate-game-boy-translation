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

When the user of a Game Boy inserts a game cartridge, the contents of that cartridge become available to the CPU right after these 255 bytes. We'll talk later about where other things such as the contents of the screen and graphics data live in memory later in the book. For now we just need to know that the contents of memory starting at index 0x100 until index 0x3FFF include the contents of the cartridge.
当 Game Boy 用户插入了一个“游戏卡”时，CPU 执行完这 255 字节的指令后，该游戏卡的内容对 CPU 来讲也是可执行的。我们稍后将在书中讨论如屏显和图形数据等内容在内存中存储的位置。现在，我们只需要知道游戏卡中从 0x100 开始到 0x3FFF 的内存包含的内容。

所以我们的内存是简单的 8 位整形长数组（确切的说是其中的 0xFFFF 或 65,536）。这些数字中的每个都能被解码为 CPU 能“懂”的指令。但是 CPU 如何判断执行哪一个呢？

## 程序计数器
除了寄存器数据，我们的 CPU 还保存了一个 16 位的数字，叫做“程序计数器”（通常缩写为 PC），它告诉 Game Boy 当前正在执行哪条指令。这个 16 位数字能够寻址到内存中的 0xFFFF 数据。事实上，我们谈到内存数组时，我们通常不使用“索引”这个词，而是使用“地址”这个词。

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

现在我们有了程序计数器，它可以告诉我们当前执行的指令在内存中的哪个地址。到本书的后面，我们不会再过多地讨论内存的内容或者内存中存储的数据。现在你应该将内存描绘成一个大数组，并且我们可以从中读取数据。

现在我们需要给 CPU 添加方法，也就是 CPU 使用计数器从内存中读取指令并执行。

主要设置步骤如下:
    - 使用程序计数器在内存中读取指令字节
    - 将字节转换为 `Instruction` 枚举中的某个实例。
    - 如果转换成功，指令调用 `execute` 方法，否则就 panic，并返回下一个程序计数器
    - 把下一个程序计数器设定给 CPU 

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

因此，我们需要添加方法来完成上面提到的工作。我们需要更改执行方法来返回下一个程序计数器，我们还需要添加一个函数，该函数接收一个字节的参数并返回一个 `Instruction`。我们从后者开始。将指令字节解码为一个 `Instruction` 不难。指令是通过字节数值表示的唯一标识。例如，`A` 寄存器的逻辑或（`OR`）由字节 0x87 作为标识。要实现 `H` 寄存器的逻辑或（`OR`）该怎么做呢？结果是 0xB4。`SCF`（或 Set Carry Flag）指令由字节 0x37 作为标识。我们可以使用 [instruction guide](https://blog.ryanlevick.com/DMG-01/public/book/appendix/instruction_guide/index.html) 来找出字节值对应哪个 `Instruction`。

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

And now let's change our execute method so that it now returns the next program counter:

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
          _ => { /* TODO: support more targets */ self.pc }
        }
      }
      _ => { /* TODO: support more instructions */ self.pc }
    }
  }
}
```

Now we have the ability to read the instruction byte from memory that's pointed to by our program counter, decode that instruction byte as one of the variants of our `Instruction` enum, execute that instruction and get back the new program counter and finally set the new program counter on our CPU. This is how all instructions in the Game Boy get executed! Well, except...

## Prefix Instructions
The process we've laid out for how instructions get executed is true for roughly half of the total instructions the Game Boy can perform. The other half of instructions work the same way except that instead of being identified by a single byte they're first indentified by a prefix byte. This prefix byte tells the CPU, "Hey! The next instruction byte you read shouldn't be interpreted as a normal instruction, but rather as a prefix instruction".

This prefix byte is the number "0xCB". So, we'll need to add logic that first checks to see if the byte we read from memory is 0xCB. If it is, we then need to read one more byte and interpret this byte as an "prefix instruction". For example, if we read 0xCB from memory, we know that we're going to be decoding a prefix instruction. We then read another byte. If that byte is, say, 0xB4, we should not interpret this as `OR` with `H` as the target like we normally would but rather as a `RES` instruction with the 6th bit of the `H` register as the target. Again we can use the [instruction guide](https://blog.ryanlevick.com/DMG-01/public/book/appendix/instruction_guide/index.html) to help us know what a given byte should decode as.

Let's put it in code!

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
      _ => /* TODO: Add mapping for rest of instructions */ None
    }
  }

  fn from_byte_not_prefixed(byte: u8) -> Option<Instruction> {
    match byte {
      0x02 => Some(Instruction::INC(IncDecTarget::BC)),
      _ => /* TODO: Add mapping for rest of instructions */ None
    }
  }
}
```

The amount the program counter goes forward after each step of execution is determined by how "wide" the instruction - i.e. how many bytes it takes to describe the instruction in its entirety. For simple instructions, this is one byte - the byte the uniquely identifies the instruction. So far all the instructions we've seen either are 1 or 2 bytes wide (prefix instructions are two bytes - the prefix and the instruction identifier - while the other instructions are only one 1 byte - just for indentifier). In the future we'll see other instructions which have "operands" or data the instruction needs to execute. These instructions can sometimes be even 3 bytes wide.

However, the program counter doesn't have to go forward by a set amount. In fact, there are instructions that manipulate the program counter in arbitrary ways sometimes sending the program counter to somewhere far away from its previous location.

## Jump Instructions
The real power of computers are their ability to "make decisions" - i.e., do one thing given one condition and do another thing given another condition. At the hardware level this is usually implemented with "jumps" or the ability to change where in our program we are (as indicated by the program counter) based on certain conditions. In the case of the Game Boy's CPU these conditions are specificed by the flags register. For example, there is an instruction that says to "jump" (i.e., set the program counter) to a certain location if the flags register's zero flag is true. This gives the game a way to perform certain instructions and then change to different parts of the game code if the result of the instruction resulted in setting particular flags. Let's list out the types of jumps there:

    - JP: Jump to a particular address dependent on one of the following conditions: the zero flag is true, the zero flag is flase, the carry flag is true, the carry flag is false, or always jump.
    - JR: Jump a certain amount relative to the current program counter dependent on the same conditions above.
    - JPI: Jump to the address stored in HI

You can find the specifics of how these jump instructions work in the [instruction guide](https://blog.ryanlevick.com/DMG-01/public/book/appendix/instruction_guide/index.html).

Implementation of jump is pretty trivial:

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
      // Gameboy is little endian so read pc + 2 as most significant bit
      // and pc + 1 as least significant bit
      let least_significant_byte = self.bus.read_byte(self.pc + 1) as u16;
      let most_significant_byte = self.bus.read_byte(self.pc + 2) as u16;
      (most_significant_byte << 8) | least_significant_byte
    } else {
      // If we don't jump we need to still move the program
      // counter forward by 3 since the jump instruction is
      // 3 bytes wide (1 byte for tag and 2 bytes for jump address)
      self.pc.wrapping_add(3)
    }
  }
}
```

It's important to note that the address we jump to is located in the two bytes following the instruction identifier. As the comment in the code example explains, the Game Boy is little endian which means that when you have numbers that are larger than 1 byte, the least significant is stored first in memory and then the most significant byte.

```other
+-------------+-------------- +--------------+
| Instruction | Least Signif- | Most Signif-
| Identifier  | icant Byte    | icant Byte   |
+-------------+-------------- +--------------+
```

We're now succesfully executing instructions that are stored in memory! We learned that the current executing instruction is kept track of by the program counter. We then read the instruction from memory and execute it, getting back our next program counter. With this, we were even able to add some new instructions that let the game conditionally control exactly where the next program counter will be. Next we'll look at bit closer at instructions that read and write to memory.
