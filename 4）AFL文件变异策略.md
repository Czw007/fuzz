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

## 文件变异方法详解

### 1.`bitflip`，位反转，顾名思义按位进行翻转，0变1，1变0。

- `STAGE_FLIP1` 每次翻转一位(`1 bit`)，按一位步长从头开始。
- `STAGE_FLIP2` 每次翻转相邻两位(`2 bit`)，按一位步长从头开始。
- `STAGE_FLIP4` 每次翻转相邻四位(`4 bit`)，按一位步长从头开始。
- `STAGE_FLIP8` 每次翻转相邻八位(`8 bit`)，按八位步长从头开始，也就是说，每次对一个byte做翻转变化。
- `STAGE_FLIP16`每次翻转相邻十六位(`16 bit`)，按八位步长从头开始，每次对一个word做翻转变化。
- `STAGE_FLIP32`每次翻转相邻三十二位(`32 bit`)，按八位步长从头开始，每次对一个dword做翻转变化。

变异的具体实现部分在大约 `5135` 行的  #define FLIP_BIT(_ar, _b) do {***}  有详细的代码实现。

#### <1.1 `token` - 自动检测>

源码中有一段关于这部分的注释，意思是说在进行为翻转的时候，程序会随时注意翻转之后的变化。比如说，对于一段 `xxxxxxxxIHDRxxxxxxxx` 的文件字符串，当改变 `IHDR` 任意一个都会导致奇怪的变化，这个时候，程序就会认为 `IHDR` 是一个可以让fuzzer很激动的“神仙值”--token。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
/*
While flipping the least significant bit in every byte, pull of an extra trick to detect possible syntax tokens. In essence, the idea is that if you have a binary blob like this:
    xxxxxxxxIHDRxxxxxxxx
...and changing the leading and trailing bytes causes variable or no changes in program flow, but touching any character in the "IHDR" string always produces the same, distinctive path, it's highly likely that "IHDR" is an atomically-checked magic value of special significance to the fuzzed format.
We do this here, rather than as a separate stage, because it's a nice way to keep the operation approximately "free" (i.e., no extra execs).
Empirically, performing the check when flipping the least significant bit is advantageous, compared to doing it at the time of more disruptive changes, where the program flow may be affected in more violent ways.
The caveat is that we won't generate dictionaries in the -d mode or -S mode - but that's probably a fair trade-off.
This won't work particularly well with paths that exhibit variable behavior, but fails gracefully, so we'll carry out the checks anyway.
*/
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

其实token的长度和数量都是可以控制的，在 `config.h` 中有定义，但是因为是在头文件宏定义的，修改之后需要重新编译使用。

```
/* Maximum number of auto-extracted dictionary tokens to actually use in fuzzing (first value), and to keep in memory as candidates. The latter should be much higher than the former. */
 #define USE_AUTO_EXTRAS 50
 #define MAX_AUTO_EXTRAS (USE_AUTO_EXTRAS * 10)
```

#### <1.2 `effector map` - 生成>

在这里值得一提的是 `effector map`，在看源码数据变异这一部分的时候，一定会注意的是在 bitflip 8/8 的时候遇到一个叫 `eff_map` 的数组，这个数组的大小是 `EFF_ALEN(len)` （也就是【？多大还不清楚？】），数组元素只有 0/1 两种值，很明显是标记的意思，到底是标记什么呢？
要想明白 `effector map` 的原理需要了解三个点：

> \> 为什么是 8/8 的时候出现？个人理解：因为这里设置 eff_map 是为了之后的启发式判断，而后面的文件数据变异都是 8/8 起步的，所以这里在 8/8 处进行判断也就合情合理了。还有一点点个人猜测：这里是 8bit 对应着 1byte，应该是byte级别下的启发判断的效率最高。
>
> \> 标记是干什么用的？根据上面的分析，就很好理解了，标记好的数组可以为之后的变异服务，相当于提前“踩雷（踩掉无效byte的雷）”，相当于进行了启发式的判断。无效为0，有效为1。
>
> \> 达到了怎样的效果？要知道判断的时间开销，对不停循环的fuzzing过程来说是致命的，所以 `eff_map` 利用在这一次8/8的判断中，通过不大的空间开销，换取了可观的时间开销。(暂时是这样分析的，具体是否真的节约很多，不得而知)

