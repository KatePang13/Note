帮助你的邮箱域抵御欺诈，钓鱼，垃圾邮件

## 邮件身份验证的最佳实践

建议你长期为你的域建立以下的验证措施

- SPF 帮助 收件服务器 验证 从一个域名发来的邮件是否是从域名所有者授权的服务器发出的

- DKIM 为每封邮件添加一个电子签名，收件服务器可以通过这个电子签名，来验证 一封邮件是否是伪造的，在传输过程中是否被篡改
- DMARC 加强了 SPF 核 DKIM 的验证，它允许管理员获取 邮件验证 和 邮件投递 的报告



## 保证邮件投递 & 防止域名欺诈  （SPF）

SPF，即 Sender Policy Framework, 是一种 email 身份验证方法，为你的邮箱域 指定 发件服务器。

SPF 保护你的邮箱域不被伪造发件，并且确保你外投的邮件能够正确地投递。当目的服务器收到你的邮箱域发送的邮件时，使用SPF来验证来源是否是真实的。

### SPF 的 2个主要的功能

- 防止邮件欺诈

  - 垃圾邮件服务器可以伪造你的域名或组织来发送虚假邮件，这种方式叫 spoofing,即邮件欺诈。伪造的邮件可以用以 恶意的意图，比如传递错误的消息，传播有害的软件，或者骗取敏感信息等。SPF帮助接收服务器验证从您的域发送的邮件确实是从您的组织发送的，并且是由您授权的邮件服务器发送的。

- 保证邮件正确投递到收件人的收件箱

  - SPF 防止从你的域外投的邮件被扔到垃圾箱。如果你的域没有使用SPF，收件服务器无法验证来自你的域的邮件是否真的是你发的。收件服务器可能会将 合法的邮件丢到收件人的垃圾箱中，甚至拒收

  

### 部署 SFP



## 增强外投邮件的安全性 (DKIM)

