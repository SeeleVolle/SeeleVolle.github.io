# Git Notes

### Session 1 基本用法

+ 初始化：
  + `git init`

+ 提交修改
  + `git add readme.txt`
  + `git commit -m "wrote a readme file"`

+ 撤销修改
  + `git checkout -- <文件名> / git restore <文件名>`: 撤销当前所做的修改(还未 git add )
  + `git reset HEAD <文件名字> / git restore --staged <文件名>`: 撤销当前 git add 的暂存区中的内容

+ 查看状态

  + git status: 显示当前工作区的状态
    + 若git status告诉你文件被修改过，用git diff "filename"：可以查看修改内容

  + git log: 查看历史版本，`git log`命令显示从最近到最远的提交日志，如果嫌输出信息太多，看得眼花缭乱的，可以试试加上`--pretty=oneline`参数：
    +  git log的参数 
      > --pretty=oneline只显示版本号  
      > --graph显示分支合并图
      > --abbrev-commit 显示Author和Date并且详细显示merge的版本号
      > 注意到Git用`(HEAD -> master)`和`(origin/master)`标识出当前分支的HEAD和远程origin的位置分别是`582d922 add author`和`d1be385 init hello`

  ```shell
  $ git log --pretty=oneline（显示的全为版本号）
  ```

+ 版本回退

  + `git reset --hard HEAD^`  回退一个版本
  + `git reset --hard HEAD^^`, 回退两个
