# Instructions for Reading and Writting to Memory
* 读写内存的指令

>* 原文链接 https://blog.ryanlevick.com/DMG-01/public/book/
>* 译文出处 https://github.com/suhanyujie/emulate-game-boy-translation

# 寄存器数据指令
>Instructions for Reading and Writting to Memory 译文

Now that we've seen how instructions get executed and the very basics of reading from memory in order to fetch the instructions to be read, we'll look now at instructions that are used to read and write from different parts of memory.
现在我们呢已经了解了指令是如何执行的，以及从内存中获取要读取的指令的基本知识，接下来我们要研究从内存的不同地方度读写指令。

## 内存加载
First, when we talk about reading and writing memory, we usually use the term "load". We'll be loading data from some place to some place - for example, loading the contents of register A into memory at location 0xFF0A or loading register C with the contents from memory location 0x0040. Loading doesn't have to be between a register and a place in memory, it can also be between two registers or even two places in memory.
首先，当我们谈论内存的读写时，我们通常使用“加载”这个词。我们把数据从一个地方加载到另一个地方 —— 例如，将寄存器 A 的内容加载到地址为 0xFF0A 的内存中，或者从地址为 0x0040 的内存中加载数据到寄存器 C 中。“加载”不一定是寄存器和某个内存之间发生，它可以发生在两个寄存器之间甚至内存中的两个区域。

All of the instructions we'll be looking are called `LD` instructions. We'll be differentiating between the types of loads with the `LoadType` enum. The enum will describe what kind of load we're doing.
我们看到的所有指令都成为 `LD` 指令。我们将使用 `LoadType` 的枚举类型来区分加载的类型。枚举将描述我们正在使用的加载类型。

Let's take a look at the implementation of the `LD` instruction with the `LoadType` of `Byte` which loads a byte from one place to another.
我们先看看 `LD` 指令的实现，它的 `LoadType` 是 `Byte`，它将“一个字节”的内容从一个地方加载到另一个地方。

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
当把寄存器作为源的情况，加载时，我们只需读取寄存器的值。而如果源是 `D8`（意思是“直接 8 位值”），那么该值在调用指令后将直接存储起来，因此我们可以简单地调用 `read_next_byte`，它在程序计数器当前指向的字节后直接读取字节。最后，如果源是 `HLI`，我们使用 `HL` 寄存器作为地址，从内存中读取 8 位值。

The target is merely the reverse of the source (except that we can't have `D8` as a target). If the target is a register, we write the source value into that register, and if the target is `HLI` we write to the address that is stored inside of the `HL` register.
目标仅仅是源的反面（除非我们不能将 `D8` 作为目标）。如果目标是一个寄存器，我们将源中的值写入该寄存器，如果目标是 `HLI`，我们呢将写入 `HL` 寄存器中的地址空间。

The use of the 16-bit registers `BC`, `DE`, and `HL` to store addresses is very common.
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

These instructions have been for writing and writing to anywhere in memory, but there are a set of instructions that deal with a specific piece of memory called the stack. Let's take a look at what the stack is and the instructions that are used to manipulate the stack.

## The Stack
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

We have a stack pointer now so we know where our stack is, but how do we push and pop from this stack?

The Game Boy's CPU understands two insructions for doing just that. `PUSH` will write the contents of any 16-bit register into the stack and `POP` writes the head of stack into any 16-bit register.

Here's what's actually happening when a `PUSH` is performed:

* Decrease the stack pointer by 1.
* Write the most significant byte of the 16 bit value into memory at the location the stack pointer is now pointing to
* _Decrease_ the stack pointer by 1 again.
* Write the least significant byte of the 16 bit value into memory at the location the stack pointer is now pointing to

Notice that the stack pointer is decresed by 1 and not increased. This is because the stack grows downward in memory. This is extra helpful since the normal place for the stack to live is at the very end of memory. In a later chapter we'll see that it's actually the Game Boy's boot ROM that sets the stack pointer to the very end of memory. Thus, when the stack grows it grows away from the end of memory towards the beginning of memory.

Let's implement `PUSH`:

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

We can now push elements on to the stack. Here's what's actually happening when a `PUSH` is performed:

* Read the least significant byte of the 16 bit value from memory at the location the stack pointer is pointing to
* _Increase_ the stack pointer by 1.
* Read the most significant byte of the 16 bit value from memory at the location the stack pointer is now pointing to
* _Increase_ the stack pointer by 1 again.
* Return the value with the most and least significant byte combined together

Let's write `POP`:

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

And there we have it! We have a working stack that we can used. But what sort of things is the stack used for? One built in use for the stack is creating a "call" stack that allows the game to "call" functions and return from them. Let's see how that works.

## Calling Functions
In most programming languages when you call a function, the state of the calling function is saved somewhere, execution is allowed to happen for the called function, and then when the called function returns, the state of the called function is restored. It turns out the Game Boy has built in support for this mechanism where the state that is saved is simply just what the program counter was when the called function was called. This means we can "call a function" and that function itself can call functions, and when all of that is done, we'll return right back to the place we left off before we called the function.

This functionality is handled by two types of instructions `CALL` and `RET` (a.k.a return). The way `CALL` works is by using a mixture of two instructions we already know about `PUSH` and `JP` (a.k.a jump). To execute the `CALL` instruction, we must do the following:

* `PUSH` the next program counter (i.e. the program counter we would have if we were not jumping) on to the stack
* `JP` (a.k.a jump) to the address specified in the next 2 bytes of memory (a.k.a the function).

And that's it! We've called into our function. But what happens when we call `RET` (a.k.a return) from our called function? Here's what will happen:

* `POP` the next program counter off the stack and jump back to it.

Well that's easy! Let's see it in code:

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

Now we can easily call functions and return from them. We're now done with the vast majority of our CPU instructions!
