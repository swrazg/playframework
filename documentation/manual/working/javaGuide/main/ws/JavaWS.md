<!--- Copyright (C) 2009-2015 Typesafe Inc. <http://www.typesafe.com> -->
# The Play WS API

Sometimes we would like to call other HTTP services from within a Play application. Play supports this via its [WS library](api/java/play/libs/ws/package-summary.html), which provides a way to make asynchronous HTTP calls.

There are two important parts to using the WS API: making a request, and processing the response. We'll discuss how to make both GET and POST HTTP requests first, and then show how to process the response from the WS. Finally, we'll discuss some common use cases.

## Making a Request

To use WS, first add `javaWs` to your `build.sbt` file:

```scala
libraryDependencies ++= Seq(
  javaWs
)
```

Now any controller or component that wants to use WS will have to add the following imports and then declare a dependency on the `WSClient`:

@[ws-controller](code/javaguide/ws/Application.java)

To build an HTTP request, you start with `ws.url()` to specify the URL.

@[ws-holder](code/javaguide/ws/JavaWS.java)

This returns a [`WSRequest`](api/java/play/libs/ws/WSRequest.html) that you can use to specify various HTTP options, such as setting headers. You can chain calls together to construct complex requests.

@[ws-complex-holder](code/javaguide/ws/JavaWS.java)

You end by calling a method corresponding to the HTTP method you want to use.  This ends the chain, and uses all the options defined on the built request in the [`WSRequest`](api/java/play/libs/ws/WSRequest.html).

@[ws-get](code/javaguide/ws/JavaWS.java)

This returns a [`Promise<WSResponse>`](api/java/play/libs/F.Promise.html) where the [`WSResponse`](api/java/play/libs/ws/WSResponse.html) contains the data returned from the server.

### Request with authentication

If you need to use HTTP authentication, you can specify it in the builder, using a username, password, and an [`WSAuthScheme`](api/java/play/libs/ws/WSAuthScheme.html).  Options for the `WSAuthScheme` are `BASIC`, `DIGEST`, `KERBEROS`, `NONE`, `NTLM`, and `SPNEGO`.

@[ws-auth](code/javaguide/ws/JavaWS.java)

### Request with follow redirects

If an HTTP call results in a 302 or a 301 redirect, you can automatically follow the redirect without having to make another call.

@[ws-follow-redirects](code/javaguide/ws/JavaWS.java)

### Request with query parameters

You can specify query parameters for a request.

@[ws-query-parameter](code/javaguide/ws/JavaWS.java)

### Request with additional headers

@[ws-header](code/javaguide/ws/JavaWS.java)

For example, if you are sending plain text in a particular format, you may want to define the content type explicitly.

@[ws-header-content-type](code/javaguide/ws/JavaWS.java)

### Request with timeout

If you wish to specify a request timeout, you can use `setRequestTimeout` to set a value in milliseconds. A value of `-1` can be used to set an infinite timeout. 

@[ws-timeout](code/javaguide/ws/JavaWS.java)

### Submitting form data

To post url-form-encoded data you can set the proper header and formatted data.

@[ws-post-form-data](code/javaguide/ws/JavaWS.java)

### Submitting JSON data

The easiest way to post JSON data is to use the [[JSON library|JavaJsonActions]].

@[json-imports](code/javaguide/ws/JavaWS.java)

@[ws-post-json](code/javaguide/ws/JavaWS.java)

### Streaming data

It's also possible to stream data.

Here is an example showing how you could stream a large image to a different endpoint for further processing:

@[ws-stream-request](code/javaguide/ws/JavaWS.java)

The `largeImage` in the code snippet above is an Akka Streams `Source<ByteString, ?>`.

## Processing the Response

Working with the [`WSResponse`](api/java/play/libs/ws/WSResponse.html) is done by mapping inside the `Promise`.

### Processing a response as JSON

You can process the response as a `JsonNode` by calling `response.asJson()`.

@[ws-response-json](code/javaguide/ws/JavaWS.java)

### Processing a response as XML

Similarly, you can process the response as XML by calling `response.asXml()`.

@[ws-response-xml](code/javaguide/ws/JavaWS.java)

### Processing large responses

Calling `get()`, `post()` or `execute()` will cause the body of the response to be loaded into memory before the response is made available.  When you are downloading a large, multi-gigabyte file, this may result in unwelcomed garbage collection or even out of memory errors.

`WS` lets you consume the response's body incrementally by using an Akka Streams `Sink`.  The `stream()` method on `WSRequest` returns a `CompletionStage<StreamedResponse>`. A `StreamedResponse` is a simple container holding together the response's headers and body.

Any controller or component that wants to levearge the WS streaming functionality will have to add the following imports and dependencies:

@[ws-streams-controller](code/javaguide/ws/MyController.java)

Here is a trivial example that uses a folding `Sink` to count the number of bytes returned by the response:

@[stream-count-bytes](code/javaguide/ws/JavaWS.java)

