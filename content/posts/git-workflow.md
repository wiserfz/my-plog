+++
title = "git workflow"
date = "2025-02-01T15:37:55+08:00"
draft = false
categories = ["workflow"]
tags = ["git"]
author = ["wiser"]
description = "Some advice for git workflow"
ShowWordCount = true
+++

# Overview

在工作，不管是个人开发维护一个项目还是多人协作开发维护一个项目，版本控制工具是绕不开且十分重要的工作流，记录一下自己在工作中对于 workflow 的一些想法以及做法。

## Why VCS

在软件项目中使用版本控制系统（**V**ersion **C**ontrol **S**ystem）的好处非常多，许多问题在今天看来是显而易见的，这里列举出几个最重要的点

### 协作

假如没有 VCS，我们在写程序的时候该如何协作？我们可能会在服务器或者网络中建一个共享的目录，让每个人都可以直接在那里修改。或者我们可能会维护一份特定机器上的目录，同一时刻可能由一个人在修改。如果一不小心或者没有沟通好，某个人的工作就可能被其它同事覆盖掉。这显然是一个非常糟糕、低效的协作方式。

通过 VCS，团队中的每个人就可以完全自由地参与同一个项目同一份代码甚至是同一份文件同一行代码的修改。 VCS 工具会在之后将这些改动合并起来。一个源文件或者整个源代码项目的最终版本就在那，一个版本控制系统。每个人都可以同时，或者独立地参与到项目的开发中。

我们还能从 VCS 中找出每个地方的修改是哪个同事完成的，而不需要通过 IM 或者隔空喊话问同事。

设想一下，假如是设计师团队，在协作共同完成一个设计图，每个设计师画自己的部分，最后他们还要找人把不同的部分画在一起。或者，一个设计师要加一条线，另一个要在这里放一个实心矩形色块，他们只能一个人画完自己的部分，把结果交给另一个同事，让他在此基础上画矩形，这就只能是同步操作了。（当然，现在的设计师也有自己的协作工具，只是没有文本协作这么强大）

总之，VCS 的好处太多了。

### 记录每个版本

还是拿设计图纸举例，多个版本的设计图，或者之后的改动，都要生成另一个版本的文件，比如用不同的时间或者版本号命名的文件、文件夹。怎么保存、要保存几份版本（中间版本），要怎么知道每个版本之间的不同？除了图片，还有 Excel 表格，Word 文档，或者其它。没有强大的版本控制工具，这些文件就像垃圾文件一样充满了项目目录。

有个版本控制系统，我们只需要在当前的工作环境下放一份当下正在进行的复本。其它历史版本或者变体，都可以随时从 VCS 中取出。

### 找回先前的版本

因为记录了版本和整个历史，这让我们可以轻松得回滚到之前的任何一个版本，或者说状态，不管是项目中的某个文件，还是整个项目。好的 VCS 可以让这件事变得非常轻松、快速、方便。

### 理解为什么这么做、发生了什么事

每次保存（提交），我们都记录下了当时的描述，比如需求、bug 修复或者优化改进等等。这让我们可以很轻松地了解到，为什么要这么写，当时发生了什么事，这个版本做了哪些改动。这就像有了个时间机器，不仅是代码可以回滚到先前的版本，还可以更好地让你理解现在为什么会是这样。这对程序开发和团队协作来说，非常重要。

### 备份

像 Git 这种分布式的 VCS 有个不太明显的好处，就是备份。即使中心仓库丢失，某个同事的硬盘故障，分布式的 VCS 都能够将这种情况的损失减小到最低限度。不只是当前版本的备份，还是整个历史记录的备份。

在本地，像 Git 这种牛逼的工具，甚至还能像时间机器一样记录你的操作，帮你找回几个月前，我从 develop 切换到 feature 分支的时候做了什么操作。


## Git & GitLab

Git 是目前最流行，也是最强大的 VCS。 Git 强大到可以做太多的事，太过灵活。而不同的人有不同的习惯，这会导致同一个 Git 仓库中会产生混乱，或者低效率的情况。因此，Git 社区总结了一些比较好的工作模式，称之为「工作流」(Work Flow)。

GitLab 是一个集成了 Git 版本控制系统的 issue 跟踪系统和团队协作系统。除了 Git 中心仓库之外，还有更为强大的团队协作工具，从文档记录到 issue 跟踪，到集成环境，自动化……

工作流主要涉及两部分，一个是 GitLab 工作流，即和 GitLab Issues, Merge Request, CI 相关；一个是 Git 工作流，和 Git branch, commit, rebase 相关。

