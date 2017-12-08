---
title: how to encrypt in iOS and decrypt in .net?
date: 2017-12-08 15:03:38
tags:
---

# 如何用 iOS 加密以及用 .net 解密（反之亦然）

## RSA
### 1. iOS RSA 加密/解密
参考[此处代码](https://github.com/ideawu/Objective-C-RSA)

### 2. 公钥/秘钥格式转换
iOS 采用 PEM 格式，.net 采用 xml 格式。 怎么做转换呢？很简单，在这个[网站](https://superdry.apphb.com/tools/online-rsa-key-converter)提供了在线转换。

### 3. .net RSA 加密/解密
采用这个[代码段](https://gist.github.com/simazhao/df3aa962c2b545736fb1fd88a1db3e05)