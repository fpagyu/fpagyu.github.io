---
layout: go
title: golang中的rsa
date: 2018-03-20 22:26:17
tags: go,rsa
---

### RSA

RSA 加密是一种非对称加密算法，使用公钥进行加密，私钥进行解密。因此，有的时候也称为公钥加密算法。当然有的情况下也使用RSA的私钥对数据进行签名，使用RSA公钥来对签名数据进行验签，这在一些开放api中经常使用。之前一直搞不清楚这两种情况有什么区别，加之对RSA的数学原理并不十分了解，导致在go中使用RSA时特别痛苦。当然这篇文章不会对RSA的原理做太多解释(如有需要可以参考[知乎回答]((https://www.zhihu.com/question/25912483/answer/31653639)))，只是就如何在go中使用RSA进行加密签名等操作做一个整理，同时对部分概念(比如加密和签名)进行区分。



[完整代码](https://gitee.com/fpgayu/codes/qkj3y2ubhgnroc1spa80t52)

### 在go中使用RSA加密/解密

简单理解：公钥就是公开的密钥，其公开了大家才能用它来加密数据。私钥是私有的密钥，谁有这个密钥才能够解密密文。否则大家都能看到私钥，就都能解密，那不就乱套了。

下面以代码为例，展示如何在go中使用rsa加密:

```go
func RSAEncrypt(src, key []byte) ([]byte, error) {
	// block 代表的是PEM编码的结构
	block, _ := pem.Decode(key)
	if block == nil {
		return nil, errors.New("public key error")
	}

	var pubInterface interface{}
	pubInterface, err := x509.ParsePKIXPublicKey(block.Bytes)
	if err != nil {
		return nil, err
	}

	var pub = pubInterface.(*rsa.PublicKey)

	// 根据EncryptPKCS1v15源码可以看到, len(src) < (pub.N.BitLen()-7)/8 -11
	// 所以需要根据src的长度进行分片处理
	chunkSize := pub.N.BitLen()/8 - 11

	var buf bytes.Buffer
	var i, j = 0, chunkSize
	for i < len(src) {
		if j > len(src) {
			j = len(src)
		}

		b, err := rsa.EncryptPKCS1v15(rand.Reader, pub, src[i:j])
		if err != nil {
			return nil, err
		}

		i, j = j, j+chunkSize
		buf.Write(b)
	}

	return buf.Bytes(), nil
}
```



rsa解密:

```go
func RSADecrypt(src, key []byte) ([]byte, error) {
	block, _ := pem.Decode(key)
	if block == nil {
		return nil, errors.New("private key error")
	}

	priv, err := x509.ParsePKCS1PrivateKey(block.Bytes)
	if err != nil {
		return nil, err
	}

	// priv.N.BitLen()/8 > len(src)
	var chunkSize = priv.N.BitLen() / 8
	var buf bytes.Buffer
	var i, j = 0, chunkSize

	for i < len(src) {
		if j > len(src) {
			j = len(src)
		}
		
		b, err := rsa.DecryptPKCS1v15(rand.Reader, priv, src[i:j])
		if err != nil {
			return nil, err
		}

		i, j = j, j+chunkSize
		buf.Write(b)
	}

    return buf.Bytes(), nil
}
```



### 在go中使用RSA签名/验证签名

简单理解：对一个文件签名，当然要用私钥，因为我们希望只有自己才能完成签字。验证过程当然希望所有人都能够执行，大家看到签名都能通过验证证明确实是我自己签的。

rsa 签名验证函数：

```go
func SignPKCS1v15(src, key []byte, hash crypto.Hash) ([]byte, error) {
	var h = hash.New()
	h.Write(src)
	var hashed = h.Sum(nil)

	var err error
	var block *pem.Block
	block, _ = pem.Decode(key)
	if block == nil {
		return nil, errors.New("private key error")
	}

	var pri *rsa.PrivateKey
	// 此处解析的私钥文件应该是以-----BEGIN RSA PRIVATE KEY开头的
  	// 以------BEGIN PRIVATE KEY开头的私钥文件则需要使用PKCS#8标准
  	// priInterface, err = x509.ParsePKCS8PrivateKey(block.Bytes)
  	// priv, err := priInterface.(*rsa.PrivateKey)
	pri, err = x509.ParsePKCS1PrivateKey(block.Bytes)
	if err != nil {
		return nil, err
	}
	return rsa.SignPKCS1v15(rand.Reader, pri, hash, hashed)
}

func VerifyPKCS1v15(src, sig, key []byte, hash crypto.Hash) error {
	var h = hash.New()
	h.Write(src)
	var hashed = h.Sum(nil)

	var err error
	var block *pem.Block
	block, _ = pem.Decode(key)
	if block == nil {
		return errors.New("public key error")
	}

	var pubInterface interface{}
	pubInterface, err = x509.ParsePKIXPublicKey(block.Bytes)
	if err != nil {
		return err
	}
	var pub = pubInterface.(*rsa.PublicKey)

	return rsa.VerifyPKCS1v15(pub, hash, hashed, sig)
}
```



### 小结

在go中使用rsa加密解密并没有什么疑惑的地方，关键在与对RSA的理解，x509, PKCS标准都明白，那么对调用哪一个库函数就不会有太的疑惑。之前就是因为不理解"BEGIN RSA PRIVATE KEY" 和 "BEGIN PRIVATE KEY"两种RSA私钥文件的区别，导致不清楚调用哪一个PKCS函数,走了不少弯路。

#### 参考引用

[知乎-RSA的公钥和私钥到底哪个才是用来加密和哪个用来解密](https://www.zhihu.com/question/25912483)