早先 issues track 和 CVS 相关性并不大，这些看随着 Git，GitHub，GitLab 的流行，Git 本身对代码仓库版本管理的方便性，issue 跟踪，协作，merge request 等功能慢慢在开发社区形成了一些主流的工作流。最早的是 Git Flow，并附带了工具，随后的 GitHub 和 GitLab 基本上都采用了 Git Flow 的思路，只是在细节上有些不同。

Git 本身功能非常强大，也非常适合多人协作和版本管理。我们基本上参照这几个主流的工作流，结合我们每个具体的项目，把工作流简化了下。这种流程思路基本上是适用于所有的项目，举例。


| 分支        | 功能     | 说明 |
|-------------|----------|--------------------------------------------------------|
| `master`    | 主分支   | 主分支保持了所有已发布的特性，包括所有已发布的版本 tag |
| `develop`   | 开发主干 | 所有的开发都从 `develop` 分支开出来，所有的修改最终也都合并到 `develop` |
| `feature/*` | 特性分支 | 所有的特性都从直接、或者间接基于 `develop`，在合并到 `develop`、或者废弃之前，都不影响 `develop` 上的内容 |
| `hotfix/*`  | 紧急修复 | `hoftix/*` 分支是用于紧急修复线上问题，因此是从 `master` 上当前发布的版本开出来，完成后同时合并到 `master` 和 `develop`，如需紧急发布，仍然走 `release/*` 分支流程 |
| `release/*` | 发布分支 | 所有发布到生产环境的代码都是从 `release/*` 分支取得的，将需要发布的改动/特性从 `develop` 分支开出，添加发布准备工作、集成测试，修复 bug 等，合并到 `master` 之后，从 `master` 发布，并打上版本号 |
| `support/*` | 特殊支持 | 例如，特殊客户的需求，单独私有化部署等。可以从 `feature/*` 分支、`hotfix/*` 分支合并特性和 Bug 修复，但原则上不合到 `master` |

![Git\_Flow][image-1]

[Git Flow][git-flow-model] 是第一个完整提出的 Git 工作流建议，这是一个非常优秀的方案，但事实上这种方式并不适用于所有的项目，它更适合那种比较传统的软件发布流程。类似现今的 Web 应用或者其它类似的项目，是自动持续发布的，所以一般它们在主干/默认分支上都保证可用就行，甚至，这些持续发布的系统并没有像传统意义上的发布版本号。所以我们对这种工作流作了个简化的约定，以适用于我们的实际情况。

首先，项目在 GitLab 上的 **默认分支** 并不是 master，而是 **develop**，但同时存在 master 分支。这两个分支在 GitLab 的设置里都是受保护的分支。受保护的意思是，原则上是不允许直接往这两个分支提交 commit 的。我们实际的情况是，develop 作为开发主干分支，master 作为发布分支，我们并不采用 release/ 分支。

## 基本 Git 工作流约定

- **master** 分支作为生产环境分支(production)，所有的发布 tag 都要打在 master 分支上。并且，master 分支上禁止直接 commit 修改。
- **develop** 分支作为开发分支，算是整个代码仓库的主干。所有的特性 (feature) 分支都要基于 develop 分支，或者是间接基于 develop 分支。原则上也 **不建议直接在 develop 分支上直接 commit 修改** 。 develop 分支上包含所有已经发布的代码，和即将要发布的改动。如果一个 feature 开发完成，但暂未发布，不建议合并到 develop，保留在 feature 分支，并及时关注 develop 上的改动，尽早 rebase 解决冲突。
- **feature/xxx** 所有的特性分支用 `feature/` 作为前缀，base 在 develop 分支，或者其它 feature 分支。
- **hotfix/xxx** 生产环境上发现的问题需要紧急修复的，用 `hotfix/xxx` 分支来处理。即从 master 分支上当前的 tag 处开出一个 hotfix 分支，在此分支上提交修改，完成后 merge 到 master 作为修复，并同时将这个 hotfix 分支的改动也 merge 到 develop，以保证 develop 上也能得到此问题的修复（不要从 master 向 develop merge）。
- 所有正在开发的 feature 分支需要（只要）关注 develop 分支（或者自己 base 的 feature 分支）的变动，及时 rebase。
- 保留 develop 和 master 分支上完整的 merge commit 记录，即 **不使用 fast forward 的方式合并** 。这便于追溯历史，可以完整得保留主干的历史记录，否则得话，主干就有可能产生跳跃，例如突然间从一个地方跳跃到了别一个地方，中间不知道为什么产生了这些提交。手动操作的时候加上 `--no-ff` 选项，如 `git checkout develop; git merge feature/a --no-ff`
- 记录有意义的 commit message
- 特性分支需要及时关注 based 的分支的改动，尽早跟上，**尽早**处理冲突，即 rebase
- 小特性可以单人开发，大的特性可以多人在一个或者多个特性分支上开发，注意 base

