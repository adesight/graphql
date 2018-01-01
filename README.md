# GraphQL 技术栈揭秘

> 本文转载自：[众成翻译](http://www.zcfy.cc)
> 译者：[billyma](http://www.zcfy.cc/@billyma)
> 链接：[http://www.zcfy.cc/article/4549](http://www.zcfy.cc/article/4549)
> 原文：[https://dev-blog.apollodata.com/the-graphql-stack-how-everything-fits-together-35f8bf34f841](https://dev-blog.apollodata.com/the-graphql-stack-how-everything-fits-together-35f8bf34f841)

## 本文整理自2017年 GraphQL 峰会上的演讲，详述缓存、追踪、模式拼接和 GraphQL 未来发展等有关话题。

Facebook 开源 GraphQL 至今已两年有余。两年来，社区成倍增长，成千上万的公司在产品中使用 GraphQL。在今年10月份举办的 GraphQL 峰会上，我有幸在第二天发表开幕主题演讲。读者可以在 YouTube 上观看[完整的演讲](https://www.youtube.com/watch?v=ykp6Za9rM58)，或阅读本文浏览概要。

首先，我将介绍一下 GraphQL 的现状，然后讨论在不久的将来如何发扬它的主要优势。具体来说，我们将介绍三个完整的 GraphQL 集成示例：缓存，性能追踪和模式拼接。让我们开始吧！

------------

### GraphQL 的特别之处？

有三个主要因素使得 GraphQL 从其他 API 技术中脱颖而出：

1. GraphQL 拥有明确的查询语言，这是描述**数据需求**的好方法，还有定义良好的模式，能够暴露 **API 能力**。它是唯一能够指定方程两侧（译者注：视图和模型）的主流技术，它的所有优势都源于这两个概念的相互作用。

2. GraphQL 能够帮你将 API 提供者与消费者解耦。在像 REST 这样基于端点的 API 中，返回数据的结构由服务器决定。在 GraphQL 中，响应的结构与使用它的 UI 代码保持关联，这样会更加自然，使得你可以专注于**关注业务而不是技术**。

3. 由于 GraphQL 查询会被附加到使用它的代码上，因此可以将该查询视为**数据获取单元**。GraphQL 能够预先知道 UI 组件的所有数据需求，从而启用新的服务器功能。例如，在单个查询中对底层 API 调用进行批处理和缓存，代表了 UI 中某一部分所需的数据，使用 GraphQL 将会令其非常简单。

![](http://p0.qhimg.com/t0159f934d506b65adc.png)

关注业务，而不是技术：GraphQL 将数据需求放在它们所属的客户端。

现在，我们来看看人们对于数据请求经常关注的三个方面，以及 GraphQL 如何利用上面提到的属性进行全方位改进。

![](http://p0.qhimg.com/t01b29e0b65fe2535b2.png)

请注意，虽然下面讨论的许多功能是你现在就能够使用的，但其中一部分仍属于未来的愿景。如果你和我一样对这件事感到兴奋，那么请滚动到底部参与其中。

### 1. 交叉请求缓存

人们经常问的第一件事情就是——我怎么用我的 GraphQL API 做交叉请求缓存？将常规 HTTP 缓存应用于 GraphQL 时会出现这样一些问题：

* HTTP 缓存通常不支持 POST 请求或长缓存键

* 越复杂的请求可能意味着更少的缓存命中

* GraphQL 的传输是独立的，因此 HTTP 缓存并不总是可行

但是，GraphQL 也带来了许多新的可能性：

* 访问后端时，在模式和解析器旁声明缓存控制信息的可能性

* 基于模式的自动化细粒度缓存控制，而不必单独考虑每个请求

我们应该如何使用 GraphQL 来生效缓存，以及如何利用好这些新的可能性呢？

#### 究竟应该在哪里执行缓存？

首先，我们得确定缓存功能应该设计在哪。一开始的直觉可能是缓存逻辑应该放在 GraphQL 服务器本身内。然而，像 DataLoader 这样简单的工具在同时执行多个 GraphQL 请求时并不能很好地工作，而将缓存功能放在我们的服务器代码中会使实现变得非常复杂。因此我们应该把它放在别的地方。

事实证明，就像在 REST 中一样，在 API 层的两端都进行缓存是有意义的：

1. 在位于基础设施层中的 GraphQL API 之外缓存整个响应。

2. 缓存 GraphQL 服务器底层对数据库和微服务的请求。

对于第二点，需要你现有的缓存基础设施工作良好。对于第一点，我们需要一个位于 API 之外的层，并且能够以 GraphQL 的方式执行缓存等操作。本质上讲，这种架构能够将复杂度从 GraphQL 服务器中分离出来：

![](http://p0.qhimg.com/t019a14168aa374a14f.png)

将复杂度转移到位于客户端和服务器之间新的分层。

我将这个组件称之为 GraphQL 网关。在 Apollo 团队内部，我们认为这个新的网关层非常重要，每个人都需要将其作为 GraphQL 基础架构的一部分。

**这就是为什么在今年的 GraphQL 峰会期间，我们推出了首个 GraphQL 网关 [Apollo Engine](https://dev-blog.apollodata.com/introducing-apollo-engine-insights-error-reporting-and-caching-for-graphql-6a55147f63fc)**。

[![](http://p0.qhimg.com/t01a68af2361981d5a6.png)](https://www.apollographql.com/engine)

#### 用于缓存控制的 GraphQL 响应扩展

正如我在开始时提到的那样，GraphQL 的一大优点是其拥有巨大的工具生态系统，所有工具都围绕着 GraphQL 的查询和模式来工作。我认为像缓存这样的功能也应该以同样的方式工作。这就是为什么我们要介绍 [Apollo 缓存控制](https://github.com/apollographql/apollo-cache-control)，它使用了 GraphQL 规范中内置的一个名为 **扩展** 的特性，将缓存控制信息包含在响应中。

使用我们的 [JavaScript 参考实现](https://github.com/apollographql/apollo-cache-control-js)，可以很容易在你的 schema 中添加缓存控制标识：

[![](http://p0.qhimg.com/t01bd5ec9aa954ef764.png)](https://github.com/apollographql/apollo-cache-control-js)

使用 apollo-cache-control-js 的缓存控制标识

这个新的缓存控制规范建立在 GraphQL 的主要优势上，我对此感到非常兴奋。它使你能够细粒度地指定相关数据的信息，并利用 GraphQL 执行将相关的缓存控制标识发回给调用者，而且这与语言和传输方式无关。

自从我在 GraphQL Summit 上发表这个演讲后，[Oleg Ilyenko](https://medium.com/@oleg.ilyenko) 已经发布了[ Sangria 的缓存控制工作版本](https://twitter.com/easyangel/status/925037659482976257)，由他维护的 Scala GraphQL 实现。

#### 使用网关缓存

现在我们可以在 GraphQL 服务器中返回缓存控制标识，所以我们有一个明确的方法来在网关中进行缓存。技术栈中的每一部分都起着作用：

![](http://p0.qhimg.com/t0121e7ab4c0ab6cd9f.png)

技术栈中所有部分之间的协作可以借助缓存来完成。

需要注意的一件很酷的事情是，大多数人已经在他们的 GraphQL 栈中拥有了一个缓存：比如 Apollo Client 和 Relay 这样的库在前端缓存你的数据。在 Apollo 客户端的未来版本中，来自响应的缓存控制信息将被用于自动在前端过期旧数据。所以，就像 GraphQL 的其他部分一样，服务器描述它的功能，客户端指定它的数据需求，然后一切工作配合正常。

现在，我们来看看另一个贯穿整个技术栈的 GraphQL 功能的例子。

### 2. 追踪

借助 GraphQL，前端开发人员能够以相较基于端点的系统更精细的方式处理数据。他们可以确切地请求他们需要的数据，并跳过他们不打算使用的字段。受益于此，我们可以展示详细的性能信息，并以前所未有的方式使其具有可操作性。

![](http://p0.qhimg.com/t0154cffc62a1f1b42b.png)

不要满足于不透明的总查询时间—— GraphQL 使你能够获晓字段级别的详细耗时。

你可以认为 GraphQL 是首个内置细粒度查看的 API 技术。这不是借助某个特定的工具—— GraphQL 第一次合理地让前端开发人员能够获得逐个字段的执行时间，然后调整他们的查询来解决问题。

#### 跨栈追踪

事实证明，追踪跟缓存一样，对于整个栈的协作大有裨益。

![](http://p0.qhimg.com/t013b9827662f68efc4.png)

每一部分在提供追踪数据和使其具有可操作性方面发挥着作用。

服务器能够提供相关信息作为响应结果的一部分，就像提供缓存标识一样，网关可以提取和聚合这些信息。重申，使用网关组件来处理复杂的功能，而不是在服务器进程中处理。

在这种情况下，客户端的主要功能是将查询与 UI 组件连接起来。这非常关键，因此你可以将 API 层的性能与其对前端的影响联系起来。这将是首次，你可以直接将后端请求的性能与其在页面上受影响的 UI 组件关联起来。

#### GraphQL 追踪扩展

就像缓存一样，通过利用 GraphQL 的响应扩展功能，能够以服务器无关的方式实现上述功能。[Apollo Tracing](https://github.com/apollographql/apollo-tracing) 规范已经在 [Node](https://github.com/graphql/graphql-js)，[Ruby](https://github.com/rmosolgo/graphql-ruby)， [Scala](https://github.com/sangria-graphql/sangria)，[Java](https://github.com/graphql-java/graphql-java) 和 [Elixir](https://github.com/absinthe-graphql/absinthe) 上实现了，其定义了一种 GraphQL 服务器以标准方式为解析器返回耗时数据的方式，能够被任何工具所使用。

想象一下，你所有的 GraphQL 工具都可以访问性能数据：

![](http://p0.qhimg.com/t01e19b5f4ebe8bae87.png)

共享抽象允许工具使用诸如追踪数据等信息。

借助 Apollo Tracing，你可以在 GraphiQL，编辑器或其他任何地方获取性能数据。

目前为止，我们一直在谈论单个客户端和单个服务器之间的交互。我们的最后一个例子，一起来看看我们如何通过 GraphQL 来模块化我们的架构。

### 3. 模式拼接

GraphQL 最好的地方就是可以在一个地方访问所有的数据。但是，这样做的代价是：你需要将整个 GraphQL 模式作为单个代码库来实现，以便能够在一个请求中查询它。 如果你拥有一个模块化的架构，而该架构同时具有单个统一 GraphQL API 的好处呢？

模式拼接的概念很简单：GraphQL 可以很容易地将多个 API 合并为一个，因此你可以将模式的不同部分实现为独立的服务。这些服务可以分开部署，用不同的语言编写，甚至可以由不同的组织维护。

这有[一个例子](https://launchpad.graphql.com/130rr3r49)：

[![](http://p0.qhimg.com/t01bb5408811071ad9e.png)](https://launchpad.graphql.com/130rr3r49)

在单个查询中结合来自 GraphQL 峰会的票务系统和天气 API 的数据：
[https://launchpad.graphql.com/130rr3r49](https://launchpad.graphql.com/130rr3r49)

在上面的截图中，你可以看到对于一个拼接 API 的单个查询是如何将提供不同服务的两个独立查询合并在一起，这种方式对客户端来说是完全不可见的。通过这种方式，你可以像使用乐高积木般合并多个 GraphQL 模式。

我们实现了一个能用的版本，放到了 Apollo graphql-tools 库里，你可以现在去试用。[具体查看文档。](https://www.apollographql.com/docs/graphql-tools/schema-stitching.html)

#### 在网关中拼接

模式拼接的概念在整个栈中也可以很好地工作。我们认为新的网关层长期来看会是一个执行拼接操作的好地方，你可以使用任何你想要的技术构建你的模式，如 [Node.js](https://www.howtographql.com/graphql-js/1-getting-started/)，[Graphcool](https://www.graph.cool/) 或 [Neo4j](https://neo4j.com/developer/graphql/)。

![](http://p0.qhimg.com/t018ce4a1d81b8b49f9.png)

事实证明，模式拼接与 GraphQL 栈的每一部分都息息相关。

客户端也可以加入进来！就像你使用一个查询从多个后端加载数据一样，你也可以在客户端上组合数据源。最近发布的 [Apollo Client 2.0](https://dev-blog.apollodata.com/apollo-client-2-0-5c8d0affcec7) 中的新客户端状态管理功能使你可以在单个查询中获取来自于客户端状态和任意数量后端的数据。

### 结论

希望你通过阅读这篇文章或者看完演讲后能知道一件事情，那就是尽管现今的 GraphQL 工具已经很棒了，未来还有更多的潜力。我们只是抓住了 GraphQL 提供的抽象和功能的表面。

我想以上述提到的概念的待办事项清单来结束本篇文章：

![](http://p0.qhimg.com/t018407b494cb714d23.png)

整合这些新功能还有很多工作要做，特别是在开发者工具和编辑器方面。

要发掘 GraphQL 的全部潜力，还有很多事情要做。在 Apollo 团队，我们正在尽我们所能来做这件事情，但单独的一个人，团队或组织不可能独自完成这个目标。为了实现未来的愿景，我们需要共同合作，共同构建出所有这些解决方案。

无论我们在哪里，有一点是明确的：GraphQL 已经成千上万的公司的变革技术，而这只是一个开始！ 我迫不及待地想知道在接下来的两年，五年和十年中编写应用程序会是什么样子，因为这将是不可思议的。

![](http://p0.qhimg.com/t013e2a02e9eab54bd1.png)

* * *

### 参与其中

如果你像 Apollo 一样对 GraphQL 的潜力感兴趣，请考虑加入我们的社区。我们已经组建了一个[帮助页面](https://www.apollographql.com/docs/community/) 来帮助你入门。

GraphQL 峰会上的其他会议即将发布！在Twitter上关注 [@graphqlsummit](https://twitter.com/graphqlsummit) 或订阅 [Apollo 的 Youtube 频道](http://youtube.com/apollographql) 以获取最新内容。

终于写完了……
