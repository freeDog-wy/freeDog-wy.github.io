---
title: C语言中那些奇奇怪怪的宏
date: 2024-04-14 11:00:47
tags: 
- C语言
- 宏
categories: 
- [编程语言, C语言]
mathjax: false
---

今天整理了一下C语言一些奇奇怪怪的宏，给大家瞧一瞧。

<!-- more -->

## TO_STR 添加双引号
通过在一个标识符前添加 # 号，可以给这个标识符加上双引号。
```C
#define TO_STR(x) #x

printf("%s\n", TO_STR(hello world));
printf("%s\n", TO_STR(234));
```
```bash
hello world
234
```

## TO_CHAR 添加单引号
```C
#define TO_CHAR(x) #@x

printf("%c\n", TO_CHAR(a));
```
```bash
a
```
`a`并不是变量，啥也不是，只是通过预处理器给它套了一个单引号，它就变成字符了。


## PRIMITIVE_CAT 字符拼接
符号 ## 可以拼接两个标识符。
至于为什么这里给它取名叫PRIMITIVE_CAT宏，是因为宏扩展的一个机制，往下看就知道了。
```C
#define PRIMITIVE_CAT(x,y)   x ## y
int age = 10;

printf("age: %d\n", PRIMITIVE_CAT(a, ge));
```
```bash
age: 10
```

## CAT 字符拼接
可能你会感到奇怪，这不就是套了一层皮吗，有啥区别呢？
```C
#define CAT(x,y) PRIMITIVE_CAT(x,y)
#define X a
int age = 10;
printf("%d\n", CAT(X, ge));
```
```bash
10
```
但是如果把上面的 `CAT` 替换成 `PRIMITIVE_CAT`，看看会发生啥？
```C
#define CAT(a,b) PRIMITIVE_CAT(a,b)
#define X a
int age = 10;
printf("%d\n", PRIMITIVE_CAT(X, ge));
```
```bash
找不到变量 Xge
```
**报错了**
预处理器直接把 `X` 拿来拼接了，并没有把 `X` 替换成 `a`.
那原因是什么呢？

首先宏函数展开有两种情况：
1. 参数参与了 ## 或者 # 符号的操作
    此时，由于 `PRIMITIVE_CAT` 宏函数中的参数`X Y`参与了## 拼接，那么就不会对宏 `X Y` 进行展开后传入，而是直接传入。
    ```C
        #define X 0
        #define Y 1
        #define XY 2
        #define PRIMITIVE_CAT(a,b) a##b
        printf("%d\n", PRIMITIVE_CAT(X,Y));
    ```
    所以拼接结果是 `PRIMITIVE_CAT(X,Y) -> XY -> 2`. 
    拼接后得到了宏`XY`，直接展开成 `2`.
    ```bash
        2
    ```
2. 参数没有参与 ## 或者 # 符号的操作
    此时，`CAT` 宏函数的参数并没有参与拼接，则先对参数 `X Y`进行展开，分别展开为 `0 1`，然后传入宏函数。
    ```C
        #define X 0
        #define Y 1
        #define CAT(a,b) PRIMITIVE_CAT(a,b)
        printf("%d\n", CAT(X,Y));
    ```
    所以拼接结果是`CAT(X,Y) -> PRIMITIVE_CAT(0,1) -> 01`
    ```bash
        1
    ```
可以看出这两个拼接宏函数，都是很常用的。

## OFFSET_OF 得到一个字段在结构体struct中的偏移量(字节数)
```C
#define OFFSET_OF(type,field) ( (size_t) &(( type *) 0)-> field )

#define NAME_SIZE 20
struct Person
{
    char name[NAME_SIZE];
    int score;
    int age;
};

int main()
{
    printf("age offset of struct Person: %u\n", OFFSET_OF(struct Person, age));
    return 0;
}
```
```bash
24
```
在我的电脑上，`int`类型是`4`字节的，`char`类型是`1`字节的。所以字段 `age` 排在 `name` 和 `score`的后面，他到结构体首地址偏移就是 `24`字节。

什么？你要问这个宏函数原理是啥？
这个很简单的，首先上面那个宏函数展开的结果是这个：
```C
(size_t) &((struct Person *)0)->age
```
他把地址为`0`，强转为一个`struct Person *`类型，所以它变成了一个指针，然后就可以访问结构体中的 `age`字段了，再然后通过取地址符 `&` 得到`age`字段的地址，很显然，因为结构体首地址是`0`，所以`age`字段的地址就是偏移量啦。


## FILED_SIZE 得到一个结构体中字段的大小（字节数）
```C
#define FILED_SIZE( type, field ) sizeof( ((type *) 0)->field )

printf("filed size of age: %u\n", FILED_SIZE(struct Person, age));
```
```bash
4
```
在我的电脑上`int`类型`4`个字节.

