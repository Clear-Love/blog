---
title: C语言基础
date: 2024-04-10T10:02:28+08:00
draft: false
tags:
  - C语言
categories:
  - C语言
series:
  - C language
hidden: false
toc:
  enable: true
---
## 环境配置
### 1.安装mingw

```shell
scoop install mingw
```

### 2.验证

```shell
gcc -v
```
![](https://pan.lmio.xyz/pic/98257bac12069f45b8e2d80d5b41a6fa.png)

## 数据类型
---
###  定义：一段固定大小数据内存的别名

| 关键字           | 名称    | 大小（64位系统） | 值域            |
| ------------- | ----- | --------- | ------------- |
| int           | 整型    | 4字节       | -2^31, 2^31-1 |
| float         | 浮点型   | 4字节       | ...           |
| long int      | 长整型   | 8字节       | -2^63, 2^63-1 |
| long long int | 长长整型  | 8字节       |               |
| short         | 短整型   | 2字节       | -2^15, 2^15-1 |
| bool          | 布尔类型  | 1字节       | 0, 1          |
| unsiged int   | 无符号整型 | 4字节       | 0, 2^32-1     |
- 1字节=8比特
## 计算过程
---

1. 整型数据在内存中存储都是以补码的方式存放
2. 数据在运算的时候是以补码的方式运算
3. 整数反码补码都是一样的
4. 补码运算的时候，符号位也参与运算
5. 运算结果为补码，*printf*函数打印时，补码还原为原码


- 原码：数据转换成二进制的样子
- 反码：在原码的基础上，`0`变成`1`，`1`变成`0`，符号位不变
- 补码： 在反码的基础上加1

- 一个有符号整数，使用`%u`方式输出，直接输出补码

## 变量
---

- mian函数外部只用来定义和声明全局变量
- 全局变量在定义时没有赋值，默认为0
- 赋值--实际上就是将数据存储变量所在的内存空间
- =赋值符号的左边称之为左值，指的是变量的内存空间
- =赋值符号的右边称之为右值，指的是变量的值

## printf函数
---
```c
#include<stdio.h>
int main() {
	printf("hello, world");
	while(1);
	return 0;
}
```

- 编译运行后输出为空

### 输入输出缓冲机制
---
>当我们程序运行的时候，将数据输出到屏幕终端，首先数据会先放入一个缓冲区中，然后当达到一定条件时，数据才输出到终端

![image.png](https://pan.lmio.xyz/pic/789add7d18893d0bd866ef8aa01e3f04.png)

**满足条件**

1. 遇到换行符时，系统会将这个程序的输出缓冲区的数据冲刷到屏幕终端上
2. fflush(stdout)刷新缓冲区
3. 缓冲区满

**占位符**

| 占位符 | 输出类型  | 扩展                 |
| --- | ----- | ------------------ |
| %d  | 整型    | %-{x}d 左对齐x个字符     |
| %f  | 浮点型   | %.{x}f  表示保留小数点后x位 |
| %e  | 科学计数法 | 以科学计数法的方式输出浮点数     |
| %o  | 八进制   | %#o 按格式输出          |
| %x  | 十六进制  | %#x 按格式输出          |
| %c  | 字符    |                    |
| %s  | 字符串   |                    |
|     |       |                    |
|     |       |                    |

### 带颜色输出
---

```c
#include <stdio.h>
int main() {
	printf("\033[1;31m Hello, World! \033[0m\n"); // 红色文本 
	return 0;
}
```
## scanf函数
---
- 原型：int scanf(const char *format,...)
- 作用：从键盘上获取数据，存储到变量的内存空间中。
- 参数：const char\*format是格式控制字符串，它指定了输入数据的格式。
- 返回值：返回的是整型数据，代表获取成功的个数

```c
#include<stdio.h>

int main() {
	int age;
	float score;
	int ret;
	ret = scanf("%d %f", &age, &score);
	// 获取失败，重新处理
	while(ret < 2) {
		printf("获取失败，请重新输入\n");
		ret = scanf("%d%f", &age, &score);
	}
	printf("ret:%d age:%d score:%f\n", ret, age, score);
}
```

程序进入死循环，在*while*循环中*scanf*函数没有等待输入

- *scanf*函数没有正确读取，不会从输入缓冲区拿走数据
- 在第二次输入前清空缓冲区

```c
#include<stdio.h>

int main() {
	int age;
	float score;
	int ret;
	ret = scanf("%d %f", &age, &score);
	// printf("%d", ret);
	// 获取失败，重新处理
	while(ret < 2) {
		// 清空输入缓冲区
		while (getchar() != '\n');
		printf("获取失败，请重新输入\n");
		ret = scanf("%d%f", &age, &score);
	}
	printf("ret:%d age:%d score:%f\n", ret, age, score);
}
```

> %c会把空格和换行符在内的字符读入，

```c
#include<stdio.h>

int main() {
	int age;
	char sex;
	printf("请输入年龄：");
	scanf("%d",&age);
	while(getchar() != '\n');
	printf("请输入性别：");
	scanf(" %c", &sex);
	printf("age:%d sex:%c", age, sex);
	return 0;
}
```

## 算术运算符
---
### 除法运算 /

- 除法运算时，除数不能为0
- 整型/整型 结果只能是整型，小数部分直接舍弃，即向下取整
- 若参与运算的值有一个是浮点数，运算过程中把所有数转化为浮点数，运算结果也是浮点数

### 取余运算 %

- 取余运算中参与运算的除数只能是整数

### 自增运算符 ++

- 自增属于单目运算符，用于一个数的左端或者右端
- 如果++在后，先参与运算，再自增
- 如果++在前，先自增，再参与运算

```c
#include<stdio.h>

int main() {
	int x = 2;
	int y = 10 + x++;
	printf("x=%d, y=%d\n", x, y);
	// x=3, y=12
	int z = 10 + (++x);
	printf("x=%d, z=%d\n", x, z);
	// x=4, z=14
	return 0;
}
```

```c
#include<stdio.h>

int main() {
	int i = 3;
	int ret = i++ + i++; // 3 + 4
	printf("i=%d, ret=%d\n", i, ret);
    // i=5, ret=7
    i = 3;
	ret = i++ + (++i); // 3 + 5
	printf("i=%d, ret=%d\n", i, ret);
    // i=5, ret=8
    return 0;
}
```

### 关系运算符 > < == >= <= !=

- 由关系运算符组成的表达式称之为关系表达式
- 关系表达式的结果只有两种，要么关系成立为`1`，不成立为`0`

```c
#include<stdio.h>

int main() {
	int a = 10;
	printf("%d\n", 0<a<2); // 1
	int b = a == 10;
	printf("%d\n", b); // 1
	return 0;
}
```

### 逻辑运算符 && ||

>由逻辑运算符组成的表达式称之为逻辑表达式，由逻辑运算符组成的表达式称之为逻辑表达式，逻辑表达式的结果是布尔值，要么为*真*（**非0为真**），要么为*假*

- 逻辑表达式中存在惰性运算（短路特性），如果前面的表达式已经能表达出明确的结果了，那么后面就不会算了。

### 位运算符 << >> ~ | & ^

> 位运算指将一个数据进行bit位运算，也就是说，要将这个数据转成二进制（补码），然后再进行运算位操作。

- 按位或|：至少有一位为1，结果就为1
- 按位与&：两个都为1，结果才为1
- 按位取反~：把每个bit位的0变为1，1变为0
- 按位异或^：两个比特位不同为1，相同为0
- 按位左移<<：除符号位，所有的bit位左移x位
- 按位右移>>：除符号位，所有的bit位右移x位