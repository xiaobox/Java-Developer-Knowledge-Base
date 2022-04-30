# Collections

## 创建空集合

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h0zybqtlb9j20v80av0ui.jpg)

**注意：返回的空集合是不可变集合，无法向其中添加或删除元素**。此外，也可以用各个集合接口提供的`of(T...)`方法创建空集合。例如，以下创建空`List`的两个方法是等价的：

```java
List<String> list1 = List.of();
List<String> list2 = Collections.emptyList();
```

## 创建单元素集合

*   创建一个元素的List：`List<T> singletonList(T o)`

*   创建一个元素的Map：`Map<K, V> singletonMap(K key, V value)`

*   创建一个元素的Set：`Set<T> singleton(T o)`

要注意到返回的单元素集合也是不可变集合，无法向其中添加或删除元素。

此外，也可以用各个集合接口提供的`of(T...)`方法创建单元素集合。例如，以下创建单元素`List`的两个方法是等价的：

```java
List<String> list1 = List.of("apple");
List<String> list2 = Collections.singletonList("apple");
```

实际上，使用`List.of(T...)`更方便，因为它既可以创建空集合，也可以创建单元素集合，还可以创建任意个元素的集合：

```java
List<String> list1 = List.of(); // empty list
List<String> list2 = List.of("apple"); // 1 element
List<String> list3 = List.of("apple", "pear"); // 2 elements
List<String> list4 = List.of("apple", "pear", "orange"); // 3 elements
```

## 创建不可变集合

`Collections`还提供了一组方法把可变集合封装成不可变集合：

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h0zymcl3cdj20yx09276i.jpg)

如果我们希望把一个可变`List`封装成不可变`List`，那么，返回不可变`List`后，最好立刻扔掉可变`List`的引用，这样可以保证后续操作不会意外改变原始对象，从而造成“不可变”`List`变化了：

```java
  public static void main(String[] args) {
        List<String> mutable = new ArrayList<>();
        mutable.add("apple");
        mutable.add("pear");
        // 变为不可变集合:
        List<String> immutable = Collections.unmodifiableList(mutable);
        // 立刻扔掉mutable的引用:
        mutable = null;
        System.out.println(immutable);
    }
```

利用JDK类库中提供的unmodifiableXXX方法最少存在以下几点不足：

*   笨拙：因为你每次都得写那么多代码；

*   不安全：如果没有引用到原来的集合，这种情况之下才会返回唯一真正永恒不变的集合；

*   效率很低：返回的不可修改的集合数据结构仍然具有可变集合的所有开销。

可以使用Guava 提供的不可变集合Immutable ：

*   当你往immutableList 中添加元素，也会抛出java.lang.UnsupportedOperationException异常；

*   **修改原集合后，immutable集合不变**

*所有Guava不可变集合的实现都不接受null值*，*如果你需要在不可变集合中使用null，请使用JDK中的Collections.unmodifiableXXX方法*

### Guava集合和不可变对应关系

