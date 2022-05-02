# new 一个对象占多少个字节？

可以参考  [对象头（mark word）](../../JAVA%20技术栈/JVM/对象头（mark%20word）/对象头（mark%20word）.md "对象头（mark word）") ，如果用

```java
Object obj = new Object();
```

创建一个空白对象，则占用 16个字节，如果对象有其他属性，则可能更大，但由于填充对齐的默认要求都是 8 字节的倍数，
