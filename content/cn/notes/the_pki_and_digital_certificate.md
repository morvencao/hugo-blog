---
title: "深入理解PKI系统的与数字证书"
date: 2014-11-12
type: "notes"
draft: false
---

在上一篇文章[数字签名与数字证书](/notes/the_digital_signature_and_digital_certificate/)中介绍了数字签名以及数字证书的一些基础知识以及数字证书的作用，但是并没有提到数字证书的管理，比如数字证书的申请，数字证书的文件格式等知识。这篇文章将继续探讨数字证书的管理知识。

说到数字证书的管理，不得不提到一个词，`PKI`(Publick Key Infrastructure)，即公钥基础设施，是一种遵循既定标准的密钥管理平台，它能够为所有网络应用提供加密和数字签名等密码服务及所必需的密钥和证书管理体系。简单来说，可以理解为利用之前提到过的供私钥非对称加密技术为应用提供加密和数字签名等密码服务以及与之相关的密钥和证书管理体系。

> PKI既不是一个协议，也不是一个软件，它是一个标准，在这个标准之下发展出的为了实现安全基础服务目的的技术统称为PKI

## PKI的组成

PKI作为一个实施标准，有一系列的组件组成：

1. **CA(Certificate Authority)中心(证书签发)**：是PKI的”核心”，即数字证书的申请及签发机关，CA必须具备权威性的特征，它负责管理PKI结构下的所有用户(包括各种应用程序)的证书，把用户的公钥和用户的其他信息捆绑在一起，在网上验证用户的身份，CA还要负责用户证书的黑名单登记和黑名单发布。

2. **X.500目录服务器(证书保存)**：X.500目录服务器用于"发布"用户的证书和黑名单信息，用户可通过标准的LDAP协议查询自己或其他人的证书和下载黑名单信息。

3. **基于SSL(Secure socket layer)的安全web服务器**：Secure Socket Layer(SSL)协议最初由Netscape企业发展，现已成为网络用来鉴别网站和网页浏览者身份，以及在浏览器使用者及网页服务器之间进行加密通讯的全球化标准。

4. **Web(安全通信平台)**：Web有Web Client端和Web Server端两部分，分别安装在客户端和服务器端，通过具有高强度密码算法的SSL协议保证客户端和服务器端数据的机密性、完整性、身份验证。

5. **自开发安全应用系统**：自开发安全应用系统是指各行业自开发的各种具体应用系统，例如银行、证券的应用系统等。完整的PKI包括:

    1) 认证政策的制定
    2) 认证规则
    3) 运作制度的制定
    4) 所涉及的各方法律关系内容
    5) 技术的实现等

### CA

认证中心CA(Certificate Authority)，是一个负责发放和管理数字证书的权威机构，它作为电子商务交易中受信任的第三方，承担公钥体系中公钥的合法性检验的责任。CA为每个使用公开密钥的用户发放一个数字证书，以实现公钥的分发并证明其合法性。作为PKI的核心部分，CA实现了PKI中一些很重要的功能:

1. 接收验证最终用户数字证书的申请
2. 确定是否接受最终用户数字证书的申请-证书的审批
3. 向申请者颁发、拒绝颁发数字证书-证书的发放
4. 接收、处理最终用户的数字证书更新请求-证书的更新
5. 接收最终用户数字证书的查询、撤销
6. 产生和发布证书废止列表(CRL)
7. 数字证书的归档
8. 密钥归档
9. 历史数据归档

### X.509标准

整个PKI体系中有很多格式标准，PKI的标准规定了PKI的设计、实施和运营，规定了PKI各种角色的"游戏规则"。如果两个PKI应用程序之间要想进行交互，只有相互理解对方的数据含义，交互才能正常进行，标准的作用就是提供了数据语法和语义的共同约定。PKI中最重要的标准，它定义了公钥证书的基本结构。