使用  域密钥标识邮件( [DomainKeys Identified Mail (DKIM) standard](http://www.dkim.org/) ) ，可以加强对 域名被伪造发件的防范。

邮件伪造，一个邮件实际上不是从一个域名发送出来的，但是内容伪造成是来自这个域名的。伪造是一种常见的未经授权的电子邮件使用方式，因此某些电子邮件服务器需要DKIM来防止电子邮件伪造。

DKIM 在 外投的邮件的邮件头上添加一个 加密的电子签名，收件服务器收到后，使用DKIM 解码文件投，并确认邮件是否被篡改。

### DKIM 签名 

#### 一个签名长什么样

```
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed; d=asuswebstorage.com; s=default; t=1572282571; bh=NFzBvJ/pEmf+yUHDd/Y7dYNH9pE+Bx6o95KcxhwFL78=; h=From:To:Subject:From; b=QwgINKqwcBu3GbeWm2Be81qXks6Pq9yMmDZl9C6mT8moXVBeokpEmDN+0RyZFiOmNH30kbe6HbS2lY3b1Pf726UH/V/0VAH0nigTuir4TWdN/IUePV+goQdEJ2+sDQ1fHlVjyyJCRwCiFiZpBIjhTBNN0vrgNJZ/gSLLOvq6k3s=

```

这里我们找了一个常规的DKIM 签名， 它包含了以下tag:

- `v=1` – the version (always equals to 1)
- `a=` – a signing algorithm used for the creation of a DKIM record
- `c=` – a canonicalization algorithm for the header and the body
- `d=` – a domain where the DKIM is signed
- `s=` – a DKIM selector
- `t=` – a timestamp of when the email was signed
- `bh=` – a hashed email body
- `h=` – a list of headers
- `b=` – a digital signature 

创建一个 DKIM 签名，只需要指定2个tag

- 一个认证的域名  `d=`

- 一个 选择器  `s=`  

  (选择器的用途是，当一个域名要发送多种类型的邮件时，可以为每个类型指定一个选择器，便于区分和管理)

  

#### 一个签名是如何工作的

指定 域名和选择器， 生成一个domain key，这是一个密钥对，包含公钥和私钥。

- 公钥添加到DNS TXT 记录上

- 私钥放在  域名所在的 MTA 上，用于给每个外投邮件写签名

  

#### dkim domain key 生成工具

市面上有很多生成工具可供选择

- [DKIM Core](https://dkimcore.org/tools/keys.html) – the selector is assigned automatically.
- [DKIM Generation Wizard by SocketLabs](https://www.socketlabs.com/domainkey-dkim-generation-wizard/) – allows you to assign a selector and generate 1024 and 2048 bit key pairs.
- [DKIM Wizard by SparkPost](https://www.sparkpost.com/resources/tools/dkim-wizard/) – allows you to assign a selector and generate 1024 and 2048 bit key pairs. Previously known as **Port25**.
- [DKIM Record Generator by Easy DMARC](https://easydmarc.com/tools/dkim-record-generator) – allows you to assign a selector and lookup DKIM. 
- [DKIM Wizard by Unlock The Inbox](https://www.unlocktheinbox.com/dkimwizard/) – allows you to assign a selector and generate 512, 768 (keys smaller than 1024 bits are subject to off-line attacks), 1024, and 2048 bit key pairs.
- [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) – an installable tool for generating public-private key pairs on Windows and Linux.
- [ssh-keygen](https://www.ssh.com/ssh/keygen/) – an installable tool for generating public-private key pairs on Linux.

使用某些工具，您可以生成2048位域密钥。它们比1024位的安全性更高。但是，只有在您的DNS系统支持它们的情况下，您才能使用它们。



### 部署 DKIM

这里我们使用  https://www.sparkpost.com/resources/tools/dkim-wizard/ 来生成 dkim key

如下所示，只需要指定  域名，选择器， 指定大小，然后 CREATE KEYS 就可以了。

![image-20201021222446821](C:\Users\liuyh\AppData\Roaming\Typora\typora-user-images\image-20201021222446821.png)

生成如下的一组密钥对：

```
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDgGaoEm5Uujx/NtWUGAgaPQ0Kd
/uD0Gs2hTJHRQ6KKSd7LykdoAFt/31nEZONw0cuQrCiWpYajolEyfVVJYm8np2qq
JLZSdIUFDOUQdHzhgxZy8KQ8RS02DBTh3o2H35iROaHu78FMUm1y4cRq9qyXUW0Q
P/DDU7tdaFSEyXlHIwIDAQAB
-----END PUBLIC KEY-----
-----BEGIN RSA PRIVATE KEY-----
MIICXQIBAAKBgQDgGaoEm5Uujx/NtWUGAgaPQ0Kd/uD0Gs2hTJHRQ6KKSd7Lykdo
AFt/31nEZONw0cuQrCiWpYajolEyfVVJYm8np2qqJLZSdIUFDOUQdHzhgxZy8KQ8
RS02DBTh3o2H35iROaHu78FMUm1y4cRq9qyXUW0QP/DDU7tdaFSEyXlHIwIDAQAB
AoGAcYkvFQSJ8Tu73ilflEqkbiKido9yAtotgeHcIoxEphFE2jSSNsOvl7pdrV17
yWXQ32wJaEFWVELhJlZPRk2jiB2+iY4SliFfsijTwZt6ynCxXSctfgRMiJJP4OEM
5M9oxVIsBwJzYo3RTUobhnylXokPJWHkpd208rNJHL++/ZkCQQD297mpGgY6qKPo
8OolYeCOEOoisxiFU2wOqyAmL7GdQN5laXOrcofaMFH6A2cQ6E1o8ptxksfzY+MF
dVtRK9zfAkEA6EvWw8q2Fi38vCEoe0AuxF9CMNXEYlxoWYu/0PqloQDWjulT5yCl
NBcENnH1YMSQm1swL33ZjpDC5fn8PkkaPQJAPjg1FyxOS3L3MJWZd+eLyl7qjelv
EQ/uVle4lsZHSiXwob4KfTQyk76+uG0pBzJvZjRRAzEGnQQaSuLBKdcSIwJBANX0
/FQMAti89Lsm401aWXj/sEygqChcqrRHlp5aLnHz/qtU17XbiK5IwNWQ8wx1ICgn
vmMPzHGWfh0qup134ZUCQQCzGWi8+hn2xkOA/cXgC72EhuHdOL5Ce1zRqFWNkY1Z
YoiaAKDZef6qGF7SpGTrlepb6PZTyobfvX5Pr4Zr79XK
-----END RSA PRIVATE KEY-----
```

#### Step 1:  DNS服务器上配置 公钥

你需要添加在DNS服务器上，为  test._domainkey.sofia7.com 以下内容 

注意：这里 是 为 选择器 添加 记录，而不是为域名添加记录

```
test._domainkey.sofia7.com IN TXT
“v=DKIM1\; k=rsa\; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDgGaoEm5Uujx/NtWUGAgaPQ0Kd/uD0Gs2hTJHRQ6KKSd7LykdoAFt/31nEZONw0cuQrCiWpYajolEyfVVJYm8np2qqJLZSdIUFDOUQdHzhgxZy8KQ8RS02DBTh3o2H35iROaHu78FMUm1y4cRq9qyXUW0QP/DDU7tdaFSEyXlHIwIDAQAB”
(以上是BIND DNS服务器的语法, 其他DNS服务器分号 ; 可能不需要转移字符)
```

配置完DNS记录后，你可以使用这个链接来检查这个选择器记录的状态
http://www.dnswatch.info/dns/dnslookup?la=en&host=test._domainkey.sofia7.com&type=TXT&submit=Resolve

#### Step 2:  密钥保存到 你的 MTA 服务器上

你可以创建一个新文件，比如 test.sofia7.com.pem，并把以下内容拷贝到文件中。

```
-----BEGIN RSA PRIVATE KEY-----
MIICXQIBAAKBgQDgGaoEm5Uujx/NtWUGAgaPQ0Kd/uD0Gs2hTJHRQ6KKSd7Lykdo
AFt/31nEZONw0cuQrCiWpYajolEyfVVJYm8np2qqJLZSdIUFDOUQdHzhgxZy8KQ8
RS02DBTh3o2H35iROaHu78FMUm1y4cRq9qyXUW0QP/DDU7tdaFSEyXlHIwIDAQAB
AoGAcYkvFQSJ8Tu73ilflEqkbiKido9yAtotgeHcIoxEphFE2jSSNsOvl7pdrV17
yWXQ32wJaEFWVELhJlZPRk2jiB2+iY4SliFfsijTwZt6ynCxXSctfgRMiJJP4OEM
5M9oxVIsBwJzYo3RTUobhnylXokPJWHkpd208rNJHL++/ZkCQQD297mpGgY6qKPo
8OolYeCOEOoisxiFU2wOqyAmL7GdQN5laXOrcofaMFH6A2cQ6E1o8ptxksfzY+MF
dVtRK9zfAkEA6EvWw8q2Fi38vCEoe0AuxF9CMNXEYlxoWYu/0PqloQDWjulT5yCl
NBcENnH1YMSQm1swL33ZjpDC5fn8PkkaPQJAPjg1FyxOS3L3MJWZd+eLyl7qjelv
EQ/uVle4lsZHSiXwob4KfTQyk76+uG0pBzJvZjRRAzEGnQQaSuLBKdcSIwJBANX0
/FQMAti89Lsm401aWXj/sEygqChcqrRHlp5aLnHz/qtU17XbiK5IwNWQ8wx1ICgn
vmMPzHGWfh0qup134ZUCQQCzGWi8+hn2xkOA/cXgC72EhuHdOL5Ce1zRqFWNkY1Z
YoiaAKDZef6qGF7SpGTrlepb6PZTyobfvX5Pr4Zr79XK
-----END RSA PRIVATE KEY-----
```

#### Step 3:  在你的MTA上引用密钥，用以 签名

1. 使用 开源  MTA  Postfix + opendkim 方案，验证 dkim 是否可用

2. 目前我们的MTA还未支持该功能，后续准备参考 [opendkim](https://sourceforge.net/projects/opendkim/) 的实现。

## Reference:

https://support.google.com/a/topic/9061731?hl=en&ref_topic=9202

https://blog.mailtrap.io/create-dkim-tutorial/

https://sourceforge.net/projects/opendkim/