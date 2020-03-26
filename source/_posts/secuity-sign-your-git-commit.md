---
title: "安全故事会 - 第二期：Sign Your Work"
date: 2020-03-26T05:54:46Z
tags: ["安全故事会"]
categories: ["安全故事会"]
author: "Chen Xiyu"
---
原文发表于: https://blog.94xychen.net/posts/sign-your-git-commit/

最近在Github 上发现一个有意思的功能: 如果一个提交被作者签名了, 并且签名可被验证的话, 提交上会显示一个绿色的`Verified`的标志.如下图所示:

![](/images/secuity-sign-your-git-commit/1.png)
这篇文章, 我们就来聊聊为什么要签名你的提交以及如何去做.
## 什么是签名?
在真正的进入正文之前, 我想先简单的普及一下签名这个密码学的概念, 有些同学听到密码学这个词就觉得好复杂而心生畏惧, 其实大可不必, 如果只是应用的话, 你大可不需要了解每一个算法的细节, 你只需要知道他们的特点以及应用场景就完全足够了, 这里我就会以一个非常简化的模型来解释数字签名是如何实现的.

签名, 在字面意思上就是给某个东西署上名字, 而表面这个内容我们是同意的, 或者说, 这就是我写的. 在现实生活中, 我们常用的签名方式就是1)签字, 2)摁手印. 当我们通过这种方式来给一个东西签名以后, 别人就可以通过字迹以及指纹比对的方式来确认这个东西到底是不是我签署的.

而在现实生活中, 对签名的校验(字迹以及指纹比对)通常需要权威机构来做, 而在数字世界, 我们通常没有一个权威机构来帮我们校验某个东西是否真的是来自于某个人的, 于是, 采用密码学技术的签名就派上用场了. 在数字世界里, 数字签名的作用和现实生活中还是一模一样的, 只是我们实现的技术有不同而已.

下面我们就来看一下, 密码学工具如何做到的吧.
首先, 我们先要从密码学工具箱中挑出几件工具出来, 在我接下来构想的简单模型中, 数字签名会用到的工具有两种: hash散列算法(数字世界的指纹), 非对称密码. 我们接下来看一下他们的特性, 具体的更多细节还请自行学习.

### hash散列算法
hash 散列算法可以将一个任意长度的内容转化成一个固定长度的内容,(不同的算法实现有不同的长度, 我们假设是64个字节),我们称它为内容的指纹(finger print). 而且没有办法没有办法直接通过转换后的内容直接推算出原内容.

并且, hash算法都有雪崩效应, 也就是说, 即使是改动原内容的一个bit, 生成出来的指纹将会有很大的差别.
用它, 我们可以校验内容的完整性.

### 非对称密码算法
相对于对称密码算法, 非对称密码算法的密钥有一对而不是一个.
它的特性主要是: 用其中一个密钥加密的内容, 不能通过同一个密钥解密, 只能通过相对应的另一个密钥来解密, 反之亦然(当然, 不同的算法有不同的特性.).

于是, 我们可以将这一对密钥区分成公钥和私钥, 公钥直接放在互联网上, 而私钥自己保管好(这是关键, 私钥就代表你自己), 如果别人需要和你进行加密通信, 就可以直接用你的公钥加密, 而不需要像对称密码算法一样先和你交换密钥(当然, 这里又涉及到了公钥可行度的问题, 密码学通常通过证书来解决. 而且, 由于非对称密码算法的性能还是不如对称密码算法, 所以非对称密码在实际的加解密过程中通常扮演安全密钥交换通道的作用, 密码学有太多的东西可以讲, 以后有机会再说.).

### 简单的签名实现
![](/images/secuity-sign-your-git-commit/2.jpeg)
我们先有请密码学中的李雷和韩梅梅: Alice 和 Bob. 假设Alice 想给Bob 发一条消息说: `Hello Bob. Do you have time tonight?`. 如果Bob 收到这条消息, 他怎么知道这条消息就是Alice 发的呢? 万一是Bob的情敌要骗他出来干仗呢? 于是, Alice 做了两件事:
1. 用hash散列算法计算出了这条消息的指纹: `c01228362b0b8f707c018fe24cca6ac179e2619d1fcfa47cdd19fa1235feb251`
2. 用自己的私钥给这条指纹加密: `ZmI3NmY3M2M4ZTQ3YmRlMGE3ZDI1ZGM2MjViOGUzNDg=`

