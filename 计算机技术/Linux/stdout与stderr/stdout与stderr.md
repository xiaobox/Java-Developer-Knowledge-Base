# stdout与stderr

## 摘要

`1>&2`的表达式，就是把stdout里的内容，重定向到stderr中

`2>&1`  就是把stderr里的内容，重定向到stdout中

## 简介stdout与stderr&#x20;

> stdin, stdout, stderr - standard I/O streams

stdout和stderr都是标准输出，只是stream不同。
stdout就是正常的终端打印输出，而stderr是错误信息。 从终端上看，二者并无直观分别。 但是，由于stream不同，在进行组合操作时，会有较大差异。
作为Unix标准的一部分，这哥俩一开始是从C语言出来的。 其它更早的语言，虽然也有区分正常打印与错误信息的，比如Fortran。 但是，stdout和stderr这两个词，直接出自C语言的stdio.h。

```bash
#include<stdio.h>
int main(void)
{
  printf("stdout\n");
  fprintf(stdout, "stdout\n");
  fprintf(stderr, "stderr\n");
  return 0;
}
```

以上C语言代码编译后运行，会在终端显示三行。 两行stdout会打印到stdout，一行stderr会打印到stderr。

## 打印到stdout与stderr&#x20;

默认情况下，echo就是打印到stdout：

```bash
echo "hello"
```

如果需要打印到stderr，则要麻烦一些：

```bash
echo "error" 1>&2
```

其中，末尾的1可以省略，可以写作>&2。 这里，1就代表stdout，2就代表stderr。 含义就是，把stdout的输出内容，重定向（redirect to）到stderr中去。

## stdout与stderr的相互重定向&#x20;

前面**1>&2的表达式，就是把stdout里的内容，重定向到stderr中**。 1可以省略，是因为>符号前面，默认是1，也就是指>默认只操作stdout。
同理，**2>&1就是把stderr里的内容，重定向到stderr中**。 这个更常用一些。
这种表达式，不仅对echo生效，而是对任何Bash调用都生效，包括自定义脚本与程序。

**注意**：2>&1这类表达式中间不能有空格。

## 从stdout与stderr里重定向到文件&#x20;

有时，俺们需要把终端里的输出，重定向到一个文件中去。 比如，把make的编译过程，重定向至build.log文件中。

### 重定向stdout&#x20;

```bash
make > build.log
```

上面一行，相当于：

```bash
make 1> build.log
```

这时，stdout的内容不再打印，而是重定向到build.log文件中。 而在终端里，仍然有可能会有stderr的输出。 有时，这正是俺们需要的。

### 重定向stderr&#x20;

有些时候，俺们的需求是相反的。 不显示任何stderr的输出，只显示stdout的正常输出。

```bash
make 2> /dev/null
```

### 同时重定向stdout与stderr&#x20;

同时重定向stdout与stderr，有两种方法。

1.  先把stderr重定向到stdout中，然重定向stdout到文件

2.  直接把stdout和stderr都重定向到文件

一般网上流行的是第一种方法。

```bash
make > build.log 2>&1
```

这种方法，不仅逻辑上麻烦，而且很容易出错。 例如，有时会写成`make 2>&1 > build.log`，这是无效的，与make > build.log等价。 2>&1必须要放到一个完整Bash表达式的最后，才能生效。 这一点，非常容易出错。
第二种方法，直接把stdout和stderr都重定向到文件。 不仅形式简单，而且不会出错。

```bash
make &> build.log
```

把&>替换成>&，也是等价的。 另外要注意，这种用法仅在Bash中有效，在标准的sh无效，其它shell则没试过。 而1>、2>、2>&1即使在标准的sh中，也是有效的。

## 既打印又重定向&#x20;

有时，俺们既需要打印，又需要重定向。 还是以make举例，俺们通常既需要看到编译的过程，又需要分析build.log。 这时，tee就可以满足需求。

```bash
make | tee build.log
```

这一句的意思是，把make的stdout输出，传给tee； 而tee则会打印到stdout的同时，重定向到build.log中。 所以结果是，stdout和stderr都会被打印，而只有stdout才会被重定向到build.log中。
这是不是不合用？ 通常，俺们需要完整的结果作为log。 如果只有stdout，反而找不出真正的问题。

```bash
make 2>&1 | tee build.log
```

这一句就满足了保存完整log的需求。
咦？等等！ 那个2>&1的位置，似乎有些奇怪？
这是因为，管道|分割了Bash表达式，这一句实际上是两个表达式。 所以，2>&1确实是在表达式的末尾，没错。
如果换成`make | tee build.log 2>&1`，反而无法生效。 因为，管道|只会传递stdout的输出，不会传递stderr的输出。

## 其它用法展示&#x20;

知道了相关原理，一些其它的复杂用法，就可以轻松地组合出来。 这里随意举几例。

### 丢弃stdout的打印输出&#x20;

这种情况，通常是希望终端输出干净一些。 有错误就说话，没错就什么也别说——这也是符合Unix哲学的一种设计。

```bash
cmd > /dev/null
```

### 丢弃stderr的打印输出&#x20;

不想看到stderr，只要stdout，这通常是在一个脚本里组合多个命令时使用。

```bash
cmd 2> /dev/null
```

### 丢弃所有输出&#x20;

把Unix哲学贯彻到底！

```bash
cmd &> /dev/null
```

### 分别重定向stdout和stderr&#x20;

这是一个比较常用的日志转存策略。

```bash
cmd 1> stdout.log 2> stderr.log
```

写1>而不是>，略微清楚一些。

### 打印并重定向stdout，重定向stderr&#x20;

```bash
cmd 2> stderr.log | tee stdout.log 
```

要注意，由于管道|与tee的特性，反过来无法做到。 并且，同时打印并重定向stdout与stderr，也无法做到。 出现这两种情况，只能让cmd自己去解决。

在Linux系统中，自行查看相关帮助文档：

*   man stdout

*   man echo

*   man tee

*   man null

## 参考&#x20;

*   [Standard streams - Wikipedia](https://en.wikipedia.org/wiki/Standard_streams "Standard streams - Wikipedia")

*   [Standard Output Definition](http://www.linfo.org/standard_output.html "Standard Output Definition")

*   [Standard Error Definition](http://www.linfo.org/standard_error.html "Standard Error Definition")

*   [https://note.qidong.name/2017/07/bash\_stdout\_stderr/](https://note.qidong.name/2017/07/bash_stdout_stderr/ "https://note.qidong.name/2017/07/bash_stdout_stderr/")