X.509是定义了公钥证书结构的基本标准，是目前非常通用的证书格式。X.509标准在PKI中起到了举足轻重的作用，PKI由小变大，由原来网络封闭环境到分布式开放环境，X.509起了很大作用，可以说X.509标准是PKI的雏形。PKI是在X.509标准基础上发展起来的。理论上只要为一个网络应用程序创建的证书符合ITU-T X.509国际标准，就可以用于任何其他符合X.509标准的网络应用。对于符合X.509标准的数字证书，必须保证公钥及其所有者(Subject)的姓名是一致的，同时，认证者(Issuer)总是CA或由CA指定的人。X.509数字证书是一些标准字段的集合，这些字段包含有关用户或设备及其相应公钥的信息。X.509标准定义了证书中应该包含哪些信息，并描述了这些信息是如何编码的(即数据格式)，所有的X.509证书包含以下数据：

1. **版本号(Version)**：指出该证书使用了哪种版本的X.509标准（版本1、版本2或是版本3），版本号会影响证书中的一些特定信息，目前的版本为3
2. **序列号(Serial Number)**： 标识证书的唯一整数，由证书颁发者分配的本证书的唯一标识符
3. **签名算法标识符**： 用于签证书的算法标识，由对象标识符加上相关的参数组成，用于说明本证书所用的数字签名算法。例如，SHA-1和RSA的对象标识符就用来说明该数字签名是利用RSA对SHA-1杂凑加密
4. **认证机构的数字签名**：这是使用证书发布者私钥生成的签名，以确保这个证书在发放之后没有被撰改过
5. **认证机构**： 证书颁发者的可识别名（DN），是签发该证书的实体唯一的CA的X.500名字。使用该证书意味着信任签发证书的实体。(注意：在某些情况下，比如根或顶级CA证书，发布者自己签发证书)
6. **有效期限(Validity)**： 证书起始日期和时间以及终止日期和时间；指明证书在这两个时间内有效
7. **主题信息(Subject)**：证书持有人唯一的标识符(或称DN-distinguished name)这个名字在 Internet上应该是唯一的
8. **公钥信息(Public-Key)**： 包括证书持有人的公钥、算法(指明密钥属于哪种密码系统)的标识符和其他相关的密钥参数
9. **颁发者唯一标识符(Issuer)**：标识符—证书颁发者的唯一标识符，仅在版本2和版本3中有要求，属于可选项

除了以上字段之外，X.509证书还可以包括可选的标准和专用的扩展（仅在版本2和版本3中使用），扩展部分的元素都有这样的结构：

```
Extension ::= SEQUENCE {
    extnID OBJECT IDENTIFIER,  # extnID：表示一个扩展元素的OID
    critical BOOLEAN DEFAULT FALSE, # 表示这个扩展元素是否极重要
    extnValue OCTET STRING # 表示这个扩展元素的值，字符串类型
}
```

扩展部分包括：

1. **发行者密钥标识符**：证书所含密钥的唯一标识符，用来区分同一证书拥有者的多对密钥
2. **密钥使用**：一个比特串，指明（限定）证书的公钥可以完成的功能或服务，如：证书签名、数据加密等。如果某一证书将 KeyUsage 扩展标记为“极重要”，而且设置为“keyCertSign”，则在 SSL通信期间该证书出现时将被拒绝，因为该证书扩展表示相关私钥应只用于签写证书，而不应该用于SSL
3. **CRL分布点**：指明CRL的分布地点
4. **私钥的使用期**：指明证书中与公钥相联系的私钥的使用期限，它也有`Not Before`和`Not After`组成（若此项不存在时，公私钥的使用期是一样的）
5. **证书策略**：由对象标识符和限定符组成，这些对象标识符说明证书的颁发和使用策略有关
6. **策略映射**：表明两个CA域之间的一个或多个策略对象标识符的等价关系，仅在CA证书里存在
7. **主体别名**：指出证书拥有者的别名，如电子邮件地址、IP地址等，别名是和DN绑定在一起的
8. **颁发者别名**：指出证书颁发者的别名，如电子邮件地址、IP地址等，但颁发者的DN必须出现在证书的颁发者字段
9. **主体目录属性**：指出证书拥有者的一系列属性。可以使用这一项来传递访问控制信息

下面是我使用`openssl`创建的个人证书信息：

