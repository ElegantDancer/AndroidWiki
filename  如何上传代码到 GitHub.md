##  如何上传代码到 GitHub

### 1. 直接用 git clone 

可以直接在 GitHub 上新建一个仓库，然后 clone 到本地，再把文件放到这个仓库里面。之后 git add 、git commit 、git push 即可将文件传送到 GitHub 仓库。



### 2. git remote

#### 2.1 如果待上传的文件还不是 git 仓库

~~~java
git init  //初始化 git
git add somefile
git commit -m"commit message"
git remote add origin https://github.com/用户名/仓库名.git
git push -u origin master
~~~

`git remote add origin https://github.com/用户名/仓库名.git`添加后，远程库的名字就是`origin`，这是 Git 默认的叫法，也可以改成别的，但是`origin`这个名字一看就知道是远程库。

`git push -u origin master`由于远程库是空的，我们第一次推送`master`分支时，加上了`-u`参数，Git不但会把本地的`master`分支内容推送的远程新的`master`分支，还会把本地的`master`分支和远程的`master`分支关联起来，在以后的推送或者拉取时就可以简化命令。

#### 2.2 如果待上传的文件已经是 git 仓库了

~~~java
git remote add origin https://github.com/用户名/仓库名.git
git push -u origin master
~~~

#### 2.3 小结

- 1. 要关联一个远程库，使用命令`git remote add origin git@server-name:path/repo-name.git`；
- 2. 关联后，使用命令`git push -u origin master`第一次推送master分支的所有内容；
- 3. 此后，每次本地提交后，只要有必要，就可以使用命令`git push origin master`推送最新修改；

![](https://ww3.sinaimg.cn/large/006tKfTcgy1ff1dcrwm3kj30r40d2taf.jpg)



### 3. 如何删除一个 GitHub 仓库

[如何删除 GitHub 仓库](http://blog.csdn.net/deng0zhaotai/article/details/38535251)