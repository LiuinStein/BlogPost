### Idempotent REST APIs

在REST API上下文中，当发送的多个请求产生的效果和一个请求相同时，这个REST API就是幂等的（**idempotent**）。

当你设计一个REST API的时候，你必需意识到使用API的人是可能会犯错误的。他们写出来的客户端代码，可能会将一个请求发送多遍，这些重复的请求可能是在无意间发出来的（例如，客户端程序错误），当然有时也是故意发送的（例如，网络超时）。你应当设计一个具有容错性的API使得当重复请求发生时并不会使系统变得不稳定。

> 一个幂等的HTTP方法是那种当被请求多次时并不会产生不同结果的HTTP方法。这与这个方法到底被请求了多少遍没有关系。结果应该是相同的。简要来说，这意味着请求成功执行的结果与执行次数无关。举个例子，在数学里，把0加到任意一个数字的操作都是幂等的。

如果你遵循REST准则来设计API，你将有如下的几个幂等的REST API， GET, PUT, DELETE, HEAD, OPTIONS以及TRACE。只有`POST` API不是幂等的。

1. `POST`不是幂等的
2. `GET`, `PUT`, `DELETE`, `HEAD`, `OPTIONS`以及`TRACE`是幂等的。

让我们来分析一下为什么上述的HTTP方法时幂等的，以及为什么POST不是幂等的

### HTTP POST

通常情况下（不是必须的），POST API用来在服务端创建一个新的资源。所以当你多次调用一个POST请求的时候，你将会在服务端创建多个新的资源。所以，**POST不是幂等的**

### HTTP GET, HEAD, OPTIONS and TRACE

`GET`, `HEAD`, `OPTIONS`以及`TRACE`方法并不改变服务器上资源的状态，它们仅是单纯地用来获取资源的表现层或者元数据。所以多次调用这些请求并不会在服务端产生任何写操作，所以`GET`, `HEAD`, `OPTIONS`以及`TRACE`是幂等的。

### HTTP PUT

通常情况下，当然也不是必须的，PUT API被用来更新一个资源的状态。如果你多次调用一个PUT请求，一开始的那个请求将会更新资源，然后剩下的那些请求仅仅是重复地执行跟第一遍一样的更新，后面的这些请求实际上并没有改变任何事情。所以，**PUT是幂等的**

### HTTP DELETE

当你请求N次相同的`DELETE`请求时，第一个请求将会从服务器删除资源并且返回`200 (OK)`或者`204 (No Content)`，剩下的N-1个请求将会返回`404 (Not Found)`。显然，剩下的N-1个请求的响应与第一个不同，但是此时在服务端并没有任何资源状态的改变，因为源资源已经在第一个请求完成时被删除了。所以，**DELETE是幂等的**。

请看，有些系统的DELETE API是这样的：

```
DELETE /item/last
```

> 译者注：
>
> 这个API的用途是删除某个资源集合的最后一个元素。

在上述的例子中，调用N次这个API将会删除N个资源，所以在这个例子中`DELETE`请求不是幂等的。在这个例子中，建议将此处API请求方式改为POST，因为POST不是幂等的。

```
POST /item/last
```

这样才更贴切HTTP规范，也因此更符合REST。

引用：

[Rfc 2616](https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html) 

[SO Thread](https://stackoverflow.com/questions/7016785/is-put-delete-idempotent-with-rest-automatic) 