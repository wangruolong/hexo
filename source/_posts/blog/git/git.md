---
title: Git
author: 大大白
date: 2018-01-03 12:00:00
categories:
- 其他
- 版本管理
tags:
- Git
---

在使用Git过程中的一些积累总结。主要有Sourcetree的安装使用、Gerrit基本用法、Submodule的基本用法等。

<!-- more -->

## Git
在Linux安装和使用Git
```
//生成公钥和私钥
ssh-keygen
//公钥添加到github
//私钥添加到本地git
ssh-add ~/.ssh/id_rsa
//如果执行ssh-add时出现Could not open a connection to your authentication agent
//先执行eval
eval `ssh-agent`
//再重新ssh-add就能成功了
ssh-add ~/.ssh/id_rsa
```

## Sourcetree
git的可视化的图形界面，里面有各种方便的操作。

## Gerrit
主要用来代码评审。
推送
```
git push origin HEAD:refs/for/[branch]
```

遇到no newchange的问题，可以重新commit一次。
```
git commit --amend
```
如果还不行，可以回退一个版本再提交。
```
git reset HEAD~1
```

遇到没有changeId的问题需要查看git目录下是否有gerrit的hooks

## Submodule
### 什么是 Submodule
有种情况我们经常会遇到：某个工作中的项目需要包含并使用另一个项目。 也许是第三方库，或者你独立开发的，用于多个父项目的库。 现在问题来了：你想要把它们当做两个独立的项目，同时又想在一个项目中使用另一个。
Git 通过子模块来解决这个问题。 子模块允许你将一个 Git 仓库作为另一个 Git 仓库的子目录。 它能让你将另一个仓库克隆到自己的项目中，同时还保持提交的独立。

### 快速上手 Submodule

#### 在引用方
检出项目地址
```
git clone
```

进入到子目录初始化子模块。
```
git submodule init
git submodule update
```

检出分支，拉取代码。
```
git checkout develop
git pull --rebase
```

主模块会记录子模块的git操作，因此需要在主模块把这个变化做一下提交。
```
git commit -m '拉取子模块代码'
git push
```

#### 在提供方
在提供方按照git的正常操作即可。
克隆代码
```
git clone
```

提交代码
```
git checkout develop
git commit -m '提交代码'
git push
```

### 在主模块加入子模块
先进入到你想要存放子模块的目录，然后把子模块的 git 仓库加进来。
```
$ git submodule add aaa
```

现在`git status`会发现，多了一个`.gitmodules`文件，和一个新的文件夹 xxx
```
$ git status
On branch feature/submodule
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        new file:   .gitmodules
        new file:   src/main/js/src/components/xxx
```

`.gitmodules`文件保存了项目 URL 与已经拉取的本地目录之间的映射，用`cat`命令可以查看到该文件的具体内容。如果有多个子模块，该文件中就会有多条记录。要重点注意的是，该文件也像`.gitignore`文件一样受到（通过）版本控制。它会和该项目的其他部分一同被拉取推送。这就是克隆该项目的人知道去哪获得子模块的原因。
```
$ cat .gitmodules
[submodule "src/main/js/src/components/xxx"]
        path = src/main/js/src/components/xxx
        url = xxx
        branch = stable
```

`path = src/main/js/src/components/xxx`，表示是 submodule 的目录，同时它被作为 key 来唯一标识，因为一个项目可能会存在多个 submodule。
`url = xxx`，是远程仓库的地址。
`branch = stable`，是跟踪的分支，可以设置跟踪分支也可以不设置跟踪分支。这里没有设置所以没有显示。

特别需要注意的是：默认情况下是没有跟踪分支的。它既不是跟踪 master 也不是跟踪任何分支，他是出于游离状态的 HEAD。那么这种游离状态的分支他要怎么跟新代码呢？你只需要`git merge origin/master`把 master 的代码合并到自己的游离分支就好了。当前游离的 commitId 和 master 的最新的 commitId 一致，则表示拉取成功！

虽然`xxx`是工作目录中的一个子目录，但 Git 还是会将它视作一个子模块。当你不在那个目录中时，Git 并不会跟踪它的内容， 而是将它看作该仓库中的一个特殊提交。

现在我们已经让`aaa`和`xxx`关联在一起了。那么别的小伙伴怎么用呢？

### 检出

#### 检出。根据检出的时机不同分两种情况。
第一种：在主项目未添加 submodule 的时候已经检出，这时候要获取 submodule 的代码需要如下操作。
```
git submodule init
git submodule update
```

第二种：直接重新检出项目，可以用递归检出的方式，把子模块也一起检出。
```
git clone --recursive aaa
```

### 拉取
#### 在主模块拉取
`git submodule update --remote`默认会更新 master 分支的代码，但是它本身又不处于 master 分支。它会更新 master 最新的代码。然后让自己处于游离状态。
```
$ git submodule update --remote
```

