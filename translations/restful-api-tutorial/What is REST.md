### What is REST

REST是**RE**presentational **S**tate **T**ransfer（表述性状态传递）的缩写，是2000年由Roy Fielding在他[论文](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)中提出的一种适用于分布式超媒体系统（distributed hypermedia systems）的体系结构。

像任何其他架构风格一样，REST也有它自己的[6个指导性约束](https://restfulapi.net/rest-architectural-constraints/)，如果接口需要被称为RESTful，则必须满足这些约束。这些原则列在下面。

### Guiding Principles of REST

1. 客户端服务器模式（**Client–server**）：通过将用户界面问题与数据存储问题分离开来，我们可以提升跨多个平台的用户界面的可移植性，并通过简化服务器组件来提高可伸缩性。

2. 无状态（**Stateless**）：从客户端到服务器的每个请求都必须包含处理请求所需的所有信息，并且不能利用服务器存储任何的上下文，因此，会话状态需要完全保留在客户端上。

3. 可缓存（**Cacheable**）：缓存约束要求对请求响应中的数据隐式或明确标记为可缓存或不可缓存。如果响应是可缓存的，则客户端缓存有权重用该响应数据以用于以后等效的请求。

4. 统一接口（**Uniform interface**）：通过将一般的软件工程原理应用于组件接口，使得整个系统的体系结构得到简化，交互性得到提高。为了获得统一接口，需要多个体系结构的约束来共同指导组件的行为。REST定义了4个接口约定：

   * 资源标识符（identification of resources）

   * 通过资源的表现层来操纵资源（manipulation of resources through representations）

     > 译者注：
     >
     > 在某一特定的时间点处资源的状态称之为资源的表现层（resource representation）
     >
     > The state of resource at any particular timestamp is known as **resource representation**. 

   * 自我描述的信息（self-descriptive messages）

   * 以超媒体作为应用状态的驱动（hypermedia as the engine of application state）

5. 分层系统（**Layered system**）：分层系统由多个受约束且功能不同的组件进行的层级划分而组成，每个组件均不能被那些与他正在交互的组件之外的组件“看到”

6. 按需求编写代码【非必需】（**Code on demand (optional)**）：REST允许客户端通过从服务器下载和执行以applet或script形式的代码来扩展客户端的功能，这个设计允许通过减少需要预先实现的功能的数量来简化客户端。

### Resource

对于REST来说，对信息最关键的抽象就是资源（resource），任何可以被命名的信息均可以作为一个资源：一份文档或图像，一个临时的服务，其他资源的集合，一个非虚拟的物体（例如，一个人）等等。REST使用资源标识符（**resource identifier**）来区分组件间进行交互的特定的资源。

在某一特定的时间点处资源的状态称之为资源的表现层（resource representation），表现层通常包含数据，描述数据的元数据（metadata）以及可以帮助客户端转换到下一期望状态的超媒体链接。

> 译者注：
>
> 此处将timestamp译为时间点，timestamp主要指UNIX时间戳，UNIX时间戳是指从1970年1月1日（UTC/GMT的午夜）开始所经过的秒数

对数据格式的表述就是[media type](https://www.iana.org/assignments/media-types/media-types.xhtml)，media type标识定义了一个资源的表现层是如何被处理的，一个真正的Restful API看上去应该像超文本一样，每一个可寻址的信息单元应当显式（例如在链接或属性id）或非显式（例如包含在media type定义或资源表现层结构里）地携带着一个地址。

> 依据Roy Fielding的话:
>
> Hypertext（超文本）意味着同时呈现信息和控制，使得信息成为用户（或自动机）成为获取选择和以及选择动作的可供件。超文本不需要非得是HTML（或XML或JSON），机器可以通过跟踪链接来理解数据格式及关系类型。

此外，**资源的表现层应当是可以自我描述的（self-descriptive）**：客户端不需要知道一个资源到底是职工还是设备，它应当依据media type与资源间的关联来进行操作。所以，在实践中，你会创建大量的自定义media type，一般情况下，一种media type对应一种资源。

> 对每一个media type定义一个默认的处理模块，例如，HTML为每一个元素（element）定义了一组超文本的渲染过程以及浏览器行为，这与对资源的操作方法例如GET/PUT/POST/DELETE/并没有任何的关系。除了一些媒体元素定义了一组带有`href`标签的锚元素处理过程模型将会在被选中时创建一个超链接以及打开一个队应URI的数据获取请求（GET）

### Resource Method

另一个与REST有关的事情是用来执行预期过渡（transition）的资源方法（resource methods），很多人错误地将这些资源方法与HTTP的请求方法GET/PUT/POST/DELETE关联起来。

Roy Fielding从来都没有提过任何有关在某种情况下具体使用哪种（HTTP请求）方法的建议。他强调的仅有应当统一接口（uniform interface）。如果你决定使用POST请求来更新一个资源，而不是大多数人推荐的PUT，这其实也是没有问题的，而且应用程序的接口也同样会是RESTful。

在理想状况下，改变资源状态信息所需的一切都应成为对该资源API响应（response）的一部分，包括方法以及他们将在何种状态下离开当前的状态。

> 一个REST API在进入的时候应当没有除了为预定使用者约定的URI以及一组标准化的media type之外的其他的任何预先的了解，即预计任何可能使用这个API的客户端都应当可以理解。从那时起，所有的应用状态过渡必需由客户端对服务端提供的选择来进行驱动，这些选择存在于接收到的表现层中，或者是对用户操作的描述。这些过渡可能由客户端对media type的了解以及资源的通信机制决定，这两者都可以被即时修改（例如：需求的变化）
>
> [这里的失败意味着其他的信息正在驱动着交互而不是超文本]

> 译者注：
>
> 这里可以参考一下[GitHub API](https://api.github.com/)的设计，当我们访问这个域名时，即会获得一份当前可用的API列表，其中包含了每个API的访问地址
>
> > 这其实就是上文中所说的REST API在进入的时候没有除了为预定使用者约定的URI以及一组标准化的media type之外的其他的任何预先的了解。
>
> 当我们进一步访问其中的某一个API地址时，我们可以获取有关信息，以及API文档地址，产生一个过渡，例如访问[current_user_url](https://api.github.com/)这个API我们可以得到如下信息：
>
> ```json
> {
>   "message": "Requires authentication",
>   "documentation_url": "https://developer.github.com/v3/users/#get-the-authenticated-user"
> }
> ```
>
> `message`字段告诉我们需要授权，授权也就相应产生了一个应用状态上的过渡。

另一个可以帮助你建立RESTful API的事情是基于查询的API结果应当由包含摘要信息的链接来表示，而不是一个源资源表现层的数组，因为查询并不是对资源标识的替代品。

### REST and HTTP are not same !!

很多人喜欢拿HTTP与REST相比较，但其实HTTP和REST是两样不同的东西。

> **REST != HTTP**

虽然REST同样倾向于使WEB（internet）更加精简和高效，他（代指REST发明人Roy fielding）主张更加严格地遵循REST准则。同样，这也是人们试图将REST与Web（HTTP）进行比较的地方。Roy fielding在他的博士论文里从来没有提到过任何（有关REST的）直接的实现方法-包括任何协议选择以及HTTP。直到现在，你只要履行REST的6个基本准则，你就可以将你的接口称作RESTful。

简单来说，在REST体系风格中，数据和功能通常被认为是一种可以通过统一资源定位符（URI）访问的一种资源。通过使用一组简单，定义明确的操作来操纵这些资源。客户端与服务端通过使用标准化的接口和协议（通常是HTTP）来交换资源表现层。

资源与其表现层进行分离后即可以使用多种形式来访问其内容，就比如说HTML, XML, plain text, PDF, JPEG, JSON等等。资源的metadata同样是可以访问并使用的，例如，进行缓存的控制、发现传输中的错误、协商适当的资源表现形式以及执行授权与访问控制。最重要的是，每一次带有资源的交互都是无状态的。

所有的这些准则使得RESTful应用变得简单、轻量和高效。

参考资料：

<http://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven>
[http://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)