## GitLab 规范

- master 作为发布分支，发布版本的 tag 只能打在 master 上（目前是不带前缀的版本号，建议增加发布 tag 的前缀，如 v0.1.0）
- 结合 gitlab 的 issue 和 merge request 管理，记录开发历史
- 尽量开 issue，并给出具体描述，并尽早开出 Merge Request，加上 WIP
- WIP 状态下，尽早 push 到远程，很多好处，比如，团队的同事可以更早了解到，可以有更多的机会沟通，尽早发现问题等
- Merge 之前需要用 rebase 理事好 commit
- 尽量保持**原子提交**
- issue 中需要有详细描述，无论是 feature 还是 bug，并给出相关的资源链接地址，如果是站内，使用 [GitLab 引用链接格式][gitlab-markdown]
- merge request 中使用使 [GitLab 引用链接][gitlab-markdown] 指定关联的 issue，需要代码 Review 的时候（我们规定 Review 是从去掉 WIP 前缀之后）在 merge request 里讨论
- 合并 merge request 的时候注意删除远程仓库中已合并的 feature 或者 hoxfix 分支名
- Merge request 强制使用 `--no-ff` 方式合并，产生一个 merge commit，方便代码仓库回溯和问题查找
- 分支不宜有过多改动，尽量拆分成小的改动
- 避免长期在分支上开发而不合并，也避免落后 base 很久，这必然会引入冲突，相当于让这个分支上的一部分工作失效
- 建议直接在团队仓库中开发，开分支并提交 Merge Request，而不是 fork 一份到自己的命名空间下，然后从自己的库发起 Merge Request（Github 上的开源项目一般是这么干）

## Git 建议

- 避免使用 git pull，使用 `git fetch`，然后视具体情况选择操作 `git merge` `git reset` `git rebase` 等(2.27 版本引入了一个[警告][git-warning]，需要显式指定 pull 是什么操作或者允许什么操作)
- 本地仓库可以进行任意操作，不用担心提交的改动丢失。例如可以将未 push 到远程仓库的改动暂时 push 到自己的私有仓库备份，或者从 reflog 中找回
- 在本地执行 rebase 的时候需要留意，如果 commit 的作者不是自己，在产生冲突的时候，需要保留作者信息，commiter 可以是你
- **理解 git 操作和内部原理，知道自己在做什么**
- 本地 `git config --global user.name` `git config --global user.email` 需要和 gitlab 上的用户名邮件相同，避免在代码仓库里同一个人有两个甚至是多个不同的显示（如果公司内部的 git 名和自己对外的名字不一样，可以只在当前项目里设置，即不需要 `--global`）
- 熟练使用 `git rebase -i` 调整分支 commit，可以拆分或者合并，或者是调整顺序等
- 本地代码未提交的情况下，需要切换分支的时候，可以用 `git stash` 或者是 `git stash save -u 'some message'` 暂时放入本地 stash 空间
- 删除代码的时候用删除，而不是注释掉，不用担心删除掉的代码以后可能会再用到，可以从历史记录中找回
- 了解一下 **Reflog** 时光机器
- 读一下 [Pro Git][pro-git-book]
- 更多……

## 对于开源项目的建议

公司内部会有很多项目是基于开源项目改造而来的，但很多时候由于仓库管理不佳，导致无法跟上原始仓库的改动，造成一些 Bug 无法/难以从社区的修复中获取。

保持原来项目的 commit 信息，而不是基于原始仓库的某个版本重新 init 一个仓库，这样就丢失了原来的提交信息，不利于后期维护。

一个 Git 仓库可以有多个 remote，我们在改造开源项目的时候，可以在原始仓库的某个点开启一个分支，然后将这个新的分支的改动 push 到内部仓库。例如，

```sh
git clone https://github.com/<namespace>/<repository name>.git
cd <repository name>
git remote add internal ssh://git@<registry address>/<namespace>/<repository name>.git
# push master 分支到 internal
git push internal master
```

基于开源项目，如果改动不大，尽量以通用的方式改动，同时把 patch 回馈到开源社区，这样内部的代码就与社区的代码在最大程度上保持一致，也有利于从社区获取更新。

