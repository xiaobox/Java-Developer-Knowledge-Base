# docker 基础知识

### CMD和 ENTRYPOINT 的区别

The main purpose of a CMD is to provide defaults for an executing container. These defaults can include an executable, or they can omit the executable, in which case you must specify an ENTRYPOINT instruction as well.

意思是，cmd给出的是一个容器的默认的可执行体。也就是容器启动以后，默认的执行的命令。重点就是这个“默认”。意味着，如果docker run没有指定任何的执行命令或者dockerfile里面也没有entrypoint，那么，就会使用cmd指定的默认的执行命令执行。同时也从侧面说明了entrypoint的含义，它才是真正的容器启动以后要执行命令。

所以这句话就给出了cmd命令的一个角色定位，它主要作用是默认的容器启动执行命令。（注意不是“全部”作用）

这也是为什么大多数网上博客论坛说的“cmd会被覆盖”，其实为什么会覆盖？因为cmd的角色定位就是默认，如果你不额外指定，那么就执行cmd的命令，否则呢？只要你指定了，那么就不会执行cmd，也就是cmd会被覆盖。

### 可以使用 docker commit 指令，把一个正在运行的容器，直接提交为一个镜像。

一般来说，需要这么操作原因是：这个容器运行起来后，我又在里面做了一些操作，并且要把操作结果保存到镜像里。但注意VOLUME的内容不会打到镜像里。
