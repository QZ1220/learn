# git命令行学习笔记 #
## 一、创建本地项目和remote仓库的连接 ##
### 添加之前可以先使用如下命令查看一下目前remote的状况： ###
    $ git remote -v
    origin  http://wangquanzhou:Zhaoxxx@code.cmschina.com.cn/glyyyfwkfz/Customer-Relationship-Management-System---Server.git (fetch)
    origin  http://wangquanzhou:Zhaoxxx@code.cmschina.com.cn/glyyyfwkfz/Customer-Relationship-Management-System---Server.git (push)
### 因为我之前已经添加了remote，所以这里会看到origin ###

### 可以使用如下命令删除remote ###
    $ git remote rm origin

### 下面开始添加remote，http方式，@符号之前的用户名和密码可以免去每次提交的时候再手动输入用户名和密码 ###
    $ git remote add origin http://wangquanzhou:Zhaoxxx@code.cmschina.com.cn/glyyyfwkfz/Customer-Relationship-Management-System---Server.git

### 添加好remote以后，第一次push代码到remote，需要使用如下命令： ###
    $ git push -u origin master
### 加上了-u参数，Git不但会把本地的master分支内容推送的远程新的master分支，还会把本地的master分支和远程的master分支关联起来，在以后的推送或者拉取时就可以简化命令。也就是如下： ###
    $ git push origin master

### 如果要从remote拉取代码，直接使用如下命令 ###
    $ git pull
    From http://code.cmschina.com.cn/glyyyfwkfz/Customer-Relationship-Management-System---Server
     * [new branch]  master -> origin/master
    There is no tracking information for the current branch.
    Please specify which branch you want to merge with.
    See git-pull(1) for details.
    git pull <remote> <branch>
    If you wish to set tracking information for this branch you can do so with:
    git branch --set-upstream-to=origin/<branch> master

## 二、添加本地更改到本地仓库并推送到remote ##
### 将更改添加到本地仓库，这里我们新增了一个test.Java文件，./的意思是当前目录下所有有更改的文件都会add到本地仓库，并使用git status命令查看目前本地仓库状态 ###
    $ git add ./
    warning: LF will be replaced by CRLF in test.java.
    The file will have its original line endings in your working directory.
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git status
    On branch master
    Changes to be committed:
      (use "git reset HEAD <file>..." to unstage)
    modified:   test.java

add以后，还需要将更改commit到本地仓库
        Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git commit -m "添加test.java文件"
    [master c33986b] 添加test.java文件

### 最后就是push操作将本地所有的修改同步到remote生效，origin就是我们第一步添加的remote连接，master表示本地推送时所处的branch分支 ###
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git push origin master

## 三、分支管理 ##
### GitHub上想要在master下新建一个分支的话。可以直接在本地新建一个dev分支，然后直接推送到origin，相应的origin也会产生一个dev分支，这样就完成在remote创建新分支的过程。
### 查看分支状态，表明当前我只有一个master分支 ###
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git branch
    * master

### 一般我们不直接在master分支修改代码，而是新建一个类似名为dev的分支来修改代码，然后在dev分支add和commit所做的修改，然后和master分支合并，然后再push到remote ###
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git branch
    * master
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git checkout -b dev
    Switched to a new branch 'dev'
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (dev)
    $ git branch
    * dev
      master
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (dev)
    $ ls
    cpack.dat  fop/  lib/  pager-taglib.tld  README.md  server-config.wsdd  src/  test.java  test2.java  web.xml  weblogic.xml  WebRoot/
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (dev)
    $ vi test.java
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (dev)
    $ git add ./
    warning: LF will be replaced by CRLF in test.java.
    The file will have its original line endings in your working directory.
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (dev)
    $ git commit -m "dev分支修改了test.java文件"
    [dev 3ecd5f5] dev分支修改了test.java文件
     1 file changed, 1 insertion(+)
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (dev)
    $ git branch
    * dev
      master
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (dev)
    $ git checkout master
    Switched to branch 'master'
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ cat test.java
    hello world
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git merge dev
    Updating c33986b..3ecd5f5
    Fast-forward
     test.java | 1 +
     1 file changed, 1 insertion(+)
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ cat test.java
    hello world
    this is dev branch
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git branch -d dev
    Deleted branch dev (was 3ecd5f5).
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git branch
    * master