### 2.`arithmetic`，算术，实际操作就是加加减减。

`bitflip` 结束之后，就进入 `arithmetic` 阶段，目标大小和阶段与 `bitflip` 非常类似：

- arith 8/8，每次8bit进行加减运算，8bit步长从头开始，即对每个byte进行整数加减变异；
- arith 16/8，每次16bit进行加减运算，8bit步长从头开始，即对每个word进行整数加减变异；
- arith 32/8，每次32bit进行加减运算，8bit步长从头开始，即对每个dword进行整数加减变异；

其中对于加减变异的上限，在 `config.h` 中有所定义：

```
/* Maximum offset for integer addition / subtraction stages: */
#define ARITH_MAX 35
```

*注：跟bitflip相同的，如果需要修改此值，在头文件中修改完之后，要进行编译才会生效。*

在这里对整数目标进行+1，+2，+3...+35，-1，-2，-3...-35的变异。由于整数存在大端序和小端序两种表示，AFL会对这两种表示方式都进行变异。
前面也提到过AFL设计的巧妙之处，AFL尽力不浪费每一个变异，也会尽力让变异不冗余，从而达到快速高效的目标。AFL会跳过某些arithmetic变异：

1. 在 `eff_map` 数组中对byte进行了 0/1 标记，如果一个整数的所有 bytes 都被判为无效，那么就认为整数无效，跳过此数的变异；
2. 如果加减某数之后效果与之前某bitflip效果相同，认为此次变异在上一阶段已经执行过，此次不再执行；

### 3.`interest`，把一些“有意思”的特殊内容替换到原文件中。

interest的三个步骤跟arithmetic相似：

- interest 8/8，每次8bit进行加减运算，8bit步长从头开始，即对每个byte进行替换；
- interest 16/8，每次16bit进行加减运算，8bit步长从头开始，即对每个word进行替换；
- interest 32/8，每次32bit进行加减运算，8bit步长从头开始，即对每个dword进行替换；

用于替换的叫做 `interesting values` ，是AFL预设的特殊数：

```
/* Interesting values, as per config.h */
static s8 interesting_8[] = { INTERESTING_8 }; 
static s16 interesting_16[] = { INTERESTING_8,INTERESTING_16 };
static s32 interesting_32[] = { INTERESTING_8, INTERESTING_16, INTERESTING_32 };
```

其中 `interesting values` 在 `config.h` 中的设定：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
/* List of interesting values to use in fuzzing. */
#define INTERESTING_8 \
  -128,          /* Overflow signed 8-bit when decremented  */ \
  -1,            /*                                         */ \
   0,            /*                                         */ \
   1,            /*                                         */ \
   16,           /* One-off with common buffer size         */ \
   32,           /* One-off with common buffer size         */ \
   64,           /* One-off with common buffer size         */ \
   100,          /* One-off with common buffer size         */ \
   127           /* Overflow signed 8-bit when incremented  */

#define INTERESTING_16 \
  -32768,        /* Overflow signed 16-bit when decremented */ \
  -129,          /* Overflow signed 8-bit                   */ \
   128,          /* Overflow signed 8-bit                   */ \
   255,          /* Overflow unsig 8-bit when incremented   */ \
   256,          /* Overflow unsig 8-bit                    */ \
   512,          /* One-off with common buffer size         */ \
   1000,         /* One-off with common buffer size         */ \
   1024,         /* One-off with common buffer size         */ \
   4096,         /* One-off with common buffer size         */ \
   32767         /* Overflow signed 16-bit when incremented */

#define INTERESTING_32 \
  -2147483648LL, /* Overflow signed 32-bit when decremented */ \
  -100663046,    /* Large negative number (endian-agnostic) */ \
  -32769,        /* Overflow signed 16-bit                  */ \
   32768,        /* Overflow signed 16-bit                  */ \
   65535,        /* Overflow unsig 16-bit when incremented  */ \
   65536,        /* Overflow unsig 16 bit                   */ \
   100663045,    /* Large positive number (endian-agnostic) */ \
   2147483647    /* Overflow signed 32-bit when incremented */
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

