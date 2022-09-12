---
title: "[总结] 加解密算法 -《图解密码技术》"
date: 2020-10-04T22:26:00+08:00
draft: false
---

无论你是使用什么语言，都应该遇到过使用加解密的使用场景，比如接口数据需要加密传给前端保证数据传输的安全；HTTPS使用证书的方式首先进行非对称加密，将客户端的私匙传递给服务端，然后双方后面的通信都使用该私匙进行对称加密传输；使用MD5进行文件一致性校验，等等很多的场景都使用到了加解密技术。

这篇文章就来总结一下目前主流的加解密技术，我认为程序员应该需要了解的。既然是总结就不涉及具体算法分析。

日常使用的加解密大致可以分为以下四类：

- 散列函数(也称信息摘要)算法
- 对称加密算法
- 非对称加密算法
- 组合加密技术
  
# 1. 散列函数算法
听名字似乎不是一种加密算法，类似于给一个对象计算出hash值。所以这种算法一般用于数据特征提取。常用的散列函数包括：MD5、SHA1、SHA2（包括SHA128、SHA256等）散列函数的应用很广，散列函数有个特点，它是一种单向加密算法，只能加密、无法解密。

## 1.1 MD5
先来看MD5算法，MD5算法是广为使用的数据特征提取算法，最常见的就是我们在下载一些软件，网站都会提供MD5值给你进行校验，你可以通过MD5值是否一致来检查当前文件是否被别人篡改。MD5算法具有以下特点：

1. 任意长度的数据得到的MD5值长度都是相等的；
2. 对原数据进行任一点修改，得到的MD5值就会有很大的变化；
3. 散列函数的不可逆性，即已知原数据，无法通过特征值反向获取原数据。（需要说明的是2004年的国际密码讨论年会（CRYPTO）尾声，王小云及其研究同事展示了MD5、SHA-0及其他相关杂凑函数的杂凑冲撞。也就是说，她找出了第一个 两个值不同，但 MD5 值相同的碰撞的例子。这个应该不能称之为破解）
## 1.2 MD5用途：

1. 防篡改。上面说过用于文件完整性校验。
2. 用于不想让别人看到明文的地方。比如用户密码入库，可以将用户密码使用MD5加密存储，下次用户输入密码登录只用将他的输入进行MD5加密与数据库的值判断是否一致即可，这样就有效防止密码泄露的风险。
3. 用于文件秒传。比如百度云的文件秒传功能可以用这种方式来实现。在你点击上传的时候，前端同学会先计算文件的MD5值然后与服务端比对是否存在，如果有就会告诉你文件上传成功，即完成所谓的秒传。
   
在JDK中提供了MD5的实现：java.security包中有个类MessageDigest，MessageDigest 类为应用程序提供信息摘要算法的功能，如 MD5 或 SHA 算法。信息摘要是安全的单向哈希函数，它接收任意大小的数据，输出固定长度的哈希值。

MessageDigest 对象使用getInstance函数初始化，该对象通过使用 update 方法处理数据。任何时候都可以调用 reset 方法重置摘要。一旦所有需要更新的数据都已经被更新了，应该调用 digest 方法之一完成哈希计算。

对于给定数量的更新数据，digest 方法只能被调用一次。digest 被调用后，MessageDigest 对象被重新设置成其初始状态。

下面的例子展示了使用JDK自带的MessageDigest类使用MD5算法。如果使用了update方法后没有调用digest方法，则会累计当前所有的update中的值在下一次调用digest方法的时候一并输出：

```java

package other;

import java.security.MessageDigest;

/**
 * @author: rickiyang
 * @date: 2019/9/13
 * @description:
 */
public class MD5Test {

    static char[] hex = {'0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'A', 'B', 'C', 'D', 'E', 'F'};

    public static void main(String[] args) {
        try {
            //申明使用MD5算法
            MessageDigest md5 = MessageDigest.getInstance("MD5");
            md5.update("a".getBytes());//
            System.out.println("md5(a)=" + byte2str(md5.digest()));
            md5.update("a".getBytes());
            md5.update("bc".getBytes());
            System.out.println("md5(abc)=" + byte2str(md5.digest()));
            //你会发现上面的md5值与下面的一样
            md5.update("abc".getBytes());
            System.out.println("md5(abc)=" + byte2str(md5.digest()));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 将字节数组转换成十六进制字符串
     *
     * @param bytes
     * @return
     */
    private static String byte2str(byte[] bytes) {
        int len = bytes.length;
        StringBuffer result = new StringBuffer();
        for (int i = 0; i < len; i++) {
            byte byte0 = bytes[i];
            result.append(hex[byte0 >>> 4 & 0xf]);
            result.append(hex[byte0 & 0xf]);
        }
        return result.toString();
    }
}
```
输出：