### 上面代码注意的是，我们一定要在dev分支将所做的修改add并且commit以后，再删除dev分支，否则一切在dev分支的修改都不会同步到master分支。
### 合并dev分支的时候可以使用--no-ff参数，表示禁用Fast forward，这样merge操作就可以被记录下来否则merge操作就没有记录 ###
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git log --graph --pretty=oneline --abbrev-commit
    * 3ecd5f5 (HEAD -> master) dev分支修改了test.java文件//这里也做过merge，但是没有任何记录
    * c33986b 添加test.java文件
    * 39af982 test2
    * e744267 hhh
    * 3eb1ede (origin/master) Init Commit
    * 059ab5f first commit
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git branch
    * master
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git checkout -b dev
    Switched to a new branch 'dev'
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (dev)
    $ vi test.java
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (dev)
    $ git add ./
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (dev)
    $ git commit -m "commit with --no-ff"
    [dev 972dc0f] commit with --no-ff
     1 file changed, 1 insertion(+)
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (dev)
    $ git checkout master
    Switched to branch 'master'
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git merge --no-ff -m "merge with no-ff" dev
    Merge made by the 'recursive' strategy.
     test.java | 1 +
     1 file changed, 1 insertion(+)
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ cat test.java
    hello world
    this is dev branch
    merge with --no-ff
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git log --graph --pretty=oneline --abbrev-commit
    *   4aab4e1 (HEAD -> master) merge with no-ff
    |\
    | * 972dc0f (dev) commit with --no-ff//这里就有合并的记录了
    |/
    * 3ecd5f5 dev分支修改了test.java文件
    * c33986b 添加test.java文件
    * 39af982 test2
    * e744267 hhh
    * 3eb1ede (origin/master) Init Commit
    * 059ab5f first commit
    
### 如果使用上面分支管理仍然由于某些原因发现，两个分支存在差异，可以使用如下命令将，例如使用dev分支的文件覆盖master分支的文件或者文件夹，下面的lib文件夹也可以换成一个文件名 ###
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git checkout dev -- lib
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git status
    On branch master
    Changes to be committed:
      (use "git reset HEAD <file>..." to unstage)
    new file:   lib/PDFSignAsian.jar
    new file:   lib/PDFSignPro.jar
    new file:   lib/bcpkix-jdk15on-152.jar
    new file:   lib/bcprov-jdk15on-152.jar
    new file:   lib/poi-3.0.2-FINAL-20080204.jar
    new file:   lib/poi-contrib-3.0.2-FINAL-20080204.jar

### 关于远程分支与本地分支的对应关系：
### 1、如果远程分支不存在，而本地分支存在，例如本地有分支feature，远程没有，那么直接使用如下命令，就可以实现在远程建立一个feature分支，并且远程分支feature和本地feature分支是有对应关系的 ###
    Administrator@HP MINGW64 /d/test/Customer-Relationship-Management-System---Server (master)
    $ git checkout -b feature
    Switched to a new branch 'feature'
    Administrator@HP MINGW64 /d/test/Customer-Relationship-Management-System---Server (feature)
    $ git branch
      dev
    * feature
      master
    Administrator@HP MINGW64 /d/test/Customer-Relationship-Management-System---Server (feature)
    $ git remote -v
    origin  http://code.cmschina.com.cn/glyyyfwkfz/Customer-Relationship-Management-System---Server.git (fetch)
    origin  http://code.cmschina.com.cn/glyyyfwkfz/Customer-Relationship-Management-System---Server.git (push)
    Administrator@HP MINGW64 /d/test/Customer-Relationship-Management-System---Server (feature)
    $ git push origin feature
    Total 0 (delta 0), reused 0 (delta 0)
    To http://code.cmschina.com.cn/glyyyfwkfz/Customer-Relationship-Management-System---Server.git
     * [new branch]  feature -> feature

### 2、如果远程分支已经存在，本地分支不存在，那么可以使用如下命令来建立对应关系 ###
    Administrator@HP MINGW64 /d/test/Customer-Relationship-Management-System---Server (master)
    $ git checkout -b feature origin/feature
    Switched to a new branch 'feature'
    Branch feature set up to track remote branch feature from origin.

### 3、如果远程分支已经存在，本地分支也存在，但是二者没有对应关系，那么可以使用如下命令来建立对应关系 ###
    Administrator@HP MINGW64 /d/test/Customer-Relationship-Management-System---Server (dev)
    $ git branch --set-upstream-to=origin/dev dev
    Branch dev set up to track remote branch dev from origin.

