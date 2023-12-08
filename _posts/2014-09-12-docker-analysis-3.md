---
title: "Docker源码分析3-daemon启动（API）"
date: 2014-09-12T17:30+08:00
categories:
  - docker
tags:
  - golang
  - docker
---
在该系列的
[上一篇文章](http://blog.hamobai.com/2014/08/31/docker-analysis-2/)中，
 我们介绍了Docker daemon启动流程的前半部分，后半部分的代码会完成daemon
 启动中最重要的两个步骤：

* Daemon结构的初始化
* API的初始化

在这篇文章中，我们将首先分析API初始化相关的代码，其利用Golang标准库
 "net/http"生成一个RESTful的HTTP接口与client通信。
<!--more-->
上一篇文章中，我们介绍了Docker daemon能执行的动作都是封装在其engine之
 中的，那么具体如何执行这些动作呢？这里就需要engine.Job接口。其代码位于
 ```$SRC/engine/job.go```。

job struct的定义如下：

```go
type Job struct {
	Eng     *Engine
	Name    string
	Args    []string
	env     *Env
	Stdout  *Output
	Stderr  *Output
	Stdin   *Input
	handler Handler
	status  Status
	end     time.Time
}
```
从Job结构封装的内容不难看出，Job的设计想法来自于Unit进程：每个Job包含
自己的名字，调用的参数，执行的环境变量，标准输入、输出、错误流以及执行
状态。这里不再详细展开介绍Job的实现，有兴趣的读者可以自行研究。

回到API的初始化。之前我们介绍过，```builtins.Register()```会向
engine中注册serveapi命令，其对应的handler为apiserver.ServeApi。该命令
会完成API的初始化工作。

首先，以命令名"serveapi"及其参数（监听地址）调用```engine.Job()```函数，
 会返回指向该命令的Job结构。之后我们会根据命令行参数中指定的对API的选项，
 如是否启用tls等设置Job的环境变量。在这些工作都完成后，调用
 ```Job.Run()```函数将实际调用"serveapi"对应的handler。

当然，```Job.Run()```并非简单调用handler，其还会处理与任务同步，日志与
 状态等相关的工作，有兴趣的同学可以自行阅读```Job.Run()```的代码。下面
 我们直接进入到handler ```apiserver.ServeApi```，其代码位于
 ```$SRC/api/server/server.go```。

对于daemon来说，Docker支持绑定多个监听地址，他们通过Job的args传递，格
式为```PROTO://ADDR```。Docker支持3种PROTO，分别是tcp,unix和fd，分别对
应tcp连接，unix socket和systemd fd。不失一般性的，我们将以unix socket
为例继续后面的代码分析，有兴趣的同学可以自己研究下另外两种proto的代码。

ServeApi首先会初始化```activationLock chan```。上篇文章中我们
 介绍了API的初始化分为两个部分，第一部分是今天我们介绍的"serveapi"命令，
 另一部分是在daemon结构初始化完成后调用的"acceptconnections"命令，其作
 用是令初始化完成的API可以接受连接。而这个名为"activationLock"的
 chan就是实现API初始化与接受连接分离的关键。同时，Docker实现了
 ```listenbuffer```package，通过包装Golang标准库```net```来实现对
 延迟接受连接的支持。```listenbuffer```package的代码位于
 ```$SRC/pkg/listenbuffer/buffer.go```。在实际调用```net.Accpet()```接
 受连接之前，会首先从管道```activationLock chan```中进行读取操作，该操
 作会立即被阻塞（因为管道还没有被写入），直到管道有写入操作或者管道被
 关闭。写入或者关闭的区别在于，如果是写入操作，那么必须写入N次才能唤醒
 N个读取操作，而关闭管道将唤醒所有阻塞在该管道的读取操作。说到这里，读
 者不难想到"acceptconnections"命令的具体作用，对，他会调用
 ```close(activationLock)```从而将所有阻塞在```activationLock
 chan```的函数唤醒，进而实际调用```net.Accept()```开始接受连接。

API的初始化工作在```ListenAndServe```函数中完成。首先是生成router。
router即从HTTP方法和URL到对应的handle函数的映射。Docker包含一组
Profiler API，只在打开DEBUG模式时才有```AttachProfiler```函数添加到
router中。Docker主要的API router在```createRouter```函数中生成。这里利
用了gorilla/mux库，好处是：1. ```mux.Router```兼容```http.Handler```接
口，可以直接作为handler传入```http.Server```；2. ```mux.Vars```函数可
以将单次连接中的router变量生成```map[string]string```，方便在handler函
数中获取。

router生成之后，会利用它和之前传递的监听地址来初始化```http.Server```，
并调用```http.Serve```函数传入之前生成的```Listenbuffer```。这样，
daemon的API便初始化完成。在运行"acceptconnections"命令之后，daemon的
API便可接受外部连接来响应client的请求了。

在[下一篇文章](http://blog.hamobai.com/2014/09/20/docker-analysis-4/)
中，我们将开始介绍Docker daemon最重要的Daemon结构的初始化，它包含了
docker网桥、containers repo、images graph、volumes graph、repository
list以及execDriver等的初始化。
