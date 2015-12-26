title:  "将网站打造为https"
date: 2015-05-05
updated: 2015-12-26
categories: 
- 后端
tags:
- security
- tls
- ssl
---

> 为什么需要https，以及如何迁移到https

## 引言

最近看到一个系列好文 [百度全站https实践](http://op.baidu.com/2015/04/https-index/)，其中描述了百度全站port到https。
主要的好处是：
- 防止ISP劫持
- 保护用户信息
- https的访问方式是未来的大趋势，http/2默认是使用这种方式的。

## HTTPS的基本原理
- 公钥私钥 `ssh-keygen -t rsa -C "new email"`
- 证书，公钥加上CA的认证
- CA是颁发受信任的证书的机构
- TLS1.2是目前最高版本，还没有发现bug。不要选择SSL，TLS是SSL的后续版本，比SSL更加安全。OpenSSL是支持TLS的。
- 关于一系列的RSA加密解密，包括对称加解密，SHA摘要签名等，注意理解其原理即可。其中SHA1已被山东大学的王小云教授破解，改用SHA2.
![https握手链接建立](/images/2015/5/将网站打造为https/handshake.png)
上图中的几个要点
- 302浏览器端跳转需要重新进行tcp握手
- 服务器端发送的证书，浏览器需要到其CA进行验证是否可信
- 证书私钥主要用来协商对称加密秘钥

### StartSSL.com提供的免费CA认证证书和私钥
[StartSSL](http://www.startssl.com/)是一个免费的全球范围认证的证书提供商。具体的注册以及使用参考其官网,[这篇博文](http://www.freehao123.com/startssl-ssl/)讲解得还是比较细致的。

### HTTPS证书秘钥补充
- 不要纠结于证书的格式，这些格式都是可以互相转换的。
- 证书里面一般都包含公钥，某些证书格式甚至包含了私钥，如p12,pfx格式的证书。
- 最重要的是知道，在传给浏览器的证书里面不要包含私钥，还有证书是需要CA颁发了才具有trust性的。

## webserver配置证书和私钥，实现https方式
主要是通过nginx来做
1 在nginx中配置好证书，这样nginx可以返回这个证书给客户端。
2 客户端浏览器拿到证书了就可以进行认证，上文提到的StartSSL颁发的证书在受信任列表内。
3 如果客户端访问非https的url，就先让其进行redirect到对应https的url




