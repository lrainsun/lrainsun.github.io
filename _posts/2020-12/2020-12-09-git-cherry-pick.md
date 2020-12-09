---
layout: post
title:  "git cherry-pick"
date:   2020-12-09 23:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: git cherry-pick
mathjax: true
typora-root-url: ../
---

# git cherry-pick

`git cherry-pick`可以理解为”挑拣”提交，它会获取某一个分支的单笔提交，并作为一个新的提交引入到你当前分支上。 当我们需要在本地合入其他分支的提交时，如果我们不想对整个分支进行合并，而是只想将某一次提交合入到本地当前分支上，那么就要使用`git cherry-pick`了。

```shell
git cherry-pick [<options>] <commit-ish>...

常用options:
    --quit                退出当前的chery-pick序列
    --continue            继续当前的chery-pick序列
    --abort               取消当前的chery-pick序列，恢复当前分支
    -n, --no-commit       不自动提交
    -e, --edit            编辑提交信息
```

## cherry-pick使用方法

当我们有commit想要从dags branch提交到dags-staging branch时

可以这样

```shell
git checkout dags-staging
git cherry-pick c6ae27eeafd6ac89ae557a8185b344a0fae3b646
git push origin dags-staging
```

# cherry-pick冲突解决办法

但是有时候，我们在cherry-pick的时候会发生冲突

`Sorry, we cannot cherry-pick this commit automatically. This commit may already have been cherry-picked, or a more recent commit may have updated some of its content.`

这个时候，我们需要手工地去做这个cherry-pick，假如我们还是想要把dags branch的commit提交到dags-staging branch

1. 基于dags-staging branch建立一个本地branch
2. 把dags的commit cherry-pick到本地branch，这个时候会提示有冲突，把冲突解决
3. 把本地branch的修改Merge到dags-staging branch

下面有个例子

```shell
# 创建本地branch
MINSU-M-M1RW:ocp_dags minsu$ git checkout -b cherry-pick-branch
Switched to a new branch 'cherry-pick-branch'

# cherry-pick commit，发现冲突
MINSU-M-M1RW:ocp_dags minsu$ git cherry-pick 88a6a89c4f6158e018af7287ba5c3009a8394ebd
error: could not apply 88a6a89... add db_instance
hint: after resolving the conflicts, mark the corrected paths
hint: with 'git add <paths>' or 'git rm <paths>'
hint: and commit the result with 'git commit'

# 查看冲突的地方在哪里
MINSU-M-M1RW:ocp_dags minsu$ git cherry-pick --continue
U    common/ocp_apply_deployment_patch/ocp_apply_deployment_patch_dag.py
error: commit is not possible because you have unmerged files.
hint: Fix them up in the work tree, and then use 'git add/rm <file>'
hint: as appropriate to mark resolution and make a commit.
fatal: Exiting because of an unresolved conflict.

# 解决冲突，在我这个例子里，ocp_apply_deployment_patch_dag.py这个文件在dags-staging里不存在所以会有问题，所以git rm一下，解决冲突的办法具体情况具体分析
MINSU-M-M1RW:ocp_dags minsu$ git rm common/ocp_apply_deployment_patch/ocp_apply_deployment_patch_dag.py
common/ocp_apply_deployment_patch/ocp_apply_deployment_patch_dag.py: needs merge
rm 'common/ocp_apply_deployment_patch/ocp_apply_deployment_patch_dag.py'

# 解决冲突之后，继续cherry-pick
MINSU-M-M1RW:ocp_dags minsu$ git cherry-pick --continue
[cherry-pick-branch 6823986] add db_instance
 Date: Wed Dec 9 14:17:20 2020 +0800
 3 files changed, 26 insertions(+), 25 deletions(-)
 
# 切换到dags-staging branch
MINSU-M-M1RW:ocp_dags minsu$ git checkout dags-staging
Switched to branch 'dags-staging'
Your branch is up-to-date with 'origin/dags-staging'.

# 把本地branch Merge进来
MINSU-M-M1RW:ocp_dags minsu$ git merge cherry-pick-branch
Updating 241998a..49ba560
Fast-forward
 ocp_2x/ocp_2x_add_compute_node/ocp_2x_add_compute_node_dag.py           |  2 +-
 ocp_2x/ocp_2x_add_compute_node/templates/ocp_2x_add_compute_node_dag.j2 | 38 +++++++++++++++++++-------------------
 zerotouchlib/db.py                                                      | 11 ++++++-----
 3 files changed, 26 insertions(+), 25 deletions(-)
```

# References

[1] [https://gitlab.com/gitlab-org/gitlab/-/issues/22059](https://gitlab.com/gitlab-org/gitlab/-/issues/22059)

