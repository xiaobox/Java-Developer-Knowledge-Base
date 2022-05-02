# java 方法重载与静态分派

先来看一段代码，想一想输出结果：

```java

public class StaticDispatch {
static abstract class Human{}
static class Man extends Human{}
static class Woman extends Human{}

public void sayHello(Human guy){
        System.out.println("hello,guy!");
    }
public void sayHello(Man guy){
        System.out.println("hello,gentleman!");
    }
public void sayHello(Woman guy){
        System.out.println("hello,lady!");
    }

public static void main(String[] args){
        Human man=new Man();
        Human woman=new Woman();

        StaticDispatch sr=new StaticDispatch();

        sr.sayHello(man);
        sr.sayHello(woman);
    }
}
```

它的输出结果为： &#x20;

```java
hello,guy!
hello,guy!
```

这个结果有可能跟你想的不一样，那么是为什么呢？

由代码可知sayHello方法是重载方法，**重载是静态分派典型的应用**。

先来说一下什么是分派？**根据对象的类型而对方法进行的选择,就是分派(Dispatch)**

分派调用可能是静态的也可能是动态的，根据分派依据的宗量数（方法的调用者和方法的参数统称为方法的宗量）又可分为单分派和多分派。两类分派方式两两组合便构成了静态单分派、静态多分派、动态单分派、动态多分派四种分派情况。

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxD9ia5IkE4Yk4vPlUZicdDm8qohbWSzwcbzWpmyOIYVYybbZu7TLFt0ibWX4tjn2TH5MKaibmJ7WSY6qpg/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

*   动态分派的一个最直接的例子是重写。对于重写，我们已经很熟悉了。

*   静态分派(Static Dispatch) 发生在编译时期，分派根据静态类型信息发生。

方法的接受者（亦即方法的调用者）与方法的参数统称为方法的宗量。分派是根据一个宗量对目标方法进行选择，多分派是根据多于一个宗量对目标方法进行选择，**Java 语言的静态分派属于多分派类型**。

方法重载(Overload)就是静态分派。（所谓的：编译时多态）

所以，**重载的时候是通过参数的静态类型而不是实际类型作为判断依据的。**
