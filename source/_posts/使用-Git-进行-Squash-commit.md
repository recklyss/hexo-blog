---
title: 使用 Git 进行 Squash commit
date: 2018-10-11 22:37:07
tags: [Git, 开发工具]
categories: [Git]

---

---

## Git 更改 commit 的操作

1. ` git rebase -i HEAD~<number 代表需要处理几个 commit>`

2. ```shell
   # Rebase ddebba2..a54dc28 onto 9d9ba60 (15 commands)
   #
   # Commands:
   # p, pick <commit> = use commit
   # r, reword <commit> = use commit, but edit the commit message
   # e, edit <commit> = use commit, but stop for amending
   # s, squash <commit> = use commit, but meld into previous commit
   # f, fixup <commit> = like "squash", but discard this commit's log message
   # x, exec <command> = run command (the rest of the line) using shell
   # b, break = stop here (continue rebase later with 'git rebase --continue')
   # d, drop <commit> = remove commit
   # l, label <label> = label current HEAD with a name
   # t, reset <label> = reset HEAD to a label
   # m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
   # .       create a merge commit using the original merge commit's
   # .       message (or the oneline, if no original merge commit was
   # .       specified). Use -c <commit> to reword the commit message.
   #
   # These lines can be re-ordered; they are executed from top to bottom.
   #
   # If you remove a line here THAT COMMIT WILL BE LOST.
   #
   # However, if you remove everything, the rebase will be aborted.
   #
   # Note that empty commits are commented out
   ```

3. 根据上面每一个指令操作，更改以下类似内容：

   ```shell
   pick 54f205a Update README.md
   pick e1deb05 Update README.md
   pick 3a33ad2 Update README.md
   pick 225a513 Update README.md
   pick d44d34b Update README.md
   pick 657d8c2 Update README.md
   ```

   ```shell
   pick 54f205a Update README.md
   pick e1deb05 Update README.md
   squash 3a33ad2 Update README.md
   squash 225a513 Update README.md
   squash d44d34b Update README.md
   squash 657d8c2 Update README.md
   ```

4. 这样就可以把最上面两个 Message 保留，把后面的 Message 去掉

5. 最后`git push --force`
