---
title: C语言极简的双向链表实现
mathjax: false
date: 2024-04-18 16:54:22
tags:
- C
- 链表
- 双向链表
- quickjs
categories:
- [编程语言,C语言]
- [数据结构,链表]
---

最近在阅读 `Fabrice Bellard` 巨佬的 `quickjs` 项目源码，阅读之前我还复习了现代的JavaScript的语法知识，以期望对阅读源码有所帮助。
先从最简单的开始，在 `quickjs` 的源码中有一个 `Fabrice Bellard` 实现的双向链表实现，整个实现就是一个头文件 `list.h`。

> 在本文最下面提供了完整源码，希望对大家有所帮助。
<!-- more -->

## 链表结构体
```C
struct list_head {
    struct list_head *prev;
    struct list_head *next;
};
```
非常的简单，结构体中一共就两个成员，分别是**指向上一个链表节点的指针**、**指向下一个链表节点的指针**.
这让我想起来，`Linux`内核中链表的实现好像也是这样的，但是当时的我太菜了，没有继续去了解。

### 如何使用呢？
```C
#define NAME_SIZE 20

typedef struct {
    struct list_head link;  // 包含这个链表结构体
    char name[NAME_SIZE];
    int age;
} Person;

// 初始化一个person
Person * init_person(const char *name, int age) {
    Person *p = (Person *)malloc(sizeof(Person));
    strcpy(p->name, name);
    p->age = age;
    return p;
}
```
这里我们为了展示如何使用链表，建立了一个简单的结构体 `Person`，用来表示一个人。
可以看到，我们在结构体里面添加了成员 `struct list_head link;`，这样我们就可以依靠这个 `link` 成员来组成一个`person` 链表。

## 链表头
```C
static inline void init_list_head(struct list_head *head)
{
    head->prev = head;
    head->next = head;
}

int main()
{
    struct list_head person_list;
    init_list_head(&person_list);
}
```
而一个链表，需要一个头节点，这个头节点是不存储数据的，在这里也就是头节点不会被包含在 `person` 的结构体成员中，而是直接定义一个 `struct list_head` 变量。
然后调用 `init_list_head` 函数来初始化它，这里初始化完毕之后，`person_list` 变量它的 `prev` 指针和`next` 指针均指向自己，这种状态可以用来表示这个链表目前是**空**的。


## 向链表添加数据
现在我们向这个链表添加一个`person`，在添加之前呢，还有一些问题需要解决。
```C
/* insert 'el' between 'prev' and 'next' */
static inline void __list_add(struct list_head *el,
                              struct list_head *prev, struct list_head *next)
{
    prev->next = el;
    el->prev = prev;
    el->next = next;
    next->prev = el;
}
```
上面这个函数，通俗易懂，简而言之，**在两个节点之间插入一个新的节点**。

依靠 `__list_add` 函数，我们就可以实现下面两个功能：
1. `list_add`：在链表头部位置插入一个节点（注：在链表头的后一个位置插入，因为链表头是不保存数据的）。
2. `list_add_tail`：在链表尾部位置插入一个节点。
```C
/* add 'el' at the head of the list 'head' (= after element head) */
static inline void list_add(struct list_head *el, struct list_head *head)
{
    // 在头节点，和头节点的下一个节点之间插入新的节点 el
    __list_add(el, head, head->next);
}

/* add 'el' at the end of the list 'head' (= before element head) */
static inline void list_add_tail(struct list_head *el, struct list_head *head)
{
    // 在头节点的上一个节点，头节点之间插入新的节点 el
    __list_add(el, head->prev, head);
}

int main()
{
    struct list_head person_list;
    init_list_head(&person_list);

    // 在person_list头部插入一个person
    Person *p = init_person("dog", 23);
    list_add(&p->link, &person_list);
    
    Person *p1 = init_person("cat", 22);
    list_add(&p1->link, &person_list);

    Person *p2 = init_person("pig", 77);
    list_add_tail(&p2->link, &person_list);
}
```

大家是不是对`list_add_tail` 函数的实现感到奇怪？他明明实在头节点的上一个节点和头节点之间插入一个元素啊，为什么就插到链表的尾部去了呢？？？
其实你只要稍微想一想就会明白，这`tm`是一个循环链表啊！！！
头节点的上一个节点，就是链表的最后一个节点，这是一个循环！！！
接下来就很简单了。

## 链表for循环
`Fabrice Bellard` 贴心的封装了好几个宏，方便使用。我测试了前面两个宏。
```C
#define list_for_each(el, head) \
  for(el = (head)->next; el != (head); el = el->next)

#define list_for_each_safe(el, el1, head)                \
    for(el = (head)->next, el1 = el->next; el != (head); \
        el = el1, el1 = el->next)

#define list_for_each_prev(el, head) \
  for(el = (head)->prev; el != (head); el = el->prev)

#define list_for_each_prev_safe(el, el1, head)           \
    for(el = (head)->prev, el1 = el->prev; el != (head); \
        el = el1, el1 = el->prev)

int main()
{
    struct list_head person_list;
    init_list_head(&person_list);

    Person *p = init_person("dog", 23);
    list_add(&p->link, &person_list);
    
    Person *p1 = init_person("cat", 22);
    list_add(&p1->link, &person_list);

    Person *p2 = init_person("pig", 77);
    list_add_tail(&p2->link, &person_list);

    struct list_head *node, *node1;
    list_for_each(node, &person_list) {
        Person *tp = list_entry(node, Person, link);// 下文会提到list_entry宏是啥
        printf("name: %s, age: %d\n", tp->name, tp->age);
    }

    list_for_each_safe(node, node1, &person_list) {
        list_del(node); // 删除节点，后文也会提到
        free(node); // 释放内存
    }
    assert(person_list.next == person_list.prev);
    return 0;
}
```

