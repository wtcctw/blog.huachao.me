title:  "Git一站式"
date: 2015-04-12
updated: 2015-12-26
categories:
- tool
tags:
- tool
- git
---

> Git，版本控制神器

## Git的基本命令

``` bash
git init
git remote add <远程仓库别名> git@github.com:username/reponame.git
```

``` bash
git clone git@github.com:username/reponame.git  #推荐
git clone https://github.com/username/reponame.git
```

``` bash
git status                                    #查看当前分支状态
```

``` bash
git branch                                    #查看本地分支
git branch -a                                 #查看所有分支
git branch -D <branch_name>                   #删除branch_name分支
```

``` bash
git checkout -b <new_branch> <exist_branch>   #根据start_branch创建一个新分支
git checkout <exist_branch>                   #切换到exist_branch分支
```

``` bash
git log                                       #查看当前分支的提交历史
git log --oneline                             #在一行中显示log
git log <branch>                              #显示某个分支的提交历史
git log <file_name>                           #查看某个文件的改动历史（commit）
```

``` bash
git fetch                                     #获取远程分支更新
git pull origin                               #获取远程分支更新，并且merge远程分支到本地分支
git pull                                      #同上
git pull origin next                          #获取远程分支更新，并且merge远程的next分支到本地当前分支
git pull --rebase                             #获取远程分支更新，并且在当前分支上进行rebase
```

``` bash
git push origin                               #推送本地分支到远程对应分支
git push origin -f                            #强制推送本地分支到远程的对应分支
git push origin test:master                   #将本地的test分支推送到远程的master分支上
```

``` bash
git merge <branch>
git merge --squash
      A---B---C topic
     /         \
D---E---F---G---H master
注意，关于git merge只有看git log，能看到merge进来的分支上的commit，
但是实际上对于当前分支来讲，也就只有一个合并的commit而已,就是H。
要想看不到A,B,C,可以用 --squash
```

``` bash
# 方法1
git cherry-pick <commit>                      #把commit挂到当前分支最后
fix conflict
git add -A
git commit -am "comments"

git rebase <branch>                           #连续的cherry-pick
fix conflict
git add -A
git rebase --continue
```

``` bash
git reset --hard
git reset --soft
git reset --mixed
注意：理解working directory, index, HEAD区别。index指已经git add过的文件
```

``` bash
git diff                                      #当前修改和最新的commit比较
git diff <commit>                             #当前修改和某次commit比较
git diff <branch_a> <branch_b>                #两个分支的比较
```

``` bash
git stash -a                                  #暂存所有改动，包括新加入的没有index的文件
git stash pop                                 #弹出所有改动
```

``` bash
git tag -a tag_name -m "tag descprition"      #创建一个tag，基于当前分支的当前commit
git tag -d tag_name                           #删除一个tag
git push origin tag_name                      #push一个tag
```

## Git的基本概念
- 分清本地分支和远程分支。 所谓远程分支其实也是拷贝到本地的，并且每次得fetch了才能更新，一般在前面加上了origin/
- 四个重要的文件或目录：`HEAD` 及 `index` 文件，`objects` 及 `refs` 目录。这些是 Git 的核心部分。`objects`目录存储所有数据内容，`refs` 目录存储指向数据 (分支) 的提交对象的指针，`HEAD` 文件指向当前分支，`index` 文件保存了暂存区域信息。

## Git的其他方面
[gitignore模板](https://github.com/github/gitignore)

## 多个git账号如何配置

- 生成了多个ssh key `ssh-keygen -t rsa -C "new email"`
- 配置.ssh/config文件
``` bash
#Default Git
Host github.com
  HostName github.com
  User name1
  IdentityFile ~/.ssh/id_rsa_first
 
#Second Git
Host domain_name
  HostName domain_name
  User name2
  IdentityFile ~/.ssh/id_rsa_second
```
