---
title: Git使用
date: 2017-04-10 10:42:35
tags: 
	- Git
---
## Git 常用命令
克隆一个Repository
```sh
git clone git@github.com:Uwingame/Uwingame.github.io.git
```

获取版本库的更新
```sh
git fetch
```

查看log
```sh
git log
```

查看修改
```sh
git diff {文件}
```
不带参数时显示所有修改

查看修改的文件
```sh
git status
```

拉取远程版本库的更新并且合并到当前分支
```sh
git pull origin master
```

推送本地版本库修改到远程分支
```sh
git push origin master
```

从当前分支新建一个分支
```
git checkout -b {分支名称}
```

合并一个本地的分支
```sh
git merge {分支名称}
```

合并一个远程分支
```sh
git fetch
git merge origin/{分支名称}
```

初始化Git，并且链接到远程版本库
```sh
git init
git add --all
git commit -m "first commit"
git remote add origin git@github.com:Uwingame/Uwingame.github.io.git
git push -u origin master
```

抛弃所有本地没有commit的修改，慎重使用
```
git clean -df
git reset --hard
```

## Git Tag功能
Commit的名字，使用Tag可以快速切换到对应的分支

添加Tag
```sh
git tag v1.0.0 {Commit}
```
如果不带Commit参数，默认是当前分支的Commit

删除一个本地Tag
```sh
git tag -d v1.0.0
```

上传Tag更改到远程服务器
```sh
git push origin --tags
```

删除远程Tag
```sh
git push origin :refs/tags/v1.0.0
```

## Git 其他功能

### Submodule
为项目添加一个子项目

### rebase
git pull 实现的是拉取和merge功能，git rebase 还原到一个版本，然后一个一个应用修改

使用实例，rebase 远程master分支
```sh
git fetch
git rebase origin/master
```
如果没有冲突，直接rebase成功，如果发现冲突，rebase会中断，这时需要手动解决冲突并进行add，解决完冲突后，继续rebase
```sh
git add --all
git rebase --continue
```
如果想放弃这次rebases，可以
```
git rebase --abort
```

## 恢复到指定commit

查看reflog
```sh
git reflog
```

恢复
```
git reset HEAD@{25}
```



