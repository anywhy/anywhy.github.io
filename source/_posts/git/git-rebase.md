---
title: Git命令-rebase
categories: Git
tags:
  - git
  - rebase
abbrlink: 2868751694
---

## git rebase 命令使用

`git rebase`命令在另一个分支基础之上重新应用，用于把一个分支的修改合并到当前分支，通常在对一个分支进行“变基”操作时使用。这意味着改变这个 branch 的初始 commit(我们知道 commits 本身组成了一颗树），它会在新的 base 上一个一个地运行这个分支上的所有 commits。

### 使用语法

```bash
git rebase [-i | --interactive] [<options>] [--exec <cmd>]
    [--onto <newbase> | --keep-base] [<upstream> [<branch>]]
git rebase [-i | --interactive] [<options>] [--exec <cmd>] [--onto <newbase>]
    --root [<branch>]
git rebase (--continue | --skip | --abort | --quit | --edit-todo | --show-current-patch)
```

### 示例

假设存在以下历史，有两个分支一个是`master`,一个是`topic`, 当前分支是“topic”:

```bash
          A---B---C topic
         /
    D---E---F---G   master
```

如果我们对`topic`变基操作会变成一下结果：

```bash
                  A'--B'--C' topic
                 /
    D---E---F---G master
```

这样能够维护一个清晰的语义化的暴露给大众的变更历史。

### 变基操作过程

- 第一步，确认自己所在分支
- 第二步：确定要 rebase 的 commit 的数量：
  - 使用 git log 命令，查看 git 操作记录，确定要 rebase 的 commi 数量。
- 第三步：确定 rebase 的 commit 数量，执行命令
  git rebase -i HEAD^^
- 第四步：进入 rebase 的 vi 编辑模式

```bash
  * p, pick = use commit
  * r, reword = use commit, but edit the commit message
  * e, edit = use commit, but stop for amending
  * s, squash = use commit, but meld into previous commit
  * f, fixup = like “squash”, but discard this commit’s log message
  * x, exec = run command (the rest of the line) using shell
  * d, drop = remove commit
```

> 注意，只需要，在第二个（此处是在两个 commit 合并成一个 commit 的情况下执行的！）的 commitId 前面，将 pick 修改为 s（编译器会自动将 s 变成 squash），然后，wq！退出保存即可。

- 第五步：在第四步顺利的情况下，可以直接 git push -f 即可。
- 但是如果第五步失败了！
  - 执行以下的操作！
  - 继续执行 git rebase -i HEAD^^ --allow-empty
    - 此处的允许空，是根据提示报错执行的，如果继续报错，但是其实可以看见，提示会变成，使用命令 git rebase —continue，继续执行。
    - 此时因为执行过 git rebase —continue ，则继续进入 vi 编辑模式，会看见，有两个对应 commitId 的 message。
  - 可以开始修改自己的 message，然后保存退出。
  - 执行命令 git log，发现最新的 commit 已经变换
  - 执行命令 git status，查看当前 git 的命令
  - 最后执行 git push -f
  - 查看远程的 GitHub 的想对应的 pr 界面，发现 commit 已经变换。
