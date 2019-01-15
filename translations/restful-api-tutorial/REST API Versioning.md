### REST API Versioning

版本控制可帮助您在识别所需更改时快速迭代。 

> 随着你的知识面以及系统的提升，API的修改是不可避免的。当它打破了现有客户端的整合时，管理这种变化带来的影响可能是一个挑战。

### When to Version

当发生重大更改之后，API需要提升版本号。重大更改一般是指以下情况

* 返回数据格式的变化
* 返回值数据类型的变化（比如从整数变为浮点数）
* 从当前API中移除任一部分

重大更改后应当修改API的主要版本号或者内容[响应类型](https://www.iana.org/assignments/media-types/media-types.xhtml)

非重大更改，例如增加一个新的终结点或者新的响应参数，不应当修改主要的版本号。然而，当进行更改以支持客户可能遇到的一些问题时，比如收到缓存后的数据或者其他的一些API问题，跟踪这些次要版本的API可能会有很大帮助。

### How to Version

REST并没有提供任何特定的版本控制方法，但是通常情况下会从下面3个角度来进行版本控制：

#### URI Versioning

在URI中直接进行版本控制是目前最常用的方法，但是其违反了URI必需用来定义唯一资源的准则。当版本更新时，这同样会打破客户端的集成。

例如：

```
http://api.example.com/v1
http://apiv1.example.com
```

版本号不一定非得是个数字，也不必非得使用`v[x]`这样的语法格式。替代方案包括日期、项目名称、季节或其他标识符，这些标识符对于API的开发者团队来说足够有意义，并且可以随着版本的变化灵活地进行修改。

#### Versioning using Custom Request Header

自定义HTTP头字段（比如`Accept-version`）允许你保留不同版本之间的URI，尽管实际上它是使用`Accept`头进行内容协商行为的重复。

例如：

```
Accept-version: v1
Accept-version: v2
```

#### Versioning using Accept header

内容协商可能会使你的URL看上去不那么冗杂，但是你仍然需要处理一些在不同地方使用了不同版本的API所带来的复杂问题。这个负担同样会带入到你API的Controller里面，你需要使你的Controller识别传送过来的到底是什么版本的资源。最后的结果可能就是使得API更加复杂，客户端必需要知道在请求资源之前要设置哪种请求头。

例如：

```
Accept: application/vnd.example.v1+json
Accept: application/vnd.example+json;version=1.0
```

在实际中，一个API永远不会完全稳定。所以，如何管理这些更改将变得非常重要。一份好的文档以及逐步淘汰掉一些API对大多数API设计来说将会是一个可接受的方法。