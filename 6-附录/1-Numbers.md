# numbers-in-computers

>* numbers-in-computers 译文（计算机中的数字）
>* 原文链接：https://rylev.github.io/DMG-01/public/book/appendix/numbers.html
>* 译文出处：https://github.com/suhanyujie/emulate-game-boy-translation
>* 译者：[suhanyujie](https://github.com/suhanyujie)
>* ps：水平有限，翻译不当之处，还请指正，谢谢！

In this guide, we'll look at how numbers are stored in the Gameboy's CPU, RAM, and ROM. In this guide we'll be using different types of number notations: binary, decimal and hexadecimal. If you're unfamiliar with these different ways of writing numbers, check out our [guide on number notations](https://rylev.github.io/DMG-01/public/book/appendix/number_notations.html).
>在这篇指南中，我们要会了解数字如何存储在 Gameboy 的 CPU、RAM 和 ROM 中的。通过本文，我们将会用到不同类型的数字符号：二进制、十进制和十六进制。如果你对这些不同形式的数字不太了解，可以先查看你这篇[数字符号指南](https://rylev.github.io/DMG-01/public/book/appendix/number_notations.html)。

## Bits
At a very basic level, computers can only read and write two different values that we'll call 1 and 0. This piece of data is called a bit. Computer memory is really just a long array of bits that the computer can read or write.
>在非常基础的层面上，计算机只能读取和写入两种不同的值 —— 0 和 1。这种数据被称为比特（bit）。计算机内存实际上只是一个可以被计算机读和写的长数组（bit 数组）。

>* ### 了解更多
>* Computers normally represent bits as either one voltage (e.g., five volts) or as some other, typically lower, voltage (e.g., zero volts). Again, a great resource for learning about how computers actually deal with bits, check out Ben Eater's series on [making an 8-bit Breadboard Computer](https://www.youtube.com/user/eaterbc).
>* 计算机也经常用位来表示电压（例如，五伏）或一些其他东西，通常较低的（例如，0 伏）。同样，了解计算机如何利用 bit 做一些实际处理的案例，请查看 Ben Eater 的关于[制作 8 位面包板计算机](https://www.youtube.com/user/eaterbc)的系列文章。

Bits can represent only two different values 0 or 1. If we want to numbers larger than one we need to compose bits together. To get to three for instance we would write 0b11. The total count of numbers we can represent is equal to 2^(# of bits). So one bit can represent 2^1 a.k.a two numbers and 7 bits can represent 2^7 a.k.a 128 numbers.
>位只能表示两个不同的值 0 或 1.如果我们想要大于 1 的数，我们需要将位进行组合表示。例如，要形成三，我们会写做 0b11。我们可以表示的数字总数等于 2^n（n 是位的个数）。所以以为可以表示 2^1 即 2 个数，7个位可以表示 2^7 即 128 个数字。

Since being able to manipulate numbers larger than 1 is pretty useful, we normally talk about and the computer typically reads and writes bits in large chunks called bytes.
>由于能够操作大于 1 的数字更有用，因此，我们通常谈论计算机通常是以字节（bytes）这种大块形式来进行读取和写入的。

## 字节（Bytes）
Bytes are defined as a collection of 8 bits. Our Gameboy, as an 8-bit machine, typically deals with one byte at a time and each compartment in memory stores one byte. However, the Game Boy also has 16-bit instructions which act on two bytes at a time. A byte can represent numbers 2^8 a.k.a 256 numbers (0 to 255) while 8 bytes (composed of 64 bits) and can represent 2^64 a.k.a 9,223,372,036,854,775,808 numbers (0 to 9,223,372,036,854,775,807).
>字节被定义为 8 个位的集合。我们的 Gameboy，也是 8 位机器，即一次处理一个字节，内存中也是以一个字节作为存储单元。然而，Game Boy 也有 16 位的指令，一次作用于两个字节。一个字节可以表示 2^8 即 256 个数字（0 到 255），而 8 个字节（由 64 位组成）可以表示 2^64 即 9,223,372,036,854,775,808 个数字（0 到 9,223,372,036,854,77）。

>* ## 了解更多
>* Some times we'll actually only deal with half a byte (i.e., 4 bits) at a time. This is usually referred to as a "nibble".
>* 有时候，我们实际上只处理半个字节（即 4 位）。这种称为“半字节（nibble）”

Since writing out bytes in binary can be quite tedious, we normally write out bytes in hexadecimal notation: So while we could write out the byte representing the number 134 as "0b10000110" we typically write it as "0x86". These two notations specify the same number, "0x86" is just shorter so it's more often used.

When disucssing numbers composed of multiple bytes, for example 0xFFA1 (composed of three bytes), we'll often need to talk about which byte is "most significant" (MSB - most significant byte) and which is "least significant" (LSB - least significant byte). Going back to math class, you may remember that when writing numbers like "178", the digit on the right (i.e., the "8") is the least sigificant, it adds the least amount to the total sum of the number (just eight) while the digit on the left (i.e., the "1") is the most significant since it adds the most to the sum of the number (one hundred!). Bytes work the same way - in 0xFFA1, 0xFF is the most significant byte and 0xA1 is the least significant.

## Endianess
Let's take the example of two bytes sitting next to each other in memory: first at address 0 there is 0xFF and then at address 1 there is 0x16. If we want to read these two bytes together as a 16 bit number, should it be read as 0xFF16 or as 0x16FF? Even if one way or the other makes more sense to you, the answer is: it depends on the machine. In the case of the Game Boy the order is 0xFF16 - in other words the least significant byte is first in memory. This scheme is known as little-endian and its opposite is known as big-endian.

## Signed Numbers
Ok so we know how to conceivably represent any number from 0 to some very large positive number. We can just keep adding bytes until we have enough to represent our number. But what about negative numbers? Well one way we could chose to do it (and the way the Game Boy does it) is using something called the "two's complement".

Let's say we have the number 0b00111011 a.k.a. 59 and we want to represent -59 instead. In two's complement, we do the following:

* Invert every digit - 1s become 0s and 0s become 1s
  * 0b00111011 becomes 0b11000100
* Add 1 to the number
  * 0b11000100 becomes 0b11000101

So -59 is 0b11000101. But wait is 0b11000101 already 197? Yes it is! Whether we chose to interpret a byte as a number from 0 to 255 or as two's complement number capable of representing -128 to 127 is up to programmer! Interpreting a number as only positive means it is "unsigned" and interpeting as being possibly negative with two's complement means it is "signed".

## Overflow and underflow
When doing arithmetic on numbers, sometimes the result is too large or small to be represented. For example if you add two 8 bit numbers 253 and 9 together you would expect to get 262. But 262 cannot be represented by 8 bits (it requires 9 bits). When this happens the number simply is what the first 8 bits of 262 would be just with the final 9th bit missing: 0b0000_0110 a.k.a 6. This phenomenon is called overflow. The opposite can occur when subtracting. This is called underflow

## Rust
In Rust, the various number types tell us both how many bits are used to represent that particular integer and whether the integer is in two's complement or not. For example, the number type `u8` is a number composed of 8 bits (i.e., 1 byte) and is unsigned while `i64` is a number composed of 64 bits (i.e., 8 bytes) and is signed.