```
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

### 证书废除列表CRL

证书废除列表CRL(Certificate revocation lists)为应用程序和其它系统提供了一种检验证书有效性的方式。任何一个证书废除以后，证书机构CA会通过发布CRL的方式来通知各个相关方。符合X.509标准的CSL基本结构如下所示：

1. CRL的版本号
    1) 0: 表示X.509 V1 标准
    2) 1: 表示X.509 V2 标准
    3) 2: 表示X.509 V3标准
目前常用的是V3标准

2. 签名算法：包含:
    1) 算法标识
    2) 算法参数
用于指定证书签发机构用来对CRL内容进行签名的算法。

3. 证书签发机构名：签发机构的DN名，由
    1) 国家(C)
    2) 省市(ST)
    3) 地区(L)
    4) 组织机构(O)
    5) 单位部门(OU)
    6) 通用名(CN)
    7) 邮箱地址

4. 此次签发时间：此次CRL签发时间，遵循ITU-T X.509 V2标准的CA在2049年之前把这个域编码为UTCTime类型，在2050或2050年之后年之前把这个域编码为GeneralizedTime类型。

5. 下次签发时间：下次CRL签发时间，遵循ITU-T X.509 V2标准的CA在2049年之前把这个域编码为UTCTime类型，在2050或2050年之后年之前把这个域编码为GeneralizedTime类型。

6. 用户公钥信息，其中包括:
    1) 废除的证书序列号: 要废除的由同一个CA签发的证书的一个唯一标识号，同一机构签发的证书不会有相同的序列号
    2) 证书废除时间 

7. 签名算法：对CRL内容进行签名的签名算法。

8. 签名值：证书签发机构对CRL内容的签名值

同样，举个例子：

```
Certificate Request:
    Data:
        Version: 0 (0x0)
        Subject: C=CN, ST=Beijing, L=XX, O=XXX, OU=XXX, CN=XXX/emailAddress=
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
            RSA Public Key: (2048 bit)
                Modulus (2048 bit):
                    00:be:2c:a4:fc:9f:f7:b3:2a:6b:c8:2f:ec:8d:59:
                    ba:12:ed:8e:c1:82:e0:6b:5d:12:99:ff:a1:54:3f:
                    64:d5:31:7f:26:b6:70:95:a7:1e:7f:89:77:3b:c9:
                    cd:00:7c:9a:cc:32:c9:2f:56:f5:36:8d:2b:65:d9:
                    73:0c:a8:6f:03:46:1e:97:76:66:5c:93:a4:2c:00:
                    99:0e:b0:38:e5:43:22:ae:6e:c6:0c:f7:b5:ef:59:
                    9b:c8:d3:af:5a:35:9b:78:1d:e3:bd:c5:7e:08:9e:
                    fc:de:73:fd:2a:fc:f6:11:97:ca:60:30:f4:37:0f:
                    e7:d6:b7:36:d4:84:3e:e2:81:02:27:24:96:16:6d:
                    da:97:7c:d9:bf:5b:79:51:f1:ba:4e:e9:17:44:1e:
                    7c:fe:2d:b3:ec:62:34:2b:4d:ce:84:49:9f:0a:ec:
                    1e:fe:ee:69:60:e5:14:73:cd:8f:3d:75:d7:d9:c5:
                    b3:dc:c6:d7:d2:df:e6:ba:3a:a3:da:97:dd:24:cf:
                    6b:e4:00:df:64:13:22:da:25:e2:4b:47:d3:12:39:
                    60:0e:ab:a3:bc:54:c9:c3:36:80:9d:e5:f0:be:83:
                    d4:b5:d4:73:70:15:42:6e:74:04:06:ab:12:3e:02:
                    45:1f:02:20:79:fd:b5:00:48:b1:78:f0:a7:76:a5:
                    94:2f
                Exponent: 65537 (0x10001)
        Attributes:
            a0:00
    Signature Algorithm: sha1WithRSAEncryption
        82:1a:93:02:7d:42:02:91:7f:59:31:75:84:49:8a:d4:4a:90:
        ec:ad:c9:f7:3b:75:68:23:f4:d0:9b:de:ab:0e:4e:60:7c:46:
        be:26:35:38:68:6b:1e:d0:61:19:86:b2:b6:a6:94:5e:8a:c1:
        90:01:63:df:a7:c2:b0:79:75:bd:01:72:30:9a:08:21:83:82:
        51:e7:79:07:7b:c8:27:9d:fa:5d:38:89:3a:97:87:87:21:65:
        a7:00:3d:4b:c6:2f:ac:0c:45:57:8c:1a:bd:89:78:2b:7a:00:
        4d:48:09:c5:55:22:9e:92:6b:f9:c8:dd:8f:de:5c:61:c7:3d:
        20:6a:a3:6b:e5:32:00:2f:dd:68:d8:a5:66:be:19:fb:95:e1:
        e2:cc:18:1e:96:2e:e5:2f:58:d9:4c:f8:d5:92:1d:34:ed:79:
        52:a2:3d:02:2e:58:2f:86:d3:29:b7:5c:66:27:25:61:d3:0e:
        5e:86:77:12:0f:4f:12:3c:bf:95:85:5c:b7:77:05:11:9e:bb:
        06:ac:f8:cc:c3:42:84:f7:a7:84:b3:6c:fe:fe:66:92:31:32:
        dc:47:8d:a2:04:e0:2e:43:74:de:9f:03:c6:7e:f0:90:1d:0f:
        8a:f3:bc:5c:2c:5c:0b:db:d9:7d:69:05:31:a9:13:f4:18:1f:
        7d:69:f4:26
