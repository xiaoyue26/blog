title: pyqt绘图库
date: 2014-12-23 17:52:55
tags:
- 配置
categories:
- 配置

---


先记录下ubuntu下pygal的安装命令:
```shell
sudo pip install pygal
sudo pip install cssselect
sudo pip install tinycss
sudo apt-get update
sudo apt-get install libxml2 libxml2-dev
sudo apt-get install python-libxml2 python-libxslt1
sudo apt-get install python2.7-dev
sudo apt-get install idle 
sudo apt-get install libxslt-dev  
sudo apt-get install libxml2 libxml2-dev
sudo pip install lxml 
sudo apt-get install python-lxml
sudo apt-get install libffi-dev 
sudo pip install cairosvg
sudo pip install cssselect 
```


pycharm下配置Pyqt4:
1. 
```
sudo apt-get install python-pip
sudo apt-get install build-essential  
```

2. pyqt4:
```
sudo apt-get install libxext6 libxext-dev libqt4-dev libqt4-gui libqt4-sql qt4-dev-tools qt4-doc qt4-designer qt4-qtconfig "python-qt4-*" python-qt4
```

3.安装pycharm(官网下载)专业版支持django
解压到program下(用VNC图形界面下操作)
用户名：
`EMBRACE`
密钥：
```
14203-12042010
0000107Iq75C621P7X1SFnpJDivKnX
6zcwYOYaGK3euO3ehd1MiTT"2!Jny8
bff9VcTSJk7sRDLqKRVz1XGKbMqw3G
```
脚本默认启动目录：`/usr/local/bin/charm`

4.
```
apt-get install python-lxml
pip install lxml
apt-get install libffi-dev
pip install cairosvg
pip install tinycss
pip install cssselect
pip install pygal
```
5.`pycharm+pyqt4`配置：
http://www.jb51.net/softjc/127737.html
```
sudo apt-get install qdevelop 
sudo apt-get install libqwt5-qt4 libqwt5-qt4-dev
sudo apt-get install libqt4-sql-mysql
sudo apt-get install pyqt4-dev-tools
```
6.sip安装：
`pip install sip`不成功的话：
`easy_install sip`
估计还是不行 绿色安装吧：
http://www.riverbankcomputing.com/software/sip/download
解压缩，进文件夹然后：
`python configure.py`
检查`sip -V`是否成功。

7.用ubuntu的软件中心安装软件cx_Freeze失败所以：
```
sudo pip install cx_Freeze
```
8. pycharm连接qtdesigner:
pycharm中：
`File->settings->Tools->external tools`
'+'增加一个工具，
Program选择qtdesigner的运行路径；
`Working directory`中输入`$ProjectFileDir$`

9. pycharm连接pyuic:
先安装：
```
sudo apt-get install pyqt4-dev-tools
```
测试pyuic命令能否运行。
```
pyuic
```
pycharm中：
`File->settings->Tools->external tools`
'+'增加一个工具，
Program输入：
`pyuic4  `
参数输入：
`-x $FileName$ -o $FileNameWithoutExtension$.py`
`Working directory`中输入:
`$ProjectFileDir$`


示例命令:
```
pyuic4 -x untitled.ui -o untitled.py
```


三种绘制表格的库: 

1.第一种pygal:
http://pygal.org/
```
apt-get install python-lxml
pip install lxml
pip install cairosvg
pip install tinycss
pip install cssselect
pip install pygal
```

2. 第二种matplotlib:
```
apt-get install python-numpy
apt-get install python-scipy
apt-get install libpng-dev
apt-get install python-matplotlib
pip install matplotlib
```

3. 第三种pycha:
```
pip install pycha
```