## CONTAINER_OF 根据结构体变量字段地址获取整个结构体的存储空间首地址

- `ptr`：结构体字段的指针
- `type`：结构体类型
- `filed`: 字段名称
```C
#define CONTAINER_OF(ptr, type, filed) \
    (type *)((char *)(ptr) - (char *) &((type *)0)->filed )

int main()
{
    struct Person p = {
        .name = "dog",
        .score = 30,
        .age = 2332432
    };
    struct Person *pp = CONTAINER_OF(&p.age, struct Person, age);
    printf("name: %s\n", pp->name);

}
```
```bash
dog
```
虽然这个样例看起来多此一举，但是这个宏函数在操作系统底层用是很常用的。
可以用于C的面向对象设计，类的继承，从父类访问子类。

原理是啥？
其实只要理解了C语言的内存布局，这个就不是问题。


## UPPERCASE 字母大小写转换
```C
#define UPPERCASE(c) ( ((c) >= 'a' && (c) <= 'z') ? ((c) - 0x20) : (c) )
#define LOWERCASE(c) ( ((c) >= 'a' && (c) <= 'z') ? (c) : ((c) + 0x20))

printf("Uppercase: %c -> %c\n", 'x', UPPERCASE('x'));
printf("Lowercase: %c -> %c\n", 'X', LOWERCASE('X'));
```
```bash
Uppercase: x -> X
Lowercase: X -> x
```


## ARRAY_SIZE 获取一个数组元素的个数
```C
#define ARRAY_SIZE(arr) (sizeof((arr)) / sizeof((arr[0])))

struct Person p_arr[10] = {0};
printf("p_arr size: %u\n", ARRAY_SIZE(p_arr));
```
```bash
p_arr size: 10
```

## LOG 打印日志
```C
#define LOG_ON 1
#if LOG_ON

#define LOG(fmt, ...) \
    printf("[FILE: %s] [FUNCTION: %s] [LINE: %d] " fmt "\n", \
        __FILE__, __FUNCTION__, __LINE__, ##__VA_ARGS__)

#else

#define LOG(fmt, ...)

#endif

LOG("一次LOG测试");
LOG("测试可变参: %d", 4);
```
```bash
[FILE: D:\xx.c] [FUNCTION: main] [LINE: 72] 一次debug测试
[FILE: D:\xx.c] [FUNCTION: main] [LINE: 73] 测试可变参: 4
```




## FOR_EACH 函数宏递归实现For循环
- `count`： 循环次数，我的这个实现最多只能循环三次，不过你可以继续增加
- `marco`： 相当于回调宏
    - `idx`： 当前枚举下标的，顺序是 `count, count - 1, count - 2, ..., 2, 1`, 倒序下标
    - `x`：这个是枚举的值
- `...`：可变参数，枚举的值
```C
#include <stdio.h>

#define PRIMITIVE_CAT(a,b)   a ## b
#define CAT(a,b) PRIMITIVE_CAT(a,b)

#define INT_DEFINES(idx, x) int CAT(a_, x) = idx
#define PRINT(idx, x)    printf(#x": %d idx: %d\n", CAT(a_, x), idx)

#define FOR_EACH(count, marco, ...) CAT(FOR_EACH_, count)(marco, ##__VA_ARGS__)
#define FOR_EACH_3(marco, arg, ...) marco(3, arg); FOR_EACH_2(marco, ##__VA_ARGS__)
#define FOR_EACH_2(marco, arg, ...) marco(2, arg); FOR_EACH_1(marco, ##__VA_ARGS__)
#define FOR_EACH_1(marco, arg, ...) marco(1, arg)


int main()
{
    // IF_ELSE (0) ( \
    //     printf("YES\n"); \
    // )( \
    //     printf("NO\n"); \
    // )
    FOR_EACH(3, INT_DEFINES, dog, cat, pig);
    
    FOR_EACH(3, PRINT, dog, cat, pig);
    
    return 0;
}
```
宏展开后真实的样子是：
```C
int a_dog = 3; int a_cat = 2; int a_pig = 1;
printf("dog"": %d idx: %d\n", a_dog, 3); printf("cat"": %d idx: %d\n", a_cat, 2); printf("pig"": %d idx: %d\n", a_pig, 1);
```
```bash
dog: 3 idx: 3
cat: 2 idx: 2
pig: 1 idx: 1
```
不得不说，宏的代码可读性确实太差了，哈哈哈！
> **注意！！！** 我用`MSVC`编译上述`FOR_EACH`宏代码出现了奇怪的问题： `x86` 的 `Microsoft (R) C/C++` 优化编译器 `19.39.33521` 版。
> 但是在 `GNU C`, `Clang` 上没有问题，大家也可以自己实验一下。

