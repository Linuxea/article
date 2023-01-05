# RESP protocol spec

> Redis serialization protocol (RESP) specification

Redis clients use a protocol called RESP (REdis Serialization Protocol) to communicate with the Redis server. While the protocol was designed specifically for Redis, it can be used for other client-server software projects.

RESP is a compromise between the following things:

Simple to implement.
Fast to parse.
Human readable.



Redis client 与 server 之间采用的是 RESP 协议 (REdis Serialization Protocol) 通信。
虽然这个协议是为 Redis 专门设计的，但是也可以用在其他 client-server (CS) 架构的项目中。

RESP 是根据不同的需求做出的折衷设计：
- 实现简单
- 快速解析
- 可读性高


RESP can serialize different data types like integers, strings, and arrays. There is also a specific type for errors. Requests are sent from the client to the Redis server as arrays of strings that represent the arguments of the command to execute. Redis replies with a command-specific data type.

RESP 支持的序列化有 integer, string, array 和一种异常类型。

Redis 客户端将请求执行命令以字符数组形式发送给服务端， Redis 服务端返回的是跟请求命令特定的数据类型。


## RESP protocol description

The RESP protocol was introduced in Redis 1.2, but it became the standard way for talking with the Redis server in Redis 2.0. This is the protocol you should implement in your Redis client.

RESP 协议是在 Redis 1.2 引进的，但是是在 Redis 2.0 版本才成为标准 CS 通信协议。
这也是你在创建自己的 Redis 客户端时需要实现的协议。

RESP is actually a serialization protocol that supports the following data types: Simple Strings, Errors, Integers, Bulk Strings, and Arrays.

RESP 实际上支持的序列化数据类型包含如下：simple strings，Errors，Integers, Bulk Strings 与 Arrays.


Redis uses RESP as a request-response protocol in the following way:

Redis 通过以下方式使用 RESP 作为请求-响应协议：

client 将请求命令以 Bulk Strings 的数组形式发送给 redis server.

Redis Server 根据命令的实现以 RESP 类型进行回复。

在 RESP 中，第一个字节决定了其数据类型:
- Simple String：响应的第一个字节是 "+"
- Erros: 响应的第一个字节是 "-"
- Integers: 响应的第一个字节是 ":"
- Bulk Strings: 响应的第一个字节是 "$"
- Arrays: 响应的第一个字节是 "*"

RESP 可以使用稍后指定的批量字符串或数组的特殊变体来表示 Null 值

Clients send commands to a Redis server as a RESP Array of Bulk Strings.
The server replies with one of the RESP types according to the command implementation.
In RESP, the first byte determines the data type:

For Simple Strings, the first byte of the reply is "+"
For Errors, the first byte of the reply is "-"
For Integers, the first byte of the reply is ":"
For Bulk Strings, the first byte of the reply is "$"
For Arrays, the first byte of the reply is "*"
RESP can represent a Null value using a special variation of Bulk Strings or Array as specified later.

In RESP, different parts of the protocol are always terminated with "\r\n" (CRLF).

在 RESP 中，协议中的不同部分总是以 "\r\n"(CRLF) 结束。


## RESP Simple Strings

Simple Strings are encoded as follows: a plus character, followed by a string that cannot contain a CR or LF character (no newlines are allowed), and terminated by CRLF (that is "\r\n").

Simple Strings 编码规则如下：一个 "+" 字符，后面跟随一个不能包含 CR 或者 LF (不允许换行) 的字符串并以 CRLF（"\r\n"） 结束。


Simple Strings are used to transmit non binary-safe strings with minimal overhead. For example, many Redis commands reply with just "OK" on success. The RESP Simple String is encoded with the following 5 bytes:


Simple Strings 用来最小负载代价传输非二进制安全的字符串。
例如有许多 Redis 命令仅仅以 "OK" 表示的场景，RESP 简单字符串 "OK" 使用以下 5 个字节进行编码：

```console
"+OK\r\n"
```

In order to send binary-safe strings, use RESP Bulk Strings instead.