```

### 数字证书格式

数字证书体现为一个或一系列相关经过加密的数据文件。常见格式有：

- 符合`PKI ITU-T X509`标准，传统标准（`.DER .PEM .CER .CRT`）
- 符合`PKCS#7`加密消息语法标准(`.P7B .P7C .SPC .P7R`)
- 符合`PKCS#10`证书请求标准(`.p10`)
- 符合`PKCS#12`个人信息交换标准（`.pfx *.p12`）

当然，这只是常用的几种标准，其中，X509证书还分两种编码形式：

- `X.509 DER`(Distinguished Encoding Rules)编码，后缀为：`.DER .CER .CRT`
- `X.509 BASE64`编码，后缀为：.PEM .CER .CRT

`X.509`是数字证书的基本标准，而`P7`和`P12`则是两个实现规范，`P7`用于数字信封，`P12`则是带有私钥的证书实现规范。采用的标准不同，生成的数字证书，包含内容也可能不同。下面就证书包含/可能包含的内容做个汇总，一般证书特性有：

- 存储格式：二进制还是`ASCII`
- 是否包含公钥、私钥
- 包含一个还是多个证书
- 是否支持密码保护（针对当前证书

其中：

- `DER、CER、CRT`以二进制形式存放证书，只有公钥，不包含私钥
- `CSR`证书请求
- `PEM`以`Base64`编码形式存放证书，以`—–BEGIN CERTIFICATE—–`和`—–END CERTIFICATE—–`封装，只有公钥
- `PFX`、`P12`也是以二进制形式存放证书，包含公钥、私钥，包含保护密码。`PFX`和`P12`存储格式完全相同只是扩展名不同
- `P10`证书请求
- `P7R`是CA对证书请求回复，一般做数字信封
- `P7B/P7C`证书链，可包含一个或多个证书

凡是包含私钥的，一律必须添加密码保护， 比如`PFX`、`P12`也是以二进制形式存放证书。`DER`表示证书且有签名，实际使用中，还有`DER`编码的私钥不用签名，实际上只是个“中间件”。另外：证书签名请求一般采用`CSR`扩展名，但是其格式有可能是`PEM`也可能是`DER`格式，但都代表证书请求，只有经过CA签发后才能得到真正的证书。

## SSL证书生成

有些情形只需要公私密钥对就够了，不需要数字证书，比如私有的`SSH`服务。但是对于一些要求身份认证的情形，则需要对公钥进行数字签名形成数字证书。

有两种常见的工具来生成`RSA`公私密钥对：

- `ssh-keygen`
- `openssl genrsa`

其实`ssh-keygen`底层也是使用`openssl`提供的库来生成密钥。

### ssh-keygen

举例来说，我们要生成`2048`位RSA密钥对：

$ ssh-keygen -b 2048 -t rsa -f foo_rsa
 
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in foo_rsa.
Your public key has been saved in foo_rsa.pub.
The key fingerprint is:
b8:c4:5f:2a:94:fd:b9:56:9d:5b:fd:96:02:5a:7e:b7 user@oasis
The key's randomart image is:
+--[ RSA 2048]----+
|                 |
|                 |
|                 |
|     . +         |
|      * S .  . ..|
|     o o + +. o o|
|      o o *..  oo|
|       . ..o o.oo|
|         .. . oE.|
+-----------------+

