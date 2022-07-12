

### 第一步 插桩编译测试

1. 新建一个项目文件夹afl_demo1
2. 进入项目文件
3. 写个c程序afl_test.c，准备编译，代码如下：

```
#include <stdio.h> 
int main(int argc, char *argv[])
{
    char buf[100]={0};
    gets(buf);//存在栈溢出漏洞
    printf(buf);//存在格式化字符串漏洞
    return 0;
}
```

4.使用命令 **afl-gcc -g -o afl_test afl_test.c** 编译插桩

![image-20220205155514296](/Users/changzw/Library/Application Support/typora-user-images/image-20220205155514296.png)

![image-20220205155739610](/Users/changzw/Library/Application Support/typora-user-images/image-20220205155739610.png)

### 第二步：开始fuzz

创建两个文件夹，一个做为输入，一个做为输出

在fuzz_in文件夹中创建一个文件input_file作为输入文件，随便输入一点内容

![image-20220205161330132](/Users/changzw/Library/Application Support/typora-user-images/image-20220205161330132.png)

输入指令：**afl-fuzz -i fuzz_in -o fuzz_out ./afl_test** 开始fuzz

这时候应该会出现一些问题

![img](https://img2018.cnblogs.com/blog/806432/201910/806432-20191025174802631-1397540032.png)

###  解决方案

1.查看报错，发现有个问题，需要 **core_pattern**

2.切换**root**

3.按提示输入指令切换 **echo core >/proc/sys/kernel/core_pattern**

4.重新输入指令 **afl-fuzz -i fuzz_in -o fuzz_out ./afl_test**



当cycles done中的数字变绿时，可以结束 ，CTRL + C 停止

![image-20220205161148079](/Users/changzw/Library/Application Support/typora-user-images/image-20220205161148079.png)

#### 结果分析：