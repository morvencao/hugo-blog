---
title: "PKI 系统的与 CA 中心"
date: 2014-11-12
categories: ['note', 'tech']
draft: false
---

在上一篇文章[数字签名与数字证书](/posts/the_digital_signature_and_digital_certificate/)中介绍了数字签名、数字证书的一些基础知识，但是并没有提到数字证书如何进行管理，比如数字证书的文件格式、数字证书的申请以及轮换等知识，这篇文章将介绍数字证书的管理。

说到数字证书的管理，不得不提到一个专有名词： PKI(Publick Key Infrastructure) ，翻译过来就是公钥基础设施，是一种遵循既定标准的密钥管理平台，它能够为所有网络应用提供加密和数字签名等密码服务及所必需的密钥和证书管理体系。简单来说，可以理解为利用之前提到过的公私钥非对称加密技术为应用提供加密和数字签名等密码服务以及与之相关的密钥和证书管理体系。

> PKI 既不是一个协议，也不是一个软件，它是一个标准，在这个标准之下发展出的为了实现安全基础服务目的的技术统称为 PKI 。

## PKI的组成

PKI 作为一个实施标准，包含一系列的组件：

1. **CA(Certificate Authority) 中心**：是 PKI 的”核心”，即数字证书的申请及签发机关， CA 必须具备权威性的特征，它负责管理 PKI 结构下的所有用户(包括各种应用程序)的证书的签发，把用户的公钥和用户的其他信息捆绑在一起，在网上验证用户的身份， CA 还要负责用户证书的黑名单登记和黑名单发布；

2. **X.509 目录服务器**：X.500 目录服务器用于"发布"用户的证书和黑名单信息，用户可通过标准的 LDAP 协议查询自己或其他人的证书和下载黑名单信息；

3. **基于 SSL(Secure Socket Layer) 的安全 web 服务器**：Secure Socket Layer(SSL) 协议最初由 Netscape 企业发展，现已成为网络用来鉴别网站和网页浏览者身份，以及在浏览器使用者及网页服务器之间进行加密通讯的全球化标准；

4. **Web 安全通信平台**：Web 有 Web 客户端和 Web 服务端两部分，通过具有高强度密码算法的 SSL 协议保证客户端和服务器端数据的机密性、完整性以及身份验证；

5. **自开发安全应用系统**：自开发安全应用系统是指各行业自开发的各种具体应用系统，例如银行、证券的应用系统等。完整的 PKI 包括:

    - 认证政策的制定
    - 认证规则
    - 运作制度的制定
    - 所涉及的各方法律关系内容
    - 技术的实现等

## 证书中心

认证中心也就是 CA ，是一个负责发放和管理数字证书的权威机构，它作为电子商务交易中受信任的第三方，承担公钥体系中公钥的合法性检验的责任。 CA 为每个使用公开密钥的用户发放一个数字证书，以实现公钥的分发并证明其合法性。作为 PKI 的核心部分， CA 实现了 PKI 中一些很重要的功能:

1. 接收验证最终用户数字证书的申请；
2. 确定是否接受最终用户数字证书的申请 - 证书的审批；
3. 向申请者颁发、拒绝颁发数字证书 - 证书的发放；
4. 接收、处理最终用户的数字证书更新请求 - 证书的更新；
5. 接收最终用户数字证书的查询、撤销；
6. 产生和发布证书废止列表；
7. 数字证书的归档；
8. 密钥归档；
9. 历史数据归档；

## X.509 证书标准

整个 PKI 体系中有很多格式标准， PKI 的标准规定了 PKI 的设计、实施和运营，规定了 PKI 各种角色的"游戏规则"。如果两个 PKI 应用程序之间要想进行交互，只有相互理解对方的数据含义，交互才能正常进行，标准的作用就是提供了数据语法和语义的共同约定。 PKI 中最重要的标准，它定义了公钥证书的基本结构。

