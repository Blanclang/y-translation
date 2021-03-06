>原文地址：[how-push-works](https://developers.google.com/web/fundamentals/push-notifications/how-push-works)

>译文地址：[推送是怎么工作的?](https://github.com/yued-fe/y-translation/blob/master/todo/push-notifications/how-push-works.md)

>译者：[刘鹏](https://github.com/git-patrickliu)

>校对者：[任家乐](https://github.com/jennyrenjiale)、[张卓](https://github.com/Zhangdroid)


# 推送是怎么工作的?

在我们接触这个 API 之前，让我们先从一个高层次从头到尾来看一下推送。稍后我们将通过逐一介绍各个主题和 API 让你知道为什么推送是这么重要。

下面将介绍实现推送的三个关键步骤：

1. 添加客户端侧的逻辑来给用户订阅推送（也就是使用 Web App 当中的 JavaScript 和 UI，帮助用户注册推送消息）。
2. 从你的后台/应用调用 API 来触发推送消息到用户的设备。
3. 当推送到达用户的设备时，service worker 可以接收到“推送事件”，通过使用 service worker 你将展示出一个通知。

让我们更加详细的来看一下这三个步骤。

## Step 1: 客户端侧

第一步是 “订阅” 一个用户来推送消息。

订阅一个用户需要2个条件。第一，从用户那里获取给他们发送消息的**许可**。第二，从浏览器那里获取 `PushSubscription`。

`PushSubscription` 包含了我们需要推送消息给目标用户的所有信息。这个信息可以某种程度上认为是用户设备的 ID。

这些全部可以通过 JavaScript 调用 [Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API) 来实现。

在我们订阅一个用户之前，我们需要生成一套应用服务器密钥。这个我们后续会解释。

应用服务器密钥，也被称为 VAPID 密钥，对你的服务器来说是独一无二的。有了它，推送服务知道是哪一个应用服务器注册了一个用户，并且确保了是同一个服务器触发了推送消息给那个用户。

一旦你订阅了一个用户，并且拿到了 `PushSubscription`，你需要将关于这个 `PushSubscription` 的详细信息发送至你的后台/服务器。在服务器上，你需要将这个订阅保存至数据库当中，并使用它给用户推送消息。

![确保你发送了 `PushSubscription` 到你的后端](https://developers.google.com/web/fundamentals/push-notifications/images/svgs/browser-to-server.svg)

## Step 2: 发送一个推送消息

当你想要发送一个推送消息到目标用户，你需要执行一个 API 调用到推送服务。这个 API 包括：发送的数据、发送的对象和任何关于如何发送这条消息的标准。一般情况下这个 API 调用是由你的服务器来完成的。

你可能会有一些疑问：

- 推送服务是谁/是什么？

- 这个 API 长什么样？它是 JSON 格式？XML 格式？还是其他什么格式？

- 这个 API 能干什么？

### 推送服务是谁/是什么?

推送服务接收到一个网络请求，校验该请求的正确性，然后发送一个推送消息到合适的浏览器。如果浏览器正好离线，这条消息会排队直到这个浏览器重新在线之后发送给它。

每个浏览器都能使用他们想用的任何一个推送服务，这个是开发者管控不了的。这其实并不是一个问题，因为每一个推送服务都接受**相同**的 API 调用。也就是说你不必担心这个推送服务是谁。你只需要确保你的 API 调用是正确的。

要获得合适的 URL 来触发推送消息（也就是推送服务的 URL），你只需要查看一下前面获得的 `PushSubscription` 中 `endPoint` 的值。

下面是 **PushSubscription** 的一个示例：

	{
	  "endpoint": "https://random-push-service.com/some-kind-of-unique-id-1234/v2/",
	  "keys": {
	    "p256dh" :
	"BNcRdreALRFXTkOOUHK1EtK2wtaz5Ry4YfYCA_0QTpQtUbVlUls0VJXg7A8u-Ts1XbjhazAkj7I99e8QcYP7DkM=",
	    "auth"   : "tBHItJI5svbpez7KI4CCXg=="
	  }
	}
	
这个示例当中的 **endpoint** 是 “https://random-push-service.com/some-kind-of-unique-id-1234/v2/ ”。 推送服务应该是 “random-push-service.com”，如 `some-kind-of-unique-id-1234` 所示，每个 endpoint 对用户来说都是独一无二的。
当你开始着手于推送之后，你会注意到这个模式。

关于上述示例当中的 **key** 这个字段，我们后续会讲到，这里就先不解释了。

### 这个 API 是什么样的?

我前面提到每个 Web 推送服务需要的是相同的 API 调用。这个 API 就是 [**Web 推送协议**](https://tools.ietf.org/html/draft-ietf-webpush-protocol)。
它是一个 IETF 标准，定义了如何向一个推送服务执行一个 API 调用。

这个 API 调用需要设置一些头部，并且需要以字节流的方式发送数据。我们将看一下如何用库来执行这个 API，以及自己如何来实现这个 API。

### 这个 API 能做什么？

这个 API 提供了发送消息到用户的一种方法，消息可以携带，也可以不携带数据。同时它也提供了**如何**发送消息的指导方法。

你通过推送服务发送的数据必须是加密的。原因是推送服务可以是任何人，你必须防止他能够看到数据明文。这很重要，因为是浏览器决定（而不是开发者）该用哪一个推送服务，而那些不安全的推送服务可能打开浏览器的安全大门。

当你触发一个推送消息，推送服务接收了 API 调用，然后将消息放到队列当中。这个消息会一直呆在队列当中，直到用户的设备上线，然后推送服务就可以将消息发送过去。你能给推送服务指示来决定推送消息是如何排队的。

这些指示包含如下：

- 推送消息的 TTL（生存时间）。这个定义了一条消息在没有发送并移除前，能在队列当中排多长时间。

- 设置消息的紧急度。这在推送服务为了保持用户的电量情况下，只推送高优先级的消息时很有用。

- 给推送消息一个话题名，拥有相同话题的新消息将替换掉仍在排队的老消息。

![当你的服务器希望发送一个推送消息，它需要发送一个 Web 推送协议请求至推送服务](https://developers.google.com/web/fundamentals/push-notifications/images/svgs/server-to-push-service.svg)

## Step 3: 用户设备上的推送事件

一旦我们发出一个推送消息，推送服务会保留我们的消息，直至下面的任一一种情况发生：

1. 设备在线，然后推送服务发送消息。

2. 消息过期。如果消息过期，推送服务会将这条消息从它的队列当中移除，这样消息就永远不会再发送了。

当推送服务确实发送了一条消息，浏览器会接收到这条消息，解密数据，并且会在你的 **service worker** 中触发一个**推送**事件。

[service worker](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API) 是一个特殊的 JavaScript 文件。即使浏览器没有打开，浏览器依然可以执行这个 JavaScript。**service worker** 也拥有它自己的 API，比如**推送**，这些 API 在 Web 页面是不存在的（也就是说这些 API 在 service worker 脚本之外无法被使用）。

秘密就是位于 **service worker** 的推送事件当中，它能让你能执行任何后台任务。你可以执行分析调用、缓存离线页面和弹出通知等。

![当推送服务发送一条推送消息给用户的设备，设备的 service worker 将收到一个推送事件](https://developers.google.com/web/fundamentals/push-notifications/images/svgs/push-service-to-sw-event.svg)

这就是整个推送消息的流程。让我们更详细的来看一下其中的每一步吧。