### 4、远程分支的删除，参考如下链接 ###
#### https://blog.zengrong.net/post/1746.html ####
## 三、多人协作开发 ##
### 多人协作的工作模式通常是这样： ###

### 	1. 首先，可以试图用git push origin branch-name推送自己的修改；
### 	2. 如果推送失败，则因为远程分支比你的本地更新，需要先用git pull试图合并；
### 	3. 如果合并有冲突，则解决冲突，并在本地提交;###
###  4.解决冲突的办法：  ###
#### 一般出错信息如下所示： ####
    error: Your local changes to the following files would be overwritten by merge:
    protected/config/main.php
    Please, commit your changes or stash them before you can merge.
#### 如果希望保留生产服务器上所做的改动,仅仅并入新配置项, 处理方法如下(依次执行下列命令): ####
    git stash
    git pull
    git stash pop
###5. 没有冲突或者解决掉冲突后，再用git push origin branch-name推送就能成功！ ###

### 如果git pull提示“no tracking information”，则说明本地分支和远程分支的链接关系没有创建，用命令 ###
    git branch --set-upstream branch-name origin/branch-name。
## 四、版本回退和撤销修改 ##
### 如果发现文件更改有问题，那么可以进行版本回退 ###
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git reflog
    4aab4e1 (HEAD -> master) HEAD@{0}: merge dev: Merge made by the 'recursive' strategy.
    3ecd5f5 HEAD@{1}: checkout: moving from dev to master
    972dc0f (dev) HEAD@{2}: commit: commit with --no-ff
    3ecd5f5 HEAD@{3}: checkout: moving from master to dev
    3ecd5f5 HEAD@{4}: merge dev: Fast-forward
    c33986b HEAD@{5}: checkout: moving from dev to master
    3ecd5f5 HEAD@{6}: commit: dev分支修改了test.java文件
    c33986b HEAD@{7}: checkout: moving from master to dev
    c33986b HEAD@{8}: commit: 添加test.java文件
    39af982 HEAD@{9}: commit: test2
    e744267 HEAD@{10}: commit: hhh
    3eb1ede (origin/master) HEAD@{11}: reset: moving to 3eb1ede
    431f4d3 HEAD@{12}: reset: moving to 431f4d3
    431f4d3 HEAD@{13}: reset: moving to 431f4d3
    431f4d3 HEAD@{14}: commit: 测试
    3eb1ede (origin/master) HEAD@{15}: reset: moving to 3eb1ede
    d7a2fd3 HEAD@{16}: commit: 删除原CRMserver端老版本的代码
    3eb1ede (origin/master) HEAD@{17}: clone: from http://code.cmschina.com.cn/glyyyfwkfz/Customer-Relationship-Management-System---Server.git
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git reset --hard 3ecd5f5
    HEAD is now at 3ecd5f5 dev分支修改了test.java文件

### 命令git checkout -- readme.txt意思就是，把readme.txt文件在工作区的修改全部撤销，这里有两种情况：###
### 一种是readme.txt自修改后还没有被放到暂存区，现在，撤销修改就回到和版本库一模一样的状态； ###
### 一种是readme.txt已经添加到暂存区后，又作了修改，现在，撤销修改就回到添加到暂存区后的状态。 ###
### 总之，就是让这个文件回到最近一次git commit或git add时的状态。 ###
### 那如果修改的文件以及add到了暂存区了，怎么回退？  ###
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ cat test.java
    hello world
    this is dev branch
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git status
    On branch master
    nothing to commit, working tree clean
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ vi test.java
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git status
    On branch master
    Changes not staged for commit:
      (use "git add <file>..." to update what will be committed)
      (use "git checkout -- <file>..." to discard changes in working directory)
    modified:   test.java
    no changes added to commit (use "git add" and/or "git commit -a")
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git add ./
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git status
    On branch master
    Changes to be committed:
      (use "git reset HEAD <file>..." to unstage)
    modified:   test.java
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git reset HEAD test.java
    Unstaged changes after reset:
    M   test.java
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git status
    On branch master
    Changes not staged for commit:
      (use "git add <file>..." to update what will be committed)
      (use "git checkout -- <file>..." to discard changes in working directory)
    modified:   test.java
    no changes added to commit (use "git add" and/or "git commit -a")
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ cat test.java
    hello world
    this is dev branch
    rollback
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git checkout -- test.java
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ cat test.java
    hello world
    this is dev branch

### 更进一步的，如果你add了，还commit了，那么就需要进行版本回退了。 ###

