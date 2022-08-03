---
layout: post
title: git的PR与MR两种工作方式
categories: git
description: the difference between MR and PR of git
keywords: git MR PR
---

# git的PR和MR提交代码的工作方式

​	首先无论 pull request 还是 merge request 都不是 git 本身的功能，而是 GitHub/GitLab 等服务提供的额外功能。

​	push request 不存在，因为 push 是把你本地的 commit 推到 remote，要么你有权限 push（比如，我的这个github博客，我有权限push，所以每次写完笔记都直接将本地的commit推到远程仓库。），要么没有，且你没法请别人代你做，因为多数情况下别人没有你本地的 commit。但如果你已经把自己的 commit 给 push 到某处（你有权限的 repo 或分支，且能被其他人访问到），然后想把这些修改合入另一个你没有权限的 repo 或分支，你就可以请求有权限的人帮你 pull 或 merge，所以就有了 pull/merge request。

**pull/merge request** 分别代表了两种不同的工作流程：

**pull request** 用于 fork 出来的 repo 之间（**用于多个仓库之间**），一个 repo 希望给另一个 repo 提交代码，常见于 GitHub上公开的代码仓库间协作开发。比如，我想要向某项目 Microsoft/vscode 提交代码，那么我先 fork 出来一个自己的 zhouyue/vscode，修改完成后 git push 到 zhouyue/vscode 某分支上（显然我没有 Microsoft/vscode 的 push 权限），然后通过 GitHub 的网页或者 API 向 Microsoft/vscode 发起一个 pull request，对方如果同意则会从他那边拉取（pull） 我在 zhouyue/vscode 上的代码，我的代码就过去了。因此得名 pull request：我请求对方拉取我（在另一个仓库上）的代码

**merge request** 用于同一个 repo 不同分支之间需要合并代码的情况（**用于同一个仓库，多个分支之间**），常见于 GitLab 上一个团队共享同一 repo 合作开发。比如我是新员工，没有权限提交代码到 master 分支上，但我可以把修改 push 到一个自己的 zhouyue 分支上，然后我发起一个从 zhouyue 分支到 master 分支的 merge request，如果被批准后，zhouyue 分支则会合并到 master 分支上。因此得名 merge request：我请求有权限的人合并某个分支的代码到同一个仓库的另一个分支上。

一般，我们公司都会搭建自己的gitlab，代码提交采用的就是**Merge Request**方式：

```c
1. git clone gitlab-url				/* 从gitlab仓库clone最新代码 */
2. git checkout -b zmy master			/* 在本地新建一个开发分支zmy */
3. 本地在新建的zmy分支开始开发并提交代码，可以有多个本地commit，最后使用git rebase -i来合并为一个提交，这个过程中为了避免本地机器硬件故障导致开发丢失，可以随时将本地的zmy分支push推送到远程的gitlab仓库上，这样本地的开发随时都可以同步到远程分支。
4. 开发&测试完毕，git rebase -i合并多个本地commit为一个commit。
5. git checkout master & git pull	/* 切换到本地的master分支，拉取远程其他人的提交 */
6. git checkout zmy					/* 切换到本地的zmy分支，准备merge最新的代码 */
7. git rebase master				/* 将我们的代码与远程最新代码做merge，并解决冲突 */
8. git push -f origin zmy:zmy		/* 将本地开发分支推动到远程的同名分支，也可以不同名 */
9. 登录到gitlab网站，选择zmy分支，提交MR请求.
```

以上，是一个标准的工作流程，更安全可靠，如果开发周期很短，比如几个小时就可以搞定的bug，甚至可以：

```c
1. git clone gitlab-url				/* 从gitlab仓库clone最新代码 */
2. 无需新建本地开发分支，直接在本地master分支上修改bug，同样可以有多个commit，最后使用git rebase -i来合并为一个commit；修改完毕后准备提交前，也要先拉取最新的代码，虽然可能时间窗口很短，远程不一定有最新的提交，为了安全必须执行以下。
3. git pull --rebae	/* 拉取远程的最新代码，与上面不同，这个会自动做merge，有冲突会中止merge，并提交你解决冲突，解决完毕git rebae --continue继续执行git pull操作，注意，执行这个命令前本地代码必须要先commit，再pull代码。 */
4. git push origin master:zmy	/* 将本地master分支推送到远程的zmy分支 */
5. 登录到gitlab网站，选择zmy分支，提交MR请求.
```



### 原文链接

> 作者：周越
> 链接：[https://www.zhihu.com/question/334230718/answer/743900086](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.zhihu.com%2Fquestion%2F334230718%2Fanswer%2F743900086)
> 来源：知乎