`-b`指定密钥位数
`-t`指定密钥类型：`rsa`，`dsa`, `ecdsa`

如果密钥对用于`ssh`，那么习惯上私钥命名为`id_rsa`,对应的公钥就是`id_rsa.pub`，然后可以使用`ssh-copy-id`把密钥追加到远程主机的`.ssh/authorized_key`文件里：

```
$ ssh-copy-id -i ~/.ssh/id_rsa.pub user@mc-lnx
```

默认生成的密钥对是`RFC 4716/SSH2`格式，可以转到`PEM`格式公其他程序使用：

```
$ ssh-keygen -f id_rsa.pub -e -m pem
-----BEGIN RSA PUBLIC KEY-----
...
-----END RSA PUBLIC KEY-----
```

`-m`指定转换后的格式，支持`RFC4716`(RFC 4716/SSH2 public or private key), PKCS8(PEM PKCS8 public key), PEM(PEM public key)

### openssl

`openSSL`是一个强大的安全套接字层密码库，整个软件包大概可以分成三个主要的功能部分：

1. 密码算法库
2. 常用的密钥和证书封装管理功能
3. SSL通信API接口
4. 丰富的应用程序供测试或其它目的使用

使用openSSL开发套件可以完成以下功能：

1. 建立`RSA`、`DH`、`DSA` key参数
2. 生成`X.509`证书、证书签名请求(CSR)和CRLs(证书回收列表)
3. 计算消息摘要(Digest)
4. 使用各种`Cipher`加密/解密
5. `SSL/TLS`客户端以及服务器的测试
6. 处理`S/MIME`或者加密邮件

openssl提供了很多不同的命令，每个子命令有很多的选项和参数，这里就不详细列举。

### SSL证书生成过程

使用`openssl`这个套件可以完成CA的搭建、SSL证书从生成到签发的全部过程，在使用openssl的指令的时候，需要记住几点:

1. 在生成过程中有很多文件扩展名(`.crt`、`.csr`、`.pem`、`.key`等)，从本质上讲，扩展名并不具有任何强制约束作用，重要的是这个文件是由哪个命令生成的，它的内容是什么格式的。
使用这些特定的文件扩展名只是为了遵循某些约定俗称的规范，让人能一目了然。
2. openssl的指令之间具有一些功能上的重叠，所以我们会发现完成同样一个目的(例如SSL证书生成)，往往可以使用看似不同的指令组达到目的
3. 理解CA、SSL证书最重要的不是记住这些openssl指令，而是要理解CA的运行机制，同时理解openssl指令的功能，从原理上去理解整个流程

#### 创建CA根证书

CA根证书是PKI体系中重要的组成部分，要建立树状的受信任体系，首先CA自身需要表明身份，并建立树根，其他的中级证书都依赖该根来签发证书，并通过根证书建立彼此的信任关系。CA证书由私钥(Root Key)和公钥(Root Cert)组成。CA根证书并不直接颁发服务器证书和客户端证书，仅用来颁发一个或多个中级证书，中级证书代理CA根证书向用户和服务器颁发证书。同时，为了保证CA根证书的可靠性，应保证CA根证书对应的私钥 (Root Key)处于离线并安全保存。

1. 创建`Root CA`目录

选择一个目录用于存储所有的私钥和证书公钥：

```
$ mkdir /root/ca
```

创建目录结构，生成`index.txt`和`serial`文件，用于作为文件数据库追踪以前发的证书：

```
$ cd /root/ca
$ mkdir certs crl newcerts private
$ chmod 700 private
$ touch index.txt
$ echo 1000 > serial
```

- certs：存放已签发的证书
- newcerts：存放CA生成的新证书
- private：存放私钥
- crl：存放以吊销的证书
- index.txt：OpenSSL定义的已签发证书的文本数据库文件，通常该文件在初始化时为空
- serial：签发证书时使用的序列号参考文件，该文件中的序列号为16进制数，初始化时必须提供并包含一个有效的序列号

2. 创建openssl Root CA的配置文件

为openssl创建Root CA配置文件，保存在`/root/ca/openssl.cnf`，文件中`[ ca ]`的部分是必须存在的，用于告诉openssl CA使用的配置信息：

```
[ ca ]
# `man ca`
default_ca= CA_default
```

