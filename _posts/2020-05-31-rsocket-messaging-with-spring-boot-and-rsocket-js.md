---
layout: post
title: RSocket messaging with Spring Boot and rsocket-js
tags: [spring, rsocket, reactive-programming]
---

After a quick introduction to RSocket, in this post we build an RSocket server which uses all the four interaction models of RSocket to communicate with a React frontend backed by [rsocket-js](https://github.com/rsocket/rsocket-js).

![RSocket client implemented with rsocket-js](/img/posts/rsocket-client-in-js.png "RSocket client implemented with rsocket-js"){: .center-block :}

## What is RSocket?

RSocket is a binary application protocol providing support for reactive streams and flow control. With flow control a client is able to request as many messages as it can process. RSocket can be used over any byte stream transports such as Websockets or TCP. RSocket provides the following interaction models:

+ request-response (stream of one message)
+ fire-and-forget (no response)
+ request-stream (finite/infinite stream of many messages)
+ channel (bi-directional streams)

## Create RSocket server

Creating an RSocket server in Spring Boot is simple and straightforward, you just need to add the `spring-boot-starter-rsocket` artifact to your `pom.xml` and set the RSocket port and transport in the `application.yaml`.

```xml
<dependencies>
    ...
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-rsocket</artifactId>
    </dependency>
    ...
</dependencies>
```

The default transport in RSocket is TCP which is ideal for communication between services, but in this tutorial the client is a browser, so we need to choose Websocket transport.

```yaml
spring:
  rsocket:
    server:
      transport: websocket
      port: 7000
```

## Connect to and disconnect from an RSocket server

We have to use the `RSocketClient` object provided by rsocket-js to connect to the RSocket server. We can create our own data and metadata serializers, but we use the basic `JsonSerializer` and `IdentitySerializer`.

When the client requested `n` messages from the server, but the server doesn't have that many messages to send, the server will respond with a keepalive message in every 10 seconds to make sure the connection is open. If the client doesn't get any (neither the requested nor the keepalive) messages for 20 seconds, the connection will be closed. You can control the frequency of the keepalive messages and the period after the connection will be timed out with the `keepAlive` and `lifetime` properties.

```javascript
import {IdentitySerializer, JsonSerializer, RSocketClient} from "rsocket-core";
import RSocketWebSocketClient from "rsocket-websocket-client";

this.client = new RSocketClient({
    serializers: {
        data: JsonSerializer,
        metadata: IdentitySerializer
    },
    setup: {
        keepAlive: 10000,
        lifetime: 20000,
        dataMimeType: 'application/json',
        metadataMimeType: 'message/x.rsocket.routing.v0',
    },
    transport: new RSocketWebSocketClient({url: address})
});
```

By calling the `connect` method of the `client` object we created recently we establish a connection to the server and subscribe to the `onComplete` event in which we create a class variable called `socket` of type `RSocketClientSocket`. We will use the socket variable later to send requests to the server. Arbitrarily, we can subscribe to the connection status of the socket to inform the users if the connection was lost or to the `onError` and `onSubscribe` event of the connection.

```javascript
this.client.connect().subscribe({
    onComplete: clientSocket => {
        this.socket = clientSocket;
        this.socket.connectionStatus().subscribe(status => {
            console.log(status);
        });
    },
    onError: error => {
        // ...
    },
    onSubscribe: cancel => {
        // ...
    }
});
```

The connection can be closed by calling the `close` method of the `client` object.

```javascript
this.client.close();
```

## Request-response

The request-response interaction model is identical to the interaction model of HTTP. The client sends a request to the server and waits until the response arrives. We send the data in the `data` property. Its value will be serialized to a JSON object by serializer which we set before. The route of the message, the security tokens or other information can be sent as metadata. Now the route of the data is `request-response`, so on the server side we have to create a message mapping on this route.

```javascript
this.socket.requestResponse({
    data: new Message('client', 'request'),
    metadata: String.fromCharCode('request-response'.length) + 'request-response'
}).subscribe({
    onComplete: msg => {
        console.log('Message received: ', msg);
    },
    onError: error => {
        // ...
    }
});
```

We have to add the following `requestResponse` method to a Java class annotated as a `@Controller`. The `requestResponse` method is invoked when a message received from the client and it returns a new message object as a response.

```java
@MessageMapping("request-response")
public Message requestResponse(final Message request) {
    log.info("Received request-response request: {}", request);
    return new Message("server", "response");
}
```

## Fire-and-forget

The fire-and-forget interaction model is similar to the request-response model, in this case the server doesn't respond to the client.

```javascript
this.socket.fireAndForget({
    data: new Message('client', 'fire'),
    metadata: String.fromCharCode('fire-and-forget'.length) + 'fire-and-forget'
});
```

As can be seen below in the mapping, the server doesn't return any value.

```java
@MessageMapping("fire-and-forget")
public void fireAndForget(final Message request) {
    log.info("Received fire-and-forget request: {}", request);
}
```

## Request-stream

With the request-stream interaction model, as its name suggests, the client can request a stream of messages from the server. When the client subscribed to the stream, the `onSubscribe` event is fired and the client have to request messages by invoking the `request(n)` method, where `n` is the number of requested messages. After the client received the requested `n` messages the stream is terminated and the `onComplete` event is fired.

In this example we create an infinite stream and to do so, in the callback of the `onNext` event we request new messages every time the client received the previously requested `n` messages. The `onNext` event is fired when the client received a new message.

```javascript
let requestedMsg = 10;
let processedMsg = 0;

this.socket.requestStream({
    data: new Message('client', 'request'),
    metadata: String.fromCharCode('stream'.length) + 'stream'
}).subscribe({
    onSubscribe: sub => {
        this.requestStreamSubscription = sub;
        this.requestStreamSubscription.request(requestedMsg);
    },
    onError: error => {
        // ...
    },
    onNext: msg => {
        processedMsg++;

        if (processedMsg >= requestedMsg) {
            this.requestStreamSubscription.request(requestedMsg);
            processedMsg = 0;
        }
    },
    onComplete: msg => {
        // ...
    },
});
```

The stream can be terminated by calling the `cancel` method of the subscription.

```javascript
this.requestStreamSubscription.cancel();
```

The server responds to the stream request with a `Flux` of messages. One message is generated in every second.

```java
@MessageMapping("stream")
public Flux<Message> stream(final Message request) {
    log.info("Received stream request: {}", request);
    return Flux
            .interval(Duration.ofSeconds(1))
            .map(index -> new Message("server", "stream", index))
            .log();
}
```

## Channel

With channel it is possible to stream messages in both directions. In order to stream messages from the client to the server, on the client side we have to instantiate a `Flowable` object. We have to implement the `request` method which is invoked when the server requests new messages. In this tutorial we create an infinite stream and we send a new message in every second. The `Flowable` stream can be terminated by the `cancel` method.

```javascript
let cancelled = false;
let index = 0;

let flow = new Flowable(subscriber => {
    this.requestChannelClientSubscription = subscriber;
    this.requestChannelClientSubscription.onSubscribe({
        cancel: () => {
            cancelled = true;
        },
        request: n => {
            let intervalID = setInterval(() => {
                if (n > 0 && !cancelled) {
                    const msg = new Message('client', 'stream', index++);
                    subscriber.onNext(msg);
                    n--;
                } else {
                    window.clearInterval(intervalID);
                }
            }, 1000);
        }
    });
});
```

The client initiates the channel request by invoking the `requestChannel` method and putting the recently created `Flowable` object in the argument of the method. The return value of the `requestChannel` method is an other `Flowable` object and the client have to subscribe on it in the same way as in case of the request-stream interaction model. When the `onSubscribe` event is fired we have to request messages from the server, as well as we have to take care with the new message requests when the `onNext` event is fired.

```javascript
let requestedMsg = 10;
let processedMsg = 0;

this.socket.requestChannel(flow.map(msg => {
    return {
        data: msg,
        metadata: String.fromCharCode('channel'.length) + 'channel'
    };
})).subscribe({
    onSubscribe: sub => {
        this.requestChannelServerSubscription = sub;
        this.requestChannelServerSubscription.request(requestedMsg);
    },
    onError: error => {
        // ...
    },
    onNext: msg => {
        processedMsg++;

        if (processedMsg >= requestedMsg) {
            this.requestChannelServerSubscription.request(requestedMsg);
            processedMsg = 0;
        }
    },
    onComplete: msg => {
        // ...
    },
});
```

The inbound and outbound streams created above can be terminated by calling the `cancel` method of their subscriptions.

```javascript
this.requestChannelClientSubscription._subscription.cancel();
this.requestChannelServerSubscription.cancel();
```

The server have to subscribe to the incoming `Flux` object, because it will not receive any message without that. The return value of the `@MessageMapping` method is a `Flux` object which generates one message in every second.

```java
@MessageMapping("channel")
public Flux<Message> channel(final Flux<Message> settings) {
    settings.subscribe(message -> log.info(message.toString()));

    return Flux.interval(Duration.ofSeconds(1))
            .doOnCancel(() -> log.warn("The client cancelled the channel."))
            .map(index -> new Message("server", "stream", index));
}
```

## Conclusion

In this tutorial we learned

+ how to create an RSocket server with Spring Boot
+ how to create a client in rsocket-js
+ the four interaction models of RSocket
+ how to create infinite message streams.

The source code with a React UI can be found on [GitHub](https://github.com/sstremler/rsocket-demo).
