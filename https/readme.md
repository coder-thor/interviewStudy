# https

> https保证数据在传输过程中不被窃取和篡改

当我们没有HTTPS的时候, 一般传输数据的过程如下:

1. 客户端给服务端发送一个message说我要给客户端B转账100块钱
2. 但是在传输过程中因为是明文传输的, 被黑客截获以后, 他说: 诶 你要给客户端B转100块钱, 我给你改了, 你直接给我转10000块钱
3. 服务端接收到被篡改过后的请求, 于是给黑客赚了10000块钱, 用户直接一脸懵逼

为了避免这种情况的发生, 我们就得**加密**, 加密是干嘛呢, 就是黑客你可以截获, 但是你看不懂我说的是什么, 你看不懂你就没法改, 你可以拦截不发出去, 不发出去无所谓, 客户端也不会有什么损失

## 加密手段

加密分为对称加密和不对称加密

#### 对称加密

对称加密: 产生一个秘钥, 可以用其加密, 也可以用其解密, 常用算法有des, 3des, aes, blowfish等

有了对称加密以后, 上面的交互过程就会变成如下:

1. 在客户端具体请求前, 先请求服务端拿到这个秘钥key, 并保存在客户端
2. 每次客户端和服务端进行交互的时候, 客户端都会用这个秘钥key将行为和数据进行加密, 然后再发送给服务端
3. 服务端拿到请求以后, 用秘钥key进行解密, 拿到解密过后的数据, 将响应结果用密钥key进行加密发送给客户端, 客户端再用秘钥key解密得到响应结果

但是这样还有一个漏洞, 就是当第一次通信的那个秘钥key就被截获了, 那后续的请求对于黑客来说又是明文了, 是明文他又可以篡改了, 所以这样也是不太安全的

#### 非对称加密

非对称加密: 产生一对秘钥(公钥+私钥), 一个用于加密, 一个用于解密, 常用的算法有: rsa, elgamal, d-h等

使用非对称加密以后, 上面的交互过程就变成如下了

1. 在客户端发送具体请求前, 先请求服务端拿到公钥key用于加密, 并保存在客户端
2. 每次客户端和服务端进行交互时, 客户端都会用这个公钥key将行为和数据进行加密, 然后再发送给服务端
3. 服务端拿到请求以后, 用私钥key进行解密, 得到解密后的请求数据, 将响应结果用公钥key进行加密发送给客户端
4. 客户端没有私钥key, 导致解密失败, GG

直接使用非对称加密的话我们发现确实是防住了黑客进行解密, 但是好像也把客户端搞凉了


#### 非对称加密 + 对称加密组合

将两种加密方式相结合, 你看我怎么搞:

1. 在客户端发送具体请求前, 先请求服务端拿到公钥key用于加密, 并保存在客户端
2. 客户端本地生成一个对称加密的秘钥, 将该秘钥用公钥key进行加密, 并发送给服务端
3. 服务端拿到对应的对称加密的秘钥, 从此以后客户端和服务端进行交互都使用客户端生成的那个秘钥

但是这样就有一个风险, 你看黑客怎么搞: 
1. 直接拦截服务端的公钥key, 然后用自己的生成的公钥发送给客户端
2. 客户端收到黑客伪造的公钥, 将自己生成的对称加密的秘钥用假公钥进行加密发送出去
3. 该请求被黑客拦截, 黑客用自己的私钥解密得到客户端生成的秘钥, 然后装作无视发生, 用服务端的公钥加密发给服务端
4. 到此时此刻, 客户端的对称加密的秘钥已经在黑客手里了, 你说他是不是想干什么就干什么了

#### CA

到了这个时候, 好像就没有更好的方案了一样, 陷入死循环了, 于是, CA(Certificate Authority)证书颁发机构站出来了

当我们安装操作系统的时候, 在系统文件夹里就已经对世界上已知的所有CA**权威**机构进行了记录, 证书颁发机构主要就是用来颁发证书的, 如果有了CA的参与, 我们的交互就变成如下这样:

1. 服务端给CA交钱然后将自己的域名+生成的公钥给CA并申请一个CA颁发的证书, CA机构有一个自己的公钥（公钥全世界公开）和私钥, 这个证书包含如下的东西:
    - 服务器地址（明文存储）
    - 证书颁发机构
    - CA机构用自己的私钥把服务器给他的公钥进行加密, 这个加密后的东西也是证书的一份子
    - 使用CA私钥加密以后证书签名
        > 证书签名的算法一般是(服务器的域名 + CA公钥key + 服务端自己生成的公钥), 这个证书签名的算法也是公开的, 这样是为了防止一个情况, 就是在客户端第一次请求证书的时候就被篡改了, 所以客户端可以在拿到证书以后, 用自己的公钥和CA公钥加上自己的域名去找这个机构的证书签名算法, 如果一旦发现算法得出的结果不一样, 则是被篡改了
2. 客户端在交互前先获取证书, 然后验证证书签名
3. 由于证书中的服务器公钥, 证书签名都是通过CA的私钥进行加密的, 因此, 其他终端只能通过CA的公钥进行解密, 但是无法重新伪造, 因为伪造你得要CA的私钥啊家人们, 你这没有他的私钥你就只能看着
4. 于是在首次交互的时候, 服务端会将CA加密过后的公钥发送给客户端
5. 客户端用众所周知的CA公钥进行解密, 如果CA公钥解密失败, 则代表被篡改, 就不会做任何事, 解密完成以后得到服务端的公钥
6. 使用服务端公钥将自己本地产生的秘钥进行加密, 发送给客户端, 这时候黑客就算拦截了又怎样, 他没有服务端的私钥 他解不了密, 只能傻看着
7. 服务端和客户端后续使用客户端生成的秘钥进行交互, 整个过程天衣无缝