对于内部项目，在 Gitlab 上也建议用这种方式，在 fork 出来的仓库与原仓库之间最好的能够尽量保持一致，这样相同的 bug 修复就能够惠及到所有人。

<!-- TODO -->
<!---->
<!-- ## 注意事项 -->
<!---->
<!-- Git 仓库非常强大且灵活，但在团队协作中，过于灵活的改动会带来一些问题和冲突，应尽量避免。 -->
<!---->
<!-- - 尽量避免已经发布的 tag，因为别的项目有可能是依赖某个固定的 tag 的，比引用 reference 要直观。改动 tag 在很多工具上也不太支持 -->
<!-- - 尽量避免在主干分支上做强制改动（rebase 或者 `push --force`），与 tag 同理。一般在内部的项目不一定需要 tag 作为版本控制，但必需要以主干为稳定的发布分支（所以 Gitlab 默认是将 master 设置为 protected 分支保护起来的） -->
<!-- - 尽量不要在一个 commit 里包含过于大量的修改，能够拆分的尽量拆分，理由见上面叙述 -->
<!-- - TODO more -->

## Git 工具

Git 命令行工具已经非常强大，几乎能做所有的事，但配合优秀的 GUI 图形化工具和集成编辑器环境，能够更加有效得提升 Git 的使用效果。例如，好的 GUI 界面能够更加直观得显示提交的历史树形结构，好的 Diff 工具能够比较出行内的差异，甚至是某些二进制文件的差异。

- 本地开发环境使用 GUI 工具检视提交历史，例如 [Git Tower][git-tower]、[Gitfox][gitfox]、[Git Fork][git-fork]、[SourceTree][sourcetree] 等，参见 [Git GUIs][git-guis]
- 使用更加友好的 Three-way merge tools 查看冲突和差异，例如 [Kaleidoscope][kaleidoscope]
- 集成到编辑器中，如 Vim，Emacs，Visual Studio Code 等


## Atomic commit

代码提交符合 Atomic Commit （原子提交）原则，每个 commit 的修改内容单一，并处于可以编译运行的状态。
开发分支允许有临时的提交，但正式合并前，需要确保每个 commit 符合规范，必要的时候，进行手动 rebase 操作
（`git rebase -i based-branch`），如合并 commit、拆分 commit、编辑 commit message 等。

每个 commit 修改内容单一，指的是相关的修改。例如：

- 修改代码格式(fmt) 需要作为一个单独的提交，不应该与修改代码逻辑的提交放在同一个 commit 里。
- revert 单独一个 commit，不应该与逻辑修改放在一起。方便后续与原 commit 合并消除，也方便逻辑代码 diff review。


<!-- ## Code Review -->
<!---->
<!-- GitLab 工作流程中的代码审核指的是从 Merge Request 建立到 Merged 的这个过程。我们更进一步约定，从 Merge Request 的标题去除 `WIP` 前缀开始算 Code Review。当然，在 `WIP` 过程中也可以审核和讨论。 -->
<!---->
<!-- - 是否已经 rebase base 分支，如 develop -->
<!-- - 接口是否简单优雅 -->
<!-- - 是否存在性能问题 -->
<!-- - 是否符合编码规范 -->
<!-- - 是否运行测试代码和 Lint 等检查工具（这部分可以结合 CI 来做） -->

## Work Flow References

- [Git Flow][git-flow]
- [GitHub Flow][github-flow]
- [GitLab Flow][gitlab-flow]

# Best Practices

- [Git Repositories Best Practices][project-guidelines]

[git-flow-model]:  https://nvie.com/posts/a-successful-git-branching-model/
[gitlab-markdown]: https://docs.gitlab.com/ee/user/markdown.html#gitlab-specific-references
[git-warning]:	https://raw.githubusercontent.com/git/git/master/Documentation/RelNotes/2.27.0.txt
[pro-git-book]:	https://git-scm.com/book/zh/v2
[git-tower]:	https://www.git-tower.com
[gitfox]:	https://gitfox.app/
[git-fork]:	https://git-fork.com
[sourcetree]:	https://www.sourcetreeapp.com
[git-guis]:	https://git-scm.com/downloads/guis
[kaleidoscope]:	https://www.kaleidoscopeapp.com
[git-flow]:	https://nvie.com/posts/a-successful-git-branching-model/
[github-flow]:	https://guides.github.com/introduction/flow/
[gitlab-flow]:	https://docs.gitlab.co.jp/ee/topics/gitlab_flow.html#git-workflow
[project-guidelines]:	https://github.com/elsewhencode/project-guidelines

[image-1]:	/img/git-workflow/git_flow.png
