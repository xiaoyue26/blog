title: tesseract_ocr_centos6.5
date: 2015-9-27 12:55:02
tags:
- 配置
categories:
- 配置

---


 1.安装依赖：
```shell
sudo yum -y groupinstall "Development Tools"
sudo yum -y gcc gcc-c++ kernel-devel make
sudo yum -y install libpng-devel.x86_64
sudo yum -y install libjpeg-devel.x86_64
sudo yum -y install libtiff-devel.x86_64
sudo yum -y install zlib-devel.x86_64
```
2.下载leptonica和tesseract：

> https://code.google.com/p/leptonica/downloads/list
>https://code.google.com/p/tesseract-ocr/downloads/list
>http://www.leptonica.org/download.html
>https://code.google.com/p/tesseract-ocr/

---
3.解压下载好的压缩包：
`tesseract-ocr-3.02.02.tar.gz`
`leptonica-1.72.tar.gz`
`tesseract-ocr-3.02.eng.tar.gz`
4.进入`leptonica`目录编译安装：
```shell
./configure
make 
sudo make install
```
5.进入`tesseract-ocr`目录编译安装：
```shell
./autogen.sh
./configure
make 
sudo make install
sudo ldconfig
```
6.解压语言包`tesseract-ocr-3.02.eng.tar.gz`，
解压后将`tesseract-ocr/tessdata` 下的所有文件全部拷贝到 `/usr/local/share/tessdata` 下。

7.测试安装成功：
进入`tesseract-ocr`目录：
```shell
tesseract phototest.tif phototest -l eng
```
若生成phototest.txt,且其中内容与phototest.tif相同则成功。


