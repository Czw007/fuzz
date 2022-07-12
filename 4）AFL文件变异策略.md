## 前言：

在AFL的fuzzing过程中，维护了一个 testcase 队列 queue ，每次把队列里的文件取出来之后，对其进行变异，下面就先粗略讲一下各个阶段的变异是怎样的。

- bitflip：    按位翻转，每次都是比特位级别的操作，从 1bit 到 32bit ，从文件头到文件尾，会产生一些有意思的额外重要数据信息；
- arithmetic： 与位翻转不同的是，从 8bit 级别开始，而且每次进行的是加减操作，而不是翻转；
- interest：  把一些有意思的东西“interesting values”对文件内容进行替换；
- dictionary： 用户提供的字典里有token，用来替换要进行变异的文件内容，如果用户没提供就使用 bitflip 自动生成的 token；
- havoc：    进行很大程度的杂乱破坏，规则很多，基本上换完就是面目全非的新文件了；
- splice：    通过将两个文件按一定规则进行拼接，得到一个效果不同的新文件；

*注： `bitflip、arithmetic、interest、dictionary` 是 `deterministic fuzzing` 过程，属于dumb mode(-d) 和主 fuzzer(-M) 会进行的操作； `havoc、splice` 与前面不同是存在随机性，是所有fuzz都会进行的变异操作。**文件变异是具有启发性判断的，应注意“避免浪费，减少消耗”的原则，即之前变异应该尽可能产生更大的效果，比如 `eff_map` 数组的设计；同时减少不必要的资源消耗，变异可能没啥好效果的话要及时止损。***

师兄之前说是可以开始看afl-fuzz.c文件，然后看下来很多点都是缠在一起的，除了插桩编译部分和fuzzing过程可以稍微分开，其实很多头文件也是需要看的。完整版笔记见我的git地址（不断更新）：[afl-fuzz.c 阅读笔记](https://github.com/WayneDevMaze/Chinese_noted_AFL/blob/master/afl-fuzz.c笔记.md)，随着深入会把一些理解的比较多的部分抽离出来单独写一写。在阅读源码过程中对文件变异有了新的整体的认识，故有此文。**本文分两部分：文件变异方法详解 和 变异策略**。

------

