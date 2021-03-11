---
layout:     post
title:      "How to run C/C++ program on Mac os x using terminal"
subtitle:   ""
date:       2019-09-10 22:21:00
author:     "Echo Yuan"
tags:
    - C program 
---
下载了Visual Studio Code和JetBrains的CLion，发现不会用，哈哈~
还是手动在文本编辑器中码代码，然后命令行编译执行好了，原始但简单有效。

```c
#include <stdio.h>
int main(void)
{
    int n1, n2, result;
    printf("please input the two numbers:\n");
    scanf("%d%d", &n1, &n2);
    result = n1 + n2;
    printf("the result is: %d\n", result);
    return 0;
}
```
保存成`sum_2_nums.c`，然后打开命令行，进入到这个文件所在的目录，使用clang LLVM compiler来编译程序。
```
gcc sum_2_nums.c -o sum_2_nums
./sum_2_nums
``` 
或者
```
cc sum_2_nums.c -o sum_2_nums
./sum_2_nums.out
``` 

---
编译失败时的错误提示还是蛮精确的：

![gcc-error-tips](/img/in-post/how-to-run-c-program-on-mac-os-x-using-terminal/gcc-error-tips.png)

<hr/>

运行C++程序稍微有点儿不同
```
gcc -lstdc++ hello.cpp -o hello
```
The `-lstdc++` flag tells the linker to include the C++ Standard Library
参考自：[https://stackoverflow.com/questions/11852568/gcc-4-8-on-mac-os-x-10-8-throws-undefined-symbols-for-architecture-x86-64?#answer-11975663](https://stackoverflow.com/questions/11852568/gcc-4-8-on-mac-os-x-10-8-throws-undefined-symbols-for-architecture-x86-64?#answer-11975663)