Alternatively, you could also stream the body out to another location. For example, a file:

@[stream-to-file](code/javaguide/ws/JavaWS.java)

Another common destination for response bodies is to stream them back from a controller's `Action`:

@[stream-to-result](code/javaguide/ws/JavaWS.java)

As you may have noticed, before calling `stream()` we need to set the HTTP method to use by calling `setMethod` on the request. Here follows another example that uses `PUT` instead of `GET`:

@[stream-put](code/javaguide/ws/JavaWS.java)

Of course, you can use any other valid HTTP verb.

## Common Patterns and Use Cases

### Chaining WS calls

You can chain WS calls by using `flatMap`.

@[ws-composition](code/javaguide/ws/JavaWS.java)

### Exception recovery
If you want to recover from an exception in the call, you can use `recover` or `recoverWith` to substitute a response.

@[ws-recover](code/javaguide/ws/JavaWS.java)

### Using in a controller

You can map a `Promise<WSResponse>` to a `Promise<Result>` that can be handled directly by the Play server, using the asynchronous action pattern defined in [[Handling Asynchronous Results|JavaAsync]].

@[ws-action](code/javaguide/ws/JavaWS.java)

## Using WSClient

WSClient is a wrapper around the underlying [AsyncHttpClient](https://github.com/AsyncHttpClient/async-http-client). It is useful for defining multiple clients with different profiles, or using a mock.

The default client can be called from the WSClient class:

@[ws-client](code/javaguide/ws/JavaWS.java)

You can instantiate a WSClient directly from code and use this for making requests.  Note that you must follow a particular series of steps to use HTTPS correctly if you are defining a client directly:

@[ws-custom-client-imports](code/javaguide/ws/JavaWS.java)

@[ws-custom-client](code/javaguide/ws/JavaWS.java)

> NOTE: if you instantiate a NingWSClient object, it does not use the WS plugin system, and so will not be automatically closed in `Application.onStop`. Instead, the client must be manually shutdown using `client.close()` when processing has completed.  This will release the underlying ThreadPoolExecutor used by AsyncHttpClient.  Failure to close the client may result in out of memory exceptions (especially if you are reloading an application frequently in development mode).

You can also get access to the underlying `AsyncHttpClient`.

@[ws-underlying-client](code/javaguide/ws/JavaWS.java)

This is important in a couple of cases.  WS has a couple of limitations that require access to the client:

* `WS` does not support multi part form upload directly.  You can use the underlying client with [RequestBuilder.addBodyPart](https://asynchttpclient.github.io/async-http-client/apidocs/com/ning/http/client/RequestBuilder.html).
* `WS` does not support streaming body upload.  In this case, you should use the `FeedableBodyGenerator` provided by AsyncHttpClient.

## Configuring WS

Use the following properties in `application.conf` to configure the WS client:

* `play.ws.followRedirects`: Configures the client to follow 301 and 302 redirects *(default is **true**)*.
* `play.ws.useProxyProperties`: To use the system http proxy settings(http.proxyHost, http.proxyPort) *(default is **true**)*.
* `play.ws.useragent`: To configure the User-Agent header field.
* `play.ws.compressionEnabled`: Set it to true to use gzip/deflater encoding *(default is **false**)*.

### Timeouts

There are 3 different timeouts in WS. Reaching a timeout causes the WS request to interrupt.

* `play.ws.timeout.connection`: The maximum time to wait when connecting to the remote host *(default is **120 seconds**)*.
* `play.ws.timeout.idle`: The maximum time the request can stay idle (connection is established but waiting for more data) *(default is **120 seconds**)*.
* `play.ws.timeout.request`: The total time you accept a request to take (it will be interrupted even if the remote host is still sending data) *(default is **120 seconds**)*.

The request timeout can be overridden for a specific connection with `setTimeout()` (see "Making a Request" section).

## Configuring WS with SSL

To configure WS for use with HTTP over SSL/TLS (HTTPS), please see [[Configuring WS SSL|WsSSL]].

### Configuring AsyncClientConfig

The following advanced settings can be configured on the underlying AsyncHttpClientConfig.
Please refer to the [AsyncHttpClientConfig Documentation](https://asynchttpclient.github.io/async-http-client/apidocs/com/ning/http/client/AsyncHttpClientConfig.Builder.html) for more information.

* `play.ws.ning.allowPoolingConnection`
* `play.ws.ning.allowSslConnectionPool`
* `play.ws.ning.ioThreadMultiplier`
* `play.ws.ning.maxConnectionsPerHost`
* `play.ws.ning.maxConnectionsTotal`
* `play.ws.ning.maxConnectionLifeTime`
* `play.ws.ning.idleConnectionInPoolTimeout`
* `ws.ning.webSocketIdleTimeout`
* `play.ws.ning.maxNumberOfRedirects`
* `play.ws.ning.maxRequestRetry`
* `play.ws.ning.removeQueryParamsOnRedirect`
* `play.ws.ning.useRawUrl`
