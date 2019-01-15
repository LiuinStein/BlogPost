### HATEOAS Driven REST APIs

**HATEOAS (Hypermedia as the Engine of Application State 使用超媒体作为应用状态引擎)** 是REST应用程序架构的约束之一，用来保证RESTful体系结构与其他网络应用体系结构之间的独特性。这个术语“**hypermedia**”指的是任何包含一个超链接或者其他媒体形式（诸如图像、电影或者文本）的内容。

这个体系结构允许你在响应体中使用超链接，以便客户端可以通过超链接动态定向到所需要的资源。这在概念上等价于web用户通过点击web页面上的超链接来到达其想要的页面或寻求期望的信息。

例如，下面的是对请求`HTTP GET http://api.domain.com/management/departments/10`的一个JSON响应体：

```json
{
    "departmentId": 10,
    "departmentName": "Administration",
    "locationId": 1700,
    "managerId": 200,
    "links": [
        {
            "href": "10/employees",
            "rel": "employees",
            "type" : "GET"
        }
    ]
}
```

在这个例子中，服务端返回一个包含跳转到职工信息超链接`10/employees`的响应体，客户端可以通过这个超链接来读取这个部门的职工信息。

上述实现方法的优点在于从服务端返回的超链接驱动着这个应用程序的状态。

这里并没有一个通用的实现方法用来在JSON中表示两种资源之间的链接。你同样可以选择下面的形式来通过HTTP响应头来发送一个链接：

```
HTTP/1.1 200 OK
...
Link: <10/employees>; rel="employees"
```

这两者都是合适的解决方案。

### HATEOAS Implementation

在现实世界中，当你访问一个网站的时候，你点击了它的主页。他会现实一个网页的快照以及一些链接到其他网站部分的链接。你点击这些链接，然后你将会获得更多与上下文相关的链接。

与人类浏览网页相似，**一个REST客户端访问初始API URI，然后使用服务端提供的链接来动态寻求可使用的动作以及获取相关资源**。一个客户端并不需要对有关服务以及不同步骤有任何的预先了解。此外，**无需将不同资源的URI地址结构硬编码到客户端内**，这同样允许在无需修改客户端的情况下，服务端随着API的演变而修改URI地址。

> 译者注：
>
> 硬编码就是写死到代码里。

以上API交互必需使用HATEOAS。

每一个REST框架都会提供其自己的HATEOAS实现方法，例如在[这篇文章](https://howtodoinjava.com/resteasy/writing-restful-webservices-with-hateoas-using-jax-rs-and-jaxb-in-java/)内，链接作为资源模型类的一部分，并作为资源状态传递给客户端。

### HATEOAS References

下面是两个主流的JSON REST API超媒体链接格式：

#### RFC 5988 (web linking)

[RFC 5988](https://tools.ietf.org/html/rfc5988)提出了一个用来构建定义网络上资源间关系链接的框架。符合RFC 5988标准的链接应当包含以下几个属性：

**Target URI（目标URI）**:每个链接都应当包含一个目标[Internationalized Resource Identifiers 国际化资源标识符](https://en.wikipedia.org/wiki/Internationalized_Resource_Identifier) (IRIs)。这通常由`href`属性来表示。

**Link relation type（链接关系类型）**: 链接关系类型描述的是当前上下文与目标上下文之间的关系。通常由`rel`属性来表示。

> 译者注：
>
> ```html
> <link rel="icon" href="/img/abc.svg">
> ```
>
> 这里的`rel`定义的即为链接`href`所指向的内容与当前HTML页的关系

**Attributes for target IRI（目标IRI属性）**:这个链接属性包括`hreflang`, `media`, `title`以及`type`，以及一些其他的链接属性。 

#### JSON Hypermedia API Language (HAL)

[JSON HAL](https://en.wikipedia.org/wiki/Hypertext_Application_Language)是一个很有希望的提案，其为使用JSON或XML来控制超媒体的表示设置了约定。目前正处于草案阶段。

> 译者注：
>
> 这里说他是一个很有希望的提案是因为这个东西目前还处于草案（draft）阶段，没有正式的归入标准。

下面是两个相关联的MIME类型：

```
media type: application/hal+xml 
media type: application/hal+json
```

HAL标准下的每一个链接都应当包含以下属性：

**Target URI（目标URI）**:指出了指定资源的URI。这通常由`href`属性来表示。

**Link relation type（链接关系类型）**: 链接关系类型描述的是当前上下文与目标上下文之间的关系。通常由`rel`属性来表示。

**Type（类型）：**指定了期望的资源媒体类型。通常由`type`属性来表示。

为你的应用程序选择何种超媒体链接格式并没有对错之分。你只需要选择一种你认为最适合你应用程序最符合你需求的即可。

> 译者注：
>
> 参考资料：
>
> [Microservices中的API设计 - HAL+Json API](https://www.jianshu.com/p/faba6b57f915)