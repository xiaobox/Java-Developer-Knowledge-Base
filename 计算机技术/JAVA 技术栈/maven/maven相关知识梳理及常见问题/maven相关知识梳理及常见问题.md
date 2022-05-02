# maven相关知识梳理及常见问题

## maven 指定依赖版本范围

**有时我们为了不频繁修改依赖的版本号，会直接指定一个范围，如果你需要一直依赖最新版本就能省些事儿，当然也可以根据你的版本需求进行配置。**

*   A square bracket ( `[` & `]` ) means "closed" (inclusive).

*   A parenthesis ( `(` & `)` ) means "open" (exclusive).

| Range          | Meaning                                                                                                                                                                                           | 描述                                                       |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| 1.0            | x >= 1.0 \* The default Maven meaning for 1.0 is everything (,) but with 1.0 recommended. Obviously this doesn't work for enforcing versions here, so it has been redefined as a minimum version. | 1.0 的默认 Maven 含义是所有（，）但建议使用 1.0。如果没有 1.0 则通常表示 1.0 或更高版本 |
| (,1.0]         | x <= 1.0                                                                                                                                                                                          | 依赖小于等于 1.0 的版本                                           |
| (,1.0)         | x < 1.0                                                                                                                                                                                           | 依赖等于 1.0 的版本                                             |
| \[1.0]         | x == 1.0                                                                                                                                                                                          | 声明确切版本                                                   |
| \[1.0,)        | x >= 1.0                                                                                                                                                                                          | 依赖大于等于 1.0 的版本                                           |
| (1.0,)         | x > 1.0                                                                                                                                                                                           | 依赖大于 1.0 的版本                                             |
| (1.0,2.0)      | 1.0 < x < 2.0                                                                                                                                                                                     | 依赖大于 1.0 小于 2.0 的版本                                      |
| \[1.0,2.0]     | 1.0 <= x <= 2.0                                                                                                                                                                                   | 依赖大于等于 1.0 小于等于 2.0 的版本                                  |
| (,1.0],\[1.2,) | x <= 1.0 or x >= 1.2. Multiple sets are comma-separated                                                                                                                                           | 依赖小于等于 1.0，或大于等于 1.2 的版本（多组以逗号分隔）                        |
| (,1.1),(1.1,)  | x != 1.1                                                                                                                                                                                          | 依赖不包括 1.1 的版本                                            |

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>[1.18.8,1.18.12]</version>
</dependency>
```

## 快照

一般我们在开发阶段包的版本会定义为：**snapshot**

```xml
<dependency>
     <groupId>data-service</groupId>
      <artifactId>data-service</artifactId>
      <version>1.0-SNAPSHOT</version>
      <scope>test</scope>