> 为了发放二进制安全的字符串，应该使用 RESP Buld Strings

When Redis replies with a Simple String, a client library should respond with a string composed of the first character after the '+' up to the end of the string, excluding the final CRLF bytes.


## RESP Bulk Strings

Bulk Strings are used in order to represent a single binary-safe string up to 512 MB in length.

Bulk Strings are encoded in the following way:

A "$" byte followed by the number of bytes composing the string (a prefixed length), terminated by CRLF.
The actual string data.
A final CRLF.

Bulk Strings 用于表示单个最大长度达到 512MB 的二进制安全字符串。

Bulk Strings 根据以下方式编码：
- 一个 "$" 及一个数字用来表示跟随其后字节数量（前缀长度），以 CRLF 结束
- 实际的数据
- 一个最终的 CRLF

So the string "hello" is encoded as follows:
所以一个 "hello" 以 Bulk Strings 编码的实现如下：
```
$5\r\nhello\r\n
```
An empty string is encoded as:
一个空的字符串编码实现如下：
```
$0\r\n\r\n
```

RESP Bulk Strings can also be used in order to signal non-existence of a value using a special format to represent a Null value. In this format, the length is -1, and there is no data. Null is represented as:

RESP Bulk Strings  也能以特定的格式用来表示一个不存在的值为 Null.
在这种格式中，长度是 -1 并且没有实际数据。Null 以如下格式表示:
```
$-1\r\n
```
这被称为 Null Bulk String.


## RESP Integers

This type is just a CRLF-terminated string that represents an integer, prefixed by a ":" byte. For example, ":0\r\n" and ":1000\r\n" are integer replies.

Many Redis commands return RESP Integers, like INCR, LLEN, and LASTSAVE.

There is no special meaning for the returned integer. It is just an incremental number for INCR, a UNIX time for LASTSAVE, and so forth. However, the returned integer is guaranteed to be in the range of a signed 64-bit integer.

Integer replies are also used in order to return true or false. For instance, commands like EXISTS or SISMEMBER will return 1 for true and 0 for false.

Other commands like SADD, SREM, and SETNX will return 1 if the operation was actually performed and 0 otherwise.

The following commands will reply with an integer: SETNX, DEL, EXISTS, INCR, INCRBY, DECR, DECRBY, DBSIZE, LASTSAVE, RENAMENX, MOVE, LLEN, SADD, SREM, SISMEMBER, SCARD.


RESP Integers 就是一个以 ":" 为前缀，以 CRLF 终结的字符串。
例如，<code>:0\r\n</code> 与 <code>:1000\r\n</code> 都是整数类型的响应。



Redis 有许多命令以整型类型响应，比如 <code>INCR</code>, <code>LLEN</code> 与 <code>LASTSAVE</code>。

响应整数没有特殊的含义。在不同场景下有不同的含义。
- 对于 INCR 的响应表示增量的结果
- 对于 LASTSAVE 的响应表示一个 UNIX 时间
- 对于 EXISTS 与 SISMEMBER, 整数 1 表示 true 而整数 0 表示 false
- 对于 SADD, SREM, SETNX 整数 1 表示执行结果的确定，否则响应整数 0
- 等等

能保证的是的响应整数的范围为 64 位有符号整数。

## RESP Errors

RESP has a specific data type for errors. They are similar to RESP Simple Strings, but the first character is a minus '-' character instead of a plus. The real difference between Simple Strings and Errors in RESP is that clients treat errors as exceptions, and the string that composes the Error type is the error message itself.

对于异常 RESP 有特定的数据表示类型。

跟 RESP Simple Strings 非常相似，但是第一个字符是 "-" 而不是 "+" 。
在 client 中两者的区别是 error 会被视为一个异常，而组成异常类型本身的字符串则为异常消息。

基础格式如下：
```
"-Error message\r\n"
```

Error replies are only sent when something goes wrong, for instance if you try to perform an operation against the wrong data type, or if the command does not exist. The client should raise an exception when it receives an Error reply.

