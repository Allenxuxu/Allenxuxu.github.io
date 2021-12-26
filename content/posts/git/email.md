---
title: "Git自动设置不同邮箱"
date: 2021-12-26T10:53:19+08:00
draft: false
tags: [git]
---

日常使用时，经常出现提交公司项目不小心用的私人邮箱，或者提交 Github 项目用了公司邮箱的情况，所以希望 Git 提交时能自动根据域名选择不同的邮箱。


全局设置必须配置用户名邮箱

```bash
git config --global user.useConfigOnly true
```

并且删除全局的 user.name 和 user.email 配置，这样本地如果有一些项目之前是读取全局配置邮箱的需要手动设置一下( 可以用复制下面的脚本里的部分代码进行自动设置，即 if 分支里的逻辑)。

> 所有的全局配置都在 ~/.gitconfig 文件中
>

设置 git hooks templates 目录

```go
mkdir -p ~/.git-templates/hooks
git config --global init.templatedir ~/.git-templates
```

然后在 `~/.git-templates/hooks`目录里新建 `post-checkout`文件，内容如下：

```bash
#!/bin/bash

if [[ $1 == 00000000000* ]]; then
	  remote=`git remote -v | awk '/\(push\)$/ {print $2}'`
	  email=xxx@xx.com # default
	  name="x x"

	  if [[ $remote == *需要匹配的公司域名* ]]; then
	    email=x@cc.com
	    name="x x"
	  fi

	  echo "Configuring user <name: $name email: $email>"
	  git config user.email "$email"
	  git config user.name  "$name"
fi
```

修改上面的用户名，邮箱和需要特殊匹配的公司域名。

这样在 `git clone` 的时候，git 会自动执行 `post-checkout`从而按照域名自动设置邮箱用户名。
