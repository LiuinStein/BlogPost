### REST Resource Representation Compression

REST API可以通过XML, JSON, HTML甚至超文本等多种格式来返回一个资源的表述层。所有的返回数据格式均可以被压缩以节省网络带宽。**不同的协议使用不同的技术来启用压缩**并且告诉客户端其使用何种压缩方式，以便客户端可以在使用这些资源表述层之前解压它们。

> 压缩，或者加密，往往发生在表述层转移中，并且在客户端使用资源表述层的之前将其解压（或解密）

HTTP是目前REST使用最广泛的协议，所以在这里将用压缩HTTP响应来举个例子。

### Compression Related Request/Response Headers

#### Accept-Encoding

当请求一个资源表述层的时候，随着客户端发出一个HTTP请求，Accept-Encoding 请求头就会指示客户端可以接受哪种压缩算法。

`Accept-Encoding` 两个标准值为`compress`和`gzip`。

一个带有`accept-encoding`请求头的示例请求如下：

```
GET        /employees         HTTP/1.1
Host:     www.domain.com
Accept:     text/html
Accept-Encoding:     gzip,compress
```

`accept-encoding` 其他的取值可能为：

```
Accept-Encoding: compress, gzip
Accept-Encoding:
Accept-Encoding: *
Accept-Encoding: compress;q=0.5, gzip;q=1.0
Accept-Encoding: gzip;q=1.0, identity; q=0.5, *;q=0
```

如果请求中包含`Accept-Encoding`字段，并且如果服务端不能返回`Accept-Encoding`所指示的编码格式的数据，服务端应该向其返回406（Not Acceptable）错误。

#### Content-Encoding

如果服务端可以接受`Accept-Encoding`中所指示的压缩算法，它可以使用该压缩算法在响应之前压缩资源的表现层。当成功压缩完成之后，服务端必需通过HTTP响应头`Content-Encoding`让客户端知道其到底使用了哪种编码方式。

```
200 OK
Content-Type:     text/html
Content-Encoding:     gzip
```

如果服务器并不支持请求信息中所要求的实体的编码方式，服务端应当返回415（Unsupported Media Type）。如果实体可以支持多种编码，`Content-Encoding`必须将其一一列出。

请注意，请求或响应的源数据类型并不影响到其是否被会被压缩。

压缩数据可以节省很多带宽，并且只需要很少的附加成本。同样，你也可能知道，目前的大多数web浏览器都会使用上述的请求或响应头自动向服务器请求压缩后的数据。

> 译者注：
>
> 此处的附加成本指的是压缩数据时，CPU的多余消耗

参考资料：

<https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.11> <https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.3> 