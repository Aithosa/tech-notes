# ConcurrentModificationException 引起的问题

## 现象

这是一个由 ConcurrentModificationException 引发的问题，先看下这段程序:

```Java
List<String> tmpList1 = new LinkedList<>();
tmpList1.add("Hello");
tmpList1.add("My");
tmpList1.add("Son");

for (String curStr : tmpList1) {
    if ("My".equals(curStr)) {
        tmpList1.remove(curStr);
    }
    System.out.printf("curStr = %s, tmpList = %s\n", curStr, tmpList1.toString());
}
```

首先整体的说下这段程序，在 for-each 循环中调用`add`或者`remove`修改内容肯定是不妥的，会引发标题里的`ConcurrentModificationException`异常。但是这个异常在这段程序里并不会百分百触发，接下来我就直接说下运行结果了。

当像上面这样移除`"My"`时，程序并不会报错，而换成`"Son"`时也不会报错，除此之外移除 tmpList1 里的其他任何元素都会引起`ConcurrentModificationException`。这是 tmpList1 定义为`LinkedList`的运行结果。输出是:

```Java
# 移除My
curStr = Hello, tmpList = [Hello, My, Son]
curStr = My, tmpList = [Hello, Son]

# 移除son
curStr = Hello, tmpList = [Hello, My, Son]
curStr = My, tmpList = [Hello, My, Son]
curStr = Son, tmpList = [Hello, My]
```

而当 tmpList1 定义为`ArrayList`时，仅仅移除`"My"`时，程序不会报错，除此之外移除其他元素都会引发`ConcurrentModificationException`异常。
输出是:

```Java
# 移除My
curStr = Hello, tmpList = [Hello, My, Son]
curStr = My, tmpList = [Hello, Son]

curStr = Hello, tmpList = [Hello, My, Son]
curStr = My, tmpList = [Hello, My, Son]
curStr = Son, tmpList = [Hello, My]
Exception in thread "main" java.util.ConcurrentModificationException
```

## 原因

### ConcurrentModificationException

由于这一切和 ConcurrentModificationException 有关，所以先讲下为什么会报这个异常。

for-each 循环本质上是在内部用了迭代器，但也只是在内部调用，对用户并没有暴露，因此用户想要操作某个元素，比如删除，只能通过对象去调`remove`方法，于是就导致迭代和修改途径不一致，迭代的内容会受到修改的影响出现难以预料的情况。为了防止这种情况发生，`ArrayList`和`LinkedList`等在内部增加了一个`modCount`字段，记录修改了几次，当通过对象去调`remove`方法或者`add`方法时，每修改一次就会`modCount++`，如果不使用迭代器的话，这个字段不会产生任何影响，如果使用了迭代器，比如 for-each 循环，这个字段就会专门用于检查 ConcurrentModificationException，在获取下一个元素时，首先就会检查`modCount`和`expectedModCount`是否相等，如果有在迭代器控制范围外修改的情况就会导致`modCount != expectedModCount`，于是就会报`ConcurrentModificationException`。

### 为什么删的元素不同程序运行的结果会不同

首先讲下 for-each 循环的简单过程。这循环中主要做两件事，先是判断还有没有下一个元素(`hasNext`)，如果有的话就获取下一个元素(`next`)，获取到之后用户可以进行操作。`ConcurrentModificationException`就是在获取下一个元素(`next`)的一开始做的检查里抛出的。

也就是说只要修改了元素之后循环还会调`next`，那么就一定会抛出`ConcurrentModificationException`。因此对于`ArrayList`和`LinkedList`来说需要仔细查看的就是`hasNext`的实现，只要还判断有下个元素，就一定会调`next`。

以下是`LinkedList`的`hasNext`实现:

```Java
public boolean hasNext() {
    return nextIndex < size;
}
```

这里看出判定还是比较宽的，调用`remove`会导致`size`减小，不管是移除倒数第二个还是最后一个元素，都会判定为循环提前结束，就不会再去调用`next`了。

下面是`ArrayList`的`hasNext`实现:

```Java
public boolean hasNext() {
    return cursor != size;
}
```

相对于`LinkedList`，这里的判定会略显严格。以只移除一个元素为例，当移除的是倒数第二个的时候，`cursor`和`size`是刚好相等的，于是循环结束，就不会抛出异常。当移除的是最后一个元素时，`cursor != size`为`true`会继续尝试获取下一个元素，于是就会抛出`ConcurrentModificationException`。