在`[ CA_default ]`中CA的基础配置信息：

```
[ CA_default ]
# Directory and file locations.
dir               = /root/ca
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand
 
# The root key and root certificate.
private_key       = $dir/private/ca.key.pem
certificate       = $dir/certs/ca.cert.pem
 
# For certificate revocation lists.
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30
 
# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256
 
name_opt          = ca_default
cert_opt          = ca_default
default_days      = 375
preserve          = no
policy            = policy_strict
```

创建Root CA证书签名策略，作为Root CA仅用来签发中级CA证书：

```
[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
```

为中级CA证书创建相对宽松的证书签名策略，中级CA证书可以为第三方终端用户和服务器签发证书：

```
[ policy_loose ]
# Allow the intermediate CA to sign a more diverse range of certificates.
# See the POLICY FORMAT section of the `ca` man page.
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
```

在`[ req ]`部分，主要定义了创建证书和证书请求时默认选项：

```
[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only
 
# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256
 
# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca
```

在`[ req_distinguished_name ]`中，定义了生成证书请求时常用的信息说明和默认值：

```
[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
emailAddress                    = Email Address
 
# Optionally, specify some defaults.
countryName_default             = GB
stateOrProvinceName_default     = England
localityName_default            =
0.organizationName_default      = Alice Ltd
#organizationalUnitName_default =
#emailAddress_default           =
```

在openssl中，除了配置基本的必要信息外，还有一些可扩展的选项可用于证书签发：

```
[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
```

定义中级CA证书签发的扩展选项，其中`pathlen:0`是为了确保中级CA证书不能再签发新的中级CA证书：

```
[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
```

在`[ user_cert]`中约束了签发客户端证书策略，该证书一般用于用户身份认证：

```
[ usr_cert ]
# Extensions for client certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = client, email
nsComment = "OpenSSL Generated Client Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, emailProtection
```

在`[ server_cert ]`中，定义了签发服务器证书时的证书签发策略：

```
[ server_cert ]
# Extensions for server certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "OpenSSL Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
```

在`[ crt_ext ]`用于定义创建证书吊销列表CRL时的策略：

```
[ crl_ext ]
# Extension for CRLs (`man x509v3_config`).
authorityKeyIdentifier=keyid:always
```

定义在线证书使用策略`ocsp`：

```
[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning
```

3. 创建Root CA Key

生成的Root key必须保证其绝对的安全，并使用`aes-256`加密和高强度密码保护：

```
$ cd /root/ca
$ openssl genrsa  -aes256 -out private/ca.key.pem 4096
Enter pass phrase for ca.key.pem: secretpassword 
Verifying - Enter pass phrase for ca.key.pem: secretpassword
 
$ chmod 400 private/ca.key.pem
```

- `genrsa`：使用RSA算法产生私钥
- `-aes256`：使用256位密钥的AES算法对私钥进行加密
- `-out`：输出文件目录
- `4096`：指定密钥的长度

4. 创建Root CA证书

使用Root Key(`ca.key.pem`)创建Root CA证书(`ca.cert.pem`)。需要为Root CA指定较长的有效期，一旦Root CA证书过期，其所有签发的证书将都不可用。在使用`req`命令时，必须使用`-config`指定配置文件，否则程序默认使用`/etc/pki/tls/openssl.cnf`。

```
$ cd /root/ca
openssl req -config openssl.cnf -key private/ca.key.pem -new -x509 -days 7300 -sha256 -extensions v3_ca -out certs/ca.cert.pem
Enter pass phrase for ca.key.pem: secretpassword
You are about to be asked to enter information that will be incorporated
into your certificate request.
-----
Country Name (2 letter code) [XX]:GB
State or Province Name []:England
Locality Name []:
Organization Name []:Alice Ltd
Organizational Unit Name []:Alice Ltd Certificate Authority
Common Name []:Alice Ltd Root CA
Email Address []:
$ chmod 444 certs/ca.cert.pem
```

5. 验证签发的Root CA证书

```
$ openssl x509 -noout -text -in certs/ca.cert.pem
```

我们需要确认一下信息：

