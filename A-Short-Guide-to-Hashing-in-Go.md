# A Short Guide to Hashing in Go


>How to hash a string or file

散列函数是现代密码学最重要的特征之一。

因为我打算学习 GO， 所以为了乐趣，何不尝试在 Go 中实现散列相关函数的学习呢？
Note: 本文使用的 Go 版本是 go version go1.15.14 linux/amd64


## Introduction

散列函数是将一个输入映射为一个固定长度输出的算法。

返回的值被叫做散列值，摘要或者就是散列。

散列主要用来解决密码学的完整性原则。

消息在发送者与接收者之间是可以被改变的，而散列函数可以验证消息是否被修改。


散列函数的主要特性有如下：
- 固定长度的输出：散列函数接收任意长度的输入，但总是产生相同长度的输出。
- 高效率：绝对不能难以执行计算
- 确定性：相同的输入必须问题产生相同的散列输出

应用到密码学当中，散列函数还必须拥有如下属性：
- 阻力1：给定一个散列值，应该很难根据它去寻找原始的值
- 阻力2：给定一个输入，应该很难找到产生同样散列的另外一个输入
- 阻力3：很难找到产生相同散列的两个输入

散列函数被应用到互联网的大多数应用中，如下：

- 比如你在之前应用有过从 FTP 下载大文件的经历，这些大文件可能拥有指示散列值
- 密码存储：你的密码不是以原文而是以散列值的形式存储的数据库中 (至少好的系统是这么干的)
- 唯一 ID：由于每条消息必须产生相同的输出，同时不同的消息产生不同的输出，你可以使用散列来唯一的标识一个文档或者消息。比如 Git 就是如此实践的，用散列来标识每一次提交。
- 工作量证明：对于用户执行操作或发布某些内容，他们必须证明自己已经执行了一项任务。该证明是用户花费一些时间生成满足评估者条件的答案的保证。比如用于区块链。


一些受欢迎的散列函数包括 MD5, SHA 和 Whirpool.



## String Hash

我们需要 `crypto` 包来计算计算散列值。这里列出了可用的散列函数：
```
const (
	MD4         Hash = 1 + iota // import golang.org/x/crypto/md4
	MD5                         // import crypto/md5
	SHA1                        // import crypto/sha1
	SHA224                      // import crypto/sha256
	SHA256                      // import crypto/sha256
	SHA384                      // import crypto/sha512
	SHA512                      // import crypto/sha512
	MD5SHA1                     // no implementation; MD5+SHA1 used for TLS RSA
	RIPEMD160                   // import golang.org/x/crypto/ripemd160
	SHA3_224                    // import golang.org/x/crypto/sha3
	SHA3_256                    // import golang.org/x/crypto/sha3
	SHA3_384                    // import golang.org/x/crypto/sha3
	SHA3_512                    // import golang.org/x/crypto/sha3
	SHA512_224                  // import crypto/sha512
	SHA512_256                  // import crypto/sha512
	BLAKE2s_256                 // import golang.org/x/crypto/blake2s
	BLAKE2b_256                 // import golang.org/x/crypto/blake2b
	BLAKE2b_384                 // import golang.org/x/crypto/blake2b
	BLAKE2b_512                 // import golang.org/x/crypto/blake2b

)
```

为了从 string 或者 byte 切片中计算一个散列值，我们可以使用给定包下想要的 Sum 方法：

```golang
package main

import (
	"crypto/md5"
	"crypto/sha1"
	"crypto/sha256"
	"fmt"
)

func main() {
	s := "Foo"

	hmd5 := md5.Sum([]byte(s))
	hsha1 := sha1.Sum([]byte(s))
	hsha2 := sha256.Sum256([]byte(s))

	fmt.Printf("   MD5: %x\n", hmd5)
	fmt.Printf("  SHA1: %x\n", hsha1)
	fmt.Printf("SHA256: %x\n", hsha2)
}

==========
➜  go run .
   MD5: 1356c67d7ad1638d816bfb822dd2c25d
  SHA1: 201a6b3053cc1422d2c3670b62616221d2290929
SHA256: 1cbec737f863e4922cee63cc2ebbfaafcd1cff8b790d8cfd2e6a5d550b648afa

```

## File Hash

为了从一个文件中计算散列值，我们需要基于文件内容来创建散列值

- 从 `crypto` 包中创建我们想要的散列算法
- 添加内容到散列算法的 `io.Writer` 函数中
- 通过调用 `Sum` 函数提取 sum 

>通过块形式读取文件内容避免占用大量内存空间

```golang
package main

import (
	"crypto/sha256"
	"io"
	"log"
	"os"
)

func main() {
	file, err := os.Open("test.txt")
	if err != nil {
		log.Fatal(err)
	}
	defer file.Close()

	buf := make([]byte, 30*1024)
	sha256 := sha256.New()
	for {
		n, err := file.Read(buf)
		if n > 0 {
			_, err := sha256.Write(buf[:n])
			if err != nil {
				log.Fatal(err)
			}
		}

		if err == io.EOF {
			break
		}

		if err != nil {
			log.Printf("Read %d bytes: %v", n, err)
			break
		}
	}

	sum := sha256.Sum(nil)
	log.Printf("%x\n", sum)
}

```


## Conclusion

幸亏标准库中的 crypto 包，在 Go 中从 string 和 文件内容计算散列变得如此简单。