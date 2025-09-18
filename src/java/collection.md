---
title: 集合
author: xmy
---

9.18

### ArrayList、LinkedList和Vector的区别？

- ArrayList
  - 基于**数组**，查询较快，末尾插入O(1)，中间i处插入O(n)，需要移动元素，
  - 可通过序号快速获取对象，
  - **线程不安全**，
  - 末尾有预留的内存空间
  
- LinkedList
  - 基于**双向链表**，末尾插入O(1)，中间i处插入O(n)，但不需要移动元素，
  - 不可通过序号快速获取对象，
  - **线程不安全**，
  - 没有预留的内存空间，但每个节点都有两个指针占用了内存
  
- Vector

  - 基于数组

  - Vector是**线程安全**的，其各种增删改查方法加了synchronized修饰

### ArrayList扩容机制

> [!NOTE]
>
> - 构造时未指定大小
>   - 创建对象时，并未开辟数组空间。内部指向一个共享的静态空数组。
>   - 延迟初始化：第一次add时，扩容至10。
>   - 后面每次不够用时候扩容至1.5倍
> - 构造时指定大小
>   - 创建对象时，直接开辟数组空间
>   - 后面不够的时候扩容至1.5背倍
> - 扩容的算法
>   - 获取当前容量
>   - 计算新的容量（1.5倍）
>   - 新的容量能否满足需求
>     - 能的话，新的容量不变
>     - 不能的话，新的容量变成传入的参数（用户使用ensureCapacity(int minCapacity)手动扩容）
>   - 判断是否超出最大限制
>   - 创建新的数组并复制元素、旧数组进行垃圾回收

通过一次ArrayList创建和add的流程分析扩容机制：

首先，ArrayList有三个构造函数：

```java
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
```

```java
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```

```java
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

以及几个核心属性：

```java
    private static final int DEFAULT_CAPACITY = 10;
    private static final Object[] EMPTY_ELEMENTDATA = {};
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    transient Object[] elementData;
    private int size;
```

由此可以分析出：

如果创建空的Arraylist对象，则可以用无参构造器或其它两个分别以0和空的集合作为参数进行创建。当用无参构造器创建时，会将其内容数组elementData指向DEFAULTCAPACITY_EMPTY_ELEMENTDATA这个单例，而用其它两个构造器创建空的ArrayList对象时，会令elementData指向EMPTY_ELEMENTDATA这个单例

如果创建非空的，则只能用两个有参构造器进行创建，它们都会将elementData指向新创建的Object[]对象

再来看看add：

```java
	public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
```

```java
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```

后者在前者的基础上只是加了一些对数组越界和插入时移动部分对象的处理，扩容的核心则都在ensureCapacityInternal这个函数里

```java
    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }
```

其中`minCapacity`可以理解为扩容后容量的最小值，假设调用无参构造器创建了一个`ArrayList`对象，则初始状态下其`size=0`，`DEFAULT_CAPACITY=10`。此时我们对其调用`add(new Object())`，那么首先会调用`ensureCapacityInternal(0 + 1)`，意为新加入了一个对象，此时容量应至少为`1`。随后，判断该`ArrayList`是否是用无参构造器创建的，如果是，将`minCapacity`设为`max(DEFAULT_CAPACITY, minCapacity)`，即默认容量够用了就不用再扩容。随后调用`ensureExplicitCapacity(minCapacity)`

```java
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```

此时判断如果应该达到的容量超过现有数组的长度，则需要扩容，扩容就调用`grow(minCapacity)`方法

```java
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

首先设定新的容量为原来的`3/2`，再判定加完这部分是否够用，如果不够用就将新容量设定为刚刚好。这时如果新容量超出了系统规定的最大值，则调用`hugeCapacity(minCapacity)`将其设定为一个合理的最大值。最后将elementData拷贝到新容量的Object[]中。