```
md5(a)=0CC175B9C0F1B6A831C399E269772661
md5(abc)=900150983CD24FB0D6963F7D28E17F72
md5(abc)=900150983CD24FB0D6963F7D28E17F72
```

## 2 SHA系列算法

Secure Hash Algorithm，是一种与MD5同源的数据加密算法。

SHA算法能计算出一个数位信息所对应到的，长度固定的字串，又称信息摘要。而且如果输入信息有任何的不同，输出的对应摘要不同的机率非常高。

因此SHA算法也是FIPS所认证的五种安全杂凑算法之一。原因有两点：一是由信息摘要反推原输入信息，从计算理论上来说是极为困难的；二是，想要找到两组不同的输入信息发生信息摘要碰撞的几率，从计算理论上来说是非常小的。任何对输入信息的变动，都有很高的几率导致的信息摘要大相径庭。

SHA实际上是一系列算法的统称，分别包括：SHA-1、SHA-224、SHA-256、SHA-384以及SHA-512。后面4中统称为SHA-2，事实上SHA-224是SHA-256的缩减版，SHA-384是SHA-512的缩减版。各中SHA算法的数据比较如下表，其中的长度单位均为位：

|类别	| SHA-1	| SHA-224	| SHA-256	| SHA-384	| SHA-512|
|---|---|---|---|---|---|
|消息摘要长度	| 160	| 224	| 256	| 384	| 512|
|消息长度	| 小于264位	| 小于264位	| 小于264位	| 小于2128位	| 小于2128位|
|分组长度	| 512	| 512	| 512 | 	1024	| 1024|
|计算字长度	| 32	| 32	| 32	| 64| 	64|
|计算步骤数	| 80	| 64	| 64	| 80	| 80|

SHA-1算法输入报文的最大长度不超过264位，产生的输出是一个160位的报文摘要。输入是按512 位的分组进行处理的。SHA-1是不可逆的、防冲突，并具有良好的雪崩效应。


上面提到的MessageDigest类同时也支持SHA系列算法，使用方式与MD5一样，注意SHA不同的类型：

```
MessageDigest md = MessageDigest.getInstance("SHA");
MessageDigest md = MessageDigest.getInstance("SHA-224");
MessageDigest md = MessageDigest.getInstance("SHA-384");
```
# 2. 对称加密算法
所谓的对称加密，意味着加密者和解密者需要同时持有一份相同的密匙，加密者用密匙加密，解密者用密匙解密即可。

常用的对称加密算法包括DES算法、AES算法等。 由于对称加密需要一个秘钥，而秘钥在加密者与解密者之间传输又很难保证安全性，所以目前用对称加密算法的话主要是用在加密者解密者相同，或者加密者解密者相对固定的场景。

对称算法又可分为两类：

第一种是一次只对明文中的单个位（有时对字节）运算的算法称为序列算法或序列密码；

另一种算法是对明文的一组位进行运算，这些位组称为分组，相应的算法称为分组算法或分组密码。现代计算机密码算法的典型分组长度为64位――这个长度既考虑到分析破译密码的难度，又考虑到使用的方便性。

## 2.1 BASE64算法
我们很熟悉的BASE64算法就是一个没有秘密的对称加密算法。因为他的加密解密算法都是公开的，所以加密数据是没有任何秘密可言，典型的防菜鸟不防程序员的算法。

BASE64算法作用：

用于简单的数据加密传输；

用于数据传输过程中的转码，解决中文问题和特殊符号在网络传输中的乱码现象。

网络传输过程中如果双方使用的编解码字符集方式不一致，对于中文可能会出现乱码；与此类似，网络上传输的字符并不全是可打印的字符，比如二进制文件、图片等。Base64的出现就是为了解决此问题，它是基于64个可打印的字符来表示二进制的数据的一种方法。

BASE64原理

BASE64的原理比较简单，每当我们使用BASE64时都会先定义一个类似这样的数组：

