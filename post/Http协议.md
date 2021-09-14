## 什么是HTTP？

HTTP是 Hyper Text Transfer Protocol的简称，中文名超文本传输协议，是一个基于TCP/IP 协议来传输超文本的应用层协议。

HTTP协议工作与客户端-服务器架构上，浏览器作为HTTP的客户端通过URL向HTTP服务器端发送请求。服务端收到请求后向客户端发送相应信息。HTTP的默认端口为80.

- HTTP是无连接的。即每次只处理一个请求，服务器处理完客户端的请求，并收到客户端的应答后就断开连接。
- HTTP是无状态的。对于请求的处理没有记忆能力，如果后续处理需要前面的信息，则必须重传。

## HTTP 消息结构



### 客户端请求

客户端发送一个HTTP请求到服务器的请求消息包括以下格式：请求行（request line）、请求头部（header）、空行和请求数据四个部分组成，下图给出了请求报文的一般格式。

![](https://www.runoob.com/wp-content/uploads/2013/11/2012072810301161.png)



### 服务端响应

HTTP响应也由四个部分组成，分别是：状态行、消息报头、空行和响应正文。

![](https://www.runoob.com/wp-content/uploads/2013/11/httpmessage.jpg)



## HTTP的请求方法

HTTP1.0定义了三种方法，分别为GET、POST和HEAD方法。HTTP1.1新增了六种方法PUT、DELETE、OPTIONS、PATCH、TRACE 和 CONNECT 方法。这里只介绍常用的五种方法：

### 1.GET

用来获取资源，不进行服务器数据操作，因此请求数据中没有Body。

### 2.POST

向指定资源提交数据进行处理请求（例如提交表单或者上传文件）。数据被包含在请求体中。POST 请求可能会导致新的资源的建立和/或已有资源的修改。

### 3.PUT

向指定资源提交数据进行处理请求（例如提交表单或者上传文件）。数据被包含在请求体中。POST 请求可能会导致新的资源的建立和/或已有资源的修改。用于修改资源，有Body

### 4.DELETE

请求服务器删除指定的页面。没有Body

### 5.HEAD

类似于 GET 请求，只不过返回的响应中没Body内容，用于获取报头

## HTTP 的状态码

1xx : 临时性消息,服务器收到请求，需要请求者继续执行操作

2xx : 成功,操作被成功接收并处理

3xx : 重定向,需要进一步的操作以完成请求

4xx : 客户端错误,请求包含语法错误或无法完成请求

5xx : 服务器错误,服务器在处理请求的过程中发生了错误

## HTTP的响应头

HTTP 的请求头示例如下：

```
GET /home.html HTTP/1.1
Host: developer.mozilla.org
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:50.0) Gecko/20100101 Firefox/50.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type:text/html;charset=UTF-8
Referer: https://developer.mozilla.org/testpage.html
Connection: keep-alive
Upgrade-Insecure-Requests: 1
If-Modified-Since: Mon, 18 Jul 2016 02:36:04 GMT
If-None-Match: "c561c68d0ba92bbeb8b0fff2a9199f722e3a621a"
Cache-Control: max-age=0
```



| 应答头           | 说明                                                         |
| :--------------- | :----------------------------------------------------------- |
| Allow            | 服务器支持哪些请求方法（如GET、POST等）。                    |
| Content-Encoding | 文档的编码（Encode）方法。只有在解码之后才可以得到Content-Type头指定的内容类型。利用gzip压缩文档能够显著地减少HTML文档的下载时间。Java的GZIPOutputStream可以很方便地进行gzip压缩，但只有Unix上的Netscape和Windows上的IE 4、IE 5才支持它。因此，Servlet应该通过查看Accept-Encoding头（即request.getHeader("Accept-Encoding")）检查浏览器是否支持gzip，为支持gzip的浏览器返回经gzip压缩的HTML页面，为其他浏览器返回普通页面。 |
| Content-Length   | 表示内容长度。只有当浏览器使用持久HTTP连接时才需要这个数据。如果你想要利用持久连接的优势，可以把输出文档写入 ByteArrayOutputStream，完成后查看其大小，然后把该值放入Content-Length头，最后通过byteArrayStream.writeTo(response.getOutputStream()发送内容。 |
| Content-Type     | 表示后面的文档属于什么MIME类型。Servlet默认为text/plain，但通常需要显式地指定为text/html。由于经常要设置Content-Type，因此HttpServletResponse提供了一个专用的方法setContentType。 |
| Date             | 当前的GMT时间。你可以用setDateHeader来设置这个头以避免转换时间格式的麻烦。 |
| Expires          | 应该在什么时候认为文档已经过期，从而不再缓存它？             |
| Last-Modified    | 文档的最后改动时间。客户可以通过If-Modified-Since请求头提供一个日期，该请求将被视为一个条件GET，只有改动时间迟于指定时间的文档才会返回，否则返回一个304（Not Modified）状态。Last-Modified也可用setDateHeader方法来设置。 |
| Location         | 表示客户应当到哪里去提取文档。Location通常不是直接设置的，而是通过HttpServletResponse的sendRedirect方法，该方法同时设置状态代码为302。 |
| Refresh          | 表示浏览器应该在多少时间之后刷新文档，以秒计。除了刷新当前文档之外，你还可以通过setHeader("Refresh", "5; URL=http://host/path")让浏览器读取指定的页面。 注意这种功能通常是通过设置HTML页面HEAD区的＜META HTTP-EQUIV="Refresh" CONTENT="5;URL=http://host/path"＞实现，这是因为，自动刷新或重定向对于那些不能使用CGI或Servlet的HTML编写者十分重要。但是，对于Servlet来说，直接设置Refresh头更加方便。  注意Refresh的意义是"N秒之后刷新本页面或访问指定页面"，而不是"每隔N秒刷新本页面或访问指定页面"。因此，连续刷新要求每次都发送一个Refresh头，而发送204状态代码则可以阻止浏览器继续刷新，不管是使用Refresh头还是＜META HTTP-EQUIV="Refresh" ...＞。  注意Refresh头不属于HTTP 1.1正式规范的一部分，而是一个扩展，但Netscape和IE都支持它。 |
| Server           | 服务器名字。Servlet一般不设置这个值，而是由Web服务器自己设置。 |
| Set-Cookie       | 设置和页面关联的Cookie。Servlet不应使用response.setHeader("Set-Cookie", ...)，而是应使用HttpServletResponse提供的专用方法addCookie。参见下文有关Cookie设置的讨论。 |
| WWW-Authenticate | 客户应该在Authorization头中提供什么类型的授权信息？在包含401（Unauthorized）状态行的应答中这个头是必需的。例如，response.setHeader("WWW-Authenticate", "BASIC realm=＼"executives＼"")。 注意Servlet一般不进行这方面的处理，而是让Web服务器的专门机制来控制受密码保护页面的访问（例如.htaccess）。 |



### Content-Type

Content-Type 用于定义网络文件的类型和网页的编码，决定浏览器将以什么样的形式，什么样的编码读取这个文件。语法格式：

```
Content-Type: text/html; charset=utf-8
Content-Type: multipart/form-data; boundary=something
```

常见的媒体格式类型如下：

- text/html ： HTML格式
- text/plain ：纯文本格式
- text/xml ： XML格式
- image/gif ：gif图片格式
- image/jpeg ：jpg图片格式
- image/png：png图片格式

以application开头的媒体格式类型：

- application/xhtml+xml ：XHTML格式
- application/xml： XML数据格式
- application/atom+xml ：Atom XML聚合格式
- application/json： JSON数据格式
- application/pdf：pdf格式
- application/msword ： Word文档格式
- application/octet-stream ： 二进制流数据（如常见的文件下载）
- application/x-www-form-urlencoded ： <form encType="">中默认的encType，form表单数据被编码为key/value格式发送到服务器（表单默认的提交数据的格式）

另外一种常见的媒体格式是上传文件之时使用的：

- multipart/form-data ： 需要在表单中进行文件上传时，就需要使用该格式

## 一次完整的HTTP请求

1. 首先进行DNS域名解析（本地浏览器缓存、操作系统缓存或者DNS服务器），首先会搜索浏览器自身的DNS缓存（缓存时间比较短，大概只有1分钟，且只能容纳1000条缓存）
    - 如果浏览器自身的缓存里面没有找到，那么浏览器会搜索系统自身的DNS缓存
    - 如果还没有找到，那么尝试从 hosts文件里面去找
    - 在前面三个过程都没获取到的情况下，就去域名服务器去查找，

2. 三次握手建立 TCP 连接

在HTTP工作开始之前，客户端首先要通过网络与服务器建立连接，HTTP连接是通过 TCP 来完成的。HTTP 是比 TCP 更高层次的应用层协议，根据规则，只有低层协议建立之后，才能进行高层协议的连接，因此，首先要建立 TCP 连接，一般 TCP 连接的端口号是80；

3. 客户端发起HTTP请求

4. 服务器响应HTTP请求

5. 客户端解析html代码，并请求html代码中的资源

浏览器拿到html文件后，就开始解析其中的html代码，遇到js/css/image等静态资源时，就向服务器端去请求下载

6. 客户端渲染展示内容

7. 关闭 TCP 连接

一般情况下，一旦服务器向客户端返回了请求数据，它就要关闭 TCP 连接，然后如果客户端或者服务器在其头信息加入了这行代码 Connection:keep-alive ，TCP 连接在发送后将仍然保持打开状态，于是，客户端可以继续通过相同的连接发送请求，也就是说前面的3到6，可以反复进行。保持连接节省了为每个请求建立新连接所需的时间，还节约了网络带宽。

https://www.runoob.com/http/http-tutorial.html