---
title: "Docker源码分析2-daemon启动"
date: 2014-08-31T12:30+08:00
categories:
  - golang
  - docker
tags:
  - docker
---

在该系列的
[上一篇文章](http://blog.hamobai.com/2014/08/31/docker-analysis-1/)中，
我们简单介绍了Docker的基本信息。在这篇文章中，我们将详细分析Docker
daemon的启动代码。从这篇文章开始，我将用```$SRC```指代Docker源码所在的
目录。
<!--more-->
### 启动入口 ###
Docker的启动入口位于```$SRC/docker```目录，在分析代码之前，先介绍Golang的一个小知识。Golang的每一个package，都可以有零个，一个或者多个```func init()```，多个init函数会按照它们在源文件中出现的顺序被执行。除```main```pakcage外，init函数在package被import的时候执行；而对于```main```pakcage，其```func init()```将在程序启动时，先于```func main()```被执行。[1]

 ```$SRC/docker```目录下共有4个文件，```flags.go``` ```docker.go```
 ```daemon.go```以及```client.go```。

 ```flags.go```包含对于daemon和client都生效的命令行参数。主要是设定
client与daemon之间通讯方式相关的参数。当然，决定该次```docker```作为
client还是daemon运行的```-d```参数也在flags.go文件中被定义。具体如下：
``` go
	flDaemon      = flag.Bool([]string{"d", "-daemon"}, false, "Enable daemon mode")
```
这样，当我们用参数```-d```或者```--daemon```调用```docker```时，
 ```flDaemon```会被置为```true```（默认值为```false```），从而影响后序
 代码的执行。
 这里要注意的是，Golang的flag对所有参数都是默认使用```-```指定的，所以
 绑定在```flDaemon```上的长参数，Docker选择了```-daemon```。这样，我们
 才可以按照Unix的惯例，使用```-```指定短参数，使用```--```指定长参数。

接下来，执行会进入到真正的入口函数```func main()```，该函数位于
 ```$SRC/docker/docker.go```文件中。首先会被调用的是
 ```reexec.init()```，其作用是实现类似busybox的程序调用，即根据文件名
 决定程序的功能。这里暂且不表，有兴趣的同学可以自行研究```reexec```
 package的源码，代码位于```$SRC/reexec```目录下。

在完成对命令行参数使用```flag.Parse()```的解析后，首先会判断是否使用了
 ```-v```或```--version```参数，之后会根据```flDebug```的值设置
 ```DEBUG```环境变量，在Docker中，很多类似debug这样的全局设置都是通过
 环境变量传递的，这样做的好处是简化程序对该设置的处理，当需要修改或获
 取该项设置时，直接访问环境变量即可；坏处也显而易见，程序容易受到运行
 时已经定义的环境变量的影响。

如果用户没有通过命令行参数指定daemon与client的连接方式，则会检查
 ```DOCKER_HOST```环境变量，如果环境变量也没有指定，则将unix
 socket ```unix:///var/run/docker.sock``` 作为默认的连接方式压入
 ```flHosts```变量。之后，因为我们以```-d```或者```--daemon```参数启动
 ```docker```，```flDaemon```变量为```true```，我们会进入
 ```mainDaemon```函数进行daemon相关的动作。该函数位于
 ```$SRC/docker/daemon.go```。

进入```daemon.go```，首先会利用之前提过的```init()```注册daemon所
需要的flag，并通过引用其他package，调用各个package的```init()```函数。
这里需要着重介绍的是如下代码：
``` go
import (
	_ "github.com/docker/docker/daemon/execdriver/lxc"
	_ "github.com/docker/docker/daemon/execdriver/native"
)
```
在import package时，如果将包名重新定义为```_```，则该package无法在后面
 的代码中被调用（Golang不允许引用而不被调用的包存在）。引用一个包再将其
 重命名为```_```，其目的就是为了调用该包自己的```func init()```。比如这
 里，lxc和native就各自利用各自的```init```函数注册了一个reexec项，为
 ```/.dockerinit```和```native```设置各自的动作。而具体以这两个名字作为
 文件名运行docker程序时会有什么不同，我们会在以后的章节中介绍。

言归正传，在daemon的```init()```函数中注册的flag位于
 ```$SRC/daemon/config.go```，由于当前文件也属于main包，这些flag实际上
 在程序运行一开始就已经被注册，并不是在进入到```mainDaemon```函数后才
 被注册的。现在的叙述方法只是为了更清楚的描述代码的结构。为了简单起见，
 接下来的分析都基于默认参数， 有兴趣的同学可以自行研究一下daemon都支持
 那些参数，各自的含义是什么。

接下来，我们将进入daemon部分最核心的代码engine，engine代码位于
 ```$SRC/engine```目录下。engine将所有操作封装在其内部，通过其
 ```Register```方法向engine注册操作。engine内部还封装了标准输入，输出
 及错误流，以及操作之间相互同步的工具。简而言之，engine是所有操作运行
 的容器。

```mainDaemon```函数首先会调用```engine.New()```生成一个全新的engine，
注册一个```commands```操作，其作用是按字典顺序显示所有已经注册过的操作。
并将所有已经注册过的操作写入这个新生成的engine里。

再之后，会通过```signal.Trap()```函数将```os.Interrupt```
 ```syscall.SIGTERM```等信号绑定到新生成的engine的Shutdown成员上。这样
 当daemon收到以上信号时，engine可以得到通知并进行相应的操作。

再之后，```builtins.Register()```函数会向engine中注册如下操作：

* initnetworkdriver -> bridge.InitDriver
* serveapi -> apiserver.ServeApi
* acceptconnections -> apiserver.AcceptConnections
* events -> Events.Get
* log -> Events.Log
* subscriberscount -> Events.SubscribersCount
* version -> builtins.dockerVersion
* auth -> registry.Service.Auth
* search -> registry.Service.Search

这里不加介绍的列出builtins注册的每个操作及其对应的函数，我们会在具体用
到他们的时候再详细介绍。

再之后的流程会分成两部分，一部分用来初始化和启动RESTful API，但并不接
受连接；另一部分用来初始化daemon的服务，并在初始化结束后设置RESTful
API可以接受新连接。这样即可加快启动的速度，同时也可以保证当daemon可以
接受API连接时，daemon处于就绪状态。

我们将会在
[下一篇文章](http://blog.hamobai.com/2014/09/12/docker-analysis-3/)中
详细介绍daemon服务和RESTful API服务的启动过程。

[1]http://golang.org/ref/spec#Package_initialization