```
['A', 'B', 'C', ... 'a', 'b', 'c', ... '0', '1', ... '+', '/']
```
上面就是BASE64的索引表，字符选用了"A-Z、a-z、0-9、+、/" 64个可打印字符，这是标准的BASE64协议规定。在日常使用中我们还会看到“=”或“==”号出现在BASE64的编码结果中，“=”在此是作为填充字符出现。

JDK提供了BASE64的实现：BASE64Encoder，我们可以直接使用：

```java
//使用base64加密
BASE64Encoder encoder = new BASE64Encoder();  
String encrypt = encoder.encode(str.getBytes());  
//使用base64解密
BASE64Decoder decoder = new BASE64Decoder();  
String decrypt = new String(decoder.decodeBuffer(encryptStr));  
```

## 2.2 DES
DES (Data Encryption Standard)，在很长时间内，许多人心目中“密码生成”与DES一直是个同义词。

DES是一个分组加密算法，典型的DES以64位为分组对数据加密，加密和解密用的是同一个算法。它的密钥长度是56位（因为每个第8 位都用作奇偶校验），密钥可以是任意的56位的数，而且可以任意时候改变。

DES加密过程大致如下：

1. 首先需要从用户处获取一个64位长的密码口令，然后通过等分、移位、选取和迭代形成一套16个加密密钥，分别供每一轮运算中使用；
2. 然后将64位的明文分组M进行操作，M经过一个初始置换IP，置换成m0。将m0明文分成左半部分和右半部分m0 = (L0，R0)，各32位长。然后进行16轮完全相同的运算（迭代），这些运算被称为函数f，在每一轮运算过程中数据与相应的密钥结合；
3. 在每一轮迭代中密钥位移位，然后再从密钥的56位中选出48位。通过一个扩展置换将数据的右半部分扩展成48位，并通过一个异或操作替代成新的48位数据，再将其压缩置换成32位。这四步运算构成了函数f。然后，通过另一个异或运算，函数f的输出与左半部分结合，其结果成为新的右半部分，原来的右半部分成为新的左半部分。将该操作重复16次；
4. 经过16轮迭代后，左，右半部分合在一起经过一个末置换（数据整理），这样就完成了加密过程。

对于DES解密的过程大家猛然一想应该是使用跟加密过程相反的算法，事实上解密和加密使用的是一样的算法，有区别的地方在于加密和解密在使用密匙的时候次序是相反的。比如加密的时候是K0,K1,K2......K15，那么解密使用密匙的次序就是倒过来的。之所以能用相同的算法去解密，这跟DES特意设计的加密算法有关，感兴趣的同学可以深入分析。

## 2.3 AES
高级加密标准(AES,Advanced Encryption Standard)，与DES一样，使用AES加密函数和密匙来对明文进行加密，区别就是使用的加密函数不同。

上面说过DES的密钥长度是56比特，因此算法的理论安全强度是2^56。但以目前计算机硬件的制作水准和升级情况，破解DES可能只是山脉问题，最终NIST(美国国家标准技术研究所（National Institute of Standards and Technology))选择了分组长度为128位的Rijndael算法作为AES算法。

AES为分组密码，分组密码也就是把明文分成一组一组的，每组长度相等，每次加密一组数据，直到加密完整个明文。在AES标准规范中，分组长度只能是128位，也就是说，每个分组为16个字节（每个字节8位）。密钥的长度可以使用128位、192位或256位。密钥的长度不同，推荐加密轮数也不同，如下表所示：

AES	密钥长度（32位比特字)	分组长度(32位比特字)	加密轮数
AES-128	4	4	10
AES-192	6	4	12
AES-256	8	4	14

# 3. 非对称加密
   
非对称加密算法的特点是，秘钥一次会生成一对，其中一份秘钥由自己保存，不能公开出去，称为“私钥”，另外一份是可以公开出去的，称为“公钥”。

将原文用公钥进行加密，得到的密文只有用对应私钥才可以解密得到原文；

将原文用私钥加密得到的密文，也只有用对应的公钥才能解密得到原文；

因为加密和解密使用的是两个不同的密钥，所以这种算法叫作非对称加密算法。



与对称加密算法的对比
优点：其安全性更好，对称加密的通信双方使用相同的秘钥，如果一方的秘钥遭泄露，那么整个通信就会被破解。而非对称加密使用一对秘钥，一个用来加密，一个用来解密，而且公钥是公开的，秘钥是自己保存的，不需要像对称加密那样在通信之前要先同步秘钥。

