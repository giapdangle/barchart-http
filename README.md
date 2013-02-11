barchart-http
=============

A high performance HTTP server built on top of Netty 4.

Key features:

1. High performance NIO built using Netty 4's core HTTP classes
2. Asynchronous request/response framework as the primary API, not bolted on as an afterthought
3. Simple programmatic configuration and lifecycle management for easily embedding in other server applications
4. Hassle-free websocket support
5. Implements J2EE servlet API via a lightweight compatibility layer

High Performance
----------------

Barchart HTTP server is built on top of Netty 4, allowing it to take full advantage of Netty's high performance
NIO stack. The server is in its early stages and formal benchmarks are incomplete, but in its current state it
outperforms Apache 2.x for basic requests by 3x or more.

Asynchronous Request Framework
------------------------------

The request handling API supports asynchronous request processing natively, without needing to block on worker
threads before returning from onRequest(). Activating asynchronous request handling is done by calling
response.suspend():

```java
public void onRequest(ServerRequest request, ServerResponse response) throws IOException {
    String id = request.getParameter("id");
    // Asynchronous lookup method, which will call back when complete
    lookupUsername(id, this);
    response.suspend();
}
```
The connection will remain open until the request handler explicitly tells it to close the connection.
To do this, the callback calls response.finish():

```java
// Called by lookupUsername() on completion
public void completeRequest(String username) {
    response.write("Username is " + username);
    response.finish();
}
```

Configuration and Lifecycle
---------------------------

Configuration is done programmatically with a fluent API, to allow for easy adaptation with whatever
application configuration mechanism you want to implement.

```java
HttpServer httpServer = new HttpServer().configure(new HttpServerConfig()
    .address(new InetSocketAddress("0.0.0.0", 8080))
    .requestHandler("/login", new LoginHandler())
    .errorHandler(new PrettyErrorHandler())
    .logger(new RotatingFileLogger("/var/log/http", "test-site"))
    .maxConnections(1000)
);

httpServer.listen();
```

The configuration object is mutable, and many changes to it will be applied to the server at runtime
as long as it does not request a rebind. This allows you to dynamically add or remove request handler
paths while the server is running, for example, but not change the address or port the server is
listening on.

```java
// Add a new handler at runtime
httpServer.config().requestHandler("/userinfo", new UserInfoHandler());

// Remove a handler at runtime - future requests to this URI will fail
httpServer.config().removeRequestHandler("/userinfo");
```

HttpServerConfig.requestHandler() accepts either a RequestHandler or RequestHandlerFactory
instance, allowing you flexibility in controlling the handler lifecycle.

Websockets
----------

Websocket support is planned, building on top of Netty 4's websocket channel handlers. The API is still
under discussion.

Servlet API Compatibility
-------------------------

The server can function as a J2EE servlet container using a lightweight API wrapper built on top of the
native request API. This is currently partially implemented as part of the barchart-osgi-servlet project,
but requires some refactoring and additional feature development before it is usable.