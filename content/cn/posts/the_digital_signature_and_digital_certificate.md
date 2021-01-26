---
title: "数字签名与数字证书"
date: 2014-10-06
categories: ['note', 'tech']
draft: false
---

在之前的[密码学笔记](/posts/the_basic_of_cryptology/)中主要是了解了密码学的基础知识，包括两种加密算法的原理，上篇笔记结束的时候引入了在非对称加密算法中数字证书（`Digital Certificate`）的概念。这篇笔记将继续探讨什么是数字证书，不过在了解它之前，先得知道什么是数字签名（`Digital Signature`）。

关于数字签名和数字证书的概念，有一篇非常经典的[文章](http://www.youdzone.com/signature.html)做了详细的介绍，本文的大部分内容来自于那篇文章。

## 数字证书与数字签名

Bob 生成了自己的公钥、私钥对，将私钥自己保存，公钥分发给了他的朋友们： Pat Susan 与 Daug

![public-key-and-private-key.jpg](https://i.loli.net/2021/01/26/WZFXjmgS2aGcYry.jpg)

Susan 要给 Bob 写一封保密的信件，写完后用 Bob 的公钥加密，就可以达到保密的效果。 Bob 收到信件之后用自己的私钥来解密，就可以看到信件的内容。这里假设 Bob 的私钥没有泄露（私钥是十分敏感的信息，一定要注意保管），即使信件被别人截获，信件内容也无法解密，也就是说这封信的内容不会有第三个人知道。

![comminucation-with-public-private-key-pair.jpg](https://i.loli.net/2021/01/26/iEfklACYPxg6hKG.jpg)

Bob 给 Susan 回信，决定采用"数字签名"：他写完信之后后先用哈希函数，生成信件的摘要（`Digest`），然后，再使用自己的私钥，对这个摘要进行加密，生成"数字签名"（`Digital Signature`）。

![digital-signature.jpg](https://i.loli.net/2021/01/26/e17WqOQNCm3cldy.jpg)

Bob 将这个数字签名，附在信件里面，一起发给 Susan：

Susan 收到信件之后，对信件本身使用相同的哈希函数，得到当前信件内容的摘要，同时，取下数字签名，用 Bob 的公钥解密，得到原始信件的摘要，如果两者相同就说明信件的内容没有被修改过。由此证明，这封信确实是 Bob 发出的。

![digital-signature-validation.jpg](https://i.loli.net/2021/01/26/DWJklALOB3QngxE.jpg)

但是，更复杂的情况出现了。 Daug 想欺骗 Susan ，他伪装成 Bob 制作了一对公钥、私钥对，并将公钥分发给 Susan ， Susan 此时实际保存的是 Daug 的公钥，但是还以为这是 Bob 的公钥。因此， Daug 就可以冒充 Bob ，用自己的私钥做成"数字签名"写信给 Susan ，而 Susan 用假的 Bob 公钥进行解密。一切看起来完美无缺？

Susan 觉得有些不对劲，因为她并不确定这个公钥是不是真正属于 Bob 的。于是她想到了一个办法，要求 Bob 去找"证书中心"（`Certificate Authority`），为公钥做认证。证书中心用证书中心的私钥，对 Bob 的公钥和一些相关信息一起加密，生成"数字证书"（`Digital Certificate`）。

![digital-certificate.jpg](https://i.loli.net/2021/01/26/7zVuLOB5xWH6qIp.jpg)

一旦 Bob 拿到数字证书以后，就可以放心写信给任何人了。只需要在信件内容的后面附上数字签名的同时，再附上数字证书就行了。

![communication-with-digital-certificate.jpg](https://i.loli.net/2021/01/26/T3ANMvaidHtpS27.jpg)

Susan 收到信件之后，首先使用用 CA 的公钥（一般都是公开的）解开数字证书，就可以拿到 Bob 真实可信的公钥了，然后就能用此公钥解密数字签名进一步验证信件内容是否被篡改过。

![digital-signature-validation-with-ca.jpg](https://i.loli.net/2021/01/26/G8qOEHiPFDuQUSd.jpg)

## 总结

在非对称加密的通信过程中，采用私钥进行对原文进行加密，然后公布密文，原文和公钥，这样任何人都可以用公钥解密密文，然后核对原文和密文是否一致。而私钥是不能公开的，也就是说只有私钥的拥有者才能采用私钥对数据加密，所以这种方式可以用于证明发布这条消息的人确实是其所声称的人，同时，使用数字签名可以验证消息没有被篡改过，这就是这就是数字签名的基本原理。

有了数字签名，那我们就可以通过数字签名来证明某个人拥有特定公钥对应的私钥，因此可以把公钥看作一个数字身份标识。通过证书中心对证书拥有者的基本信息、证书颁发者的信息以及证书本身的信息（过期时间、加密算法等）进行加密来生成数字证书。

由此可见，"数字证书"就是解决身份认证的问题，就如同现实中我们每一个人都要拥有一张证明个人身份的身份证或驾驶执照一样，以表明我们的身份或某种资格。数字证书是由权威公正的第三方机构即证书中心签发的，确保信息的机密性和防抵赖性。对于一个大型的应用环境，认证中心往往采用一种多层次的分级结构，各级的认证中心类似于各级行政机关，上级认证中心负责签发和管理下级认证中心的证书，最下一级的认证中心直接面向最终用户。
