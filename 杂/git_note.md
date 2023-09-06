## git相关操作
### 配置GitHub账户

```bash
git config --global user.name "用户名"
git config --global user.email "邮箱地址"
```
### git本地库推送到远程

```git
git init -- 新建一个本地仓库
在远程创建一个仓库
git remote add origin 远程仓库地址例如  git remote add origin https://github.com/eddyfile/DataStructure.git
git pull origin master:master  //从远程分支拉取master分支并与本地master分支合并。
git add .
git commit -m "提交信息"
git push
```
### 删除git本地缓存

```git
git rm -r --cached .
```
### 提交暂存区到仓库

```git
git commit -m [说明]
```
### 显示变更信息

```
git status
```
### git push提交成功后撤销回退

```
git reflog //查看版本更新情况
git reset --hard [版本号]
git push --force //推送至远程
```

### 撤销操作

在提交完之后发现漏掉文件或者信息填写错误，可使用<font color = red> --amend </font>
```c
git commit --amend
```

### 撤销上一次`commit`

```bash
git reset --soft HEAD^
```

### 回滚至指定版本

```bash
git reset --hard id
```

### 本地推送至远程仓库新仓库

```bash
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin <远程仓库地址>
git push -u origin main
```

### 本地推送至远程已有仓库

```bash
git remote add origin git@github.com:eddyfile/learn-ldd-master.git
git branch -M main
git push -u origin main
```

### git查看本地仓库的远程推送地址

```bash
git remote -v
```

### 配置SSH

```bash
ssh-keygen -t rsa -C "xxx@xxx.com"
//执行后一直回车即可
在/home/.ssh/文件夹找到id_rsa.pub，将其配置到对应的git远程仓库的SSH配置中即可
```

### 修改commit信息

```bash
/*填写更改的信息*/
git commit --amend or git commit --amend -m "修改的提交信息"
/*强制推送到远程（覆盖上次提交信息）*/
git push --force
```

### 切到某个commit

```bash
/* 切到某个commit */
git checkout <commit号>
/* 切换到最新commit */
git checkout -
```

### 创建tag

```bash
/* 创建tag */
git tag -a <标签名称> -m "标签的注释信息"
/* 推送标签至远程 */
git push --tags
```

### 查看当前commit哈希值

```bash
git rev-parse HEAD
```

### 查看远程于本地文件更改

```bash
git diff --name-only
git diff --name-only <本地分支> <origin/远程分支>
```

### 保存工作现场

```bash
git stash
```

git stash命令可以将当前未提交的工作隐藏起来。让你的工作区变的干净清爽。

### 恢复工作现场

```bash
git stash apply
```

### 恢复并删除工作现场

```bash
git stash pop
```

### 撤销上次push

```bash
git push --force origin HEAD^:master
```

查看最近几次的提交记录

```bash
git log -p -2
```

`-2`可修改数字，查看最近几次的提交记录。

### 代理配置

#### 设置代理

设置代理，根据自己端口号设置。

```bash
git config --global http.proxy http://127.0.0.1:1080
git config --global https.proxy http://127.0.0.1:1080
```

#### <span id="取消代理">取消代理</span>

```bash
git config --global --unset http.proxy
git config --global --unset https.proxy
```



## .gitignore

### .gitignore 文件的格式规范如下：

- 所有空行或者以 ＃ 开头的行都会被 Git 忽略。
- 可以使用标准的 glob 模式匹配。
- 匹配模式可以以（/）开头防止递归。
- 匹配模式可以以（/）结尾指定目录。
- 要忽略指定模式以外的文件或目录，可以在模式前加上惊叹号（!）取反。

#### 所谓的 glob 模式是指 shell 所使用的简化了的正则表达式。

- 星号（*）匹配零个或多个任意字符；
- [abc]匹配任何一个列在方括号中的字符（这个例子要么匹配一个 a，要么匹配一个 b，要么匹配一个 c）；
- 问号（?）只匹配一个任意字符；
- 如果在方括号中使用短划线分隔两个字符，表示所有在这两个字符范围内的都可以匹配（比如 [0-9] 表示匹配所有 0 到 9 的数字）。
- 使用两个星号（`**`) 表示匹配任意中间目录，比如`a/**/z`可以匹配 a/z, a/b/z 或 a/b/c/z等。

### 常用规则

```
build/
#过滤整个build文件夹
*.zip
#过滤所有.zip文件
/build/do.c
#过滤/mtk/do.c文件

fd1/*　　　
#忽略目录 fd1 下的全部内容

/fd1/*　　　　
#忽略根目录下的 /fd1/ 目录的全部内容；

config.ini
#不需要提交config,ini文件

!/fw/bin/
!/fw/sf/
#不忽略 根目录下的 /fw/bin/ 和 /fw/sf/ 目录；
```

## 上次commit并push时未提交部分修改的文件，将这些文件再次提交至上次的commit并推送至远程

```bash
git add .
git commit --amend
git push --force-with-lease
```

## 添加忽略某个文件或文件夹后删除远程已经推送的文件或文件夹

删除远程仓库的文件，保留本地。

```bash
git rm -r --cached <文件夹>
git commit -m "del xxx"
git push
```

若需要删除远程，同时删除本地，去掉命令里的`-r`即可。

## 统计代码行数

统计当前项目代码行数

```bash
git ls-files | xargs cat | wc -l
```

细分每个文件的代码行数，相当于把上面命令细化

```bash
git ls-files | xargs wc -l
```

仓库提交者排名前 5（如果看全部，去掉 head 管道即可）

```bash
git log --pretty=’%aN’ | sort | uniq -c | sort -k1 -n -r | head -n 5
```

统计代码提交的人数，也称：统计仓库提交贡献者

```bash
git log --pretty=’%aN’ | sort -u | wc -l
```

统计总提交次数

```bash
git log --oneline | wc -l
```

## Failed to connect to github.com port 443:connection timed out

解决办法[取消代理](#取消代理)
