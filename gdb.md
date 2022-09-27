# GDB

 关于gdb的使用参考

1. [gdb小技巧](https://wizardforcel.gitbooks.io/100-gdb-tips/content/break-on-address.html)
2. [gdb手册](https://sourceware.org/gdb/onlinedocs/gdb/index.html#SEC_Contents)

## help

如果不熟悉某个命令，运行`help<command>`获取帮助

## 单步调试

* `step`一次运行一行代码，当调用函数的时候，会进入被调用的函数。   

  *简化为`s`*

  `stepi`则是单步调试一个汇编指令（或者说是一条机器指令）

  *简化为`si`*

* `next`一次运行一行代码，当调用函数的时候，不会进入被调用的函数。

  *简化为`n`*

  `nexti`则是单步调试一个汇编指令（或者说是一条机器指令）

  *简化为`ni`*   



## 检查命令

* `print` 可以自定义打印某个变量或者表达式

*简化使用`p`* 

| \fmt | 功能                           |
| ---- | ------------------------------ |
| \d   | 以有符号，十进制的形式打印整数 |
| \u   | 以无符号，十进制的形式打印整数 |
| \x   | 以十六进制打印整数             |
| \o   | 以八进制打印整数               |
| \t   | 以二进制打印整数               |
| \f   | 以浮点数的形式打印变量         |
| \c   | 以字符的形式打印变量           |

在 `p`和`fmt`直接插入数字，表示查看的数量

```sh
p $pc //打印寄存器	··
```



* 查看内存

```shell
x/2c %a1
```

以字符形式查看`a1`寄存器指向的内存存储的两个字符

* 查看指令

```shell
x/3i address
```





## 设置断点

* 在程序地址处设置断点

  ```sh
  (gdb) b *address
  ```











