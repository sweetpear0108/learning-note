# git
## 结构
![结构](https://www.runoob.com/wp-content/uploads/2015/02/1352126739_7909.jpg)

工作区：实际代码目录

版本库：.git目录，主要分为两部分：
* 暂存区（stage/index）：保存 `git add <file>` 添加的更改
* 本地仓库（respository）：保存了项目的所有历史记录和元数据（上图中的objects）

## HEAD
HEAD是一个指针，指向某个提交，用来确定当前工作目录的状态和所在位置

`git rev-parse HEAD` 查看HEAD指向的提交Hash

## branch
使用branch进行“版本控制”而不是进行“开发”，branch只是对仓库中所有commits进行管理排序，而不是将代码“提交”到分支。

```
0 <- 1 <- 2         <-------------------- master
           \
            3 <- 4 <- 5 <- 6      <------ A
                       \
                        7 <- 8 <- 9   <-- B
```

在如上所示的仓库中，master分支包括0～2提交，A分支包括0～6，B分支包括0～9

## merge
将多个开发历史合并（`Join two or more development histories together`）

主要有两种merge方法：
* `3-way merge`：创建一个“合并提交”作为两个分支的交点
* `Fast forward merge`：只改变分支指针（适用于当前分支和目标分支位于一条线性历史上，etc. rebase过后）

优势：非破坏性操作，方便协作；可以查看上游更改。

缺点：大量合并提交污染分支历史。

### 命令
```
git merge [ -ff | --no-ff | --ff-only ] <branch> [<target>]
```
等价
```
git checkout <target>
git merge [ -ff | --no-ff | --ff-only ] <branch>
```

* `<branch>` 要合并的分支

* `<target>` branch被合并到的分支

* 合并方法参数
    * `-ff`：优先快速合并，如果两分支不是线性历史则使用 `3-way merge`
    * `--no-ff`：使用 `3-way merge`
    * `--ff-only`：只使用快速合并，否则直接失败


## rebase
将提交重新应用到另一个基准点之上（`Reapply commits on top of another base tip`），称为**变基**（改变基准点）

比如，当前分支是从主分支commit1节点fork出来的，此后主分支又新增了2个提交commit2和commit3，这时候 ```git rebase master```可以将工作分支相对于主分支的“分叉点”改到commit3上。

```
                4' <- 5' <- 6'  <----  A after rebase
                /
0 <- 1 <- 2 <- 3        <-------------------- master
       \
        4 <- 5 <- 6      <------ A [abandon]
```

优势：避免了“合并提交”的污染；线性历史方便追根溯源

缺点：将一个公共的分支追加到到自己的分支会导致与其他协作者的公共分支相异；看不到当前分支中并入了上游的哪些更改。

### 命令
```
git rebase [-i] [--onto <newbase> | --keep-base] [<upstream> [<branch>]]
```

* `-i` 可以进行交互式操作，在文本编辑器内自定义提交.

* `<upstream>` 要比较的上游分支。 可以是任何有效的提交，而不仅仅是现有的分支名称。 默认为当前分支配置的上游。

* `<branch>` 工作分支；默认是HEAD。

* `--onto <newbase>` 目标分支，不指定默认为upstream

#### --onto
假设想把commits 7～9 rebase到master上，需要使用onto

```
0 <- 1 <- 2         <-------------------- master
           \
            3 <- 4 <- 5 <- 6      <------ A
                       \
                        7 <- 8 <- 9   <-- B
```

错误命令：```git rebase master B```  

output：```Current branch xx is up to date.```

原因：此命令为“将3～9提交rebase到2后边”，当然不用变了。错误在于对[branch含义](#branch)的理解。 

正确命令：```git rebase --onto master A B```

解释：`newbase`为master（commit 2），`upstream` 为A（确定了要copy的范围 7～9），`branch`为B（当前工作分支）

## cherry-pick
应用某些已有提交所引入的更改。简单来说，将**某些**提交应用于当前分支。

### 命令
`git cherry-pick A..B`

A，B是提交的Hash值，范围是 `(A, B]`，如果要包括A，则取A的父节点 `A^`

提交 A 必须早于提交 B

## 参考
[atlassian的教程](https://www.atlassian.com/git/tutorials)

[git doc](https://git-scm.com/docs)

[stackoverflow rebase onto vs rebase](https://stackoverflow.com/a/33948237)