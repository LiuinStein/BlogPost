### REST Resource Naming Guide

在REST中，主要的数据表现层称之为**资源**。使用一个健壮以及坚持使用同一个REST资源命名策略将是一个长期的最佳的设计决策。

> 在REST中，对于信息最关键的抽象即为资源。任何信息都可以被称为一个资源：一份文档或图像，一个临时的服务（例如：今天洛杉矶的天气），一个其他资源的集合，一个非虚拟化的物体（例如一个人）等等。换句话来说，任何可能成为作者超文本引用目标的概念都必须符合资源的定义。一个资源在概念上可映射为一组实体，而不是与任何特定时间点相映射的对应实体。
>
> --[Roy Fielding的博士论文](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm#sec_5_2_1_1) 

任何资源都可以是单个或者多个（一个集合）。就比如，对银行系统来说，一群顾客就是一个集合资源，一个顾客就是一个单个的资源。我们可以使用URI `/customers ` 来标识一群顾客，同样可以使用URI `/customers/{customerId} ` 来标识一个顾客。

一个资源同样可以包含子资源。例如，同样对银行系统来说，一个顾客（资源）可能含有多个银行账户（子资源），这些账户可以通过URN `/customers/{customerId}/accounts `来标识。同样，在这个子资源`accounts`下的一个单个的资源`account`可以标识为`/customers/{customerId}/accounts/{accountId} `

REST API使用[统一资源标识符 URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier)来寻址资源。REST API设计者应当使用URI来向其客户端开发者传达REST API的资源模型。只有当一个资源有良好的命名时，一个API才能直观易用。反之，这些API将会变得难用且难以理解。

有关统一接口（*uniform interface*）的约束可以通过按照标准和惯例使用的URI和HTTP动词组合来解决。

> 译者注：
>
> HTTP动词（HTTP verb）指的是诸如GET, POST, DELETE, PUT这样的HTTP请求方式。 
>
> 有关统一接口的约束请参考本目录下的文章*REST Architectural Constraints*

下面是有关如何为你的API命名资源URI的几点提示。

### REST Resource Naming Best Practices

#### Use nouns to represent resources

RESTful API应当引用为一个资源，也就是一个事物（名词）而不是一个动作（动词），因为名词可以拥有属性而动词不行——同样资源也应当有属性。下面列举一些资源的例子：

* 系统中的用户
* 用户账号
* 网络设备等等

他们的资源URI可以被设计为：

```
http://api.example.com/device-management/managed-devices 
http://api.example.com/device-management/managed-devices/{device-id} 
http://api.example.com/user-management/users/
http://api.example.com/user-management/users/{id}
```

为了更清楚起见，让我们把**资源的原型（resource archetypes）**分为4个不同的类型（document，collection，store，controller），**你应该始终将一个资源对应一种原型，然后遵循它的命名约定**。由于一致性的缘故，抵制设计不止一种原型的混合资源的诱惑。 

1. **属性** （document）
   资源属性是一个单一的概念，它类似于一个对象实例或一条数据库记录。在REST中，你可以将其视为一个资源集合中的单个资源。一个属性的状态表现层通常包含具有值的字段以及指向其他相关资源的链接。
   使用“单数”名称来表示资源属性原型。 

   ```
   http://api.example.com/device-management/managed-devices/{device-id}
   http://api.example.com/user-management/users/{id}
   http://api.example.com/user-management/users/admin
   ```
   > 译者注：
   >
   > 这里的属性英文原文写的是document，字面上翻译为文档或者文件
   >
   > 但是在这里它并不是指的传统意义上的那个文档，程序文档或代码文档。
   >
   > 依据上下文语境，**这里的document指的应该是一个资源的具体属性，它有可能是一个对象实例或者一条数据库记录**，而这个属性必需以“单数”名词的形式表现在URI里面。
   >
   > 翻阅了很多资料，都没有很好的解释方法，[Collins](https://www.collinsdictionary.com/us/dictionary/english/document)词典也没有给出有关document这个单词更好的衍生意义来，故在此将其意译为“属性”
   >
   > 如果有更好的翻译方法，请提交Issue或pull request

2. **集合**
   一个集合的资源是一个由服务器管理的资源目录。客户端可能会向其添加新的资源。然而，它取决于要创建新资源的集合。一个集合资源选择它到底包含什么样的资源并且决定每个所包含资源的URI。
   使用“复数”名词来表示集合资源原型

   ```
   http://api.example.com/device-management/managed-devices
   http://api.example.com/user-management/users
   http://api.example.com/user-management/users/{id}/accounts
   ```

3. **store**（这个真不知道怎么翻译合适了）
   store是一个由客户端维护的资源库，一个store类型的资源可以让API客户端将资源放进去或者拿出来并且决定什么时候删除它们。store从来不生成新的URI，相反，每个存储的资源在将他们放入store的时候都有一个由客户端决定的URI。
   使用“复数”名词来表示store资源原型

   ```
   http://api.example.com/cart-management/users/{id}/carts
   http://api.example.com/song-management/users/{id}/playlists
   ```

   > 译者注：
   >
   > 这个store究竟是个什么呢，这里肯定不能翻译成商店，翻译成存储库或贮藏处吧又不是很合适，不能合理地表述其意思。
   >
   > 这里的store究竟是个啥呢，打个比方啊，假设说让我们设计一个类似淘宝客户端的API，用户在客户端点击将商品添加至购物车，购物车信息就可以放进这个store里，这个**store是由客户端维护的，但实际的存储是在服务端的**，对其的增删改查都是由客户端直接决定的，例如淘宝的购物车是同步的，你在电脑端添加了购物车在手机端也可以看到你购物车里的物品，为了实现这个购物车的同步，我们就要在服务端设置一个用来存放你当前购物车信息的数据库或存储位置，这个由客户端维护的数据库或存储库就是这里的store。在设计同步购物车API的时候，我们就要用复数名词来表示store的资源类型，就是这样。

4. **控制器**
   控制器资源模型是一个程序化的概念，控制器资源就像一个可执行的函数一样，带有参数和返回值，输入和输出
   使用“动词”来表示控制器资源

   ```
   http://api.example.com/cart-management/users/{id}/cart/checkout
   http://api.example.com/song-management/users/{id}/playlist/play
   ```

   > 译者注：
   >
   > 这里的控制器资源，打个比方吧，还是那个淘宝的API，上一个store里面我们讲了一个同步购物车，现在说一下这个购物车的结算，当用户点击结账按钮的时候，调用某一个API对购物车进行结账，结账就相当于是一个控制器资源，这时这个API我们就可以设计成上面例1的形式，在URI里面加入动词`checkout`，来表示结算

#### Consistency is the key

为了使歧义最小化以及获得最大化的可读性和可维护性，我们需要使用一套一致的资源命名约定以及URI格式，你可以参考以下的设计提示来进行实现：

1. **使用斜杠（/）来区分分级关系**
   斜杠（/）字符一般用在URI上来指定一个资源间的分级关系，例如：

   ```
   http://api.example.com/device-management
   http://api.example.com/device-management/managed-devices
   http://api.example.com/device-management/managed-devices/{id}
   http://api.example.com/device-management/managed-devices/{id}/scripts
   http://api.example.com/device-management/managed-devices/{id}/scripts/{id}
   ```

2. **不要在URI的末尾使用斜杠（/）**
   作为URI路径的最后一个字符，一个斜杠（/）并没有价值并且可能会导致混乱，最好是在URI的末尾不要加斜杠（/），例如

   ```
   http://api.example.com/device-management/managed-devices/
   http://api.example.com/device-management/managed-devices 
   ```

   例2的设计相对好些

3. **使用横杠（-）来提升URI的可读性**
   为了使你的URI更加容易被人理解和解释，在路径较长的部分，使用横杠（-）来提升名称的可读性

   ```
   http://api.example.com/inventory-management/managed-entities/{id}/install-script-location
   http://api.example.com/inventory-management/managedEntities/{id}/installScriptLocationreadable
   ```

   例1的可读性相对好些

4. **不要使用下划线（_）**
   有时候可能会用下划线（_）来代替横杠（-）作为分隔符，但是取决于应用前端，下划线（\_）字符可能会在某些浏览器或屏幕中部分模糊或完全隐藏。 
   为了避免这种误解，使用横杠（-）而不是下划线（_）

   ```
   http://api.example.com/inventory-management/managed-entities/{id}/install-script-location
   http://api.example.com/inventory_management/managed_entities/{id}/install_script_location
   ```

5. **在URI中使用小写字符**
   小写字符在URI中始终是首选
   [RFC 3986](http://www.rfc-base.org/txt/rfc-3986.txt) 定义除了请求方法和主机名称之外的其他的URI部分为大小写敏感的，例如：

   ```
   http://api.example.org/my-folder/my-doc  //1
   HTTP://API.EXAMPLE.ORG/my-folder/my-doc  //2
   http://api.example.org/My-Folder/my-doc  //3
   ```

   1和2是相同的，3却是不同的，因为他的`My-Folder`用了大写字符

6. **不要使用文件扩展名（在URI中）**
   URI中的文件扩展名看上去不怎么样并且并没有任何好处。将它除去可以减少URI的长度，没有留着它的理由。
   除了上述原因外，如果你想着重使用文件扩展名来表示API的媒体类型的话，你应当通过将去放入HTTP头部的`Content-Type `字段，以此来决定如何处理body的内容。

   ```
   /*不要这么用*/
   http://api.example.com/device-management/managed-devices.xml
   /*这么用才对*/
   http://api.example.com/device-management/managed-devices
   ```

#### Never use CRUD function names in URIs

URI不应当用来指示增删改查这些操作，URI应当用来指示资源并且不应当含有任何操作信息。HTTP请求方法应当被用来指示所调用的增删改查操作。

```
// 获取所有的设备信息
HTTP GET http://api.example.com/device-management/managed-devices  
// 创建一个新的设备
HTTP POST http://api.example.com/device-management/managed-devices 
// 通过给定的设备id来检索设备信息
HTTP GET http://api.example.com/device-management/managed-devices/{id}
// 通过给定的设备id来更新设备信息
HTTP PUT http://api.example.com/device-management/managed-devices/{id}
// 通过给定的设备id来删除设备信息
HTTP DELETE http://api.example.com/device-management/managed-devices/{id}
```

> 译者注：
>
> CRUD即指Create, Read, Update, Delete，就是增删改查四项操作
>
> 详细细节可参考[维基百科](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete)

#### Use query component to filter URI collection

在多数情况下，你可能需要对一个资源集合进行排序、过滤、或者限制某一个特定的资源属性的值。你不用创建一个新的API来实现这些，而是在资源集合的API中启用排序，筛选和分页功能，并将输入参数作为查询参数传递 。例如

```
http://api.example.com/device-management/managed-devices
http://api.example.com/device-management/managed-devices?region=USA
http://api.example.com/device-management/managed-devices?region=USA&brand=XYZ
http://api.example.com/device-management/managed-devices?region=USA&brand=XYZ&sort=installation-date
```