X.509 是定义了公钥证书结构的基本标准，是目前非常通用的证书格式。X.509 标准在 PKI 中起到了举足轻重的作用，PKI 由小变大，由原来网络封闭环境到分布式开放环境，X.509 起了很大作用，可以说 X.509 标准是 PKI 的雏形。

[RFC5280](https://www.ietf.org/rfc/rfc5280.txt) 规定了 x.509 证书的标准格式如下图所示：

<img src="https://i.loli.net/2021/01/26/x324EYbUFZRyImP.png" alt="format-of-X-509-Certificate.png" style="width:60%;"/>

对于符合 X.509 标准的数字证书，必须保证公钥及其所有者(Subject)的姓名是一致的，同时，证书签发者者(Issuer)总是 CA 或由 CA 指定的人。 X.509 数字证书是一些标准字段的集合，这些字段包含有关用户或设备及其相应公钥的信息。 X.509 标准定义了证书中应该包含哪些信息，并描述了这些信息是如何编码的(即数据格式)，所有的 X.509 证书除了包含证书拥有者的公钥，证书拥有者的一些基本信息和数字签名之外，还包含了证书签发者的信息。具体来说包含以下数据：

1. **版本号(Version)**：指出该证书使用了哪种版本的 X.509 标准（版本1、版本2或是版本3），版本号会影响证书中的一些特定信息，目前的最新的版本为 `Version: 3` ；
2. **序列号(Serial Number)**： 标识证书的唯一整数，由证书颁发者分配的本证书的唯一标识符；
3. **签名算法标识符**： 用于签证书的算法标识，由对象标识符加上相关的参数组成，用于说明本证书所用的数字签名算法。例如， `sha256WithRSAEncryption` 对象标识符就用来说明该数字签名是利用 RSA 对 sha256 杂凑加密；
4. **有效期限(Validity)**： 证书起始日期和时间以及终止日期和时间；指明证书在这两个时间内有效；
5. **主题信息(Subject)**：证书持有人唯一的标识符(或称DN)这个名字在互联网上应该是唯一的；
6. **公钥信息(Subject Public Key Info)**： 包括证书持有人的公钥、算法(指明密钥属于哪种密码系统)的标识符和其他相关的密钥参数；
7. **颁发者唯一标识符(Issuer)**：标识符—证书颁发者的唯一标识符，仅在版本2和版本3中有要求，属于可选项；
8. **认证机构的数字签名**：这是使用证书发布者私钥生成的签名，以确保这个证书在发放之后没有被撰改过；
9. **认证机构**： 证书颁发者的可识别名（DN, Distinguished Name），是签发该证书的实体唯一的 CA 的 X.509 名字。使用该证书意味着信任签发证书的实体。(注意：在某些情况下，比如根或顶级 CA 证书，发布者自己签发证书)；

除了以上字段之外，X.509证书还可以包括可选的标准和专用的扩展（仅在版本2和版本3中使用），这里就不一一介绍。

下面是我使用`openssl`创建的个人证书信息：

```bash
$ openssl x509 -in certificate.pem -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 4096 (0x1000)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=CN, ST=Beijing, O=XXX, OU=XXX, CN=XXX/emailAddress=test@mail.com
        Validity
            Not Before: Mar 23 11:52:23 2019 GMT
            Not After : Apr  1 11:52:23 2020 GMT
        Subject: C=CN, ST=Beijing, L=XXXXX, O=XXX, OU=XX, CN=XXX/emailAddress=test@mail.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:a4:1a:a9:4b:aa:6e:a9:18:28:e6:35:bc:c3:6f:
                    ec:4c:7f:81:63:34:ce:57:a0:96:3f:2c:95:c8:01:
                    02:e9:75:54:f4:a4:22:d2:71:61:0e:8a:7a:bc:25:
                    d3:4d:cf:f0:9c:16:6f:0f:33:87:0e:f5:88:a4:25:
                    a8:67:b2:3e:a7:d1:b7:0d:6c:22:90:7a:ee:7a:53:
                    00:4d:5d:2e:3b:6d:48:5d:8d:a9:79:25:58:ca:0b:
                    ae:42:f6:ab:9f:a6:e8:e8:c7:06:ca:c3:92:09:ae:
                    19:a5:2b:68:1c:09:91:20:30:4f:44:5c:f0:49:fe:
                    26:29:90:7a:7b:4e:5f:93:93:49:da:a1:a1:da:4d:
                    e1:7c:86:67:05:ea:ce:64:b7:bb:fc:e0:10:f7:2e:
                    8f:9e:1f:82:4e:b6:d1:97:8d:a7:d2:0b:19:e4:bc:
                    ad:46:3f:ad:32:47:f9:47:bf:29:88:b4:3f:1d:02:
                    d4:93:e5:aa:76:13:83:52:7f:5d:33:f4:fa:73:85:
                    20:d7:7b:b9:a6:a9:35:dd:9f:e4:53:12:d3:db:33:
                    b0:1f:27:a1:9e:c0:aa:41:4b:ad:b1:74:8e:a3:c9:
                    e6:19:3d:39:2d:13:a7:e6:dc:e7:c6:88:06:70:60:
                    4b:5b:8a:cf:d3:45:23:b2:4e:17:c6:fe:06:c5:49:
                    a9:c9
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            Netscape Cert Type:
                SSL Server
            Netscape Comment:
                OpenSSL Generated Server Certificate
            X509v3 Subject Key Identifier:
                30:B7:57:E0:1A:07:69:DB:47:8C:B2:BC:21:14:7D:D2:71:A8:96:5D
            X509v3 Authority Key Identifier:
                keyid:E8:42:1C:63:92:76:AD:2C:62:2C:C1:71:92:40:24:B5:76:53:9A:1C
                DirName:/C=CN/ST=Beijing/L=XXXXX/O=XXX/OU=XXX/CN=XXX/emailAddress=test@mail.com
                serial:10:00

            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
    Signature Algorithm: sha256WithRSAEncryption
         98:8e:dd:8b:b7:af:c0:8d:41:76:0e:0a:5c:61:d6:39:a0:60:
         f4:56:4b:ea:b3:0d:09:c4:54:72:f7:45:b8:ba:f3:c9:f9:7c:
         7a:2b:81:82:dc:25:fd:b0:2c:b6:c9:d4:ad:c6:85:52:0c:0e:
         68:a2:fe:14:04:e1:95:a9:a0:b2:7a:8c:8a:25:20:ab:2d:62:
         a6:d7:e8:20:a2:9a:b4:8d:53:b1:ff:b0:d2:33:0d:ab:c3:fb:
         9c:03:59:73:cb:4f:dd:3f:bc:66:8c:c3:2b:32:23:a8:96:9e:
         a5:9d:f3:db:06:0c:a4:10:52:d6:3f:92:f8:ca:88:30:75:b6:
         d6:16:1a:5b:0f:f9:d6:2d:f2:94:2a:bd:c5:d9:b2:f9:5d:f5:
         02:01:4a:2a:10:45:a4:48:83:30:c0:a4:27:2f:73:72:73:93:
         ed:34:8f:66:53:0a:b4:93:05:61:00:22:0c:89:da:b6:46:17:
         a9:3e:28:37:2f:f3:7e:3f:69:a7:36:62:ff:c5:b2:ed:c2:0c:
         55:34:d3:c1:a6:71:77:9a:84:3a:b6:1a:9d:46:5c:6f:11:cf:
         f0:ef:52:f5:c9:ac:8f:dd:40:0f:28:69:44:3b:c9:c1:c7:45:
         9c:de:fb:48:95:89:90:9e:9e:74:e2:10:3a:52:e8:e0:30:ce:
         81:80:77:57:d9:32:02:fd:06:bd:72:e8:83:d8:3a:4e:6f:da:
         a5:07:44:cf:ba:3f:be:f1:79:ed:aa:01:65:a8:ea:f5:f6:ca:
         f0:dc:04:dd:8e:2e:78:93:75:31:c9:8f:c3:9b:40:19:fe:9b:
         fb:49:ca:b3:4e:81:39:d6:41:48:d6:80:30:a5:08:77:f6:24:
         93:d2:d0:67:5b:96:69:6a:12:f5:7c:5a:a3:f0:b2:2a:c1:69:
         76:f8:7f:b1:6c:d6:6c:a8:a9:62:aa:7a:7b:66:d0:52:bc:44:
         f5:4d:5d:1e:47:f0:00:ec:66:8a:05:43:a9:8c:23:1b:7c:ad:
         0d:24:9f:b9:5c:27:ca:e7:42:bb:10:02:e8:bf:1b:35:57:e8:
         6a:67:97:d5:dc:e0:6e:5e:fc:43:f9:26:d4:f1:32:87:15:86:
         8a:28:66:00:be:8f:fd:33:a4:50:97:35:56:a1:41:11:a4:92:
         5c:30:e3:6d:80:42:ef:a7:d7:f8:cb:ba:08:93:2e:61:73:b3:
         0c:c5:35:2f:9a:c9:9d:38:94:4c:43:15:fb:65:b7:f0:f6:7f:
         01:4b:6a:5b:a7:7c:f9:c2:a8:5c:dc:d3:30:41:1c:da:0e:e1:
         36:50:9c:6f:4a:e7:99:23:83:dd:c8:01:26:2d:e0:33:04:2a:
         0a:34:8b:78:47:bc:09:3b
```

## 数字证书格式

数字证书体现为一个或一系列相关经过加密的数据文件，常用的格式有：

- 符合 `PKI ITU-T X509` 标准，传统标准 (`.DER .PEM .CER .CRT`)
- 符合 `PKCS#7` 加密消息语法标准 (`.P7B .P7C .SPC .P7R`)
- 符合 `PKCS#10` 证书请求标准 (`.p10`)
- 符合 `PKCS#12` 个人信息交换标准 (`.pfx *.p12`)

当然，这只是常用的几种标准，其中，X509 证书还分两种编码形式：

- `X.509 DER`(Distinguished Encoding Rules) 编码，后缀为：`.DER .CER .CRT`
- `X.509 BASE64` 编码，后缀为：`.PEM .CER .CRT`

X.509 是数字证书的基本标准，而 P7 和 P12 则是两个实现规范， P7 用于数字信封， P12 则是带有私钥的证书实现规范。采用的标准不同、生成的数字证书包含内容也可能不同。下面就证书包含/可能包含的内容做个汇总，一般证书特性有：

- 存储格式：二进制还是 ASCII
- 是否包含公钥、私钥
- 包含一个还是多个证书
- 是否支持密码保护（针对当前证书）

其中：

- `DER、CER、CRT` 以二进制形式存放证书，只有公钥，不包含私钥；
- `CSR` 证书请求；
- `PEM` 以 Base64 编码形式存放证书，以 `—–BEGIN CERTIFICATE—–` 和 `—–END CERTIFICATE—–` 封装，只有公钥；
- `PFX`、 `P12` 也是以二进制形式存放证书，包含公钥、私钥，包含保护密码。`PFX` 和 `P12` 存储格式完全相同只是扩展名不同；
- `P10` 证书请求；
- `P7R` 是 CA 对证书请求回复，一般做数字信封；
- `P7B/P7C` 证书链，可包含一个或多个证书；

凡是包含私钥的，一律必须添加密码保护， 比如 `PFX`、 `P12` 也是以二进制形式存放证书；`DER` 表示证书且有签名，实际使用中，还有 `DER` 编码的私钥不用签名，实际上只是个“中间件”；另外，证书签名请求一般采用 `CSR` 扩展名，但是其格式有可能是 `PEM` 也可能是 `DER` 格式，但都代表证书请求，只有经过 CA 签发后才能得到真正的证书。