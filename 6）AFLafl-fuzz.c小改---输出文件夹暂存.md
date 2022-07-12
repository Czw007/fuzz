## 前言：

之前已经写过两篇针对tmin、cmin的修改，现在再改afl-fuzz.c已经轻车熟路了，对整个afl的源码结构也比较了解了。

afl-tmin：https://www.cnblogs.com/wayne-tao/p/11964565.html

afl-cmin：https://www.cnblogs.com/wayne-tao/p/11971922.html

此番修改是因为，对AFL关于 out_dir 文件夹的细节处理不是很满意，AFL对 out_dir 的处理是这样的：

1. 检测 out_dir 文件的合法性，不仅检测路径合法，还检测是否可用、当前状态是否可以用于处理；
2. 检查文件夹里面的内容 ，如果是之前跑的结果，检查里面的信息是不是 valuable，如果有价值，就会暂停，程序停止；
3. 经过前两步的处理，保证可以对out_dir进行简单操作了，cleanup文件夹里的信息，开始使用out_dir。

------

## 源码分析：

### 【一】需求分析

当运行时间 过长时，比如这样，虽然没有crash，但是运行了很长时间，也经过了许多变异：

![img](https://img2018.cnblogs.com/blog/806432/202001/806432-20200101171821701-599471180.png)

当停下之后，如果继续进行 afl 操作的话，会出现如下问题：

![img](https://img2018.cnblogs.com/blog/806432/202001/806432-20200101171927247-356302243.jpg)

在PROGRAM ABORT处，可看到，程序停止的位置，现在的想法是：

> **当发现 valuable out info 的时候，对文件夹进行暂存操作，让程序继续进行，当然这里的操作跟 cmin 的处理如出一辙，先重命名文件夹，而后新建与原来同名的文件夹作为 out_dir。**

### 【二】代码设计

通过**FATAL()和输出内容**很容易找到程序停止所在位置：

进行如下修改：（其中 old_out_dir 是在开始声明的跟 out_dir 相同类型的变量）

![img](https://img2018.cnblogs.com/blog/806432/202001/806432-20200101172829250-1295975195.jpg)

因为源码修改较多，所以，修改后的源码都会放在git上：https://github.com/WayneDevMaze/Chinese_noted_AFL

**此处代码稍微解释一下：**

> 1.这里的old_out_dir是之前声明过的跟out_dir同类型变量，然后 if 里面的 alloc_printf 是afl自己弄的一个拼接字符串的东西；
>
> 2.rename操作，重命名文件夹；
>
> 3.mkdir新建文件夹，用的是旧的名字；
>
> 4.如果最开始的拼接失败，可能是out_dir本身就有问题，停止程序；

ps.

```
OKF()//成功情况的输出提示
FATAL()//失败情况的输出提示，并且输出结束后，还会exit(1)
```

**代码原文**：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
//发现文件夹有价值，又不舍得删，就备份一下，跟cmin思路相同
//
if (old_out_dir = alloc_printf("%s_old", out_dir))
{
    rename(out_dir, old_out_dir);
    mkdir(out_dir, 0700);
    OKF("Success to create old_file_dir and move valuable files in it.");
}
else{
    OKF("Fail to move file!!!");
    FATAL("At-risk data found in '%s'", out_dir);
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

### 【三】运行效果

依旧用之前会停止的那个指令，发现程序继续运行，并且给与提示了：

![img](https://img2018.cnblogs.com/blog/806432/202001/806432-20200101173910082-921428131.jpg)

**而且存放out的那个文件夹，确实也暂存了**：

![img](https://img2018.cnblogs.com/blog/806432/202001/806432-20200101173944082-1754739785.jpg)

总体来说，思路还是很清楚的，也不太麻烦，如果对 Linux C 有了解，还是很容易实现的，之后还会继续对AFL的功能进行自己的修改。

------

## Reference

【1】LinuxC 文件操作指南：[linuxC文件以及目录操作函数 - yongfengnice - 博客园](https://www.cnblogs.com/yongfengnice/p/11809334.html)

【2】Linux C文件夹重命名：[linux c 重命名文件和文件夹_he_fa的专栏-CSDN博客](https://blog.csdn.net/he_fa/article/details/9373175)

【3】uint8_t意味着什么：[uint8_t / uint16_t / uint32_t /uint64_t数据类型详解 - Z--Y - 博客园](https://www.cnblogs.com/ZY-Dream/p/10029072.html)

【4】我自己git的源码地址：https://github.com/WayneDevMaze/Chinese_noted_AFL