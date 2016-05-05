---
layout: post
title: hello, world
date:  2015-11-24
author: kuga
---

关于 hello world，我们应该再熟悉不过了。
但你可能不知道，"hello, world" 的标准程序是没有感叹号，全部小写，逗号后面有空格，就像下面的例子。

```c
#include <stdio.h>

int main()
{
    printf("hello, world");
    return 0;
}
```

当然，还有一种更文艺的写法，这是之前在一本关于 Go 语言的书中看到的...

```python
def run(instructions):
    """
    运行指令
    """

    data = [0, 0]  # 数据单元
    ptr = 0        # 指针位置GG
    i = 0          # 命令下标

    # 运行指令字符串中的每一条指令(一个字符为一条指令)
    while i < len(instructions):
        # 指向下一个数据单元
        if instructions[i] == ">":
            ptr += 1

        # 指向前一个数据单元
        elif instructions[i] == "<":
            ptr -= 1

        # 数据单元值 +1
        elif instructions[i] == "+":
            data[ptr] += 1

        # 数据单元值 -1
        elif instructions[i] == "-":
            data[ptr] -= 1

        # 输出数据单元数值的 ASCII 码
        elif instructions[i] == ".":
            print(chr(data[ptr]), end="")

        # 循环处理
        elif instructions[i] == "[":
            # data[ptr] 为循环单元，若为 0
            # 则跳转到该循环的结束处退出循环，i 为跳转的下标
            if data[ptr] == 0:
                sign = 1

                # 找出当前循环对应的结束下标
                while sign != 0:
                    i += 1

                    if instructions[i] == "[":
                        sign += 1
                    elif instructions[i] == "]":
                        sign -= 1

                        # [+++[--]+++]
        # 循环处理
        elif instructions[i] == "]":
            # data[ptr] 为循环单元，若为 0，则退出循环
            if data[ptr] != 0:
                sign = -1

                # 若循环单元不为 0，则跳转到该循环对应的开始处
                while sign != 0:
                    i -= 1

                    if instructions[i] == "[":
                        sign += 1
                    elif instructions[i] == "]":
                        sign -= 1
        else:
            print("Illegal instruction: " + instructions[i])

        i += 1

# data[0] = 6   n >= 0
# data[1] = 10
run("++++++[>++++++++++<-]>+++++++.")
```

如果你已经知道输出结果，哎哟，不错哦！

分析上面的结果，其实第一个 data[0] 就是循环变量，第二个 data[1] 就是要输出的结果。
可以用一条算式总结：6*10 + 7，最后的结果就是 67 的 ASCII 码，也就是 'C'。

根据这个思想，改变指令串就可以输出 "hello, world" 了。

如果我养一只狗，名字就取 "hello"，这样我叫他的时候，就变成 "hello"，"汪!" 了。
