师兄给了第二次任务之后，因为一些奇奇怪怪的原因搁置了，现在终于想起来重新拾起来了。本文主要是对两个工具的分析：掌握AFL界面的所显示的全部信息【代表什么意思大概了解一下】。**使用afl-cmin 和 afl-tmin的功能 做出一个使用指令前后对比图。（真实的fuzz环境中，测试用例是很多的，怎么去精简化，多样化筛选是很重要的）**

------

# afl-cmin part

在**docs**的**README**中是这样描述**cmin tool**的：

Before using this corpus for any other purposes, you can shrink it to a smaller size using the afl-cmin tool. The tool will find a smaller subset of files offering equivalent edge coverage.

大概意思就是：**cmin工具可以减小corpus的大小，这个工具可以得到一个更小的子集（原case集合的子集），这个子集可以提供同样的edge coverage效果。**

举个例子就更明确了，接着上一篇的例子继续：

**第一步、**语料库，里面有很多case，但实际上有些可以精简，如图是我创建的简单的很多用例，其实就是上一次的多copy一些，改改字母，也就是说这些用例其实都差不多，需要删减；

![img](https://img2018.cnblogs.com/blog/806432/201911/806432-20191120164926761-866559190.png)

![img](https://img2018.cnblogs.com/blog/806432/201911/806432-20191120164525988-1369906887.png)

**第二步、**查看afl-cmin的操作指南

![img](https://img2018.cnblogs.com/blog/806432/201911/806432-20191120164558240-92149119.png)

根据usage的提示，输入的命令形式应该是：

```
//afl-cmin -i 测试用例文件夹 -o 筛选后的测试用例文件夹 [可能会用到其他操作] 测试程序文件
//这里的用法如下
afl-cmin -i fuzz_in -o fuzz_in_cmin ./afl_test
```

**第三步、**筛选结果，显示如图：

![img](https://img2018.cnblogs.com/blog/806432/201911/806432-20191120165037000-161574159.png)

最后筛选结果是只有一个文件，最后一个文件，推测是每次用一个文件跟之前的比较，如果能达到上一个的效果，就替换掉上一个文件。（测试后，确实如此，回头看看源代码是不是这样，挖坑，【11.21补坑，跟师兄求证之后，发现确实如此】）

![img](https://img2018.cnblogs.com/blog/806432/201911/806432-20191120165050167-695448080.png)

 

**到这里就很明显了，afl-cmin tool是为了删减，把多个用例组成的集合，删减成具有同等效果的子集。**

------

# afl-tmin part

官方**docs的README**对**tmin tool**的描述如下

Oh, one more thing: for test case minimization, give afl-tmin a try. The toolcan be operated in a very simple way:
$ ./afl-tmin -i test_case -o minimized_result -- /path/to/program [...]
The tool works with crashing and non-crashing test cases alike. In the crash mode, it will happily accept instrumented and non-instrumented binaries. In the non-crashing mode, the minimizer relies on standard AFL instrumentation to make the file simpler without altering the execution path.

 按照描述，**大概意思是：工具是为了创造一个更小的文件，同时又能达到同样的效果。当然操作也会比cmin简单。**

**第一步、**还是语料库，当然因为是对单个文件处理，所以这里我是用的crash文件进行的修改【不过这里有个问题最后会说一下】

**第二步、**查看afl-tmin工具的用法

![img](https://img2018.cnblogs.com/blog/806432/201911/806432-20191120190903897-35875500.png)

**第三步、**进行处理，可以看到命令很简单，必需的就是输入输出和测试程序

```
//afl-tmin -i 需要精简的文件 -o 精简后的文件 [其他操作] 测试程序
//这里用到的用例
afl-tmin -i fuzz_in/testcase_bin -o fuzz_in_tmin/testcase_tmin ./afl_test
```

可以看到结果如下图：

![img](https://img2018.cnblogs.com/blog/806432/201911/806432-20191120191325026-730086302.jpg)

经过tmin之后的文件对比

![img](https://img2018.cnblogs.com/blog/806432/201911/806432-20191120192821717-64541709.jpg)

**第四步、**这里遇到一个问题，这是用的crash进行的测试，但是当我用最早的testcase进行tmin的时候，会发现报错：[!] WARNING: Down to zero bytes - check the command line and mem limit!【已补坑，见最后一部分】

这里就很迷，具体情况如图，暂时还没解决为什么，挖坑待之后解决。

![img](https://img2018.cnblogs.com/blog/806432/201911/806432-20191120191847295-1263907960.jpg)

------

【补坑】

这里对tmin的新理解整理：tmin就是把我们提供的文件放到afl编译出来的程序里跑，然后再按一定方式进行精简【精简的原理感觉还是得看看源码才能更好的理解】。

那么之前为啥 down to zero 呢，估计是自己写的case太垃圾，tmin觉得都没用：“什么垃圾case，删掉删掉”，所以就全删没了，怎么印证这一观点呢？

**【1】把引起crash的bin文件用gedit打开，然后直接copy到我之前那个testcase。**

引起crash的文件打开：

![img](https://img2018.cnblogs.com/blog/806432/201911/806432-20191121223444945-752642825.png)

复制到testcase：

![img](https://img2018.cnblogs.com/blog/806432/201911/806432-20191121223529048-1919630707.png)

**【2】重新tmin testcase**，发现可以了，虽然最后精简完成了一字节，但是成功了呀。

![img](https://img2018.cnblogs.com/blog/806432/201911/806432-20191121223601302-1416871975.png)

**综上，tmin的 -i 文件不是一定要binary文件；testcase太垃圾是会被tmin全部干掉的；testcase可以是乱文件，不一定跟crash有关系，因为我之后在testcase随机加东西，都会成功tmin；**

------

# Refrenece参考资料

afl-cmin tool：http://www.tin.org/bin/man.cgi?section=1&topic=AFL-CMIN

afl-tmin tool：http://www.tin.org/bin/man.cgi?section=1&topic=afl-tmin

afl使用指南：https://www.codercto.com/a/78143.html

afl详细分析：https://blog.csdn.net/Chen_zju/article/details/80791268