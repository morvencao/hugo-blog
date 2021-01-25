---
title: "创建私有CA以及颁发SSL证书"
date: 2014-11-24
categories: ['note', 'tech']
draft: false
---

在上一篇文章[深入理解PKI系统的与数字证书](/notes/the_pki_and_digital_certificate/)中介绍了PKI系统的基本组成以及CA认证中心的主要作用，以及X.509证书基本标准。今天，我们继续应用已经学习的理论知识构建一套自己的PKI/CA数字证书信任体系。

## SSL证书生成工具

有以下两种常见的工具来生成`RSA`公私密钥对:

> Note: 有些情形只需要公私密钥对就够了，不需要数字证书，比如私有的`SSH`服务。但是对于一些要求身份认证的情形，则需要对公钥进行数字签名形成数字证书。

- `ssh-keygen`
- `openssl genrsa`

实际上`ssh-keygen`底层也是使用`OpenSSL`提供的库来生成密钥。

### ssh-keygen

举例来说，我们要生成`2048`位RSA密钥对：

```
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
```

- `-b`指定密钥位数
- `-t`指定密钥类型：`rsa`，`dsa`, `ecdsa`

> Note: 如果密钥对用于`ssh`，那么习惯上私钥命名为`id_rsa`,对应的公钥就是`id_rsa.pub`，然后可以使用`ssh-copy-id`把密钥追加到远程主机的`.ssh/authorized_key`文件里。

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

### OpenSSL

`OpenSSL`是一个强大的安全套接字层密码库，整个软件包大概可以分成三个主要的功能部分：

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

`OpenSSL`提供了很多不同的命令，每个子命令有很多的选项和参数，这里就不详细列举。

## SSL证书生成过程

使用`OpenSSL`这个套件可以完成CA的搭建、SSL证书从生成到签发的全部过程，在使用openssl的指令的时候，需要记住几点:

1. 在生成过程中有很多文件扩展名(`.crt`、`.csr`、`.pem`、`.key`等)，从本质上讲，扩展名并不具有任何强制约束作用，重要的是这个文件是由哪个命令生成的，它的内容是什么格式的。
使用这些特定的文件扩展名只是为了遵循某些约定俗称的规范，让人能一目了然。
2. openssl的指令之间具有一些功能上的重叠，所以我们会发现完成同样一个目的(例如SSL证书生成)，往往可以使用看似不同的指令组达到目的
3. 理解CA、SSL证书最重要的不是记住这些openssl指令，而是要理解CA的运行机制，同时理解openssl指令的功能，从原理上去理解整个流程

#### 创建CA根证书

CA根证书是PKI体系中重要的组成部分，要建立树状的受信任体系，首先CA自身需要表明身份，并建立树根，其他的中级证书都依赖该根来签发证书，并通过根证书建立彼此的信任关系。CA证书由私钥(Root Key)和公钥(Root Cert)组成。CA根证书并不直接颁发服务器证书和客户端证书，仅用来颁发一个或多个中级证书，中级证书代理CA根证书向用户和服务器颁发证书。同时，为了保证CA根证书的可靠性，应保证CA根证书对应的私钥 (Root Key)处于离线并安全保存。

1) 创建`Root CA`目录

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

2) 创建OpenSSL Root CA配置文件

为`OpenSSL`创建Root CA配置文件，保存在`/root/ca/openssl.cnf`，文件中`[ ca ]`的部分是必须存在的，用于告诉openssl CA使用的配置信息：

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

3) 创建Root CA Key

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

4) 创建Root CA证书

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

5) 验证签发的Root CA证书

```
$ openssl x509 -noout -text -in certs/ca.cert.pem
```

需要确认一下信息：

- 确认Signature Algorithm使用的签名算法
- 确认Validity中的证书有效期
- 确认Public-Key长度
- 确认Issuer，签发者信息
- 确认Subject，Root CA证书为自签名证书，与Issuer信息相同
- 启用x509 v3扩展信息，在输出的文件中，还将包括一些扩展的信息，如：X509v3 Subject Key Identifier/X509v3 Authority Key Identifier/X509v3 Basic Constraints等

6) 创建中级CA证书密钥对

中级CA证书用于代替Root CA签发客户端证书和服务器证书，根证书负责签发中级CA证书并维护受信任的证书链。使用中级证书主要是处于安全考虑，因为Root CA对应的私钥Root Key始终处理物理隔离并能够保障私钥的绝对安全，在中级CA证书的私钥发生泄漏后，可以通过Root CA吊销该证书，并重新签发新的中级证书密钥对。

7) 创建中级CA证书目录结构

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

8) 生成中级CA证书私钥

创建中级证书私钥，使用`AES-256`加密：

```
$ cd /root/ca
$ openssl genrsa -aes256 -out intermediate/private/intermediate.key.pem 4096
$ chmod 400 intermediate/private/intermediate.key.pem
```