然后将加密后的指纹随同那条消息一同发送给了Bob.

那么Bob如何就能知道这条消息就是Alice 发出来的呢? Bob 只需要做这几件事:
1. 用Alice 的公钥解密指纹密码(非对称密码算法的特性), 得到: `c01228362b0b8f707c018fe24cca6ac179e2619d1fcfa47cdd19fa1235feb251`
2. 用于Alice 相同的hash 算法对Alice 发送过来的内容进行计算, 得到hash指: `c01228362b0b8f707c018fe24cca6ac179e2619d1fcfa47cdd19fa1235feb251`

如果两者一样, 证明这条消息是Alice发出来的, 如果不是, 要么内容被篡改, 要么不是Alice 发出来的.

这样, 一个简易的签名和验证就完成了, 本质上, 签名的验证就是验证发布者是否持有某个私钥, 这个私钥就代表你自己, 所以私钥的保管至关重要!

## 为什么要给Commit 签名?
在了解了签名的作用以后, 我相信给git Commit签名的原因就已经很明了了: 别人可以验证这个commit 真的是你提交的, 而不是:
* 某个盗用了你的Access token的人去提交的.
* 某个盗用了你Github/Github enterprise账号的人去提交的.
* 某个同事改了你的代码, 并且 `--amend` 了你的提交, 并且`force push`了你的分支(强烈不建议这么做, 因为可能有生命危险)

总而言之, 给Commit 签名, 可以让别人验证所有的这个工作, 来自于一个可信可验证的来源.

## 如何给Commit 签名?
有两种方式可以给你的Commit 签名:
1. 在Github 上面编辑的代码, Github会自动的签名(由此可推测, Github 给每个用户都生成了一个公私钥对, 而且还没有给我们暴露出来)
2. 在自己的机器上用Git 在提交时进行签名, 并且让Github 可验证.

第一种方式不需要解释, 我们看一下第二种方式如何操作:
### 工具准备
首先我们需要有一个自己的非对称密钥对. 我们这里使用gpg 这个工具来生成和管理我们的密钥对.(gpg 是GNU Privacy Guard的缩写, 是常用的加密/解密/签名/校验等密码学操作的命令行工具, 更多详情以后介绍.)

安装GPG:
```Bash
brew install gnupg
```
安装完以后, 我们需要写一个配置到我们的shell 配置中, 不然的话, gpg 不能正常的弹出密码询问框而报错:
```Bash
echo 'export GPG_TTY=$(tty)' >> ~/.your_shell_config # For bash/zsh user. config are usually .bashrc/.zshrc
echo 'set -x GPG_TTY (tty)' >> ~/.config/fish//config.fish # For fish user
```

### 生成密钥对
安装好gpg以后, 用gpg 生成密钥对.
```Bash
gpg --full-generate-key
```
它会弹出一系列问题让你输入, 如实写入就好啦.

生成完了以后, 你就可以用如下命令查看你的密钥了:
```Bash
gpg --list-secret-keys --keyid-format LONG
```
输出类似于:
```
➜  ~ gpg --list-secret-keys --keyid-format LONG
/Users/xiyuchen/.gnupg/pubring.kbx
----------------------------------
sec   rsa4096/507BB1CAC6286AF9 2020-02-16 [SC]
      1888037BDEAA06CFDE3117FE507BB1CAC6286AF9
uid                 [ultimate] ninety-four-xychen <94xychen@gmail.com>
ssb   rsa4096/F73836ED174C17A7 2020-02-16 [E]
```
至于如何更好的管理你的公私钥(备份和导入等)以后有机会在专题介绍.

### 签名你的提交.
git commit 命令给我们提供了利用gpg来签名commit的选项: `-S[<keyid>], --gpg-sign[=<keyid>]`, 我们可以在写提交代码的时候加上`-S<keyid>` 来签名你的提交:
```
git commit -S507BB1CAC6286AF9 -m 'commit message'
```

到这一步, git签名就已经完成了, 但是, 每个提交都要写-S 加 keyid 还是有些麻烦的, 我们通过修改git 的配置(配置哪个层级自己选择, 我选择的是全局)来让git自动签名每一个提交:

```Bash
$ git config --global user.signingkey 507BB1CAC6286AF9
$ git config --global commit.gpgsign true
```

