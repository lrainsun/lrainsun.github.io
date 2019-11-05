---
layout: post
title:  "Kubernetes API对象"
date:   2019-11-05 14:50:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Kubernetes API对象
mathjax: true
typora-root-url: ../
---

# 声明式API

声明式API跟命令式API有什么不同点呢？

***Imperative（命令式）*** step-by-step的告诉计算机如何完成一项工作，如果这一系列动作被正确地执行，最终结果就达到了我们期望的状态。描述的是为了达到某一个效果或目标所需要完成的指令。

![img](/assets/images/1620.jpeg)

***Declarative（声明式）*** 有时也称为描述式，告诉计算机你想要什么，“声明”你想要的what，由计算机自己去设计执行路径，需要计算机或者是“运行时”具备一定的“智能”。描述的是目标状态（Goal State）。

![img](/assets/images/1620-20191105143533439.jpeg)

声明式 API 使系统更加健壮，在分布式系统中，任何组件都可能随时出现故障。当组件恢复时，需要弄清楚要做什么，使用命令式 API 时，处理起来就很棘手。

声明式设计的好处是：

1. **简单**，我们不需要关心任何过程细节。过程是由工具自己内部figure out的、内部执行的。
2. **self-documentation**，因为我们描述的就是希望一个事物变成什么样子，而不是“发育”过程。

Declarative是一种设计理念，是一种工作模式，透传出来的是“把方便留给别人，把麻烦留给自己”的哲学。Declarative模式的工具，设计和实现的难度是远高于Imperative模式的。作为用的人来说，Declarative模式用起来省力省心多了。

# API对象查找

在Kubernetes中，一个API对象在Etcd里的完整资源路径，是由：Group（API组）、Version（API版本）和Resource（API资源类型）三个部分组成的。

通过这样的结构，整个Kubernetes里的所有API对象，可以用如下的树形结构表示出来

![img](/assets/images/709700eea03075bed35c25b5b6cdefda.png)

kubernetes找到一个API对象的过程如下：

1. 匹配API对象的组

   1. 对于kubernetes里核心API对象，比如Pod，Node等，是不需要Group的（Group是""）

      kubernetes会在/api这个层级进行下一步匹配过程

   2. 对于非核心对象，比如CronJob等，要在/apis这个层级里查找它对应的Group

2. 匹配API对象版本号

   1. 同一种API对象可以有多个版本

3. 匹配API对象的资源类型

# 创建API对象

以CronJob为例

![img](/assets/images/df6f1dda45e9a353a051d06c48f0286f.png)

1. 发起创建CronJob的API请求之后，编写的YAML信息就被发给了APIServer
2. APIServer会过滤这个请求，并完成一些前置性的工作，比如授权、超时处理、审计等。
3. 然后请求会进入 MUX 和 Routes 流程。MUX 和 Routes 是 APIServer 完成 URL 和 Handler 绑定的场所。而 APIServer 的 Handler 要做的事情，就是按照API对象的匹配过程，找到对应的 CronJob 类型定义。
4. 接着APIServer 最重要的职责就来了：根据这个 CronJob 类型定义，使用用户提交的 YAML 文件里的字段，创建一个 CronJob 对象。
5. 而在这个过程中，APIServer 会进行一个 Convert 工作，即：把用户提交的 YAML 文件，转换成一个叫作 Super Version 的对象，它正是该 API 资源类型所有版本的字段全集。这样用户提交的不同版本的 YAML 文件，就都可以用这个 Super Version 对象来进行处理了。
6. 接下来，APIServer 会先后进行 Admission() 和 Validation() 操作。
   -  Admission Controller 和 Initializer，就都属于 Admission 的内容
   - 而 Validation，则负责验证这个对象里的各个字段是否合法。这个被验证过的 API 对象，都保存在了 APIServer 里一个叫作 Registry 的数据结构中。也就是说，只要一个 API 对象的定义能在 Registry 里查到，它就是一个有效的 Kubernetes API 对象。
7. 最后，APIServer 会把验证过的 API 对象转换成用户最初提交的版本，进行序列化操作，并调用 Etcd 的 API 把它保存起来