结果输出完全符合要求：
```bash
name: cat, age: 22
name: dog, age: 23
name: pig, age: 77
```

这里还有一个宏没有交代：`list_entry`。
```C
#define CONTAINER_OF(ptr, type, member) \
    (type *)((char *)(ptr) - (char *) &((type *)0)->member)

/* return the pointer of type 'type *' containing 'el' as field 'member' */
#define list_entry(el, type, member) CONTAINER_OF(el, type, member)
```
这个宏就是链表实现的精髓所在了。
还记得前面的Person结构体吗？我们在里面保存了链表结构体成员 `struct list_head link`
```C
typedef struct {
    struct list_head link; // 包含这个链表结构体
    char name[NAME_SIZE];
    int age;
} Person;
```
而这个list_entry 宏的用法如下：
`Person *tp = list_entry(node, Person, link);`
我们把这个list_entry宏展开看一看是啥：
`(Person *) ( (char *)(node) - (char *)&((Person *)0)->link )`
分开讲讲就是
- `(char *)(node)`：link成员变量的实际地址。
- `(char *)&((Person *)0)->link`：link成员的距离Person结构体首地址的偏移量。
把这两个指针相减，就是person变量的实际地址。

## 链表节点删除
通俗易懂，链表无需管理内存释放，需要我们手动释放，很合理。
```C
static inline void list_del(struct list_head *el)
{
    struct list_head *prev, *next;
    prev = el->prev;
    next = el->next;
    prev->next = next;
    next->prev = prev;
    el->prev = NULL; /* fail safe */
    el->next = NULL; /* fail safe */
}
```

## 链表清空
太极客了，哈哈哈
```C
static inline int list_empty(struct list_head *el)
{
    return el->next == el;
}
```


## 源码实现
```C
/*
 * Linux klist like system
 *
 * Copyright (c) 2016-2017 Fabrice Bellard
 *
 * Permission is hereby granted, free of charge, to any person obtaining a copy
 * of this software and associated documentation files (the "Software"), to deal
 * in the Software without restriction, including without limitation the rights
 * to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
 * copies of the Software, and to permit persons to whom the Software is
 * furnished to do so, subject to the following conditions:
 *
 * The above copyright notice and this permission notice shall be included in
 * all copies or substantial portions of the Software.
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 * IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 * FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
 * THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
 * LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
 * OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
 * THE SOFTWARE.
 */
#ifndef LIST_H
#define LIST_H

#ifndef NULL
#include <stddef.h>
#endif

struct list_head {
    struct list_head *prev;
    struct list_head *next;
};

#define LIST_HEAD_INIT(el) { &(el), &(el) }

#define CONTAINER_OF(ptr, type, member) \
    (type *)((char *)(ptr) - (char *) &((type *)0)->member)

/* return the pointer of type 'type *' containing 'el' as field 'member' */
#define list_entry(el, type, member) CONTAINER_OF(el, type, member)

static inline void init_list_head(struct list_head *head)
{
    head->prev = head;
    head->next = head;
}

/* insert 'el' between 'prev' and 'next' */
static inline void __list_add(struct list_head *el,
                              struct list_head *prev, struct list_head *next)
{
    prev->next = el;
    el->prev = prev;
    el->next = next;
    next->prev = el;
}

/* add 'el' at the head of the list 'head' (= after element head) */
static inline void list_add(struct list_head *el, struct list_head *head)
{
    __list_add(el, head, head->next);
}

/* add 'el' at the end of the list 'head' (= before element head) */
static inline void list_add_tail(struct list_head *el, struct list_head *head)
{
    __list_add(el, head->prev, head);
}

static inline void list_del(struct list_head *el)
{
    struct list_head *prev, *next;
    prev = el->prev;
    next = el->next;
    prev->next = next;
    next->prev = prev;
    el->prev = NULL; /* fail safe */
    el->next = NULL; /* fail safe */
}

static inline int list_empty(struct list_head *el)
{
    return el->next == el;
}

#define list_for_each(el, head) \
  for(el = (head)->next; el != (head); el = el->next)

#define list_for_each_safe(el, el1, head)                \
    for(el = (head)->next, el1 = el->next; el != (head); \
        el = el1, el1 = el->next)

#define list_for_each_prev(el, head) \
  for(el = (head)->prev; el != (head); el = el->prev)

#define list_for_each_prev_safe(el, el1, head)           \
    for(el = (head)->prev, el1 = el->prev; el != (head); \
        el = el1, el1 = el->prev)

#endif /* LIST_H */
```