9) 生成中级CA证书签名请求

使用`intermediate.key.pem`为生成中级CA证书签名请求，同时使用Root CA对证书进行签名。注意`Common Name`不能重复。如果使用中级CA签发服务器证书和客户端证书，需要指定配置文件`intermediate/openssl.cnf`：

```
$ cd /root/ca
$ openssl req -config intermediate/openssl.cnf -new sha256 -key intermediate/private/intermediate.key.pem -out intermediate/csr/intermediate.csr.pem
```

10) 生成中级CA证书

使用Root CA签发中级CA证书，并为其指定`v3_intermediate_ca`扩展属性签发证书。中级证书的有效期要短于根证书。确认使用Root CA的`openssl.cnf`签发中级证书：

```
$ cd /root/ca
$ openssl ca -config openssl.cnf -extenstions v3_intermediate_ca -days 3650 -notext -md sha256 -in intermediate/csr/intermediate.csr.pem -out intermediate/certs/intermediate.cert.pem
$ chmod 444 intermediate/certs/intermediate.cert.pem
```

> Note: `index.txt`是openssl的文件数据库，不要直接修改或删除该文件。

11) 验证中级CA证书内容

```
$ openssl x509 -noout -text -in intermediate/certs/intermediate.cert.pem
```

使用Root CA根证书验证中级证书的证书链信任关系，`OK`表示证书链受信任：

```
$ openssl verify -CAfile certs/ca.cert.pem intermediate/certs/intermediate.cert.pem
intermediate.cert.pem: OK
```

12) 创建证书链文件

当应用程序(Web浏览器)尝试验证签发的证书有效性时，会尝试验证中级CA证书和上级根证书。在证书链文件中，必须同时包含根证书和中级证书。因为客户端不知知道签发的这张私有的CA根证书，你可以在所有需要访问的客户端将Root CA导入到受信任的根证书办法机构，这样该CA签发的证书就完全受信任并且证书链文件中则只用指定中级证书即可。

```
$ cat intermediate/certs/intermediate.cert.pem certs/ca.cert.pem > intermediate/certs/ca-chain.cert.pem
$ chmod 444 intermediate/certs/ca-chain.cert.pem
```

#### 签发服务器证书和客户端证书

接下来我们将使用中级CA证书为服务器签发证书，用来加密服务器与客户端之间的数据通信。如果您在服务器使用openssl已经创建过了`csr`证书请求，则可跳过生成Server Key的过程。

1) 创建服务器证书私钥

在创建Root CA和intermediate CA使用了`4096bit`密码加密，服务器和客户端证书一般有效期为几年，可以使用`2048bit`加密。虽然`4096bit`密码要更安全，但在服务器进行TLS握手和数字签名验证时会降低访问速度，因此一般Web服务器还是建议使用`2048Bit`密码加密。如果您在创建私钥时设置了密码，则每次服务重启时都需要提供这一密码，因此不建议为服务器证书设置密码保护，可以使用`aes256`密码算法进行加密：

```
$ cd /root/ca
$ openssl genrsa -aes256 -out intermediate/private/r.wanglijie.cn.key.pem 2048
$ chmod 400 intermediate/private/r.wanglijie.cn.key.pem
```

2) 创建服务器证书请求csr

使用创建的服务器私钥生成签发证书请求`csr`文件，`csr`文件无需指定中级CA证书机构，但是必须指定`Common Name`，一般为域名或IP地址：

```
$ cd /root/ca
$ openssl req -config intermediate/openssl.cnf -new -sha256 -days 365 -key intermediate/private/r.wanglijie.cn.key.pem -out intermediate/csr/r.wanglijie.cn.csr.pem
```

2) 从证书请求`csr`创建服务器证书

```
$ cd /root/ca
$ openssl ca -config intermediate/openssl.cnf \
      -extensions server_cert -days 375 -notext -md sha256 \
      -in intermediate/csr/r.wanglijie.cn.cert.pem \
      -out intermediate/certs/r.wanglijie.cn.cert.pem 
$ chmod 444 intermediate/certs/r.wanglijie.cn.cert.pem
```

> Note: 在签发证书时，`-extensions`选项用于指定要签发的证书类型。服务器证书使用"server_cert"，客户端证书使用"usr_cert"。证书签发完毕后，将会在`intermediate/index.txt`文件中新增一条记录。

4) 查看签发的证书

```
$ openssl x509 -noout -text  -in intermediate/certs/r.wanglijie.cn.cert.pem
```

使用CA证书链(`ca-chain.cert.pem`)验证签发证书的信任关系：

```
$ openssl verify -CAfile intermediate/certs/ca-chain.cert.pem intermediate/certs/r.wanglijie.cn.cert.pem
```

5) 部署证书

在证书签发完毕后，可以将证书部署在`Nginx`, `Apache`或其他的应用服务器上。