可以看到，基本是些会造成溢出的数值。与前面的思想相同的，本着“避免浪费，减少消耗”的原则，`eff_map`数组中已经判定无效的就此轮跳过；如果 `interesting value` 达到的效果跟 `bitflip` 或者 `arithmetic` 效果相同，也被认为是重复消耗，跳过。

### 4.`distionary`，字典，会把自动生成或者用户提供的token替换、插入到原文件中。

此阶段已经是确定性变异 deterministic fuzzing 的结尾：

- user extras (over)，从头开始，将用户提供的tokens依次替换到原文件中
- user extras (insert)，从头开始，将用户提供的tokens依次插入到原文件中
- auto extras (over)，从头开始，将自动检测的tokens依次替换到原文件中

其中 “用户提供的tokens” 是一开始通过 -x 选项指定的，如果没有则跳过对应的子阶段；“自动检测的tokens” 是第一个阶段 `bitflip` 生成的。

### 5.`havoc`，“大破坏”，对原文件进行大量变异。

`havoc` 意味着随机的开始，与后面的 `splice` 是任何模式下都要进行的变异，具体来说 havoc 包含了多轮变异，每一轮都是组合拳：

- 随机选取某个bit进行翻转
- 随机选取某个byte，将其设置为随机的interesting value
- 随机选取某个word，并随机选取大、小端序，将其设置为随机的interesting value
- 随机选取某个dword，并随机选取大、小端序，将其设置为随机的interesting value
- 随机选取某个byte，对其减去一个随机数
- 随机选取某个byte，对其加上一个随机数
- 随机选取某个word，并随机选取大、小端序，对其减去一个随机数
- 随机选取某个word，并随机选取大、小端序，对其加上一个随机数
- 随机选取某个dword，并随机选取大、小端序，对其减去一个随机数
- 随机选取某个dword，并随机选取大、小端序，对其加上一个随机数
- 随机选取某个byte，将其设置为随机数
- 随机删除一段bytes
- 随机选取一个位置，插入一段随机长度的内容，其中75%的概率是插入原文中随机位置的内容，25%的概率是插入一段随机选取的数
- 随机选取一个位置，替换为一段随机长度的内容，其中75%的概率是替换成原文中随机位置的内容，25%的概率是替换成一段随机选取的数
- 随机选取一个位置，用随机选取的token（用户提供的或自动生成的）替换
- 随机选取一个位置，用随机选取的token（用户提供的或自动生成的）插入

之后AFL会生成一个随机数，作为变异组合的数量，每次从上面随机选取作用于当前文件。

### 6.`splice`，“拼接”，两个文件拼接到一起得到一个新文件。

具体来说，AFL会在文件队列中随机选择一个文件与当前文件进行对比，如果差别不大就重新再选；如果差异明显，就随机选取位置两个文件都一切两半。最后将当前文件的头与随机文件的尾拼接起来得到新文件【为什么不是当前的尾拼接随机文件的头【？？】】。当然了本着“减少消耗”的原则拼接后的文件应该与上一个文件对比，如果未发生变化应该过滤掉。

------

## 变异策略 - 循环往复的Cycle

一波变异结束后的文件，会在队列结束后下一轮中继续变异下去。AFL状态栏右上角的  cycles done 意味着完成的循环数，每次循环是对整个队列的再一次变异，不过只有第一次 cycle 才会进行  deterministic fuzzing ，之后循环的只有随机性变异了。

那么一次变异的整体看下来是一个什么步骤呢？【有许多地方不太理解，待补充】

> fuzz_one//进行一次

------

## 参考 - Reference：

【1】[文件变异一览：http://rk700.github.io/2018/01/04/afl-mutations/](http://rk700.github.io/2018/01/04/afl-mutations/)

【2】[afl-fuzz.c详解：https://bbs.pediy.com/thread-254705.htm](https://bbs.pediy.com/thread-254705.htm)