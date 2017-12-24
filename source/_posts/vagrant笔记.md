title: vagrant笔记
date: 2015-07-06 19:29:02
tags:
- 配置
categories:
- 配置

---

##操作环境为win7物理机vmware中的Ubuntu12.04.5 amd64，客户机自带virtural box.
1.直接安装：
```shell
sudo apt-get install ruby rubygems
sudo apt-get install vagrant#1.x版本
```
或者官网下载最新版然后安装：
```shell
sudo dpkg -i vagrant_1.7.4_x86_64.deb
```
从vagrant的[box列表][1]下载一个base box，例如presice32.box。
（由于笔者的虚拟机没法再次虚拟化`64`位的系统,所以选用了`32`位box。猜测应该是没有打开cpu或者bios的虚拟化设置。）
首先移动到工作目录，如`~/vagrant_workplace/boxes/ubuntu12.04`。
##从本地添加一个box,并命名为ubuntu12.04：
`vagrant box add ubuntu12.04 ./precise32.box`
##初始化box:
`vagrant init ubuntu12.04`
##启动虚拟机box:
`vagrant up`
##ssh登录到box:
`vagrant ssh`
##重启：
`vagrant reload`
##打包分发 以虚拟机ubuntu12.04为Base创建myubuntu32.box：
`vagrant package –output myubuntu32.box –base ubuntu12.04`
##关闭虚拟机
`vagrant halt`
##销毁虚拟机
`vagrant destroy` 
##移除添加的box:
`vagrant box remove ubuntu12.04`



box下载列表：
http://www.vagrantbox.es/
https://atlas.hashicorp.com/boxes/search?utm_source=vagrantcloud.com&vagrantcloud=1

下一步学stash
  [1]: http://www.vagrantbox.es/