RESP Error 响应只有在某些错误发生时会被发送到客户端。
比如，你对一个不匹配的数据类型执行操作，或者命令不存在。

当 client 接收到一个 Error 响应时应该抛出一个异常。

如下是错误响应的示例：
```
-WRONGTYPE Operation against a key holding the wrong kind of value\r\n
-ERR unknown command 'helloworld'\r\n
```
"-" 之后的第一个单词，直到第一个空格或换行符，表示返回的错误类型。 如下："WRONGTYPE", "ERR"
不过这只是 Redis 使用的约定，不是 RESP 错误格式的一部分。

For example, ERR is the generic error, while WRONGTYPE is a more specific error that implies that the client tried to perform an operation against the wrong data type. This is called an Error Prefix and is a way to allow the client to understand the kind of error returned by the server without checking the exact error message.

比如，ERR 是通用的错误，而 WRONGTYPE 是更加具体的错误，表示客户端尝试在错误的数据类型执行一个操作。
这种约定方式称为 "Error Prefix"，能够允许客户端在不需要检查确定的异常消息时，就能理解错误的类型。



## RESP Arrays

Clients send commands to the Redis server using RESP Arrays. Similarly, certain Redis commands, that return collections of elements to the client, use RESP Arrays as their replies. An example is the LRANGE command that returns elements of a list.

客户端采用 RESP Arrays 的形式发送命令给 Redis server.
类似的，某些返回元素集合给 client 的命令也使用 RESP Arrays 作为响应。
一个例子是 `LRANGE` 命令，它返回一个列表的元素们。

RESP Arrays are sent using the following format:

A * character as the first byte, followed by the number of elements in the array as a decimal number, followed by CRLF.
An additional RESP type for every element of the Array.

RESP Arrays 使用如下格式发送：
- "*" 作为第一个字节，后面跟随一个表示数组大小的十进制数字，再跟随一个 CRLF
- Array 的每个元素的 RESP 类型。


因此一个空数组表示如下：
```
*0\r\n
```

While an array of two RESP Bulk Strings "hello" and "world" is encoded as:
包含两个 RESP Buld Strings 的数组表示如下：
```
*2\r\n$5\r\nhello\r\n$5\r\nworld\r\n
```
如您所见，在数组前缀的 `*<count>CRLF` 之后，构成数组的其他数据类型只是一个接一个地连接在一起。
例如，三个整数的数组编码如下：
```
*3\r\n:0\r\n:1\r\n:2\r\n
```

Arrays can contain mixed types, so it's not necessary for the elements to be of the same type. For instance, a list of four integers and a bulk string can be encoded as follows:

Arrays 可以包含混合类型。
比如包含四个整数与一个 Buld String 的 Array 可以编码如下：
```
*5\r\n:0\r\n:1\r\n:2\r\n:3\r\n$5\r\nhello\r\n
```

为了清晰可见，手动处理换行：
```
*5\r\n
:0\r\n
:1\r\n
:2\r\n
:3\r\n
$5\r\n
hello\r\n
```


Null Arrays exist as well and are an alternative way to specify a Null value (usually the Null Bulk String is used, but for historical reasons we have two formats).

For instance, when the BLPOP command times out, it returns a Null Array that has a count of -1 as in the following example:

与 Null Buld Strings 类似，Null Arrays 同样存在，并且提供了另外的方法来表示一个空值。

比如，当 `BLPOP` 命令超时，它会返回一个拥有数量为 -1 的 Null Array，如下表示：
```
*-1\r\n
``` 


Nested arrays are possible in RESP. For example a nested array of two arrays is encoded as follows:

RESP 同样支持数组嵌套。比如一个二维数据表示如下:
```
*2\r\n*3\r\n:1\r\n:2\r\n:3\r\n*2\r\n+Hello\r\n-World\r\n
```
为了清晰可见，手动处理换行：
```
*2\r\n
*3\r\n
:1\r\n
:2\r\n
:3\r\n
*2\r\n
+Hello\r\n
-World\r\n
```
