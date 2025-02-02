## 函数

### getsockname和getpeername函数

getsockname函数用于获取与某个[套接字](https://so.csdn.net/so/search?q=套接字&spm=1001.2101.3001.7020)关联的本地协议地址
getpeername函数用于获取与某个套接字关联的外地协议地址

```c
#include<sys/socket.h>
 
int getsockname(int sockfd, struct sockaddr *localaddr, socklen_t *addrlen);
 
int getpeername(int sockfd, struct sockaddr *peeraddr, socklen_t *addrlen);
```



### strcasecmp函数

```c
strcasecmp()
    使用   :用来比较参数s1和s2字符串，比较时会自动忽略大小写的差异
    返回值 :若参数s1和s2字符串相同则返回0。s1长度大于s2长度则返回大于0 的值，s1 长度若小于s2 长度则返回小于0的值
```



### int isspace(int c)函数 

```c
检查所传的字符是否是空白字符。
    标准的空白字符包括：
        ' '     (0x20)    space (SPC) 空格符
        '\t'    (0x09)    horizontal tab (TAB) 水平制表符    
        '\n'    (0x0a)    newline (LF) 换行符
        '\v'    (0x0b)    vertical tab (VT) 垂直制表符
        '\f'    (0x0c)    feed (FF) 换页符
        '\r'    (0x0d)    carriage return (CR) 回车符

    函数的声明：
        int isspace(int c);

    返回值：
        如果 c 是一个空白字符，则该函数返回非零值（true），否则返回 0（false）。
```

### putenv()函数

```c
用于改变或增加环境变量的内容
        函数名      :putenv
        头文件      :<stdlib.h>
        函数原型    :void *putenv(char *name);
        功能        :用于改变或增加环境变量的内容
        参数        :char *name 为环境变量名
        返回值      :成功  返回0 ，失败  返回-1
```

## Post和Get 的区别

`GET`和`POST`，两者是`HTTP`协议中发送请求的方法

#### GET

`GET`方法请求一个指定资源的表示形式，使用GET的请求应该只被用于获取数据

#### POST

`POST`方法用于将实体提交到指定的资源，通常导致在服务器上的状态变化或**副作用**

本质上都是`TCP`链接，并无差别

但是由于`HTTP`的规定和浏览器/服务器的限制，导致他们在应用过程中会体现出一些区别

### 区别

从`w3schools`得到的标准答案的区别如下：

- GET在浏览器回退时是无害的，而POST会再次提交请求。
- GET产生的URL地址可以被Bookmark，而POST不可以。
- GET请求会被浏览器主动cache，而POST不会，除非手动设置。
- GET请求只能进行url编码，而POST支持多种编码方式。
- GET请求参数会被完整保留在浏览器历史记录里，而POST中的参数不会被保留。
- GET请求在URL中传送的参数是有长度限制的，而POST没有。
- 对参数的数据类型，GET只接受ASCII字符，而POST没有限制。
- GET比POST更不安全，因为参数直接暴露在URL上，所以不能用来传递敏感信息。
- GET参数通过URL传递，POST放在Request body中

#### 参数位置

貌似从上面看到`GET`与`POST`请求区别非常大，但两者实质并没有区别

无论 `GET `还是 `POST`，用的都是同一个传输层协议，所以在传输上没有区别

当不携带参数的时候，两者最大的区别为第一行方法名不同

> POST /uri HTTP/1.1 \r\n
>
> GET /uri HTTP/1.1 \r\n

当携带参数的时候，我们都知道`GET`请求是放在`url`中，`POST`则放在`body`中

`GET` 方法简约版报文是这样的

```
GET /index.html?name=qiming.c&age=22 HTTP/1.1
Host: localhost
```



`POST `方法简约版报文是这样的

```
POST /index.html HTTP/1.1
Host: localhost
Content-Type: application/x-www-form-urlencoded

name=qiming.c&age=22
```



注意：这里只是约定，并不属于`HTTP`规范，相反的，我们可以在`POST`请求中`url`中写入参数，或者`GET`请求中的`body`携带参数

#### 参数长度

`HTTP `协议没有`Body`和 `URL` 的长度限制，对 `URL `限制的大多是浏览器和服务器的原因

`IE`对`URL`长度的限制是2083字节(2K+35)。对于其他浏览器，如Netscape、FireFox等，理论上没有长度限制，其限制取决于操作系统的支持

这里限制的是整个`URL`长度，而不仅仅是参数值的长度

服务器处理长` URL` 要消耗比较多的资源，为了性能和安全考虑，会给 `URL` 长度加限制

#### 安全

**`POST `比` GET` 安全，因为数据在地址栏上不可见**

然而，从传输的角度来说，他们都是不安全的，因为` HTTP` 在网络上是明文传输的，只要在网络节点上捉包，就能完整地获取数据报文

只有使用`HTTPS`才能加密安全

#### 数据包

对于`GET`方式的请求，浏览器会把`http header`和`data`一并发送出去，服务器响应200（返回数据）

对于`POST`，浏览器先发送`header`，服务器响应100 `continue`，浏览器再发送`data`，服务器响应200 ok

并不是所有浏览器都会在`POST`中发送两次包，`Firefox`就只发送一次