### 让Github可验证
到这一步, 如果我们直接将提交推送到Github上, 提交上将会出现一个`Unverified`的标签, 这是因为, 虽然我们给提交签名了, 但是, Github还是不知道这个签名到底来自于谁. 于是, 我们就要告诉Github: 我们这个用户所对于的公钥是哪个.

这样, Github 用这个公钥校验完提交以后, 就可以说, 这个提交就是这个用户提交的了.

首先, 我们要导出我们的公钥:
```Bash
gpg --export -a <keyid>
```
输出类似:
```
-----BEGIN PGP PUBLIC KEY BLOCK-----

mQINBF5I7cwBEACxmyzZXpE8ldOqSV+RwPW3FyEj2pPY46kMbWHdbyGlm4Q2phUv
ZSYwxQTj8+MncpPQi3LUjH+VDpq9dwPzlKRVqiBrXZ4z1vjQV3YBk9cwloASLDCW
.....
X0KKdfAF6kIwIWe1jFXg76rNKly/PMj0E1kuUfTe7hHJWa/II8cloEhWSmSiuVum
do90
=syTd
-----END PGP PUBLIC KEY BLOCK-----
```
在Github 的这个页面将public key的内容上传上去:
![](/images/secuity-sign-your-git-commit/3.png)

恭喜你, 到这里, 你的提交就是`Verified`了.
你就可以像我一样帅气的拥有全Verified 提交记录了, ( •́ὤ•̀)你酸了没?.
![](/images/secuity-sign-your-git-commit/4.png)

## Advanced Tips
### 缓存密码
熟话说, 安全与便利不可兼得, 在这种情况下也不例外,

当我们配置完上面的一切以后. 提交的时候, gpg 总会弹出一个密码询问框, 让你输入你创建密钥对时使用的密码, 去解开加密保存的私钥, 从而使用私钥去签名.

如果你用了像1password这样的密码管理工具的话, 你还可以很便利的用快捷键调出密码管理界面, 找到你的私钥解密密码, 复制粘贴.

但是, 如果你没有用这种方式, 每次提交还要去输入密码还是挺痛苦的, 不然的话很多人就会设置一个弱密码...这里还是强烈建议用密码管理工具生成强密码, 并且管理密码的.

在设置了强密码的前提下, 我们可以稍微的牺牲一些安全性, 通过配置gpg-agent的 `default-cache-ttl`, 让我们解密后的私钥在内存中存在的时间稍微长一些(默认10分钟), 比如, 一天:
```
# ~/.gnupg/gpg-agent.conf

default-cache-ttl-ssh 86400
max-cache-ttl-ssh 86400
```
话说回来, 安全于便利始终是个取舍.
### 在本地查看签名
```bash
git log --show-signature
```

输出类似于:
```
commit e412a1f98b4ddc34590d7f773c5c09c5326ca62c (HEAD -> master)
gpg: Signature made Sun Feb 23 16:54:34 2020 CST
gpg:                using RSA key 1888037BDEAA06CFDE3117FE507BB1CAC6286AF9
gpg: Good signature from "ninety-four-xychen <94xychen@gmail.com>" [ultimate]
Author: ninety-four-xychen <94xychen@gmail.com>
Date:   Sun Feb 23 15:56:31 2020 +0800

    add new post: secuity-sign-your-git-commit

commit 87e8443f9880feb3d56d0bf62ed171b6c0e2d10f
gpg: Signature made Sun Feb 23 13:54:04 2020 CST
gpg:                using RSA key 1888037BDEAA06CFDE3117FE507BB1CAC6286AF9
gpg: Good signature from "ninety-four-xychen <94xychen@gmail.com>" [ultimate]
Author: ninety-four-xychen <94xychen@gmail.com>
Date:   Sun Feb 23 13:54:04 2020 +0800

    Discards building step on pipeline.

commit f182369d56f5fc8506ea843b9e1ae25fd552a60b (origin/master)
gpg: Signature made Wed Feb 19 22:46:44 2020 CST
gpg:                using RSA key 1888037BDEAA06CFDE3117FE507BB1CAC6286AF9
gpg: Good signature from "ninety-four-xychen <94xychen@gmail.com>" [ultimate]
Author: ninety-four-xychen <94xychen@gmail.com>
Date:   Wed Feb 19 22:46:44 2020 +0800

    Fixed an appearance issue on landing page.
```