`
  + `git reset --hard HEAD~100`，回退一百个
  + `git reset --hard commit_id`

  + `git reflog`用来记录你的每一次命令

     - `HEAD`指向的版本就是当前版本，因此，Git允许我们在版本的历史之间穿梭，使用命令`git reset --hard commit_id`。

     - 回退：用`git log`可以查看提交历史，以便确定要回退到哪个版本。

     - 跳转：用`git reflog`查看命令历史，以便确定要回到未来的哪个版本。

+ 删除文件
  + rm <filename>或者手动删除文件(本地删除文件)，rmdir<foldername>删除文件夹
  
  + 再次使用git rm<filename>或者git add<filename>可以从**版本库**中删除该文件，并且可以git commit.
  
  + `git checkout`其实是用版本库里的版本替换工作区的版本，无论工作区是修改还是删除，都可以“一键还原”。
    + 情况一：工作区文件删除，无其他操作
      + git restore test.txt
    + 情况二：工作区文件删除，版本库文件删除(即多了git rm的操作并且进行了git commit)
      +  git reset --hard HEAD^

+ 工作区和暂存区
  + 工作区（Working Directory): 工作目录，比如我的`learngit`文件夹就是一个工作区：
  + 版本库（Repository）(暂存区是版本库的一部分)：工作区有一个隐藏目录`.git`，这个不算工作区，而是Git的版本库。

> Git的版本库里存了很多东西，其中最重要的就是称为stage（或者叫index）的暂存区，还有Git为我们自动创建的第一个分支`master`，以及指向`master`的一个指针叫`HEAD`。  
> 所以，`git add`命令实际上就是把要提交的所有修改放到暂存区（Stage），然后，执行`git commit`就可以一次性把暂存区的所有修改提交到分支。

### Session 2 远程仓库

+ 远程仓库SSH部分创建，目录在`C:\Users\用户名\.ssh`

 创建SSH Key。在用户主目录下，看看有没有.ssh目录，如果有，再看看这个目录下有没有`id_rsa`和`id_rsa.pub`这两个文件，如果已经有了，可直接跳到下一步。如果没有，打开Shell，建SSH Key：

 ```
 $ ssh-keygen -t rsa -C "youremail@example.com"
 ```


+ 添加远程库

  + 在github上新建一个仓库：repository name

  + 本地仓库下输入：$ git remote add origin git@github.com:user name/repository name.git


  + git push -u origin 分支名 (第一次提交) （一般为main）

    > 添加后，远程库的名字就是`origin`，这是Git默认的叫法，也可以改成别的，但是`origin`这个名字一看就知道是远程库。由于远程库是空的，我们第一次推送`main`分支时，加上了`-u`参数，Git不但会把本地的`main`分支内容推送的远程新的`main`分支，还会把本地的`main`分支和远程的`main`分支关联起来，在以后的推送或者拉取时就可以简化命令。

  + git push origin 分支名//git push origin（提交当前分支）


+ git remote rm repo_name 删除远程仓库

+ git remote -v 查看远程库信息

  + `git remote rm <name>`命令。使用前，建议先用`git remote -v`查看远程库信息：

    上面显示了可以抓取和推送的`origin`的地址。如果没有推送权限，就看不到push的地址。

    ```shell
    $ git remote -v
    origin  git@github.com:michaelliao/learn-git.git (fetch)
    origin  git@github.com:michaelliao/learn-git.git (push)
    ```

    然后，根据名字删除，比如删除`origin`：

    ```shell
    $ git remote rm origin
    ```

    此处的“删除”其实是解除了本地和远程的绑定关系，并不是物理上删除了远程库。远程库本身并没有任何改动。要真正删除远程库，需要登录到GitHub，在后台页面找到删除按钮再删除


+ git clone: 克隆远程仓库到本地

 ```shell
 $ git clone git@github.com:username/repositoryname.git
 ```

+ git checkout -b branch-name origin/branch-name：创建本地和远程相对应的分支

---

#### Session 3 分支管理

+ git checkout -b <branchname>表示创建一个新分支并且切换到该分支 

  + `git checkout`命令加上`-b`参数表示创建并切换，相当于以下两条命令：

    ```shell
    $ git branch dev
    $ git checkout dev
    #Switched to branch 'dev'
    ```

+ git branch 查看当前分支，带*号标明为当前分支\

+ git branch -vv: 查看当前分支与远程分支之间的关系

+ git checkout <branch name>表示切换到该分支 //git switch master

+ git branch -d 分支name表示删除该分支

  + git branch -D <name>强行删除一个分支，比如该分支还没有合并但需要删除时

+ git merge 分支name表示将该分支合并当当前分支

+ 解决冲突

  + PS: git status查看冲突文件

  > 冲突产生的reason: 在其他branch上修改并且提交后，在main分支上再次修改添加提交，使得两者不一致。如果不在main分支上修改，则可以直接将其他branch合并到main分支上。

  + 解决方法：
    + 手动修改冲突的文件并添加提交，再进行merge

+ git merge --no-ff 分支名表示禁止`	Fast forward`模式，即**删除分支后不会丢失分支信息，很重要**。合并时候也可以创建一个新的commit。

  ```shell
   $ git merge --no-ff -m "merge with no-ff" dev
  ```

+ 分支管理策略

 在实际开发中，我们应该按照几个基本原则进行分支管理：

> 首先，`master`分支应该是非常稳定的，也就是仅用来发布新版本，平时不能在上面干活；
>
> 那在哪干活呢？干活都在`dev`分支上，也就是说，`dev`分支是不稳定的，到某个时候，比如1.0版本发布时，再把`dev`分支合并到`master`上，在`master`分支发布1.0版本；


+ git stash的使用
  + [git-stash用法小结 - Tocy - 博客园 (cnblogs.com)](https://www.cnblogs.com/tocy/p/git-stash-reference.html)
  + git stash/git stash save"branch_name"
  + git stash list
  + git stash apply / git stash pop
  + git stash drop

+ git cherry-pick 

  + delete分支的时候会显示该分支做的提交的版本号，或者用git reflog查看版本号

    > 为了方便操作，Git专门提供了一个`cherry-pick`命令，让我们能复制一个特定的提交到当前分支：方便重复之前提交过的修复

    ```shell
    $ git branch
    * dev
      master
    $ git cherry-pick 4c805e2
    [master 1d4b803] fix bug 101
     1 file changed, 1 insertion(+), 1 deletion(-)
    ```

修复bug时，我们会通过创建新的bug分支进行修复，然后合并，最后删除；

当手头工作没有完成时，先把工作现场`git stash`一下，然后去修复bug，修复后，再`git stash pop`，回到工作现场；在master分支上修复的bug，想要合并到当前dev分支，可以用`git cherry-pick <commit>`命令，把bug提交的修改“复制”到当前分支，避免重复劳动。

+ git pull处理冲突

> 推送失败，因为你的小伙伴的最新提交和你试图推送的提交有冲突，解决办法也很简单，Git已经提示我们，先用`git pull`把最新的提交从`origin/dev`抓下来，然后，在本地合并，解决冲突，再推送：

> `git pull`也失败了，原因是没有指定本地`dev`分支与远程`origin/dev`分支的链接，根据提示，设置`dev`和`origin/dev`的链接：
>
> ```
> $ git branch --set-upstream-to=origin/dev dev
> Branch 'dev' set up to track remote branch 'dev' from 'origin'.
> ```
>
> 再pull：
>
> ```
> $ git pull
> Auto-merging env.txt
> CONFLICT (add/add): Merge conflict in env.txt
> Automatic merge failed; fix conflicts and then commit the result.
> ```

**如果合并时发生冲突那我们要先进入vi 模式解决冲突。**

> 因此，多人协作的工作模式通常是这样：
>
> 1. 首先，可以试图用`git push origin <branch-name>`推送自己的修改；
> 2. 如果推送失败，则因为远程分支比你的本地更新，需要先用`git pull`试图合并；
> 3. 如果合并有冲突，则解决冲突，并在本地提交；
> 4. 没有冲突或者解决掉冲突后，再用`git push origin <branch-name>`推送就能成功！
>
> 如果`git pull`提示`no tracking information`，则说明本地分支和远程分支的链接关系没有创建，用命令`git branch --set-upstream-to <branch-name> origin/<branch-name>`。
>
> 这就是多人协作的工作模式，一旦熟悉了，就非常简单。
>

+ `git rebase -i main`：

  + 把本地未push的某个分支的分叉提交历史整理成一条直线(移动到分叉主路径的某个更新的提交)，看上去更直观。缺点是本地的分叉提交已经被修改过了。

  + rebase的目的是使得我们在查看历史提交的变化时更容易，因为分叉的提交需要三方对比。

---

#### Session 4 标签管理

+ git tag <tagname>创建一个新的标签到当前版本，也可加上commit id

+ git tag -a <tagname> -m"informaion_tag"

+ git tag 查看所有标签
+ git show<tagname> 可以查看标签
+ git tag -d tagname: 删除本地标签
+ 标签和commit挂钩如果这个commit既出现在master分支，又出现在dev分支，那么在这两个分支上都可以看到这个标签。

+ git push origin :refs/tags/<tagname>用于删除远程标签

+ git push origin <tagname>用于推送某个标签到远程库

+ git push origin --tags 一次性推送全部未推送过的标签到远程库



#### 其他

**1、git clone错误指南**

一般来说，挂上代理应该就没问题了

+ Failed to connect to github.com port 443: Timed out
  解决方法：刷新DNS缓存，ipconfig /flushdns

+ OpenSSL SSL_read: Connection was reset, errno 10054
  解决方法：忽略ssl验证，git config --global http.sslVerify "false"

**2、shell中的含空格路径处理**

+ 如果路径上有存在空格名称的路径，需要使用单引号
  比如：`E:/‘Git Repository’/Tesontainer `
+ Git上的路径分隔符是正斜杠`/`

**3、当一个Git仓库嵌套在另一个Git仓库中时clone**

此时在进行clone本仓库后，需要执行`git submodule update --init --recursive`, 作用是递归更新所有子模块，如果子模块不存在则进行初始化。
