# Graphics
* 图形
>Graphics 译文

>* 原文链接：https://github.com/rylev/DMG-01/blob/master/book/src/graphics/introduction.md
>* 译文出处：https://github.com/suhanyujie/emulate-game-boy-translation
>* 译者：[suhanyujie](https://github.com/suhanyujie)

到目前为止，我们一直在对内存进行读写，并在 CPU 中执行不同的操作。CPU 有能力做很多事情，而且从某种程度上讲它已经可以“运行”游戏了。但是，我们知道，定义电子游戏的一个重要指标就是它的视频或图形方面的能力。

在接下来的部分中，我们将深入到 Game Boy 中的图形，看它是如何工作的。首先，我们会从 Game Boy 的背景图入手。这将足以引导我们获取图标启动屏幕显示。一旦我们完成，我们会继续深入实现精灵图形的展示，进而实现字符和敌人的出现等操作。

一秒都不想耽搁了，我们直接开始吧！
