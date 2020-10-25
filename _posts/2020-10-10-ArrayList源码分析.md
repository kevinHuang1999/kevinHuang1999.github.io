---

layout:     post   				    # 使用的布局（不需要改）
title:      ArrayList源码分析 				# 标题 
subtitle:   java-jdk-ArrayList #副标题
date:       2020-10-10 				# 时间
author:     BY 	KevinHuang					# 作者
header-img: img/post-bg-2020-10-10-1.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Java
    - jdk
    

---


>ArrayList为我们经常使用的集合类，现在我们来深入分析它的实现

## 1.整体架构

​		ArrayList整体架构就是一个简单的数组。

![](https://github-blog-kevinhuang-1304012368.cos.ap-shanghai.myqcloud.com/img/20201025095043.png)

图中展示是长度为 10 的数组，从 1 开始计数，index 表示数组的下标，从 0 开始计数，elementData 表示数组本身，源码中除了这两个概念，还有以下三个基本概念：

- DEFAULT_CAPACITY 表示数组的初始大小，默认是 10，这个数字要记住；
- size 表示当前数组的大小，类型 int，没有使用 volatile 修饰，非线程安全的；
- modCount 统计当前数组被修改的版本次数，数组结构有变动，就会 +1。

接下来看下类注释

- 允许 put null 值，会自动扩容；

  ```
  <p>Each <tt>ArrayList</tt> instance has a <i>capacity</i>.  The capacity is the size of the array used to store the elements in the list.  It is always at least as large as the list size.  As elements are added to an ArrayList, its capacity grows automatically. 
  ```

- size、isEmpty、get、set、add 等方法时间复杂度都是 O (1)；

  

- 是非线程安全的，多线程情况下，推荐使用线程安全类：Collections#synchronizedList；

  ```
  * <p><strong>Note that this implementation is not synchronized.</strong> If multiple threads access an <tt>ArrayList</tt> instance concurrently, and at least one of the threads modifies the list structurally, it <i>must</i> be synchronized externally.
  ```

  ```
  If no such object exists, the list should be "wrapped" using the
  {@link Collections#synchronizedList Collections.synchronizedList}
  method.  This is best done at creation time, to prevent accidental unsynchronized access to the list
  <pre>
  List list = Collections.synchronizedList(new ArrayList(...));</pre>
  ```

​		

- 增强 for 循环，或者使用迭代器迭代过程中，如果数组大小被改变，会快速失败，抛出异常。ConcurrentModificationException

## 2.源码解析

### 2.1 初始化

三种初始化办法：无参数直接初始化、指定大小初始化、指定初始数据初始化

```java
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

//无参数直接初始化，数组大小为空
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

//指定初始数据初始化
//必须为集合的子类
public ArrayList(Collection<? extends E> c) {
    //elementData 是保存数组的容器，默认为 null
    elementData = c.toArray();
    //如果给定的集合（c）数据有值
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        //如果集合元素类型不是 Object 类型，我们会转成 Object
        if (elementData.getClass() != Object[].class) {
            elementData = Arrays.copyOf(elementData, size, Object[].class);
        }
    } else {
        // 给定集合（c）无值，则默认空数组
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

ArrayList 无参构造器初始化时，默认大小是空数组，并不是大家常说的 10，10 是在第一次 add 的时候扩容的数组值。



### 2.2 新增和扩容

新增: 往列表添加元素，分为两步

- 判断是否需要扩容，需要则执行扩容
- 直接赋值

```java
/**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return <tt>true</tt> (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        //判断是否需要扩容
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //直接赋值
        elementData[size++] = e;
        return true;
    }
```

看下扩容的源码（ensureCapacityInternal）

```java
 private static int calculateCapacity(Object[] elementData, int minCapacity) {
 				//如果初始化数组大小时，有给定初始值，以给定的大小为准，不走 if 逻辑
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }

    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }

    private void ensureExplicitCapacity(int minCapacity) {
        //记录数组被修改
        modCount++;

        // // 如果我们期望的最小容量大于目前数组的长度，那么就扩容
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```



```java
private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
  			//1.5倍扩容
        int newCapacity = oldCapacity + (oldCapacity >> 1);
  			// 如果扩容后的值 < 我们的期望值，扩容后的值就等于我们的期望值
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
  			// 如果扩容后的值 > jvm 所能分配的数组的最大值，那么就用 Integer 的最大值
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // 通过复制进行扩容
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

- 扩容的规则并不是翻倍，是原来容量大小 + 容量大小的一半，扩容后的大小是原来容量的 1.5 倍；
- ArrayList 中的数组的最大值是 Integer.MAX_VALUE，超过这个值，JVM 就不会给数组分配内存空间了。
- 新增时，并没有对值进行严格的校验，所以 ArrayList 是允许 null 值的。

我们可以知道，源码在扩容的时候，有数组大小溢出意识，就是说扩容后数组的大小下界不能小于 0，上界不能大于 Integer 的最大值，这种意识我们可以学习。

### 2.3 扩容的底层方法

扩容是通过这行代码来实现的：`Arrays.copyOf(elementData, newCapacity);`，这行代码描述的本质是数组之间的拷贝，扩容是会先新建一个符合我们预期容量的新数组，然后把老数组的数据拷贝过去，我们通过 System.arraycopy 方法进行拷贝，此方法是 native 的方法，源码如下：

```java
/**
 * @param src     被拷贝的数组
 * @param srcPos  从数组那里开始
 * @param dest    目标数组
 * @param destPos 从目标数组那个索引位置开始拷贝
 * @param length  拷贝的长度 
 * 此方法是没有返回值的，通过 dest 的引用进行传值
 */
public static native void arraycopy(Object src, int srcPos,
                                    Object dest, int destPos,
                                    int length);
```

我们可以通过下面这行代码进行调用，newElementData 表示新的数组：

```java
System.arraycopy(elementData, 0, newElementData, 0,Math.min(elementData.length,newCapacity))
```



### 2.4 删除

ArrayList 删除元素有很多种方式，比如根据数组索引删除、根据值删除或批量删除等等

我们选取根据值删除方式来进行源码说明

```java
public boolean remove(Object o) {
  	// 如果要删除的值是 null，找到第一个值是 null 的删除
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
      	// 如果要删除的值不为 null，找到第一个和要删除的值相等的删除
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
```

- 新增的时候是没有对 null 进行校验的，所以删除的时候也是允许删除 null 值的；
- 找到值在数组中的索引位置，是通过 equals 来判断的，如果数组元素不是基本类型，需要我们关注 equals 的具体实现

```java
private void fastRemove(int index) {
  // 记录数组的结构要发生变动了
  modCount++;
  // numMoved 表示删除 index 位置的元素后，需要从 index 后移动多少个元素到前面去
  // 减 1 的原因，是因为 size 从 1 开始算起，index 从 0开始算起
  int numMoved = size - index - 1;
  if (numMoved > 0)
    // 从 index +1 位置开始被拷贝，拷贝的起始位置是 index，长度是 numMoved
    System.arraycopy(elementData, index+1, elementData, index, numMoved);
  //数组最后一个位置赋值 null，帮助 GC
  elementData[--size] = null;
}
```

参考: [慕课专栏](https://www.imooc.com/read/47) 