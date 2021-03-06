[![Build status](https://ci.appveyor.com/api/projects/status/gp4qsv4xveqk2hb0?svg=true)](https://ci.appveyor.com/project/vtortola/websocketlistener-ccth8)
WebSocketListener 
=================

***
### Please read
This project is considered completed. Reported bugs will be fixed, but there is no plans for new features.

@deniszykov has continued its development, adding support for other platforms in his fork [deniszykov/WebSocketListener](https://github.com/deniszykov/WebSocketListener).
***

The **WebSocketListener** class provides simple methods that listen for and accept incoming WebSocket connection requests asynchronously. It is a lightweight listener with an API very similar to the `System.Net.TcpListener` class. It **does not use** the Microsoft's `System.Net.WebSockets` namespace.

 It works with .NET Standard 2.0 since Version 3.0.0. [Version 2.2.4](//github.com/vtortola/WebSocketListener/tree/Version-2.2.4) is  .NET/Mono 4.5 compatible.

**WebSocketListener** has been designed to provide WebSocket connectivity to other applications, in the same way that `System.Net.TcpListener` provides TCP connectivity. It is not a communication framework on its own and it does not provide any kind of publisher/subscriber pattern or reliable messaging beyond TCP.

 * It can work with both **Text or Binary** messages.
 * It supports `wss://`(secure). [More info](//github.com/vtortola/WebSocketListener/wiki/Enabling-WebSocket-Secure-(TLS)).
 * It supports **per-message deflate compression**. [More info](//github.com/vtortola/WebSocketListener/wiki/Deflate-extension). 
 * It can work with **multiple WebSocket standards** simultaneously. [More info](//github.com/vtortola/WebSocketListener/wiki/Multiple-WebSocket-standard-support)
 * It is **extensible**. [More info](//github.com/vtortola/WebSocketListener/wiki/WebSocketListener-Extensions).
 * It is **asynchronous**. 
 * It supports **Mono**. [More info](//github.com/vtortola/WebSocketListener/wiki/Mono-support)
 * It has the [**Ping/Pong** functionality](http://tools.ietf.org/html/rfc6455#section-5.5.2) **built-in**.
 * It can measure **connection latency**. [More info](//github.com/vtortola/WebSocketListener/wiki/Measuring-WebSockets-connection-latency)
 * It can work with **cookies and custom HTTP response statuses**. [More info](//github.com/vtortola/WebSocketListener/wiki/Hooking-into-the-HTTP-negotiation)
 * It detects and disconnects **half open connections**.
 * It allows to **send and receive messages as streams**. WebSocket messages are represented as delimited stream-like objects, that allows integration with other .NET objects like e.g. `StreamReader` and `StreamWriter`. Two different WebSocket messages, yield two different streams.
 * Messages reads and writes are streamed. Big messages are not held in memory during reads or writes.
 * It **handles partial frames transparently**. The WebSocket specification states that a single message can be sent across multiple individual frames. The message stream will allow to read all the message data, no matter if it was sent in a single or multiple frames.
 * It **handles interleaved control frames transparently**. The WebSocket specification states that control frames can appear interleaved with data frames, including between partial frames of the same message. The message stream will allow to read just the message data, it will skip the control frames.

Take a look on the [performance and load  tests](//github.com/vtortola/WebSocketListener/wiki/WebSocketListener-performance-tests) on a simple 'echo' server.

### Featured example
<img src="https://cdn-1.wp.nginx.com/wp-content/themes/nginx-theme/assets/img/logo.png" width="100">

[This echo server example](//github.com/vtortola/WebSocketListener/wiki/WebSocket---NGINX---SSL-Termination---Docker) uses [NGINX](//www.nginx.com/) to serve static files and WebSocket connections through the same port, providing [SSL termination](//en.wikipedia.org/wiki/TLS_termination_proxy) for both. It uses [Docker](//www.docker.com) and .Net Core 2.0.

### Quickstart

#### Install

[WebSocketListener is available through NuGet](https://www.nuget.org/packages/vtortola.WebSocketListener/)

```
PM> Install-Package vtortola.WebSocketListener
```

#### Set up
Setting up a server and start listening for clients is very similar to a `TcpListener`. An listening endpoint and a WebSocket standard is the minimum needed to set up a server.

```cs
var server = new WebSocketListener(new IPEndPoint(IPAddress.Any, 8006));
server.Standards.RegisterStandard(new WebSocketFactoryRfc6455());
server.StartAsync();
```

The class ```vtortola.WebSockets.Rfc6455.WebSocketFactoryRfc6455``` gives support to the [RFC 6455](http://tools.ietf.org/html/rfc6455), that is the WebSocket standard used at the moment. Future standards can be added in the [same way](//github.com/vtortola/WebSocketListener/wiki/Multiple-WebSocket-standard-support).

Optionally, you can also:
 * [enable TLS for secure WebSocket connections](//github.com/vtortola/WebSocketListener/wiki/Enabling-WebSocket-Secure-(TLS)).
 * [enable deflate compression for messages](//github.com/vtortola/WebSocketListener/wiki/Deflate-extension).
 * [customize memory use, subprotocols, queuing and ping behaviours](//github.com/vtortola/WebSocketListener/wiki/WebSocketListener-options).
 * [add customized extensions](//github.com/vtortola/WebSocketListener/wiki/WebSocketListener-Extensions).


#### Accepting clients
Once the server has started, clients can be awaited asynchronously. When a client connects, a `WebSocket` object will be returned:

```cs
var client = await server.AcceptWebSocketAsync(cancellationToken);
```

The client provides means to read and write messages. With the client, as in the underlying `NetworkStream`, is possible to write and read at the same time even from different threads, but is not possible to read from two or more threads at the same time, same for writing.

`AcceptWebSocketAsync` should be in a loop to continuously accept new clients, also wrapped in a `try/catch` since errors in the negotiation process will be thrown here. Take a look to the [simple host tutorial](https://github.com/vtortola/WebSocketListener/wiki/WebSocketListener-Echo-Server-Example).

#### Receiving messages
With the client we can *await* a message as a readonly stream:

```cs
var messageReadStream = await client.ReadMessageAsync(cancellationToken);
```

Messages are a stream-like objects, so is it possible to use regular .NET framework tools to work with them. The `WebSocketMessageReadStream.MessageType` property indicates the kind of content the message contains, so it can be used to select a different handling approach.

The returned `WebSocketMessageReadStream` object will contain information from the header, like type of message (Text or Binary) but not the message content, neither the message length, since a frame only contains the frame length rather than the total message length, therefore that information could be missleading.

A text message can be read with a simple `StreamReader`.  It is worth remember that according to the WebSockets specs, it always uses UTF8 for text enconding:

```cs
if(messageReadStream.MessageType == WebSocketMessageType.Text)
{
   var msgContent = string.Empty;
   using (var sr = new StreamReader(messageReadStream, Encoding.UTF8))
        msgContent = await sr.ReadToEndAsync();
}
```

```ReadMessageAsync``` should go in a loop, to read messages continuously. Writes and read can be performed at the same time. Take a look to the [simple host tutorial](//github.com/vtortola/WebSocketListener/wiki/WebSocketListener-Echo-Server-Example).

Also, a binary message can be read using regular .NET techniques:

```cs
if(messageReadStream.MessageType == WebSocketMessageType.Binary)
{
   using (var ms = new MemoryStream())
   {
       await messageReadStream.CopyToAsync(ms);
   }
}
```

#### Sending messages
Writing messages is also easy. The `WebSocketMessageReadStream.CreateMessageWriter` method allows to create a write only  message:

```cs
using (var messageWriterStream = client.CreateMessageWriter(WebSocketMessageType.Text))
```

Once a message writer is created, regular .NET tools can be used to write in it:

```cs
using (var sw = new StreamWriter(messageWriterStream, Encoding.UTF8))
{
   await sw.WriteAsync("Hello World!");
}
```    

Also binary messages:

```cs
using (var messageWriter = ws.CreateMessageWriter(WebSocketMessageType.Binary))
   await myFileStream.CopyToAsync(messageWriter);
```

#### Example
Take a look on the [WebSocketListener samples](//github.com/vtortola/WebSocketListener/wiki/WebSocketListener-Samples).

### The MIT License (MIT)

Copyright (c) 2014 vtortola

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
