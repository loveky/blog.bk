---
layout: post
title: "你可能不知道的git rerere"
description: ""
category: Git
tags: [Git]
---
{% include JB/setup %}

## rerere 是啥的缩写？
rerere = reuse recorded resolution

## rerere是干啥用的？
它会让Git记住你是如何解决某个文件的两个版本之间的conflict，这样在下次Git遇到同样的文件在相同的两个版本间发生冲突时，可以自动帮你使用相同的方法解决冲突。

## 如何启用rerere？
`git config --global rerere.enabled true`

或是

在repo目录里`mkdir .git/rr-cache`

不过推荐前者。

## 能举例说明吗？
参考inline的注释。
{% highlight bash %}
$ git init git-rerere-test
Initialized empty Git repository in /cygdrive/e/test/git-rerere-test/.git/

$ cd git-rerere-test

$ git config rerere.enabled true # 开启rerere功能

$ echo "What fruit do you like?" >> question

$ git add . && git commit -m "initial drop"
[master (root-commit) af9fdf9] initial drop
 1 files changed, 1 insertions(+), 0 deletions(-)
 create mode 100644 question

$ git branch test # 创建test分支

$ echo "I like orange" >> question

$ git add . && git commit -m "answer in master" # 在master上commit 1次
[master 3b48cca] answer in master
 1 files changed, 1 insertions(+), 0 deletions(-)

$ git checkout test
Switched to branch 'test'

$ echo "I like apple" >> question

$ git add . && git commit -m "answer in test" # 在test分支上commit 1次制造conflict
[test cda9dab] answer in test
 1 files changed, 1 insertions(+), 0 deletions(-)

$ git checkout master
Switched to branch 'master'

$ git merge test # 在master上merge test
Auto-merging question
CONFLICT (content): Merge conflict in question
Recorded preimage for 'question' # 多了这一句，表示Git已经开始track你的操作了
Automatic merge failed; fix conflicts and then commit the result.

$ vim question

$ cat question # 修改文件内容，解决冲突
What fruit do you like?
I like orange & apple

$ git status -s
UU question

$ git add question

$ git commit -m "merge test" # commit合并
Recorded resolution for 'question'. # Git记录了这次解决冲突的方法
[master 0b55608] merge test

$ git reset --hard HEAD^ # 重置这次合并，再次merge看看rerere的效果
HEAD is now at 3b48cca answer in master

$ git merge test
Auto-merging question
CONFLICT (content): Merge conflict in question
Resolved 'question' using previous resolution. # Git已经用记录的方法解决冲突了
Automatic merge failed; fix conflicts and then commit the result.

$ cat question # 查看文件内容，已经修改
What fruit do you like?
I like orange & apple

$ git status -s
UU question

$ git add .

$ git commit -m "merge test" #再次commit合并
[master ea6a966] merge test

$ git reset --hard HEAD^ # 最后看一下对rebase的支持
HEAD is now at 3b48cca answer in master

$ git branch
* master
  test

$ git checkout test # 切换到test分支
Switched to branch 'test'

$ git rebase master
First, rewinding head to replay your work on top of it...
Applying: answer in test
Using index info to reconstruct a base tree...
Falling back to patching base and 3-way merge...
Auto-merging question
CONFLICT (content): Merge conflict in question
Resolved 'question' using previous resolution. # 同样，Git使用历史记录解决了冲突
Failed to merge in the changes.
Patch failed at 0001 answer in test

When you have resolved this problem run "git rebase --continue".
If you would prefer to skip this patch, instead run "git rebase --skip".
To check out the original branch and stop rebasing run "git rebase --abort".

$ git status -s
UU question

$ cat question
What fruit do you like?
I like orange & apple

$ git add question

$ git rebase --continue
Applying: answer in test

{% endhighlight %}

## 什么情况下会用到这个功能？
假设你有如下的history
{% highlight bash %}
                         o---*---o topic
                        /
               o---o---o---*---o---o master
{% endhighlight %}
`*`表示的commit同时修改了同一个文件的同一块代码。这时你想测试一下两处修改有没有破坏build/功能。你可能会
{% highlight bash %}
                   $ git checkout topic
                   $ git merge master
{% endhighlight %}
结果是生成了如下的历史树：
{% highlight bash %}
                         o---*---o---+ topic
                        /           /
               o---o---o---*---o---o master
{% endhighlight %}
你解决了冲突并生成了`+`这个commit。测试完成后，你继续在topic分支上工作，同时master分支上也有了新的commit。最终，你在topic分支上的工作完成，merge回master分支。你执行了：
{% highlight bash %}
                   $ git checkout master
                   $ git merge topic
{% endhighlight %}
最终的历史树看起来像这样：
{% highlight bash %}
                         o---*---o---+---o---o topic
                        /           /         \
               o---o---o---*---o---o---o---o---# master
{% endhighlight %}
如果你的topic分支会存活很长时间，你可能多次从master分支merge到topic进行测试。那么你可能会看到很多`+`的commit(从master分支到topic的线)，这会让历史树看起来不那么直观。

其实你还有其他选择，在每次merge完成测试以后，丢弃掉这次merge(但是rerere已经帮你记录了如何解决冲突，不用担心日后再次费劲心思的处理)，继续原来的历史开发，直到最终topic分支完成任务，被merge到master(这时rerere会帮你处理掉merge conflict)，这样你的历史树看起来就像：
{% highlight bash %}
                         o---*---o-------o---o topic
                        /                     \
               o---o---o---*---o---o---o---o---+ master
{% endhighlight %}
干净多了。。。

## rerere帮我自动处理了conflict，但我已经忘了conflict的时候是啥样子了。。。能看看conflict的时候的样子吗
执行`git checkout --conflict=merge <path>`即可
