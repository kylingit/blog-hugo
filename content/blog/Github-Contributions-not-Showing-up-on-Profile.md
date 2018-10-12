---
title: Github 没有记录 Contributions 的解决
date: 2017-03-17 16:46:07
tags: [Github,  Git]
categories: Coding
---
<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

最近更新文章的时候发现github没有记录Contributions，也就是小绿墙没有增加小方块，看了下`git log`发现是提交的时候用户邮箱写错了，多打了一个字母...于是Github认为这些commits都不是我提交的 =_=#
网上找到了解决办法

<!-- more -->

### 1. 重新克隆一个repo

```
git clone --bare https://github.com/user/repo.git
cd repo.git
```

### 2. 新建一个脚本

```
#!/bin/sh
git filter-branch --env-filter '
OLD_EMAIL="旧的Email地址"
CORRECT_NAME="正确的用户名"
CORRECT_EMAIL="正确的Email地址"
if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_COMMITTER_NAME="$CORRECT_NAME"
    export GIT_COMMITTER_EMAIL="$CORRECT_EMAIL"
fi
if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_AUTHOR_NAME="$CORRECT_NAME"
    export GIT_AUTHOR_EMAIL="$CORRECT_EMAIL"
fi
' --tag-name-filter cat -- --branches --tags
```

### 3. 更改旧的邮箱，以及填写正确的用户名和邮箱，执行

### 4. 把正确历史 push 到 Github

```
git push --force --tags origin 'refs/heads/*'
```

### 5. 查看`git log` 检查push历史是否被更正，没有错误的话就可以删掉这个clone了

### 6. done


参考:

1. [Changing author info](https://help.github.com/articles/changing-author-info/)

2. [为什么Github没有记录你的Contributions](https://segmentfault.com/a/1190000004318632)



<script>pangu.spacingPage();</script>
