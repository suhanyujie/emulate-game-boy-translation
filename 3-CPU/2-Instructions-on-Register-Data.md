
>* Instructions on Register Data 译文
>* 原文链接 https://blog.ryanlevick.com/DMG-01/public/book/
>* 译文出处 https://github.com/suhanyujie/emulate-game-boy-translation
>* 译者：[suhanyujie](https://github.com/suhanyujie)
>* ps：水平有限，翻译不当之处，还请指正，谢谢！

# 寄存器数据的指令
我们首先研究的指令是操作符寄存器数据的指令

## ADD
以研究 `ADD` 指令如何工作作为开始。它是一个简单的指令，将指定寄存器的内容添加到另一个寄存器的内容中。一旦我们知道了这条指令的原理，扩展其它 CPU 指令以支持寄存器数据的其它操作就不需要费太多力气。

### 定义
首先，我们先定义指令。稍后会了解对应的代码是如何编码指令以及指令是存在哪儿。现在我们关注一下指令本身，以及它如何影响 CPU 寄存器。

首先定义一个名为 `Instruction` 的枚举类型。后面我们定义所有的指令都将围绕这个枚举进行。`ADD` 指令中需要包含目标寄存器的信息，因此我们要把 `ArithmeticTarget` 枚举类型的特定值和目标寄存器联系起来。`ADD` 可以将除 f 之外的所有 8 位寄存器作为目标寄存器。

```rust
enum Instruction {
  ADD(ArithmeticTarget),
}

enum ArithmeticTarget {
  A, B, C, D, E, H, L,
}
```

### 执行指令
好了，有了这个指令，我们就要想办法去执行它。我们在 CPU 上创建一个方法，它接收一条指令并执行它。这个方法接受的是 CPU 的可变引用，因为指令总是会改变 CPU 的状态。该方法还可以接收它后续要执行的指令。我们会对指令和目标寄存器进行模式匹配，然后根据指令和寄存器进行相应的操作：

```rust
struct CPU {}
enum Instruction { ADD(ArithmeticTarget), }
enum ArithmeticTarget { A, B, C, D, E, H, L, }
impl CPU {
  fn execute(&mut self, instruction: Instruction) {
    match instruction {
      Instruction::ADD(target) => {
        match target {
          ArithmeticTarget::C => {
            // TODO: implement ADD on register C
          }
          _ => { /* TODO: support more targets */ }
        }
      }
      _ => { /* TODO: support more instructions */ }
    }
  }
}
```

我们有了最基础的版本，可以计算出对应的指令和对应的目标寄存器。现在我们来看看实际上对 CPU 做了什么。给 8 位寄存器的 `ADD` 指令添加目标寄存器的步骤如下：
    * 从目标寄存器中读取当前值
    * 将这个值加到 A 寄存器中的值上，并正确处理溢出等问题。
    * 更新标志寄存器的值
    * 将更新的值写入到 A 寄存器中

我们用 C 表示目标寄存器来实现整个步骤：

```rust
struct Registers { a:u8, c: u8 }
struct CPU { registers: Registers }
enum Instruction { ADD(ArithmeticTarget), }
enum ArithmeticTarget { A, B, C, D, E, H, L, }
impl CPU {
  fn execute(&mut self, instruction: Instruction) {
    match instruction {
      Instruction::ADD(target) => {
        match target {
          ArithmeticTarget::C => {
            let value = self.registers.c;
            let new_value = self.add(value);
            self.registers.a = new_value;
          }
          _ => { /* TODO: support more targets */ }
        }
      }
      _ => { /* TODO: support more instructions */ }
    }
  }

  fn add(&mut self, value: u8) -> u8 {
    let (new_value, did_overflow) = self.registers.a.overflowing_add(value);
    // TODO: set flags
    new_value
  }
}
```

注意，我们在 8 位的值上使用了 `overflowing_add` 方法，而不是 `+`。这是因为在开发中，当加法运算结果溢出时，使用 `+` 会引发 panic。Rust 要求我们明确自己的行为，使用 `overflowing_add` 可以正确处理溢出问题，并且它告诉我们实际操作是否会溢出。这将是更新标志寄存器时的重要信息。

### Setting Flags
在标志寄存器中定义了四个标志位：
    * Zero: 如果运算结果等于 0，则设置为 true。
    * Subtract: 如果是减法运算，这个标志位设置为 true
    * Carry: 如果运算结果溢出，则将这个标志位设为 true。
    * 半进位：如果出现较低的半字节（即低四位）向较高的半字节（即高四位）溢出，则设置为 true。举个例子看看这种场景意味什么，在下面的图中，已知二进制字节 143（0b1000_1111），然后把 0b1 加到这个数上。着重看一下，1 是如何从低位进到高位的。你应该已经熟悉了数学计算中的进位。当一个数在一个特定的数位上没有足够的空间表示时，我们就把它移到下个数位上表示。

    ```other
          低半字节            低半字节
         ┌--┐                    ┌--┐
    1000 1111  +   1   ==   1001 0000
    └--┘                    └--┘
高半字节            高半字节
    ```

    如果在增加数值时，我们将 half_carry 设为 true。我们可以通过划出 A 寄存器和“被加数”值的高半字节进行测试，并测试结果值是否比 0xF 大。

看一下代码：

```rust
struct FlagsRegister { zero: bool, subtract: bool, half_carry: bool, carry: bool }
struct Registers { a: u8, f: FlagsRegister }
struct CPU { registers: Registers }
impl CPU {
  fn add(&mut self, value: u8) -> u8 {
    let (new_value, did_overflow) = self.registers.a.overflowing_add(value);
    self.registers.f.zero = new_value == 0;
    self.registers.f.subtract = false;
    self.registers.f.carry = did_overflow;
    // Half Carry is set if adding the lower nibbles of the value and register A
    // together result in a value bigger than 0xF. If the result is larger than 0xF
    // than the addition caused a carry from the lower nibble to the upper nibble.
    // 如果把值的低半字节和 A 寄存器相加，并将结果结合在一起，结果大于 0xF，则设置半进位。
    // 如果结果大于 0xF，则会导致从低四位到高四位的进位
    self.registers.f.half_carry = (self.registers.a & 0xF) + (value & 0xF) > 0xF;
    new_value
  }
}
```

## How Do We Know?
你可能会想，“我们怎么知道一个具体的指令它是做什么操作的”。简而言之，这就是芯片的设计和制造过程产生的结果。我们之所以知道这一点，是因为人们阅读了 Game Boy 的 CPU 芯片的用户手册（也就是“资料手册”），又或者给芯片编写测试程序，调用特定的指令，然后观察会发生什么。幸运的是，我们无需这么做。我们可以在[说明书](https://blog.ryanlevick.com/DMG-01/public/book/appendix/instruction_guide/index.html)中找到所有具体的说明。

>* 额外提示
>* 大多数处理寄存器数据的 CPU 指令都可以通过各种位操作来操作数据。如果你不太清楚逻辑位运算之类的知识，请参阅[位操作指南](https://blog.ryanlevick.com/DMG-01/public/book/cpu/appendix/bit_manipulation.html)

还有其他哪些作用于寄存器数据的指令呢？
    * **ADDHL** (add to HL) - 类似于 ADD 指令一样，不同的是 ADDHL 的结果是加到 HL 寄存器
    * **ADC** (add with carry) - 和 ADD 一样，知识进位标志的值也被加到数值中
    * **SUB** (subtract) - 用 A 寄存器中的值减去存储在特定寄存器中的值
    * **SBC** (subtract with carry) - 和 ADD 类似，知识进位值要从数值中减去
    * **AND** (logical and) - 对指定寄存器中的值和 A 寄存器中的值执行按位与操作
    * **OR** (logical or) - 将指定寄存器的值和 A 寄存器的值进行按位或
    * **XOR** (logical xor) - 将指定寄存器的值和 A 寄存器的值进行按位异或
    * **CP** (compare) - 和 SUB 类似，只是减运算的结果不存回 A 寄存器
    * **INC** (increment) - 将指定寄存器的值加一
    * **DEC** (decrement) - 将指定寄存器中的值减一
    * **CCF** (complement carry flag) - 切换进位标志的值
    * **SCF** (set carry flag) - 将进位标志设为 true
    * **RRA** (rotate right A register) - 通过进位标志位将 A 寄存器向右位旋转
    * **RLA** (rotate left A register) - 通过进位标志位将 A 寄存器向左位旋转
    * **RRCA** (rotate right A register) - 向右位旋转 A 寄存器的值 (不是通过进位标志位)
    * **RRLA** (rotate left A register) - 向左右位旋转 A 寄存器的值 (不是通过进位标志位)
    * **CPL** (complement) - 切换 A 寄存器的所有位
    * **BIT** (bit test) - 检查指定寄存器是否设置了特定的位
    * **RESET** (bit reset) - 将指定寄存器的指定位设为 0
    * **SET** (bit set) - 将指定寄存器的指定位设为 1
    * **SRL** (shift right logical) - 将指定寄存器的值右移 1 位
    * **RR** (rotate right) - 通过进位标志位将指定寄存器向右旋转 1 位
    * **RL** (rotate left) - 通过进位标志位将指定寄存器向左旋转 1 位
    * **RRC** (rorate right) - 将指定寄存器的值向右位旋转 1 位 (不是通过进位标志位)
    * **RLC** (rorate left) - 将指定寄存器的值向左位旋转 1 位 (不是通过进位标志位)
    * **SRA** (shift right arithmetic) - 将指定寄存器的值“算术逻辑运算右移” 1 位
    * **SLA** (shift left arithmetic) - 将指定寄存器的值“算术逻辑运算左移” 1 位
    * **SWAP** (swap nibbles) - 切换指定寄存器的高低半字节

通读指南上的说明，差不多会给你足够的信息来让你自己实现所有的指令。

接下来，我们将研究CPU 如何跟踪执行这些指令，以及更改在特定程序位置的不同类型的指令。
