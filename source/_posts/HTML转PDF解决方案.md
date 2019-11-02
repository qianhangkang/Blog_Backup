---
title: HTML转PDF解决方案
date: 2019-11-02 14:36:52
tags:
---

前阵子公司有个小需求，将HTML转成PDF。前期做了调研，虽然最后没用到，还是留个记录吧

<!--more-->

## 第三方服务

### PDFSHIFT

https://pdfshift.io/?tab=java

优点

1. 非常简单，以POST的方式就能将指定url生成为pdf
2. 转换效果非常棒

缺点

1. 每个月每个免费的KEY只能转化250次，之后需要收费
2. 转化的pdf大小要在1M之内
3. 网上资料几乎没有，只有官方文档

### Convertio

https://convertio.co/zh/

优点

1. 简单，也是以POST请求的方式
2. 文档丰富 https://developers.convertio.co/api/docs/

缺点

1. 没有PDFSHIFT转换的效果好
2. 免费的版本有转换限制：a daily limit of 25 conversion minutes.

**同一页面的转换对比：**

PDFSHIFT

![image-20190124110502754](https://cnhkblog.top/Users/qianhangkang/Library/Application%20Support/typora-user-images/image-20190124110502754.png)

Convertio

![image-20190124110610554](https://cnhkblog.top/Users/qianhangkang/Library/Application%20Support/typora-user-images/image-20190124110610554.png)

## 开源解决方案

### Flying-Saucer

https://github.com/flyingsaucerproject/flyingsaucer

优势

1. 一个解决方案

缺点

1. 对HTML格式要求严格，每个标签必须有对应的结束标签
2. 中文字体需要手动加载相应字体

### wkhtmltopdf

https://wkhtmltopdf.org/downloads.html

使用C写的一个库

优点

1. 转换效果非常棒
2. 文档丰富、功能齐全、支持中文
3. 支持多平台

缺点

1. 在java中使用只能调用相应的命令行
2. 服务器上可能无法安装对应二进制程序

### WeasyPrint

https://weasyprint.org/

使用python写的库

优点

1. 转换效果一般

缺点

1. 在java中使用只能调用相应的命令行
2. 服务器上可能无法安装对应二进制程序

### ITEXT7

老牌解决方案

缺点

1. 商业使用需要收费
2. 中文支持需要手动导入

### **pdf2htmlEX**

















> *大部分正义感都是虚伪的 聊胜于无   —————————————————————————沃德彭尤·胡某*

