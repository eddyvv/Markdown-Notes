## <a name="index"/>目录
* [git相关操作](#title) 
    * [git本地库推送到远程](#git)
    * [配置GitHub账户](#global)
    * [删除git本地缓存](#delete)
    * [提交暂存区到仓库](#commit)
    * [显示变更信息](#status)

## <a name="title"/>git相关操作
* <a name="global">配置GitHub账户
```git
git config --global user.name "用户名"
git config --global user.email "邮箱地址"
```
* <a name="git"/>git本地库推送到远程
```git
    git init -- 新建一个本地仓库
    在远程创建一个仓库
    git remote add origin 远程仓库地址例如  git remote add origin https://github.com/eddyfile/DataStructure.git
    git pull origin master:master  //从远程分支拉取master分支并与本地master分支合并。
    git add .
    git commit -m "提交信息"
    git push
```
* 删除git本地缓存<a name="delete"/>
```git
    git rm -r --cached .
```
* 提交暂存区到仓库<a name="commit"/>
```git
    git commit -m [说明]
```
* 显示变更信息<a name="status"/>
```
    git status
```
* git push提交成功后撤销回退
```
    git reflog //查看版本更新情况
    git reset --hard [版本号]
    git push --force //推送至远程
```

* 撤销操作

在提交完之后发现漏掉文件或者信息填写错误，可使用<font color = red> --amend </font>
```c
    git commit --amend
```

* 撤销上一次`commit`

```bash
git reset --soft HEAD^
```



* 回滚至指定版本

```c
    git reset --hard id
```

* 本地推送至远程仓库新仓库
```c
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin <远程仓库地址>
git push -u origin main
```

* 本地推送至远程已有仓库
```c
git remote add origin git@github.com:eddyfile/learn-ldd-master.git
git branch -M main
git push -u origin main
```

* git查看本地仓库的远程推送地址
```c
git remote -v
```

* 配置SSH

```bash
ssh-keygen -t rsa -C "xxx@xxx.com"
//执行后一直回车即可
在/home/.ssh/文件夹找到id_rsa.pub，将其配置到对应的git远程仓库的SSH配置中即可
```

* 修改commit信息

```bash
/*填写更改的信息*/
git commit --amend or git commit --amend -m "修改的提交信息"
/*强制推送到远程（覆盖上次提交信息）*/
git push --force
```

