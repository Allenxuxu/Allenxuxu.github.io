---
title: "git 修改已经 commit 的邮箱信息"
date: 2020-09-10T11:22:38+08:00
draft: false
tags: ["git"]
---

开发过程中，经常会出现提交邮箱搞错的情况。在公司项目中错误提交了自己的 GitHub 邮箱，或者在开源项目中提交了公司邮箱。

下面记录一下补救措施。

先修改 .git/config 或者 修改全局的，修改成你需要的邮箱信息。

```
[user]
    email = name@qq.com
    name = yourname
```

git log 找到要修改的那一条 commit，复制要修改的commit 的前一条 commit 的哈希值。

`git rebase -i {{刚刚复制的哈希值}}`

```javascript
# 或者最近 3 条
$ git rebase -i HEAD~3
```

然后后会出现一个 vim 打开的文本，将需要修改的 commit 信息前面的 pick 文本改成 edit，保存退出。

修改邮箱信息

`git commit --amend --author="name <name@qq.com>" --no-edit`

然后 `git rebase --continue`

> 中间也可跳过或退出 rebase 模式`git rebase --skip` `git rebase --abort`

循环执行上面两步，当输出 `Successfully rebased and updated refs/heads/master.` 修改完成。

这时候查看 git log 信息，发现邮箱已经更改了。

强制 push 到远程（注意风险）

`git push -f origin HEAD:master`

done！