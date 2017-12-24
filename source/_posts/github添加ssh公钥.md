title: github添加ssh公钥
date: 2013-09-28 10:41:52
tags:
- 配置
categories:
- 配置

---


> 可能遇到的错误：
> https://help.github.com/articles/error-permission-denied-publickey/

##原理：

> 本机生成当前电脑操作用户的公钥，然后把生成的.pub文件（也就是公钥），上传到github的账户里的ssh
> key那一项里，让自己的github账户同意这台机子的这个操作账户远程登录上传东西。

**具体做法：**
1. 配置自己的git账户：
```shell
git config --global user.name "xiaoyue26"
git config --global user.email "296671657@qq.com"
ssh-keygen -t rsa -C "296671657@qq.com"
cat ~/.ssh/id_rsa.pub
```
把屏幕输出的内容复制下来。

2. 打开github，登录https://github.com/settings/ssh，
其实就是进入自己的账户，在Personal settings里找到SSH keys,然后Add ssh key.
随便取个名字，把刚才cat的内容输入进去就好了。

3. 测试添加成功：
```shell
ssh -T git@github.com
```
此时应该不要输入密码了。
注意如果要用在sudo,也就是root用户执行操作时，生成密钥时也要加sudo,才会生成root用户的密钥。


