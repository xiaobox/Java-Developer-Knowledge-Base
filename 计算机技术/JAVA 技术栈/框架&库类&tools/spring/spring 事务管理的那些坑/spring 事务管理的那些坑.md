# spring 事务管理的那些坑

## spring的service和dao是单例的

Spring中DAO和Service默认都是以单实例的bean形式存在，Spring通过ThreadLocal类将有状态的变量（例如数据库连接Connection）本地线程化，从而做到多线程状况下的安全。

## 哪些方法不能实施Spring AOP事务

*   由于Spring事务管理是基于接口代理或动态字节码技术，通过AOP实施事务增强的。虽然Spring还支持AspectJ LTW在类加载期实施增强，但这种方法很少使用，所以我们不予关注。&#x20;

*   对于基于接口动态代理的AOP事务增强来说，由于**接口的方法都必然是public**的，这就要求实现类的实现方法也必须是public的（不能是protected、private等），**同时不能使用static的修饰符**。所以，可以实施接口动态代理的方法只能是使用“public”或“public final”修饰符的方法，其他方法不可能被动态代理，相应的也就不能实施AOP增强，换句话说，即不能进行Spring事务增强了。&#x20;

*   基于CGLib字节码动态代理的方案是通过扩展被增强类，动态创建其子类的方式进行AOP增强植入的。由于使用final、static、private修饰符的方法都不能被子类覆盖，相应的，这些方法将无法实施AOP增强。所以方法签名必须特别注意这些修饰符的使用，以免使方法不小心成为事务管理的漏网之鱼。

## @ Transactional

**private 方法上加@Transactional是不生效的**

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxD8s7Rtl6dYubBmvpkGo14K2KdWSTrz07BuyVRGd4q08JTGPoxuY1icpR3Mu5iadoFs8rUk12dIHIibKg/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

原因是如果用默认jdk的动态代理是基于接口的，private方法不能被调用，但如果是用CGlib的实现却可以。

## 回滚

*   Spring FrameWork 的事务框架代码只会将出现runtime, unchecked 异常的事务标记为回滚；也就是说事务中抛出的异常是RuntimeException或者是其子类

*   spring默认回滚的异常行为可以查看DefaultTransactionAttribute类&#x20;

*   **显式捕获异常导致spring事务回滚失效**

    *   原因：spring事务是通过aop捕获到异常后再执行回滚，如果业务代码中显式捕获了异常，会导致spring捕获不到，回滚自然失败。

    *   解决办法

        *   业务代码catch住异常后重新抛出

        *   使用编程式事务显式回滚

        *   接口下沉，将需要事务控制的代码分到另一个接口方法中

        ```java
        @Override
        public boolean rollbackOn(Throwable ex) {
        return (ex instanceof RuntimeException || ex instanceof Error);
        }

        ```

## 同一个service类中不同方法的相互调用问题

### b方法加事务，a方法不加，结论：b方法上事务不生效！

```java
public interface AService {

    public void a();
    public void b();
}

@Service()
public class AServiceImpl implements AService{

    public void a() {
        this.b();
    }
    @Transactional(rollbackFor={Exception.class})
      public void b() {

         insert();
         update();
    }
}
```

&#x20;只要给目标类AServiceImpl的某个方法加上注解@Transactional，spring就会为目标类生成对应的代理类，以后调用AServiceImpl中的所有方法都会先走代理类（即使调用未加事务注解的方法a，也会走代理类），即在通过getBean("AServiceImpl")获得的业务类时，实际上得到的是一个**代理类**，假设这个类叫做AServiceImplProxy ，spring为AServiceImpl生成的代理类类似于如下代码：

```java

public class AServiceImplProxy implements AService{

    public void a() {
      //反射调用目标类的a方法
    }

    public void b() {

      //启动事务的代码 

      //反射调用目标类的b方法 

     //事务提交的代码

    }
}
```

spring事务管理的本质是通过aop为目标类生成动态代理类，并在需要进行事务管理的方法中加入事务管理的横切逻辑代码。

![](https://mmbiz.qpic.cn/mmbiz/YZibCWq4rxD8s7Rtl6dYubBmvpkGo14K2wcialrlpiaGx0wu1ThKrTticEyaYjVLJBElThbFmOcCLpbkr4D3rWCPnw/640?wx_fmt=other\&wxfrom=5\&wx_lazy=1\&wx_co=1)

调用getBean("AServiceImpl").a()时，实际上执行的是AServiceImplProxy.a()，代理类的a方法会通过反射调用目标类的a方法， 再在目标类的a方法中调用b方法，故最终a中调用的b方法是来自于AServiceImpl中的b方法，AServiceImpl的b方法并没有横切事务逻辑代码（切记：**事务逻辑代码在代理类中，@Transactional只是标记此方法在代理类中要加入事务逻辑代码**）。**所以调用a方法时，b方法的事务会失效。**

### a、b方法都加事务 结论：a、b合并为一个事务生效

调用顺序还和上一个情形类似，区别在于在反射调用目标对象的a方法前，会对a方法开启事务管理，虽然调用的b方法还是目标对象中没有加事务逻辑的代码，spring却会把b合并到a的事务中去，此时相当于只有一个事务。

### a方法加事务，b方法不加  结论：a、b合并为一个事务生效

由于b会合并到a的事务中，所以b中的逻辑也可以被事务管理。

由于a和b都合并到了a的事务中，所以这种情形下事务传递规则不适用。

**在这种情况下，如果我想让b也执行自己的事务逻辑，即调用b时执行代理类中b方法的事务逻辑，该怎么办？**

既然只有调用代理类的方法才生效，那么我们只要获取到代理类对象就可以了：

```java
@Transactional(rollbackFor={Exception.class})

public void a() {

    ((AService) AopContext.currentProxy()).b();
    //即调用AOP代理对象的b方法即可执行事务切面进行事务增强

}
```

看一下spring的源码就明白了，是这个类 org.springframework.aop.framework.JdkDynamicAopProxy

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxD8s7Rtl6dYubBmvpkGo14K2fC7jL4hebJUw2WzJ2whEicSn8WvwtzrD3OhmohrfPx5qMbQLkw8IQOQ/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

spring boot 可以使用这个注解搞定 @EnableAspectJAutoProxy(exposeProxy=true)

**不过最好还是做接口下沉，把b方法分离到另一个接口中，从根源上避免目标对象内部方法自我调用。**

## 参考&#x20;

*   [https://juejin.im/entry/5836572767f3560065f1939b](https://juejin.im/entry/5836572767f3560065f1939b "https://juejin.im/entry/5836572767f3560065f1939b")

*   [https://docs.spring.io/spring/docs/3.0.0.M3/reference/html/ch04s04.html](https://docs.spring.io/spring/docs/3.0.0.M3/reference/html/ch04s04.html "https://docs.spring.io/spring/docs/3.0.0.M3/reference/html/ch04s04.html")
