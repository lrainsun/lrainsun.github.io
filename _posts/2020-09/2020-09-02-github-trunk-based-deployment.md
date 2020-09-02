---
layout: post
title:  "主干开发模型"
date:   2020-09-02 23:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: 主干开发
mathjax: true
typora-root-url: ../
---

# Git代码分支模型

Git代码分支模型主要有三种：

* GitFlow
* GitHub Flow
* Trunk Based Development

# GitFlow


GitFlow有两个主干分支：

* Master分支 —— 只有稳定的生产版本
* Develop分支 —— 存放待发布（不稳定），用于集成的版本

以及三类其他分支：

* Feature分支 —— 某功能的分支
* Release分支 —— 代码合并和集成
* HotFix分支 —— 产品版本代码的紧急修订

GitFlow重视管理，强调多分支开发，会造成集成滞后，合并困难，影响持续集成

![clip_image001](/../assets/images/clip_image001_thumb-2.png)

# Github Flow

GitHub Flow只有两个分支：

* Master分支 
* Feature分支

所有新功能开发直接提交到 Feature 分支，再从 Feature 分支 Pull-Request 到主分支

Github Flow相当于把GitFlow的Master和Develop分支合并为一个分支，省去了一些分支而降低了复杂度，同时也更复合持续集成的思想，以一张故事卡为集成的最小单位，相对来说集成的周期短，反馈的速度也快，能够及早的遇到问题，从而及早的解决问题。

![image-20200902145614368](/../assets/images/image-20200902145614368.png)

# Trunk Based Development

Trunk Based Development就是主干开发模型，相较于上面的Github Flow又更加简洁一些，连Feature分支都不需要了，Feature分支可以只保留在本地，直接push到远程Master分支。

使用主干开发后，代码库原则上就只能有一个 Master 分支了，所有新功能的提交也都提交到 Master 分支上，但是拉出新的分支交付。没有了分支的代码隔离，测试和解决冲突都变得简单，持续集成也变得稳定了许多。

![clip_image002](/../assets/images/clip_image002_thumb-2.png)

> Branches are made for a release. Developers are not allowed to make branches in that shared place. Only release engineers commit to those branches, and indeed create each release branch. They may also cherry-pick individual commits to that branch if there is a desire to do so. After a release has been superseded by another, the branch is most likely deleted.
>
> The release branch that will live for a short time before it is replaced by another release branch, takes everything from trunk when it is created. In terms of merges, only cherry-picks FROM trunk TO the release branch are supported. For many enterprises, only bug fixes will be merged. 

仍然还是会有release branch的，不过只会存在比较短的一段时间，一旦新的release branch产生了，旧的release branch就会被删除。只支持trunk到release branch的cherry-pick。

> `git cherry-pick`可以理解为”挑拣”提交，它会获取某一个分支的单笔提交，并作为一个新的提交引入到你当前分支上。 当我们需要在本地合入其他分支的提交时，如果我们不想对整个分支进行合并，而是只想将某一次提交合入到本地当前分支上，那么就要使用`git cherry-pick`了。

主干开发会带来一些问题：

* release的时候，对于还没完成的feature怎么办
  * 在代码库里加一个Feature开关来随时打开和关闭新Feature，但是Feature Toggle会有成本（代码设计成本，加入和移除Toggle的成本和风险）
  * 重大特性变更，如果很难做toggle，可能也不得不采用feature branch，而且要经常把Master分支merge到feature分支
* 如何进行bugfix
  * 如果Master分支没有新提交，可以直接Master分支上Hot Fix
  * 从Release Tag创建发布分支
  * 在Master上做Fix Bug提交
  * 在Release分支再做一次发布
* 如何重构
  * 代码设计增加抽象层

# References

[1] [https://www.duyidong.com/2017/10/29/trunk-base-development/]( https://www.duyidong.com/2017/10/29/trunk-base-development/)

[2] [http://www.brofive.org/?p=2165](http://www.brofive.org/?p=2165)

[3] [https://paulhammant.com/2013/04/05/what-is-trunk-based-development/](https://paulhammant.com/2013/04/05/what-is-trunk-based-development/)