| **可变集合接口**                                                                                                                              | **属于JDK还是Guava** | **不可变版本**                                                                                                                                                                                      |
| --------------------------------------------------------------------------------------------------------------------------------------- | ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Collection                                                                                                                              | JDK              | [ImmutableCollection](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/ImmutableCollection.html "ImmutableCollection")                         |
| List                                                                                                                                    | JDK              | [ImmutableList](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/ImmutableList.html "ImmutableList")                                           |
| Set                                                                                                                                     | JDK              | [ImmutableSet](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/ImmutableSet.html "ImmutableSet")                                              |
| SortedSet/NavigableSet                                                                                                                  | JDK              | [ImmutableSortedSet](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/ImmutableSortedSet.html "ImmutableSortedSet")                            |
| Map                                                                                                                                     | JDK              | [ImmutableMap](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/ImmutableMap.html "ImmutableMap")                                              |
| SortedMap                                                                                                                               | JDK              | [ImmutableSortedMap](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/ImmutableSortedMap.html "ImmutableSortedMap")                            |
| [Multiset](http://code.google.com/p/guava-libraries/wiki/NewCollectionTypesExplained#Multiset "Multiset")                               | Guava            | [ImmutableMultiset](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/ImmutableMultiset.html "ImmutableMultiset")                               |
| SortedMultiset                                                                                                                          | Guava            | [ImmutableSortedMultiset](http://docs.guava-libraries.googlecode.com/git-history/release12/javadoc/com/google/common/collect/ImmutableSortedMultiset.html "ImmutableSortedMultiset")           |
| [Multimap](http://code.google.com/p/guava-libraries/wiki/NewCollectionTypesExplained#Multimap "Multimap")                               | Guava            | [ImmutableMultimap](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/ImmutableMultimap.html "ImmutableMultimap")                               |
| ListMultimap                                                                                                                            | Guava            | [ImmutableListMultimap](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/ImmutableListMultimap.html "ImmutableListMultimap")                   |
| SetMultimap                                                                                                                             | Guava            | [ImmutableSetMultimap](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/ImmutableSetMultimap.html "ImmutableSetMultimap")                      |
| [BiMap](http://code.google.com/p/guava-libraries/wiki/NewCollectionTypesExplained#BiMap "BiMap")                                        | Guava            | [ImmutableBiMap](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/ImmutableBiMap.html "ImmutableBiMap")                                        |
| [ClassToInstanceMap](http://code.google.com/p/guava-libraries/wiki/NewCollectionTypesExplained#ClassToInstanceMap "ClassToInstanceMap") | Guava            | [ImmutableClassToInstanceMap](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/ImmutableClassToInstanceMap.html "ImmutableClassToInstanceMap") |
| [Table](http://code.google.com/p/guava-libraries/wiki/NewCollectionTypesExplained#Table "Table")                                        | Guava            | [ImmutableTable](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/ImmutableTable.html "ImmutableTable")                                        |

## 线程安全集合

`Collections`还提供了一组方法，可以把线程不安全的集合变为线程安全的集合：

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h0zz4oj14vj213p08ydi6.jpg)

## checked 集合

Checked集合具有检查插入集合元素类型的特性。

例如当我们设定checkedList中元素的类型是String的时候，如果插入其他类型的元素就会抛出ClassCastExceptions异常，Java5中提供泛型的功能，泛型功能能够在代码编译阶段就约束集合中元素的类型，但有些时候声明的集合可能是raw集合，编译阶段的类型约束就不起作用了，这个时候Checked集合就能起到约束集合中元素类型的作用。

比如：

```java
//定义成这样，会在编译期有类型检测，不对也过不了
List<Integer> list = xxx;
// 如果是这样就检测不出来了
List newList = list;
// 可以用到 checked集合
List newList = Collections.checkedList(list, Integer.class);


```

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h0zzjbts7sj21060abmzw.jpg)

## 查找替换&#x20;

*   fill 使用指定元素替换指定列表中的所有元素。

*   frequency 返回指定 collection 中等于指定对象的元素数。

*   indexOfSubList 返回指定源列表中第一次出现指定目标列表的起始位置，如果没有出现这样的列表，则返回 -1。

*   lastIndexOfSubList 返回指定源列表中最后一次出现指定目标列表的起始位置，如果没有出现这样的列表，则返回-1。

*   max 根据元素的自然顺序，返回给定 collection 的最大元素。

*   min 根据元素的自然顺序 返回给定 collection 的最小元素。

*   replaceAll 使用另一个值替换列表中出现的所有某一指定值。

## 集合排序

*   reverse 对List中的元素倒序排列

*   shuffle 对List中的元素随机排列

*   sort 对List中的元素排序

*   swap 交换List中某两个指定下标位元素在集合中的位置。

*   rotate 根据指定的距离轮换指定列表中的元素。

## 其他方法

binarySearch使用二进制搜索算法来搜索指定列表，以获得指定对象。

addAll将所有指定元素添加到指定 collection 中。

copy将所有元素从一个列表复制到另一个列表。

disjoint如果两个指定 collection 中没有相同的元素，则返回 true。

nCopies返回由指定对象的 n 个副本组成的不可变列表。

## 参考

*   [https://www.liaoxuefeng.com/wiki/1252599548343744/1299919855943714](https://www.liaoxuefeng.com/wiki/1252599548343744/1299919855943714 "https://www.liaoxuefeng.com/wiki/1252599548343744/1299919855943714")

*   [https://www.jianshu.com/p/bf2623f18d6a](https://www.jianshu.com/p/bf2623f18d6a "https://www.jianshu.com/p/bf2623f18d6a")

*   [https://blog.51cto.com/zlfwmm/1709837](https://blog.51cto.com/zlfwmm/1709837 "https://blog.51cto.com/zlfwmm/1709837")
