### Caching REST API Response

[缓存](https://tools.ietf.org/html/rfc7234)是服务端在请求-响应路径中用来存储一份经常访问的数据的副本的功能。当一个用户请求一个资源的表现层时，请求通过缓存或多级缓存（本地缓存、代理缓存、反向代理）走向托管资源的服务。如果任何一级缓存拥有请求路径的资源表现层的新鲜副本的话，那么这份副本将会用来响应这个请求。如果所有缓存都不能满足这个请求，这个请求将一路传送至服务层（源服务器）。

在HTTP请求头中，源服务器说明一个响应是不是可缓存的，如果是的话，由谁来缓存，缓存的有效期是多少。在响应路径中，缓存可以将响应信息做一个副本，当然前提是缓存数据的metadata允许他们这么做。

可以使用如下的方法来使用缓存优化网络来提高整体的服务质量

* 减少带宽
* 降低延迟
* 减轻服务器负载
* 隐藏网络失败

> 译者注：
>
> 此处的减少带宽指的是，减少数据传输量（带宽使用）以提升服务的质量。
>
> 具体做法可参考文章[Bandwidth and Data Transfer](https://www.website.com/beginnerguide/bandwidth/1/2/how-to-reduce-data-transfer.ws)
>
> 摘抄其中一段：
>
> How can I reduce my "bandwidth usage"?
>
> 我怎么样才能降低我的带宽使用呢？
>
> You certainly do not want to reduce the number of visitors. So, the most effective way is to reduce the size of the files on your website.
>
> 首先，你肯定不想降低访问你网站的用户量，所以，最有效的方法就是减少网站文件的大小。
>
> - Build simpler, more efficient websites. Remove any unnecessary text, images or other files. Keep your pages as small as possible.
> - 创建一个简单的网站，删除一切不必要的文字、图片以及其他文件，让你的网站尽可能地简单
> - Optimize your graphics. Use JPEG image format for photos and GIF form for graphics.
> - 优化图像显示，对图片使用JPEG格式，对动画使用GIF格式
> - Refrain from fancy flash presentations, audio or video. This includes any music playing the background.
> - 避免使用flash来做页面效果或者展示视频及音频，同样避免有任何背景音乐
> - Use CSS and call JavaScript externally instead of embedding in every page. This reduces the HTML file size.
> - 从外部CSS和JavaScript，避免将其嵌入至HTML页面中
> - Remove unwanted tags, white space and comments
> - 删除不想要的标签，空格以及注释
> - Limit your Meta tags to those absolutely necessary
> - 限制源标签的大小
> - Consider caching your website, but set an expiry date in the HTTP headers so your visitors' browsers will refresh the content after a certain time (this allows their browser to save a copy of your website, and each time the visitor visits your website, the pages are served from the copy on the browser and not your web server)
> - 对你的网站考虑做缓存，并且在HTTP头部设置一个过期时间

### Caching in REST APIs

可缓存本身就是REST的一个架构约束。GET请求默认就应该是可缓存的（除了一些特殊情况）。通常情况下，浏览器认为所有的GET请求都是可缓存的。POST请求默认是不可缓存的，但是如果设置了`Expires`或`Cache-Control`响应头，它就被显示定义为了可缓存的。PUT和DELETE请求在任何情况下都不应该是可缓存的。

下面是两个用来控制缓存行为的主要的HTTP响应头：

#### Expires

Expires响应头对缓存的表现层定义了一个过期时间。过了这个时间之后，缓存就会过期，并且需要重新与源服务器进行验证（缓存的有效性）。如果想指定永不过期，Expires可以设置成一个1年后的时间。

```
Expires: Fri, 20 May 2016 19:20:49 IST
```

#### Cache-Control

这个响应头字段的值是由一个或多个由逗号（,）分隔的[指令](https://tools.ietf.org/html/rfc7234#page-24)构成。这些指令决定了一个响应是否是可缓存的，如果是可缓存的话，由谁缓存，缓存多长时间，例如 `max-age` 或者 `s-maxage` 指令。

```
Cache-Control: max-age=3600
```

一个可缓存的响应（无论是对于一个GET还是一个POST请求）应当包含一个验证器（validator）--一个 `ETag` 或者 `Last-Modified ` 响应头。

> 译者注：
>
> 一个用来验证缓存有效性的验证器

#### ETag

`ETag`值是服务器与资源相关联的不透明的字符串标记，用于唯一标识资源在其整个生命周期内的状态。当资源发生改变时，`ETag`值也随之改变。

```
ETag: "abcd1234567n34jv"
```

> 译者注：
>
> 不透明的字符串标记带有的是一个由不透明的数据类型构成的字符串
>
> 不透明的数据类型的是一个不想让别人知道的数据类型（隐藏起来的数据类型），具有较强的信息隐蔽性。它的值只能通过有权访问这个信息的子程序来操作。
>
> 具体参考[维基百科](https://en.wikipedia.org/wiki/Opaque_data_type)

#### Last-Modified

一个响应的Date头指示的是这个响应在何时被生成的，然而Last-Modified响应头指示的是与其关联的资源的最后一次修改时间。Last-Modified字段的值不能晚于Date字段的值。

```
Last-Modified: Fri, 10 May 2016 09:17:49 IST
```

