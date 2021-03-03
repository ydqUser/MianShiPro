
<!-- TOC -->

- [集合](#集合)
    - [ArrayList](#arraylist)
    - [fail-fast](#fail-fast)
    - [fail-safe](#fail-safe)
- [枚举](#枚举)



<!-- /TOC -->

### base

#### Integer

Integer cache[] 存了 -128 到 127 之间的值

### 集合

#### ArrayList

扩容1.5倍左右：`newCapacity = oldCapacity + (oldCapacity >> 1)`

[refer](https://www.cnblogs.com/baichunyu/p/12965241.html)

#### fail-fast
[refer](https://www.cnblogs.com/54chensongxia/p/12470446.html)

fail-fast是一种错误检测机制，一旦检测到可能发生错误，就抛出异常，程序不继续往下执行

集合中的fail-fast机制

```
List<String> userNames = new ArrayList<String>() {{
    add("Hollis");
    add("hollis");
    add("HollisChuang");
    add("H");
}};

for (String userName : userNames) {
    if (userName.equals("Hollis")) {
        userNames.remove(userName);
    }
}
```
- 单线程
    - 使用增强的for循环执行remove/add操作(包括HashMap的put操作)，会抛出ConcurrentModificationException
    - 直接用iterator遍历时，遍历过程中向集合中添加元素或者不用iterator.remove方法删除数据。导致modeCount != expectedModCount
- 多线程
    - 当一个线程在遍历(foreach或iterator)这个集合，而另一个线程对这个集合的结构进行了修改 [示例](https://www.cnblogs.com/zhuyeshen/p/10956822.html)

- 异常产生的原因
增强的for循环使用的其实是`Iterator`迭代器，在调用iterator.next()/remove()方法时，会比较modCount和expectedModCount，如果二者不相等，抛出CMException
```
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```

- modCount是AbstractList/HashMap中的一个成员变量。它表示该集合实际被修改的次数，userName集合初始化完成之后，变量就有了，值为0
- expectedModCount是ArrayList的一个内部类Itr的成员变量。expectedModCount表示这个迭代器预期该集合被修改的次数，该值随着Itr被创建而初始化。**只有通过迭代器对集合进行操作，该值才会改变**

`userNames.remove`方法中，只修改了modCount，没有修改expectedModCount，导致modCount和迭代器中的expectedModCount不一致

- 使用普通for循环进行操作：因为普通for循环没有用到Iterator的遍历
- 直接使用Iterator进行操作：使用Iterator的remove方法
- filter过滤
- 使用增强的for循环，进行一次更新操作之后，立刻结束循环体，不再继续遍历，即不再执行下一次next方法
- fail-safe的集合类(`java.util.concurrent`包下的容器)
    - 这样的集合容器在遍历时不是直接在集合内容上访问的，而是先复制原有集合内容，在拷贝的集合上进行遍历

#### fail-safe

在遍历时不是直接在集合内容上访问的，而是先复制原有集合内容，在拷贝的集合上进行遍历(CopyOnWriteArrayList/)

- 需要复制对象，产生大量的无效对象，开销大
- 由于迭代时是对原集合的拷贝进行遍历，所以在遍历过程中对原集合所作的修改并不能被迭代器检测到
- 迭代器并不能访问到修改后的内容，即：迭代器遍历的是开始遍历那一刻拿到的集合拷贝，在遍历期间原集合发生的修改迭代器是不知道的

### 枚举

### other
### cookie

cookie和session的区别

cookie删除之后，session还可用吗