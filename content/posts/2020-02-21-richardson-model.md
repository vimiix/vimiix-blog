---
title: '[译]Richardson成熟度模型'
date: 2020-02-21 16:22:48
categories: 'translation'
tags: ['translation', 'Richardson']
---

> 原文链接：https://martinfowler.com/articles/richardsonMaturityModel.html

### 迈向 REST 的荣耀之巅

Leonard Richardson 提出的一个模型，将实现 REST 方法的主要元素分解为三个步骤，分别包括：资源（Resources）、HTTP 动词(HTTP Verbs，如`GET`、`POST`等)和超媒体控制（Hypermedia Controls）。

在[Rest In Practice](https://www.amazon.cn/dp/0596805829/ref=sr_1_1?ie=UTF8&qid=1551228104&sr=8-1&keywords=REST+in+Practice)一书中，解释了如何使用 Restful Web Service 来处理企业面临的许多集成问题。本书的核心观点是，Web 就是一个大规模可扩展的分布式系统存在、并可以很好的工作的证明，而我们可以根据这一观点更容易地构建集成系统。

![](https://static.vimiix.com/uPic/2021-04-06/5hdvom.png)

为了说明一个“Web 风格”系统的特定属性，作者使用了由 Leonard Richardson 提出的“RESTful 成熟度模型”，该模型在一次[QCon Talk](http://www.crummy.com/writing/speaking/2008-QCon/act3.html)中被谈到。通过这一模型，可以很好的思考如果使用 REST，所以我也会尝试添加一些自己的解释。（本文中所使用协议示例只是为了更好的说明，并不建议在实际生产中编码实现或测试，因为其在细节上可能会存在问题）

## Level 0

该模型的出发点是使用 HTTP 作为远程交互的传输系统，但不使用 Web 的任何机制。基本上就是使用 HTTP 作为你远程交互机中的隧道机制，通常基于“远程过程调用”（RPC，[Remote Procedure Invocation](http://www.eaipatterns.com/EncapsulatedSynchronousIntegration.html)）。

![](https://static.vimiix.com/uPic/2021-04-06/SN9yFm.png)

例如，我想预约我的医生。预约软件首先需要知道在指定日期内医生什么时间有空位，所以它需要向医院预约系统发送请求以获取该信息。在 Level 0 级场景中，医院会将某个 URI 做为一个公共服务点。然后，我就可以向该服务点发送一个文档请求。

```html
POST /appointmentService HTTP/1.1 [various other headers]

<openSlotRequest date="2010-01-04" doctor="mjones" />
```

然后，服务器会返回一个我所需信息的文档：

```html
HTTP/1.1 200 OK [various headers]

<openSlotList>
  <slot start="1400" end="1450">
    <doctor id="mjones" />
  </slot>
  <slot start="1600" end="1650">
    <doctor id="mjones" />
  </slot>
</openSlotList>
```

在本例中我使用了 XML 做为内容格式，实现可以格式的，如：JSON、YAML、键-值对、或其它自定义格式。

接下来就可以创建一个预约，预约也是向服务端发送文档实现：

```html
POST /appointmentService HTTP/1.1 [various other headers]

<appointmentRequest>
  <slot doctor="mjones" start="1400" end="1450" />
  <patient id="jsmith" />
</appointmentRequest>
```

如果一切正常的话，我会收到一个预约成功的响应：

```html
HTTP/1.1 200 OK [various headers]

<appointment>
  <slot doctor="mjones" start="1400" end="1450" />
  <patient id="jsmith" />
</appointment>
```

如果有问题，例如别人在我之前进行了预约，那么我会在响应体中收到某种错误信息：

```html
HTTP/1.1 200 OK [various headers]

<appointmentRequestFailure>
  <slot doctor="mjones" start="1400" end="1450" />
  <patient id="jsmith" />
  <reason>Slot not available</reason>
</appointmentRequestFailure>
```

至此，这都是一个简单的 RPC 风格的系统。这很简单，因为它只是来回传递普通的旧 XML（POX）。如果你使用的是 SOAP 或 XML-RPC，基本上也是相同的机制，唯一不同的是将 XML 消息包装在了不同的数据格式中。

## Level 1-资源（Resources）

在成熟度模型（RMM:Richardson Maturity Model）中，迈向 REST 的第一步就是引入资源的概念。接下来，我们所要讨论的是各个资源，而不是将所有请求发送到单一的服务端点。

![](https://static.vimiix.com/uPic/2021-04-06/1p658H.png)

因此，在我们初步查询时，可能需要指定的医生资源：

```html
POST /doctors/mjones HTTP/1.1 [various other headers]

<openSlotRequest date="2010-01-04" />
```

服务端的响应中会有相同的基本信息，但每个空闲时间段现在都是可以做为单独录址的资源：

```html
HTTP/1.1 200 OK [various headers]

<openSlotList>
  <slot id="1234" doctor="mjones" start="1400" end="1450" />
  <slot id="5678" doctor="mjones" start="1600" end="1650" />
</openSlotList>
```

有了这些资源，就可以向指定的时间段资源发送预约：

```html
POST /slots/1234 HTTP/1.1 [various other headers]

<appointmentRequest>
  <patient id="jsmith" />
</appointmentRequest>
```

如果一切正常，会收到一个与前面类似的响应：

```html
HTTP/1.1 200 OK [various headers]

<appointment>
  <slot id="1234" doctor="mjones" start="1400" end="1450" />
  <patient id="jsmith" />
</appointment>
```

现在的区别在于，任何人想对预约做任何操作（例如：预约一些检查），都需要先得到预约资源，该资源是一个类似于这样的 URI：`http://royalhope.nhs.uk/slots/1234/appointment`，然后再向该资源发送请求。

对于像我这样的对象，这就像对象标识的概念。我们不是通过传入参数来调用某些函数，而是在一个特定对象上调用一个方法，同时将其它信息做为参数传入。

## Level 2-HTTP 动词（HTTP Verbs）

在 Level 0 和 Level 1 中，我们所有的交互都使用了 HTTP POST 动词，但有些人会使用 GET 或其它动词来代替。在目前的级别上它们区别并不大，其都是作为“通道”来使你可以通过 HTTP 完成交互。但在 Level 2 级别中不再这样，而是尽可能的使用 HTTP 协议中定义的合适的 HTTP 动词。

![](https://static.vimiix.com/uPic/2021-04-06/kn3btc.png)

这时，获取医生空闲时间段列表应该使用`GET`：

```html
GET /doctors/mjones/slots?date=20100104&status=open HTTP/1.1 Host:
royalhope.nhs.uk
```

收到的响应与之前使用`POST`时是一样的：

```html
HTTP/1.1 200 OK [various headers]

<openSlotList>
  <slot id="1234" doctor="mjones" start="1400" end="1450" />
  <slot id="5678" doctor="mjones" start="1600" end="1650" />
</openSlotList>
```

在 Level 2 中，对于获取资源这样的请求使用`GET`是至关重要的，因为 HTTP 将`GET`定义为安全的操作，也就是说它不会对资源的所有状态进行任何重大的更改。这意味着我们可以任何顺序安全的多次调用`GET`，并且每次都会得到相同的结果。这样做的一个重要结果是，允许所有参与路由使用缓存，该机制是目前 Web 运行良好的关键因素之一。HTTP 中包含了很多来支持缓存的机制，这些机制可以被所有通讯参与者使用。

为了创建预约，我们需要使用一个能修改状态的 HTTP 动词，可以是`POST`或`PUT`。这里我们使用与之前相关`POST`请求：

```html
POST /slots/1234 HTTP/1.1 [various other headers]

<appointmentRequest>
  <patient id="jsmith" />
</appointmentRequest>
```

对于是使用`POST`还是`PUT`这超出了本文的讨论范畴，也许以后可以用一篇文章单独讨论。但是还需要指出，有些人将`POST`/`PUT`与`CREATE`/`UPDATE`建立对应关系是错误的，它们之间的选择与此截然不同。

以上即使我使用了与 Level1 中相同的`POST`请求，远程服务的响应还是明显不同的。如果一切顺利，服务端将返回`201`的响应代码，以表明创建了新资源。

```html
HTTP/1.1 201 Created Location: slots/1234/appointment [various headers]

<appointment>
  <slot id="1234" doctor="mjones" start="1400" end="1450" />
  <patient id="jsmith" />
</appointment>
```

`201`响应包含有 URI 的位置(Location)属性，客户端将来可以使用该 URI 获取该资源的当前状态。以上响应还包含了资源信息，以便客户端无需额外的请求来获取资源。

如果出现问题时，还有不同之处。例如，其他人预订了该资源：

```html
HTTP/1.1 409 Conflict [various headers]

<openSlotList>
  <slot id="5678" doctor="mjones" start="1600" end="1650" />
</openSlotList>
```

以上响应的重点在于，使用 HTTP 响应码来指示出错的地方。在上例中，使用`409`表示该资源已经被更新。在 Level 2 中，我们会明确使用与上面类似的某种类型的错误响应，而不是返回`200`再包含错误响应。具体使用哪种响应码由协议设计者决定，但出现错误时就不应该使用`2xx`响应。在 Level 2 引入了使用 HTTP 动词和 HTTP 响应码。

这里有一个不一致的问题，REST 倡导者建议使用所有 HTTP 动词，还试图从 Web 的成功之处来学习和借鉴。但是，在万维网的实践中并没有太多使用`PUT`或`DELETE`，的确有更多使用`PUT`和`DELETE`的合理原因，但 Web 并不是证据之一。

Web 提供的关键元素是将安全（例如`GET`）和不安全操作进行严格分离，并使用状态代码来表示你遇到的各种错误。

## Level 3-超媒体控制（Hypermedia Controls）

在最后一个级别中引入了一些你经常听到的、其缩写并不好看的“HATEOAS”(Hypertext As The Engine Of Application State)中的一些内容。它解决了如何从列表中获取时间段资源及如何创建预约的问题。

![](https://static.vimiix.com/uPic/2021-04-06/1sCFjl.png)

依然是从使用在 Level 2 中使用的`GET`开始：

```html
GET /doctors/mjones/slots?date=20100104&status=open HTTP/1.1 Host:
royalhope.nhs.uk
```

但在响应中会有一个新元素：

```html
HTTP/1.1 200 OK [various headers]

<openSlotList>
  <slot id="1234" doctor="mjones" start="1400" end="1450">
    <link rel="/linkrels/slot/book" uri="/slots/1234" />
  </slot>
  <slot id="5678" doctor="mjones" start="1600" end="1650">
    <link rel="/linkrels/slot/book" uri="/slots/5678" />
  </slot>
</openSlotList>
```

每个资源都有一个包含 URI 的`link`元素，用来告诉我该如何创建预约。

超媒体控制的关键在于它告诉我们下一步我们可以做什么，以及操作所需资源的 URI。与我们必须提前知道在哪里创建预约请求不同（Level2 中），在响应中的超媒体控件告诉了我们下一步该如何做。

创建预约请求与 Level2 中一样：

```html
POST /slots/1234 HTTP/1.1 [various other headers]

<appointmentRequest> <patient id="jsmith" /></appointmentRequest>
```

在响应中也包含一系列的超媒体控制，用以告诉我们接下来可以怎么操作：

```html
HTTP/1.1 201 Created Location: http://royalhope.nhs.uk/slots/1234/appointment
[various headers]

<appointment>
  <slot id="1234" doctor="mjones" start="1400" end="1450" />
  <patient id="jsmith" />
  <link rel="/linkrels/appointment/cancel" uri="/slots/1234/appointment" />
  <link
    rel="/linkrels/appointment/addTest"
    uri="/slots/1234/appointment/tests"
  />
  <link rel="self" uri="/slots/1234/appointment" />
  <link
    rel="/linkrels/appointment/changeTime"
    uri="/doctors/mjones/slots?date=20100104&status=open"
  />
  <link
    rel="/linkrels/appointment/updateContactInfo"
    uri="/patients/jsmith/contactInfo"
  />
  <link rel="/linkrels/help" uri="/help/appointment" />
</appointment>
```

超媒体控件的一个明显好处是，它允许服务器在不破坏客户端的情况下更改其 URI 方案。只要客户端查找“addTest”这一 URI，服务端开发团队就可以随意处理除初始入口点以外的所有 URI。

另一个好处是它可以帮助客户端开发人员探索协议。这些链接为客户端开发人员提供了下一步可操作的提示。它并没有提供所有信息：“self”和“cancel”控制都指向相同的 URI - 客户端需要自行了解哪一个使用`GET`、哪一个使用`DELETE`，但其至少为他们提供了一个起点，有需要时开发人员可以去协议文档中查找类似的 URI 以获取详细说明。

同样，它允许服务端团队在响应中添加新链接以引入新功能。如果客户端开发人员关注到未知链接，就可以能通过这些链接来进一步探索。

关于如何表示超媒体控件，目前并没有绝对的标准。我在这里所使用的是使用 REST in Practice 团队的推荐做法，即遵循 ATOM（[RFC 4287](https://tools.ietf.org/html/rfc4287)），我使用了带有`uri`属性的``元素作为目标 URI，并使用`rel`属性来描述这种关系。对于众所周知的关系（如，`self`用于引用元素本身）是不加修饰的，任何特定于该服务的关系都是一个完整的 URI。在 ATOM 中，对于众所周知的链接（linkrels）的定义是[Registry of Link Relations](http://www.iana.org/assignments/link-relations.html)。正如我所写，这些仅限于 ATOM 所做的事情，ATOM 通常被视为 Level3 级 REST 的领导者。

我应该强调，RMM 虽然是思考 REST 元素的一种好方法，但它并不是 REST 本身级别的定义。 Roy Fielding 也明确表示，3 级的 RMM 是 REST 的先决条件。像软件中的许多术语一样，REST 有很多定义，但是自从 Roy Fielding 创造了这个术语以来，他的定义应该比大多数人更具权威性。

我觉得 RMM 的有用之处在于，它提供了一个很好的一步一步的方法来理解 RESTful 思维背后的基本思路。因此，我将其视为帮助我们了解概念的工具，而不是应该在某种评估机制中使用的东西。尽管我们还没有足够的案例来证明 RESTful 方法是整合系统的正确方法，但其仍是一种非常有吸引力的方法，也是我在大多数情况下推荐的方法。

与 Ian Robinson 谈到这一点时，他强调到，当 Leonard Richardson 首次提出这个模型时，他发现这个模型很有吸引力的是它与常见设计技术的关系：

- Level 1 通过拆分来解决复杂性的问题，将大型服务端点分解为多个资源。
- Level 2 引入了一套标准动词，以便我们能以相同的方式处理类似的情况，消除不必要的变化。
- Level 3 引入了可发现性，提供了一种使协议具有自我描述能力的方法。

其结果就是，这一模型可以帮助我们思考我们想要提供的 HTTP 服务的类型，并按人们期望的交互构建。REST 层级的意义
