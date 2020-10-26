---

layout:     post   				    # 使用的布局（不需要改）
title:      LinkedList源码分析 				# 标题 
subtitle:   java-jdk-LinkedList #副标题
date:       2020-10-20 				# 时间
author:     BY 	KevinHuang					# 作者
header-img: img/post-bg-2020-10-20-1.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Java
    - jdk
    


---


>LinkedList一般使用于先进先出(FIFO)和先进后出(FILO)的场景，在队列源码实现中被频繁使用。

## 1.整体架构

LinkedList底层是一个双向链表，如图所示

![](https://github-blog-kevinhuang-1304012368.cos.ap-shanghai.myqcloud.com/img/20201025131008.png)

- first 是双向链表的头节点，它的前一个节点是 null。
- last 是双向链表的尾节点，它的后一个节点是 null；
- 当链表中没有数据时，first 和 last 是同一个节点，前后指向都是 null；
- 因为是个双向链表，只要机器内存足够强大，是没有大小限制的。

链表的元素为Node，实现：

```java
private static class Node<E> {
    E item;
    Node<E> next; //指向下一个节点
    Node<E> prev; //指向前一个节点

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

## 2.源码解析

### 2.1 添加元素

添加元素时，可以选择添加至头部或者尾部，默认的add方法为添加至尾部。

```java
public boolean add(E e) {
    linkLast(e);
    return true;
}
```

添加至尾部

```java
void linkLast(E e) {
    final Node<E> l = last; //旧的尾结点
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode; //新的尾结点指向新增结点
    //如果链表为空
  	if (l == null) 
        first = newNode;
  	//否则把前尾节点的下一个节点，指向当前尾节点
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

添加至头部

```java
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
  	// 头节点为空，就是链表为空，头尾节点是一个节点
    if (f == null)
        last = newNode;
  	//上一个头节点的前一个节点指向当前节点
    else
        f.prev = newNode;
    size++;
    modCount++;
}
```

### 2.2 节点删除

节点删除的方式和添加类似，我们可以选择从头部删除，也可以选择从尾部删除，删除操作会把节点的值，前后指向节点都置为 null，帮助 GC 进行回收。

从头部删除

```java
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException(); //不能为空链表
    return unlinkFirst(f);
}
```



```java
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    first = next; //头结点指向删除结点的下一结点
    if (next == null) //删除后链表为空
        last = null;
  	//不为空，则下一结点的前驱为null
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
```

尾部的删除和头部的删除差不多，这里就不贴出来了。

**从源码中我们可以了解到，链表结构的节点新增、删除都非常简单，仅仅把前后节点的指向修改下就好了，所以 LinkedList 新增和删除速度很快。**

### 2.3 节点查询

链表中查询一个节点是比较慢的，相对于数组的随机存储，LinkedList需要将链表遍历一遍才能查找到元素，看下源码

```java
Node<E> node(int index) {
    // assert isElementIndex(index);
		
  	//如果index处于链表的前半部，则从头开始
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else { //如果处于后半部，则从尾部开始
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

我们可以发现LinkedList并没有采取从头到尾遍历一遍的方法，而是通过size这个变量来进行二分法，首先看看index在链表的前半部分还是后半部分，若是前半部分，则从头开始，若是后半部分，则从尾开始。通过这种方式，循环的数目可以减少一半，提升了查找的性能，而正是我们记录的size变量记录链表大小，才可以进行这样的优化。

### 2.4 方法对比

LinkedList 实现了 Queue 接口，在新增、删除、查询等方面增加了很多新的方法，这些方法在平时特别容易混淆，在链表为空的情况下，返回值也不太一样

| 方法含义 | 返回异常  | 返回特殊值 | 底层实现                                         |
| :------- | :-------- | :--------- | :----------------------------------------------- |
| 新增     | add(e)    | offer(e)   | 底层实现相同                                     |
| 删除     | remove()  | poll(e)    | 链表为空时，remove 会抛出异常，poll 返回 null。  |
| 查找     | element() | peek()     | 链表为空时，element 会抛出异常，peek 返回 null。 |

PS：Queue 接口注释建议 add 方法操作失败时抛出异常，但 LinkedList 实现的 add 方法一直返回 true。
LinkedList 也实现了 Deque 接口，对新增、删除和查找都提供从头开始，还是从尾开始两种方向的方法，比如 remove 方法，Deque 提供了 removeFirst 和 removeLast 两种方向的使用方式，但当链表为空时的表现都和 remove 方法一样，都会抛出异常。

参考: [慕课专栏](https://www.imooc.com/read/47) 