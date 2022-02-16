# 配置组合生成器jenny

官网地址http://burtleburtle.net/bob/math/jenny.html#algo

源码使用c语言编写，jenny.c文件

下载文件后，

##### 编译

```
gcc -o jenny jenny.c
```

![image-20220207124653566](/Users/changzw/Library/Application Support/typora-user-images/image-20220207124653566.png)

会给出很多警告（还好不是error 哈哈哈）

##### 运行官网给的一个测试案例

```
jenny -n3 2 2 2 2 2 2 2 2 2 2 2 2 | sort
```

![image-20220207124751278](/Users/changzw/Library/Application Support/typora-user-images/image-20220207124751278.png)

##### 原理：

![image-20220207125104760](/Users/changzw/Library/Application Support/typora-user-images/image-20220207125104760.png)

解释上面这个案例：

针对12个配置项的组合情况（假设每个配置项自由两种取值，0/1，或者是on/off，enable/disable），完全枚举共有$2^{12}=4096$种组合，这个数字勉强可以接受，但是如果有100个配置项，那就有$2^{100}$组合，累死也测不完这么多情况。

根据上诉原理，一个漏洞的触发通常由一个、二个、或者三个配置的组合，并不需要枚举全部的12个配置项



假设我们想要覆盖所有的三元组特征。有 (12 choose 3) = 220 种选择维度的方法，以及 $2^3$ = 8 种选择这些维度内特征的方法，总共有 1760 个可能的特征三元组。这比详尽测试给出的 4096 测试用例要少得多。但是`jenny`比两者都好几乎两个数量级：

![image-20220207132220485](/Users/changzw/Library/Application Support/typora-user-images/image-20220207132220485.png)

`jenny`涵盖了 20 个测试用例中的所有三元组特性。如果您愿意，请运行任意三列，并确认其中包含所有 8 种可能的特征组合。每个测试用例恰好涵盖了其中一个选择，因此 8 个测试用例是涵盖所有三元组特征的下限。测试用例的最小数量可能是 8 个，或者可能更多。（在这种情况下，似乎是 12 个测试用例形成了一个[Golay 代码](http://burtleburtle.net/bob/math/lexicode.html)。不过，我不知道如何以任何有用的方式对其进行概括。） 当`-n`为 1 或方面。对于介于两者之间的 -n，它通常比最佳值略差。

##### 针对三个配置的情况：

-n1 一个配置项的组合

![image-20220207130230926](/Users/changzw/Library/Application Support/typora-user-images/image-20220207130230926.png)

-n2 二个配置项的组合

![image-20220207125851366](/Users/changzw/Library/Application Support/typora-user-images/image-20220207125851366.png)

-n3 三个配置项的组合，此时是全覆盖的情况

![image-20220207130206761](/Users/changzw/Library/Application Support/typora-user-images/image-20220207130206761.png)



##### 相关参数：

-n：指定需要组合元素的个数，取值范围1～32，默认取值2

-w：给出一个限制。-w1a2bc5d表示不允许第一维的第一个特征(1a)、第二维的第二或第三特征(2bc)和第五维的第四特征(5d)的组合。



##### 覆盖所有 n 元组的工具

详尽的测试随着要一起测试的维度的数量呈指数增长。但是，如果您限制自己只覆盖所有特征对（三元组、四元组、n 元组），并允许每个测试用例覆盖许多 n 元组，则所需的测试用例数量只会随着维度数量的增加而呈对数增长。 `jenny`只是利用这一点的一个工具。网站[pairwise.org](http://www.pairwise.org/)列出了许多此类工具，并比较了每种工具为各种输入生成的测试用例数量。（到目前为止，它没有列出哪些支持限制。我认为除非它们支持限制，否则这些工具是无法使用的。有几个支持，有几个不支持。）