```java
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

至此一次扩容算是完毕了。

**总结一下就是**

- **无参构造的第一次add扩容到10，随后每当加满了就扩容到原来的3/2。**
- **有参构造的如果构造的是空list，则第一次add扩容到1，第二次扩容到2，第三次扩容到3，第四次扩容到4，第五次扩容到6，再加满扩容到9，...，后面都是3/2了。**
- **有参构造非空list，按前面的序列来。最大容量为`Integer.MAX_VALUE=0x7fffffff`**

### HashMap、HashSet和HashTable的区别

- HashMap线程不安全，效率高一点，可以存储null的key和value，null的key只能有一个，null的value可以有多个。默认初始容量为16，每次扩充变为原来2倍。创建时如果给定了初始容量，则扩充为2的幂次方大小。底层数据结构为数组+链表，插入元素后如果链表长度大于阈值（默认为8），先判断数组长度是否小于64，如果小于，则扩充数组，反之将链表转化为红黑树，以减少搜索时间。

- HashSet是基于HashMap实现的，只是value都指向了一个虚拟对象，只用到了key

- HashTable线程安全，效率低一点，其内部方法基本都经过synchronized修饰，它基本被淘汰了，要保证线程安全可以用ConcurrentHashMap。

> [!NOTE]
>
> #### HashMap
>
> - 非线程安全
>
> - k和v允许出现null值
>
> - 初始化大小
>
>   - 默认是16（数组的默认大小）
>   - 如果用户传入的话，会向上拓展为原来的2的幂次
>   - 惰性加载：第一次插入时才初始化
>
> - 底层数据结构
>
>   - JDK1.8之前
>
>     - 数组（一个位置是一个哈希桶）+链表形式
>
>     - 计算哈希值：key的hashcode+扰动函数（使得分布更为均匀）
>     - 槽位：对计算出来的哈希值根据length求余得到数组索引
>       - 有值：key和hashcode是否相同
>         - 相同：覆盖
>         - 不相同：拉链法（产生哈希冲突）
>       - 无值：直接填
>
>   - JDK1.8之后，若产生哈希冲突
>
>     - 链表长度小于等于 **8**：继续使用链表（尾插法、头插法在多线程时可能会导致死循环的出现。但数据覆盖的问题依然存在，所以不是线程安全的）
>
>     - 哈希桶的数量小于**64**，链表长度超过**8**：进行扩容（扩容可能降低链表长度）
>     - 如果容量已经达到 **64**，并且链表长度超过 **8**：链表树化（红黑树）
>
> > length为什么是2的幂次
> >
> > - 运算速率：
> >
> >    `hash%length==hash&(length-1)`，计算哈希值运算速度快
> >
> >   > 扰动函数：`(h = key.hashCode()) ^ (h >>> 16);`
> >   >
> >   > 在length较小的情况下，哈希值高位没有参与到数组索引的计算。因此将高16位与低16位进行异或能够使得分布更均匀
> >
> > - 能够在哈希扩容时分布更均匀
> >
> >   2和4在length为2时候都在0中。length到4的时，跑到了0和2中，分布更均匀
> >
> >   **原本哈希桶i中的数据，要么保留在本地、要么跑去了新的哈希桶i+原容量当中**
>
> #### HashSet
>
> 基于HashMap实现，value都指向了一个虚拟对象





> [!NOTE]
>
> #### concurrenthashmap
>
> ##### 老版本
>
> - 数组+链表；有seg段的概念
>
> - 每个seg对应一个锁。默认大小是16，一旦初始化不能改变。不同seg的写入是可以并发执行的。
>
>   ![image-20250919005752426](C:\Users\HanFaye\AppData\Roaming\Typora\typora-user-images\image-20250919005752426.png)
>
> ##### 新版本
>
> - 数组+链表+红黑树；无seg的概念
>
> - 使用`synchronized`。锁定链表或红黑树的首节点。当发生hash冲突时候，才会获得锁
>
> ![image-20250919010015589](C:\Users\HanFaye\AppData\Roaming\Typora\typora-user-images\image-20250919010015589.png)
>
> 二者区别
>
> - 哈希冲突的解决办法
> - 并发的解决办法
> - 并发度上的差异









### HashSet、LinkedHashSet和TreeSet的区别

- LinkedHashSet是HashSet子类，在HashSet的基础上能过够按照添加的顺序遍历

- TreeSet底层使用红黑树，插入时按照默认/自定义的key排序规则指定插入位置

### 总结
- Collection下主要有List，Set，Map三个接口

- List和Set是继承了Collection接口

- Map只是依赖了Collection接口，不存在父子关系

- List主要有ArrayList，LinkedList，Vector

- Set主要有HashSet，TreeSet，LinkedHashSet

- Map主要有HashMap，HashTable，TreeMap，ConcurrentHashMap

- 当我们要存键值对，以便通过键值访问数据时，就用Map，此时需要排序就用TreeMap，不需要就用HashMap，需要线程安全就用ConcurrentHashMap

- 当我们只需要存对象时，就用Collection，需要保证唯一性就用Set，不需要就用List。用Set时，需要保证顺序就用TreeSet，不需要就用HashSet。用List时，如果是频繁查询，较少增删的场景，就用ArrayList，如果是频繁增删，较少查询的场景就用LinkedList，如果要保证线程安全就用Vector