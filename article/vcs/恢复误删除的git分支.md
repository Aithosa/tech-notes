# 恢复误删除的 git 分支

如果不小心在本地把某个本地分支 A 以及远端分支 origin/A 都删了的话...

## 目前可用的方法

1. 比较可靠的方式：问下其他人本地有没有代码，idea 会自动 fetch，所以有比较大的可能性能找到分支 A 的较新状态，可以请他重新推到远端(其他 git 工具也有可能)。
2. 远端分支还在的话可以考虑重新拉下来，如果本地没有额外提交的话
3. 用 git 恢复分支，这种方法可行但是不一定能完全恢复

## 使用 git 恢复分支

首先记得整体备份一下

### 如果删之前往 fdev 或者往其他分支合过/如果这个分支的提交没往别的分支合过

1. 查找最新 commit 信息
   - 找到 fdev 提交记录里分支 A 最新的一次 commit(非 merge)，记下 sha
   - `git log -g` 找到你在那个分支的最后一次提交的 commit,记下 sha
2. 使用`git branch recover_branch[新分支] commit_id`命令用这个 commit 重新创建一个分支

如：`git branch recover2 892a35078757471acd4e78fd8773b8e01ca0cbd5`

如果用的是 GitHub 官方客户端可以右键分支 A 最新 commit，选择`create branch from commit`(仅适用于删之前往 fdev 或者往其他分支合过，如果没合过还是用`git log -g`找 commit)

可以看到新建的分支都是分支 A 的提交，没有把 fdev 中途别的分支合过来带的提交弄过来。