如果我们希望它更新我们指定分支的代码可以这样操作。有两种方法可以修改在全局范围内的子项目跟踪的远程分支。
第一种：直接修改`.gitmodule`文件，在最后一行加入`branch = stable`。例如：
```
$ cat .gitmodules
[submodule "src/main/js/src/components/xxx"]
        path = src/main/js/src/components/xxx
        url = xxx
        branch = stable
```
要特别注意换行和 tab

第二种：本质上也是和第一种一样，只不过它是用命令行的方式实现的操作。
```
$ git config -f .gitmodules submodule.src/main/js/src/components/xxx.branch stable
```
这里要特别注意`src/main/js/src/components/xxx`这个我们在 git add submodule 的时候的别名，如果没有设置别名，git 会自动把你的文件目录路径作为别名，也就是 key 保存下来，他在全局是唯一的。

在查看一下`.gitmodules`文件发现 branch 已经变了。
这时候再`git submodule update --remote`就会发现子模块的代码已经被关联到 stable 分支了，并且拉取了 stable 分支的最新代码。
```
$ cat .gitmodules
[submodule "src/main/js/src/components/xxx"]
        path = src/main/js/src/components/xxx
        url = xxx
        branch = stable
```

#### 在子模块拉取
在项目中使用子模块的最简模型，就是只使用子项目并不时地获取更新，而并不在你的检出中进行任何更改。
如果想要在子模块中查看新工作，可以进入到目录中运行`git fetch`与`git merge`，合并上游分支来更新本地代码。
进入到 submodule 拉取。
```
cd src/main/js/src/components/xxx
git fetch
git merge origin/master
```

或者也可以
```
cd src/main/js/src/components/xxx
git checkout stable
git pull --rebase
```

这个和我们正常使用 git 是一样的。

### 推送
#### 在主模块推送
这个和平常使用的 git 一样。子模块如果有更改，主模块也要 commit 后 push。

#### 在子模块推送
如果我们在主项目中提交并推送但并不推送子模块上的改动，其他尝试检出我们修改的人会遇到麻烦，因为他们无法得到依赖的子模块改动。 那些改动只存在于我们本地的拷贝中。
为了确保这不会发生，你可以让 Git 在推送到主项目前检查所有子模块是否已推送。 git push 命令接受可以设置为 “check” 或 “on-demand” 的 --recurse-submodules 参数。 如果任何提交的子模块改动没有推送那么 “check” 选项会直接使 push 操作失败。
首先，让我们进入子模块目录然后检出一个分支。
```
$ git checkout stable
Switched to branch 'stable'
```

然后在里面修改代码之后
```
$ git add .
$ git commit -a
$ git push
```

然后查看一下子目录都是 clean 的
```
$ git status
On branch stable
Your branch is up to date with 'origin/stable'.

nothing to commit, working tree clean
```

这样我们就在子模块里面把代码提交上去了。
然后我们再回到主目录查看一下状态。
在主目录会发现还有一个变动，它把子目录的任何变动都视为`xxx`的变动，你只要把这个文件 commit 上去就可以了。
```
$ git status
On branch feature/submodule
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)
  (commit or discard the untracked or modified content in submodules)

        modified:   xxx (new commits, modified content)

no changes added to commit (use "git add" and/or "git commit -a")
```

总结来说就是，需要在子目录 push 一次，然后再回到主目录 push 一次。
这样操作变得很繁琐，而且有时候在子目录提交了会忘记在主目录再提交一次，会造成没必要的误会。
我们可以设置 push 的`--recurse-submodules`参数，它有两个值，一个是 check 另一个是 on-demand。check 是在你 push 的时候检查子目录是不是还有没提交的，如果还有没提交的你要手动到子目录提交。而 on-demand 则是直接进入子目录自动帮你提交。让我们来试一下这两种操作。

当我们在子目录修改完后 commit，然后主目录的`xxx`会显示修改，我们这时候把主目录的修改也 commit，那么就剩最后一步 push 了。这时候加上参数，他会提示我们：子目录的文件还没有 push，你可以用 on-demand 这个参数，或者手动 cd 进去 push 之后再回来主目录 push

```
$ git push --recurse-submodules=check
The following submodule paths contain changes that can
not be found on any remote:
src/main/js/src/components/xxx

Please try

        git push --recurse-submodules=on-demand

or cd to the path and use

        git push

to push them to a remote.
```

然后我们尝试用 on-demand，但是居然报错了。
```
$ git push --recurse-submodules=on-demand
fatal: src refspec 'refs/heads/feature/submodule' must name a ref
```
这是因为我们用了 gerrit。

### 遇到的问题
#### git submodule 的时候报错
```
A git directory for 'src/main/js/src/components/xxx' is found locally with remote(s):
  origin        xxx
If you want to reuse this local git directory instead of cloning again from
  xxx
use the '--force' option. If the local git directory is not the correct repo
or you are unsure what this means choose another name with the '--name' option.
```
意思是这个名字已经被占用需要换一个，或者直接删除.git/modules 下面的所有文件。

#### 克隆含有子模块的项目
方法一：进入到子模块的目录后
```
git submodule init
git submodule update
```

方法二：直接重新检出项目
```
git clone --recursive aaa
```
