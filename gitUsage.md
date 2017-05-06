# git使用笔记

## git ssh 无秘钥访问（for Mac）

通过以下两个命令确保系统中安装了Git和ssh；

```Shell
git --version
ssh -V
```

然后在执行：

```shell
ssh-keygen -t rsa -C "yourEmail@email.com"
```

这个命令会在系统的~/.ssh/中生成id_rsa,id_rsa.pub;

其中id_rsa为密钥对中的私钥，需妥善保管；id_rsa.pub为密钥对中的公钥，可任意公开。

然后将.pub的内容添加到git -> settings -> ssh keys;

在系统中执行：

```shell
ssh -T git@github.com
```

执行成功后打印如下的log：

Hi \<UserName\>! You've successfully authenticated, but GitHub does not provide shell access.



## 在github中创建自己的repo

在github中点击+号，添加自己的repo，

然后在本地的文件夹中执行下面的命令：

```shell
git init
git remote add original "http://xx"
git pull original master
```

这样就可以将创建的repo拿到本地；当我们本地修改之后，可以奖本地的东西提交上去；执行代码如下：

```shell
git add test.md
git commit -m "just for test"
git push
```

## 开始自己的操作

1. 创建并切换到自己的分支上

```Shell
git branch 'branchName'
git checkout 'branchName'
```