- 确认Signature Algorithm使用的签名算法
- 确认Validity中的证书有效期
- 确认Public-Key长度
- 确认Issuer，签发者信息
- 确认Subject，Root CA证书为自签名证书，与Issuer信息相同
- 启用x509 v3扩展信息，在输出的文件中，还将包括一些扩展的信息，如：X509v3 Subject Key Identifier/X509v3 Authority Key Identifier/X509v3 Basic Constraints等

6. 创建中级CA证书密钥对

中级CA证书用于代替Root CA签发客户端证书和服务器证书，根证书负责签发中级CA证书并维护受信任的证书链。使用中级证书主要是处于安全考虑，因为Root CA对应的私钥Root Key始终处理物理隔离并能够保障私钥的绝对安全，在中级CA证书的私钥发生泄漏后，可以通过Root CA吊销该证书，并重新签发新的中级证书密钥对。

7. 创建中级CA证书目录结构

与Root CA类是，我们需要为中级CA证书创建用于存储证书的目录。这里我们将中级证书所有内容保存在`/root/ca/intermediate`目录下面，并创建类是Root CA的目录结构：

```
$ mkdir /root/ca/intermediate
$ cd /root/ca/intermediate
$ mkdir certs crl newcerts private
$ chmod 700 private
$ touch index.txt
$ echo 1000 > serial
```

在中级CA证书目录下，新建名为`crlnumber`的文本文件，用于保存证书吊销列表：

```
$ echo 1000 > /root/ca/intermediate/crlnumber
```

创建中级CA证书OpenSSL配置文件，将中级CA证书的配置文件保存在`/root/ca/intermediate/openssl.cnf`，相比Root CA配置文件修改了证书的保存目录：

```
[ CA_default ]
dir             = /root/ca/intermediate
private_key     = $dir/private/intermediate.key.pem
certificate     = $dir/certs/intermediate.cert.pem
crl             = $dir/crl/intermediate.crl.pem
policy          = policy_loose
```

完整的配置文件内容如下：

```
# OpenSSL intermediate CA configuration file.
# Copy to `/root/ca/intermediate/openssl.cnf`.
 
[ ca ]
# `man ca`
default_ca = CA_default
 
[ CA_default ]
# Directory and file locations.
dir               = /root/ca/intermediate
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand
 
# The root key and root certificate.
private_key       = $dir/private/intermediate.key.pem
certificate       = $dir/certs/intermediate.cert.pem
 
# For certificate revocation lists.
crlnumber         = $dir/crlnumber
crl               = $dir/crl/intermediate.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30
 
# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256
 
name_opt          = ca_default
cert_opt          = ca_default
default_days      = 375
preserve          = no
policy            = policy_loose
 
[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
 
[ policy_loose ]
# Allow the intermediate CA to sign a more diverse range of certificates.
# See the POLICY FORMAT section of the `ca` man page.
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
 
[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only
 
# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256
 
# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca
 
[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
emailAddress                    = Email Address
 
# Optionally, specify some defaults.
countryName_default             = GB
stateOrProvinceName_default     = England
localityName_default            =
0.organizationName_default      = Alice Ltd
organizationalUnitName_default  =
emailAddress_default            =
 
[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
 
[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
 
[ usr_cert ]
# Extensions for client certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = client, email
nsComment = "OpenSSL Generated Client Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, emailProtection
 
[ server_cert ]
# Extensions for server certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "OpenSSL Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
 
[ crl_ext ]
# Extension for CRLs (`man x509v3_config`).
authorityKeyIdentifier=keyid:always
 
[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning
```

8. 生成中级CA证书私钥

创建中级证书私钥，使用`AES-256`加密：

```
$ cd /root/ca
$ openssl genrsa -aes256 -out intermediate/private/intermediate.key.pem 4096
$ chmod 400 intermediate/private/intermediate.key.pem
```

9. 生成中级CA证书签名请求

使用`intermediate.key.pem`为生成中级CA证书签名请求，同时使用Root CA对证书进行签名。注意`Common Name`不能重复。如果使用中级CA签发服务器证书和客户端证书，需要指定配置文件`intermediate/openssl.cnf`：

```
$ cd /root/ca
$ openssl req -config intermediate/openssl.cnf -new sha256 -key intermediate/private/intermediate.key.pem -out intermediate/csr/intermediate.csr.pem
```

10. 生成中级CA证书