</dependency>
```

每次构建项目时，Maven 将自动获取最新的快照。虽然，快照的情况下，Maven 在日常工作中会自动获取最新的快照， 你也可以在任何 maven 命令中使用 -U 参数强制 maven 现在最新的快照构建。

```bash
mvn clean package -U
```

**使用快照我们就不用频繁与依赖自己开发包的同事进行沟通了。maven 会自动更新并引用最新快照。**

## Maven 构建生命周期

![image.png](<https://cdn.nlark.com/yuque/0/2020/png/1069608/1595310383364-83e2fc83-b45b-450b-ab81-12273e58dd2a.png#align=left\&display=inline\&height=532\&margin=\[object Object]\&name=image.png\&originHeight=532\&originWidth=119\&size=15395\&status=done\&style=none\&width=119> "image.png")

| 阶段          | 处理   | 描述                             |
| ----------- | ---- | ------------------------------ |
| 验证 validate | 验证项目 | 验证项目是否正确且所有必须信息是可用的            |
| 编译 compile  | 执行编译 | 源代码编译在此阶段完成                    |
| 测试 Test     | 测试   | 使用适当的单元测试框架（例如 JUnit）运行测试。     |
| 包装 package  | 打包   | 创建 JAR/WAR 包如在 pom.xml 中定义提及的包 |
| 检查 verify   | 检查   | 对集成测试的结果进行检查，以保证质量达标           |
| 安装 install  | 安装   | 安装打包的项目到本地仓库，以供其他项目使用          |
| 部署 deploy   | 部署   | 拷贝最终的工程包到远程仓库中，以共享给其他开发人员和工程   |

## 利用持续集成工具实现自动化构建

比如你需要在本项目

*   代码提交

*   项目 build

*   项目 deploy

*   ......

这些情况下其他项目跟着一起构建，比如你的 jar 包升级需要其他项目自动构建拉取最新版本时（可以结合版本范围控制）。

**可以使用 jenkins 的自动化构建流程功能进行设计。(build after other projects are built)**

![image.png](<https://cdn.nlark.com/yuque/0/2020/png/1069608/1595312450920-56936ca6-1b9c-4928-aa58-df4f5568b8a8.png#align=left\&display=inline\&height=488\&margin=\[object Object]\&name=image.png\&originHeight=976\&originWidth=1878\&size=179532\&status=done\&style=none\&width=939> "image.png")

## 一些基础知识

### 依赖传递 (Transitive Dependencies)

依赖传递 (Transitive Dependencies) 是 Maven 2.0 开始的提供的特性，依赖传递的好处是不言而喻的，可以让我们不需要去寻找和发现所必须依赖的库，而是将会自动将需要依赖的库帮我们加进来。

例如 A 依赖了 B，B 依赖了 C 和 D，那么你就可以在 A 中，像主动依赖了 C 和 D 一样使用它们。并且传递的依赖是没有数量和层级的限制的，非常方便。

但依赖传递也不可避免的会带来一些问题，例如：

*   当依赖层级很深的时候，可能造成循环依赖 (cyclic dependency)

*   当依赖的数量很多的时候，依赖树会非常大

针对这些问题，Maven 提供了很多管理依赖的

### 依赖调节 (Dependency mediation)

依赖调节是为了解决版本不一致的问题 (multiple versions), 并采取就近原则 (nearest definition)。举例来说，A 项目通过依赖传递依赖了两个版本的 D：A -> B -> C -> ( D 2.0) , A -> E -> (D 1.0), 那么最终 A 依赖的 D 的 version 将会是 1.0，因为 1.0 对应的层级更少，也就是更近。

### scope 依赖范围

Maven 的生命周期存在编译、测试、运行这些过程，那么显然

*   有些依赖只用于测试比如 junit；

*   有些依赖编译用不到，只有运行的时候才能用到，比如 mysql 的驱动包在编译期就用不到（编译期用的是 JDBC 接口），而是在运行时用到的；

*   还有些依赖，编译期要用到，而运行期不需要提供，因为有些容器已经提供了，比如 servlet-api 在 tomcat 中已经提供了，我们只需要的是编译期提供而已。

总结说来，在 POM 4 中，中还引入了，`<dependency>`中还引入了`<scope>`，它主要管理依赖的部署。大致有 **compile、provided、runtime、test、system** 等几个。

| scope    | 说明                         | 示例              |
| -------- | -------------------------- | --------------- |
| compile  | 编译时需要用到该 jar 包（默认）         | commons-logging |
| test     | 编译 Test 时需要用到该 jar 包       | junit           |
| runtime  | 编译时不需要，但运行时需要用到            | mysql           |
| provided | 编译时需要用到，但运行时由 JDK 或某个服务器提供 | servlet-api     |

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-test</artifactId>
    <scope>test</scope>
</dependency>
```

#### scope 的依赖传递

A -> B -> C 当前项目 A，A 依赖于 B，B 依赖于 C

**知道 B 在 A 中的 scope，怎么知道 C 在 A 中的 scope ?**

即 A 需不需要 C 的问题，本质由 C 在 B 中的 scope 决定 当 C 在 B 中的 scope 是 test 或 provided 时，C 直接被丢弃，A 不依赖 C 否则 A 依赖 C，C 的 scope 继承与 B 的 scope

### 依赖管理 (Dependency management)

通过声明 Dependency management，可以大大简化子 POM 的依赖声明。
举例来说项目 A,B,C,D 都有共同的 Parent，并有类似的依赖声明如下：