缺点：非对称加密的缺点是加密和解密花费时间长、速度慢，只适合对少量数据进行加密。
在非对称加密中使用的主要算法有：RSA、Elgamal、ESA、背包算法、Rabin、D-H、ECC（椭圆曲线加密算法）等。不同算法的实现机制不同。

非对称加密工作原理
下面我们就看一下非对称加密的工作原理。

1. 乙方生成一对密钥（公钥和私钥）并将公钥向其它方公开。
2. 得到该公钥的甲方使用该密钥对机密信息进行加密后再发送给乙方。
3. 乙方再用自己保存的另一把专用密钥（私钥）对加密后的信息进行解密。乙方只能用其专用密钥（私钥）解密由对应的公钥加密后的信息。
4. 在传输过程中，即使攻击者截获了传输的密文，并得到了乙的公钥，也无法破解密文，因为只有乙的私钥才能解密密文。同样，如果乙要回复加密信息给甲，那么需要甲先公布甲的公钥给乙用于加密，甲自己保存甲的私钥用于解密。

非对称加密鼻祖：RSA

RSA算法基于一个十分简单的数论事实：将两个大质数（素数）相乘十分容易，但是想要对其乘积进行因式分解却极其困难，因此可以将乘积公开作为加密密钥。比如：取两个简单的质数：67，73，得到两者乘积很简单4891；但是要想对4891进行因式分解，其工作量成几何增加。

应用场景：

HTTPS请求的SSL层。


# 4. 组合加密
上面介绍的3种加密技术，每一种都有自己的特点，比如散列技术用于特征值提取，对称加密速度虽快但是有私匙泄露的危险，非对称加密虽然安全但是速度却慢。基于这些情况，现在的加密技术更加趋向于将这些加密的方案组合起来使用，基于此来研发新的加密算法。

MAC（Message Authentication Code，消息认证码算法）是含有密钥散列函数算法，兼容了MD和SHA算法的特性，并在此基础上加上了密钥。因此MAC算法也经常被称作HMAC算法。MAC（Message Authentication Code，消息认证码算法）是含有密钥散列函数算法，HMAC加密可以理解为加盐的散列算法，此处的“盐”就相当于HMAC算法的秘钥。

HMAC算法的实现过程需要一个加密用的散列函数（表示为H）和一个密钥。

经过MAC算法得到的摘要值也可以使用十六进制编码表示，其摘要值得长度与实现算法的摘要值长度相同。例如 HmacSHA算法得到的摘要长度就是SHA1算法得到的摘要长度，都是160位二进制数，换算成十六进制的编码为40位。


过程如下：

1. 在密钥key后面添加0来创建一个长为B(64字节)的字符串（str）；
2. 将上一步生成的字符串(str) 与ipad（0x36）做异或运算，形成结果字符串（istr）;
3. 将数据流data附加到第二步的结果字符串(istr)的末尾；
4. 做md5运算于第三步生成的数据流(istr)；
5. 将第一步生成的字符串(str) 与opad（0x5c）做异或运算，形成结果字符串（ostr），再将第四步的结果(istr) 附加到第五步的结果字符串（ostr）的末尾做md5运算于第6步生成的数据流（ostr），最终输出结果(out)

注意：如果第一步中，key的长度klen大于64字节，则先进行md5运算，使其长度klen = 16字节。

JDK中的实现：

```java
public static void jdkHmacMD5() {
    try {
        // 初始化KeyGenerator
        KeyGenerator keyGenerator = KeyGenerator.getInstance("HmacMD5");
        // 产生密钥
        SecretKey secretKey = keyGenerator.generateKey();
        // 获取密钥
        byte[] key = secretKey.getEncoded();
        //            byte[] key = Hex.decodeHex(new char[]{'1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e'});
        // 还原密钥
        SecretKey restoreSecretKey = new SecretKeySpec(key, "HmacMD5");
        // 实例化MAC
        Mac mac = Mac.getInstance(restoreSecretKey.getAlgorithm());
        // 初始化MAC
        mac.init(restoreSecretKey);
        // 执行摘要
        byte[] hmacMD5Bytes = mac.doFinal("data".getBytes());
        System.out.println("jdk hmacMD5:" + new String(hmacMD5Bytes));
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```