使用Root CA签发中级CA证书，并为其指定`v3_intermediate_ca`扩展属性签发证书。中级证书的有效期要短于根证书。确认使用Root CA的`openssl.cnf`签发中级证书：

```
$ cd /root/ca
$ openssl ca -config openssl.cnf -extenstions v3_intermediate_ca -days 3650 -notext -md sha256 -in intermediate/csr/intermediate.csr.pem -out intermediate/certs/intermediate.cert.pem
$ chmod 444 intermediate/certs/intermediate.cert.pem
```

> Note: `index.txt`是openssl的文件数据库，不要直接修改或删除该文件。

11. 验证中级CA证书内容

```
$ openssl x509 -noout -text -in intermediate/certs/intermediate.cert.pem
```

使用Root CA根证书验证中级证书的证书链信任关系，`OK`表示证书链受信任：

```
$ openssl verify -CAfile certs/ca.cert.pem intermediate/certs/intermediate.cert.pem
intermediate.cert.pem: OK
```

12. 创建证书链文件

当应用程序(Web浏览器)尝试验证签发的证书有效性时，会尝试验证中级CA证书和上级根证书。在证书链文件中，必须同时包含根证书和中级证书。因为客户端不知知道签发的这张私有的CA根证书，你可以在所有需要访问的客户端将Root CA导入到受信任的根证书办法机构，这样该CA签发的证书就完全受信任并且证书链文件中则只用指定中级证书即可。

```
$ cat intermediate/certs/intermediate.cert.pem certs/ca.cert.pem > intermediate/certs/ca-chain.cert.pem
$ chmod 444 intermediate/certs/ca-chain.cert.pem
```

13. 签发服务器证书和客户端证书

接下来我们将使用中级CA证书为服务器签发证书，用来加密服务器与客户端之间的数据通信。如果您在服务器使用openssl已经创建过了`csr`证书请求，则可跳过生成Server Key的过程。

14. 创建服务器证书私钥

在创建Root CA和intermediate CA使用了`4096bit`密码加密，服务器和客户端证书一般有效期为几年，可以使用`2048bit`加密。虽然`4096bit`密码要更安全，但在服务器进行TLS握手和数字签名验证时会降低访问速度，因此一般Web服务器还是建议使用`2048Bit`密码加密。如果您在创建私钥时设置了密码，则每次服务重启时都需要提供这一密码，因此不建议为服务器证书设置密码保护，可以使用`aes256`密码算法进行加密：

```
$ cd /root/ca
$ openssl genrsa -aes256 -out intermediate/private/r.wanglijie.cn.key.pem 2048
$ chmod 400 intermediate/private/r.wanglijie.cn.key.pem
```

15. 创建服务器证书请求csr

使用创建的服务器私钥生成签发证书请求`csr`文件，`csr`文件无需指定中级CA证书机构，但是必须指定`Common Name`，一般为域名或IP地址：

```
$ cd /root/ca
$ openssl req -config intermediate/openssl.cnf -new -sha256 -days 365 -key intermediate/private/r.wanglijie.cn.key.pem -out intermediate/csr/r.wanglijie.cn.csr.pem
```

16. 从证书请求`csr`创建服务器证书

```
$ cd /root/ca
$ openssl ca -config intermediate/openssl.cnf \
      -extensions server_cert -days 375 -notext -md sha256 \
      -in intermediate/csr/r.wanglijie.cn.cert.pem \
      -out intermediate/certs/r.wanglijie.cn.cert.pem 
$ chmod 444 intermediate/certs/r.wanglijie.cn.cert.pem
```

> Note: 在签发证书时，`-extensions`选项用于指定要签发的证书类型。服务器证书使用"server_cert"，客户端证书使用"usr_cert"。证书签发完毕后，将会在`intermediate/index.txt`文件中新增一条记录。

17. 查看签发的证书

```
$ openssl x509 -noout -text  -in intermediate/certs/r.wanglijie.cn.cert.pem
```

使用CA证书链(`ca-chain.cert.pem`)验证签发证书的信任关系：

```
$ openssl verify -CAfile intermediate/certs/ca-chain.cert.pem intermediate/certs/r.wanglijie.cn.cert.pem
```

18. 部署证书

在证书签发完毕后，可以将证书部署在`Nginx`, `Apache`或其他的应用服务器上。

