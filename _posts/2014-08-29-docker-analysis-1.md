---
title: "Docker源码分析1-简介"
date: 2014-08-29T17:31+08:00
categories:
  - docker
tags:
  - golang
  - docker
---

Docker是一款由docker, Inc发起的开源Linux容器引擎。由Golang编写完成。它
基于Linux Container技术。Linux Container技术，是一种操作系统层次的虚拟
化技术，提供了系统隔离，资源限制等功能。与Linux下传统的KVM虚拟机相比，
container技术更轻量，所有容器运行在Host的内核上，共享Host的硬件资源。

由于Docker依赖于Linux Container技术，所以现在只能运行在Linux系统上。为
了解决这个问题，
[boot2docker](https://github.com/boot2docker/boot2docker)项目应运而生。
该项目为Windows和Mac OS的用户提供了一个包含Docker的最小化的Virtualbox
虚拟机镜像，借助Virtualbox虚拟机让Docker运行在这两种操作系统上。

本系列的Docker源码分析将基于Docker v1.2.0版本进行。读者可以从Docker项
目在github的[页面](https://github.com/dotcloud/docker)获取到该项目的源
代码。

[下一篇](/docker/2014/08/31/docker-analysis-2/)，我们将首先对Docker daemon的启动代码进行分析。