*   A、B、C、D/pom.xml

    ```xml
    <dependencies>
           <dependency>
               <groupId>group-a</groupId>
               <artifactId>artifact-a</artifactId>
               <version>1.0</version>
               <exclusions>
                   <exclusion>
                       <groupId>group-c</groupId>
                       <artifactId>excluded-artifact</artifactId>
                   </exclusion>
               </exclusions>
           </dependency>
           <dependency>
               <groupId>group-a</groupId>
               <artifactId>artifact-b</artifactId>
               <version>1.0</version>
               <type>bar</type>
               <scope>runtime</scope>
           </dependency>
    </dependencies>
    ```

如果父 pom 声明了如下的 Dependency management: &#x20;

*   Parent/pom.xml

```xml
<dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>group-a</groupId>
                <artifactId>artifact-a</artifactId>
                <version>1.0</version>
                <exclusions>
                    <exclusion>
                        <groupId>group-c</groupId>
                        <artifactId>excluded-artifact</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>           
            <dependency>
                <groupId>group-a</groupId>
                <artifactId>artifact-b</artifactId>
                <version>1.0</version>
                <type>bar</type>
                <scope>runtime</scope>
            </dependency>
            <dependency>
                <groupId>group-c</groupId>
                <artifactId>artifact-b</artifactId>
                <version>1.0</version>
                <type>war</type>
                <scope>runtime</scope>
            </dependency>
           
        </dependencies>
 </dependencyManagement>
```

那么子项目的依赖声明会非常简单：

*   A、B、C、D/pom.xml

```xml
<dependencies>
        <dependency>
          <groupId>group-a</groupId>
          <artifactId>artifact-a</artifactId>
        </dependency>
        <dependency>
          <groupId>group-a</groupId>
          <artifactId>artifact-b</artifactId>
          <!-- 依赖的类型，对应于项目坐标定义的 packaging。大部分情况下，该元素不必声明，其默认值是 jar.-->
          <type>bar</type>
        </dependency>
</dependencies>
```

### 导入依赖范围

它只使用在`<dependencyManagement>`中，表示从其它的 pom 中导入 dependency 的配置，例如 (B 项目导入 A 项目中的包配置）：

想必大家在做 SpringBoot 应用的时候，都会有如下代码

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.3.3.RELEASE</version>
</parent>
```

继承一个父模块，然后再引入相应的依赖。
假如说，我不想继承，或者我想继承多个，怎么做？

**我们知道 Maven 的继承和 Java 的继承一样，是无法实现多重继承的，如果 10 个、20 个甚至更多模块继承自同一个模块，那么按照我们之前的做法，这个父模块的 dependencyManagement 会包含大量的依赖。如果你想把这些依赖分类以更清晰的管理，那就不可能了。**

import scope 依赖能解决这个问题。你可以把 dependencyManagement 放到单独的专门用来管理依赖的 pom 中，然后在需要使用依赖的模块中通过 import scope 依赖，就可以引入 dependencyManagement。例如可以写这样一个用于依赖管理的 pom：

```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.test.sample</groupId>
    <artifactId>base-parent1</artifactId>
    <packaging>pom</packaging>
    <version>1.0.0-SNAPSHOT</version>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>junit</groupId>
                <artifactid>junit</artifactId>
                <version>4.8.2</version>
            </dependency>
            <dependency>
                <groupId>log4j</groupId>
                <artifactid>log4j</artifactId>
                <version>1.2.16</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

然后我就可以通过非继承的方式来引入这段依赖管理配置

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.test.sample</groupId>
            <artifactid>base-parent1</artifactId>
            <version>1.0.0-SNAPSHOT</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
 
<dependency>
    <groupId>junit</groupId>
    <artifactid>junit</artifactId>
</dependency>
<dependency>
    <groupId>log4j</groupId>
    <artifactid>log4j</artifactId>
</dependency>
```

**注意：import scope 只能用在 dependencyManagement 里面**
这样，父模块的 pom 就会非常干净，由专门的 packaging 为 pom 来管理依赖，也契合的面向对象设计中的单一职责原则。此外，我们还能够创建多个这样的依赖管理 pom，以更细化的方式管理依赖。这种做法与面向对象设计中使用组合而非继承也有点相似的味道。

那么，如何用这个方法来解决 SpringBoot 的那个继承问题呢？
配置如下：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>1.3.3.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
 
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

这样配置的话，自己的项目里面就不需要继承 SpringBoot 的 module 了，而可以继承自己项目的 module 了。
