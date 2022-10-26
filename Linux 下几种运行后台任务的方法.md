# Linux 下几种运行后台任务的方法
### 1. 问题的引入

程序员最不能容忍的是在使用终端的时候往往因为网络，关闭屏幕，执行 CTRL+C 等原因造成 ssh 断开造成正在运行程序退出，使得我们的工作功亏一篑。

其背后的主要原因在于上述的相关操作，shell 默认会发送中断信号给该终端 session 关联的进程，从而导致进程跟随终端退出，为了弄清这个问题我们首先要了解两种中断信号：

1）sigint：signal interrupt，ctrl+c 会发送此信号，主动关闭程序

2）sighup: signal hang up，关闭终端，网络断线，关闭屏幕会发送此挂断信号。

今天就给大家介绍 linux 中几种后台任务的执行方法避免上述问题。

### 2.& 符号

这是一种把 & 放在执行命令最后，使启动的程序忽略 sigint 信号, 此时执行 ctrl+c 关闭就不会关闭此进程，但是当屏幕关闭，断网仍然会造成进程退出。

    sh test.sh &

### 3.nohup 指令

nohup（no hang up）, 意思就是不挂断运行，用 nohup 运行命令可以使命令永久执行下去，和用户终端没有关系，断开 SSH 不影响运行，nohup 捕获了 SIGHUP，并做了忽略处理，因此当屏幕关闭，断网等造成 ssh 中断时进程不会退出。但是 ctrl+c 可以关闭关闭该进程。因此大多数情况同时使用 nohup 和 & 启动的程序，ctrl+c 和关闭终端都无法关闭。在缺省情况下所有输出都被重定向到一个名为 nohup.out 的文件中。

nohup 指令基本使用格式：

    nohup Command [ Arg ... ] [　& ]

#### 举例

后台不中断执行./test.sh,stdout 输出给 out.log，stderr 输出给 err.log

    nohup ./test.sh > out.log 2>err.log  &

相关的数字含义如下：

-   0 – stdin (standard input)
-   1 – stdout (standard output), 显然 nohup command > out.log 等价于 nohup command 1> out.log，是缺省行为。
-   2 – stderr (standard error)

可能你也会见到这种写法，其含义是把 stderr 也重定向给 stdin

    nohup ./test.sh > out.log 2>&1  &

### 4.ctrl + z、jobs、fg、bg

如果我们程序在启动的时候并没有使用 &，nohup 怎么办呢，难道我们需要先执行 ctrl + c 将在前台执行的进程终止执行再重新启动吗，显然有好的方法！

#### 4.1 ctrl + z

将一个正在前台执行的作业进程放到后台，并且暂停，用术语讲就是挂起, 执行后如下：

    [1]+ Stopped ./test.sh

#### 4.2 jobs

查看当前有多少在后台运行的命令,\[jobnumber] 就是作业号。

    jobs[1]+ Stopped ./test.sh [2]+ Running ./test2.sh &

#### 4.3 fg

将后台中的作业进程调至前台继续运行, 例如把 2 号作业（./test2.sh &）调至前台运行

    fg 2 ./test2.sh

#### 4.4 bg

将后台中暂停（挂起）的作业进程继续运行, 例如把 1 号作业 (./test.sh) 放到后台运行，注意看已经带了 &

    bg 1[1]+ ./test.sh  &

### 5.screen 命令

#### 5.1 介绍

如果说上面的方法是通过 linux 相关本身命令实现了前后台任务调度，那么 screen 就提供了另外一种思路。

不说人话的版本：GNU Screen 是一款由 GNU 计划开发的用于命令行终端切换的自由软件。用户可以通过该软件同时连接多个本地或远程的命令行会话，并在其间自由切换。GNU Screen 可以看作是窗口管理器的命令行界面版本。它提供了统一的管理多个会话的界面和相应的功能。

说人话的版本: 我们可以粗略地认为 screen 是一个虚拟终端软件，直接在 linux 系统里面启动了另外一个后台程序接管（维持）了你的终端会话，当你直接连接的终端 ssh 断开时他仍然让程序认为你的 ssh 持续链接着，这样也就不会出现进程接收到中断信号而退出。

#### 5.2 安装

    yum install screen

#### 5.3 使用

**1）新建会话**

    screen -S yourname -> 新建一个叫yourname的session

**2） 列出当前所有的 session**

    screen -ls

**3）恢复会话（回到 yourname 这个 session）**

    screen -r yourname

**4） detach 某个 session**

    screen -d yourname -> 远程detach某个session screen -d -r yourname -> 结束当前session并回到yourname这个session

**5）删除会话**

    screen -S pid-X quit

 [https://mp.weixin.qq.com/s/BSuJ975D58mFsMYtYlx6aQ](https://mp.weixin.qq.com/s/BSuJ975D58mFsMYtYlx6aQ)