## 四、标签管理 ##
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git tag
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git tag v1.0.0
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git tag
    v1.0.0
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git branch
    * master
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git reflog
    77f570c (HEAD -> master, tag: v1.0.0) HEAD@{0}: merge dev: Merge made by the 'recursive' strategy.
    3ecd5f5 HEAD@{1}: checkout: moving from dev to master
    972dc0f HEAD@{2}: checkout: moving from master to dev
    3ecd5f5 HEAD@{3}: reset: moving to 3ecd5f5
    4aab4e1 HEAD@{4}: merge dev: Merge made by the 'recursive' strategy.
    3ecd5f5 HEAD@{5}: checkout: moving from dev to master
    972dc0f HEAD@{6}: commit: commit with --no-ff
    3ecd5f5 HEAD@{7}: checkout: moving from master to dev
    3ecd5f5 HEAD@{8}: merge dev: Fast-forward
    c33986b HEAD@{9}: checkout: moving from dev to master
    3ecd5f5 HEAD@{10}: commit: dev分支修改了test.java文件
    c33986b HEAD@{11}: checkout: moving from master to dev
    c33986b HEAD@{12}: commit: 添加test.java文件
    39af982 HEAD@{13}: commit: test2
    e744267 HEAD@{14}: commit: hhh
    3eb1ede (origin/master) HEAD@{15}: reset: moving to 3eb1ede
    431f4d3 HEAD@{16}: reset: moving to 431f4d3
    431f4d3 HEAD@{17}: reset: moving to 431f4d3
    431f4d3 HEAD@{18}: commit: 测试
    3eb1ede (origin/master) HEAD@{19}: reset: moving to 3eb1ede
    d7a2fd3 HEAD@{20}: commit: 删除原CRMserver端老版本的代码
    3eb1ede (origin/master) HEAD@{21}: clone: from http://code.cmschina.com.cn/glyyyfwkfz/Customer-Relationship-Management-System---Server.git
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git tag v0.0.5 431f4d3
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git tag
    v0.0.5
    v1.0.0
### 显示tag的具体信息 ###
    Administrator@HP MINGW64 /d/workspaceJDK7/ksdomain_prd/Customer-Relationship-Management-System---Server (master)
    $ git show v0.0.5
    commit 431f4d3020b533c478fe21d53dc02cf801bcf58f (tag: v0.0.5)
    Author: wangquanzhou <wangquanzhou@cmschina.com.cn>
    Date:   Sun Oct 15 14:05:02 2017 +0800
    测试
    diff --git a/test.java b/test.java
    new file mode 100644
    index 0000000..9f00003
    --- /dev/null
    +++ b/test.java
    @@ -0,0 +1,2 @@
    +
    +hello world

### 推送tag ###
    $ git push origin v1.0
### 或者，一次性推送全部尚未推送到远程的本地标签： ###
    $ git push origin --tags
### 如果标签已经推送到远程，要删除远程标签就麻烦一点，先从本地删除： ###
    $ git tag -d v0.9Deleted tag 'v0.9' (was 6224937)
### 然后，从远程删除。删除命令也是push，但是格式如下： ###
    $ git push origin :refs/tags/v0.9
    To git@github.com:michaelliao/learngit.git
    - [deleted] v0.9


## 五、设置代理 ##
### 添加http和https代理 ###
    git config --global https.proxy http://127.0.0.1:1080
    git config --global https.proxy https://127.0.0.1:1080
### 取消代理 ###
    git config --global --unset http.proxy
    git config --global --unset https.proxy

## 六、解决ignore文件不生效的问题 ##
    git rm -r --cached .
    git add .
    git commit -m 'update .gitignore'

## 七、关于git clone命令 ##
    git clone <remote-addr:repo.git> -b <branch-or-tag-or-commit>
### 实际上 clone 回来是包含了该 branch 完整历史的，所以仍然会有比较多的文件传输。###
### 你可以使用 github 的打包下载功能，tag 可以通过 release 页面找到连接，也可以直接替换 tag 为任意的branch 名来下载。  ###

![](http://image.beekka.com/blog/2014/bg2014061202.jpg)

## 八、git rebase ##

### 这个命令相当的cool,你对当前分支所作的任何改变都被保存到一个临时区域，因此你的分支将会和改变之前一样干净。如果你用git pull -rebase,git将会获取远程的改变，遍历当前本地分支，然后替换你当前分支的所有改动。
### 其实也就是**放弃**本地更改 ###