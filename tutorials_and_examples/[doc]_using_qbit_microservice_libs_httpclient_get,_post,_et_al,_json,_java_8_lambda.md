# [Doc] Using QBit microservice lib's HttpClient GET, POST, et al, JSON, Java 8 Lambda

QBit ships with an HttpClient lib. The libs tries to make it easy to make REST style calls.
The HttpClient lib is geared towards HTTP and JSON.

The nice thing about the lib is that is allows for Java 8 style lambdas for GET, POST and WebSocket calls.


Before we start up our HTTP client let's start up a server to echo back what we send the server.

##### Starting up an HTTP server

```java

        /* Create an HTTP server. */
        HttpServer httpServer = httpServerBuilder()
                .setPort(8080).build();

        /* Setting up a request Consumer with Java 8 Lambda expression. */
        httpServer.setHttpRequestConsumer(httpRequest -> {

            Map<String, Object> results = new HashMap<>();
            results.put("method", httpRequest.getMethod());
            results.put("uri", httpRequest.getUri());
            results.put("body", httpRequest.getBodyAsString());
            results.put("headers", httpRequest.getHeaders());
            results.put("params", httpRequest.getParams());
            httpRequest.getReceiver()
                .response(200, "application/json", Boon.toJson(results));
        });


        /* Start the server. */
        httpServer.start();


```

## Starting HTTP client

Now, let's try out our HTTP client.

##### Starting up an HTTP client

```java

        /* Setup an httpClient. */
        HttpClient httpClient = httpClientBuilder()
                  .setHost("localhost").setPort(8080).build();
        httpClient.start();
```

You just pass the URL, the port and then call start.

## Synchronous HTTP calls

Now you can start sending HTTP requests.

##### No Param HTTP GET

```java
        /* Send no param get. */
        HttpResponse httpResponse = httpClient.get( "/hello/mom" );
        puts( httpResponse );
```

An HTTP response just contains the results from the server.


##### No Param HTTP Response

```java
public interface HttpResponse {

    MultiMap<String, String> headers();

    int code();

    String contentType();

    String body();

}

```

There are helper methods for sync HTTP GET calls.


##### Helper methods for GET
```java


        /* Send one param get. */
        httpResponse = httpClient.getWith1Param("/hello/singleParam",
                                        "hi", "mom");
        puts("single param", httpResponse );


        /* Send two param get. */
        httpResponse = httpClient.getWith2Params("/hello/twoParams",
                "hi", "mom", "hello", "dad");
        puts("two params", httpResponse );


        /* Send two param get. */
        httpResponse = httpClient.getWith3Params("/hello/3params",
                "hi", "mom",
                "hello", "dad",
                "greetings", "kids");
        puts("three params", httpResponse );


        /* Send four param get. */
        httpResponse = httpClient.getWith4Params("/hello/4params",
                "hi", "mom",
                "hello", "dad",
                "greetings", "kids",
                "yo", "pets");
        puts("4 params", httpResponse );

        /* Send five param get. */
        httpResponse = httpClient.getWith5Params("/hello/5params",
                "hi", "mom",
                "hello", "dad",
                "greetings", "kids",
                "yo", "pets",
                "hola", "neighbors");
        puts("5 params", httpResponse );


```

The puts method is a helper method it does System.out.println more or less by the way.

The first five params are covered. Beyond five, you have to use the HttpBuilder.

```java


        /* Send six params with get. */

        final HttpRequest httpRequest = httpRequestBuilder()
                .addParam("hi", "mom")
                .addParam("hello", "dad")
                .addParam("greetings", "kids")
                .addParam("yo", "pets")
                .addParam("hola", "pets")
                .addParam("salutations", "all").build();

        httpResponse = httpClient.sendRequestAndWait(httpRequest);
        puts("6 params", httpResponse );
```

## Http Async

There are async calls for GET as well.

#### Async calls for HTTP GET using Java 8 lambda

```java

        /* Using Async support with lambda. */
        httpClient.getAsync("/hi/async", (code, contentType, body) -> {
            puts("Async text with lambda", body);
        });

        Sys.sleep(100);


        /* Using Async support with lambda. */
        httpClient.getAsyncWith1Param("/hi/async", "hi", "mom", (code, contentType, body) -> {
            puts("Async text with lambda 1 param\n", body);
        });

        Sys.sleep(100);



        /* Using Async support with lambda. */
        httpClient.getAsyncWith2Params("/hi/async",
                "p1", "v1",
                "p2", "v2",
                (code, contentType, body) -> {
                    puts("Async text with lambda 2 params\n", body);
                });

        Sys.sleep(100);




        /* Using Async support with lambda. */
        httpClient.getAsyncWith3Params("/hi/async",
                "p1", "v1",
                "p2", "v2",
                "p3", "v3",
                (code, contentType, body) -> {
                    puts("Async text with lambda 3 params\n", body);
                });

        Sys.sleep(100);


        /* Using Async support with lambda. */
        httpClient.getAsyncWith4Params("/hi/async",
                "p1", "v1",
                "p2", "v2",
                "p3", "v3",
                "p4", "v4",
                (code, contentType, body) -> {
                    puts("Async text with lambda 4 params\n", body);
                });

        Sys.sleep(100);


        /* Using Async support with lambda. */
        httpClient.getAsyncWith5Params("/hi/async",
                "p1", "v1",
                "p2", "v2",
                "p3", "v3",
                "p4", "v4",
                "p5", "v5",
                (code, contentType, body) -> {
                    puts("Async text with lambda 5 params\n", body);
                });

        Sys.sleep(100);

```

## HTTP Post synchronous

There are POST HTTP sync helper methods as well.

#### POST sync
```java

        /* Send no param post. */
        HttpResponse httpResponse = httpClient.post( "/hello/mom" );
        puts( httpResponse );


        /* Send one param post. */
        httpResponse = httpClient.postWith1Param("/hello/singleParam",
                                  "hi", "mom");
        puts("single param", httpResponse );


        /* Send two param post. */
        httpResponse = httpClient.postWith2Params("/hello/twoParams",
                "hi", "mom", "hello", "dad");
        puts("two params", httpResponse );


        /* Send two param post. */
        httpResponse = httpClient.postWith3Params("/hello/3params",
                "hi", "mom",
                "hello", "dad",
                "greetings", "kids");
        puts("three params", httpResponse );


        /* Send four param post. */
        httpResponse = httpClient.postWith4Params("/hello/4params",
                "hi", "mom",
                "hello", "dad",
                "greetings", "kids",
                "yo", "pets");
        puts("4 params", httpResponse );

        /* Send five param post. */
        httpResponse = httpClient.postWith5Params("/hello/5params",
                "hi", "mom",
                "hello", "dad",
                "greetings", "kids",
                "yo", "pets",
                "hola", "neighbors");
        puts("5 params", httpResponse );


        /* Send six params with post. */

        final HttpRequest httpRequest = httpRequestBuilder()
                .setUri("/sixPost")
                .setMethod("POST")
                .addParam("hi", "mom")
                .addParam("hello", "dad")
                .addParam("greetings", "kids")
                .addParam("yo", "pets")
                .addParam("hola", "pets")
                .addParam("salutations", "all").build();


        httpResponse = httpClient.sendRequestAndWait(httpRequest);
        puts("6 params", httpResponse );

```

## HTTP POST Async

#### HTTP POST Async with Java 8 Lambda
```java

        /* Using Async support with lambda. */
        httpClient.postAsync("/hi/async",
                     (code, contentType, body) -> {
            puts("Async text with lambda", body);
        });

        Sys.sleep(100);


        /* Using Async support with lambda. */
        httpClient.postFormAsyncWith1Param("/hi/async", "hi", "mom",
            (code, contentType, body) -> {
            puts("Async text with lambda 1 param\n", body);
        });

        Sys.sleep(100);



        /* Using Async support with lambda. */
        httpClient.postFormAsyncWith2Params("/hi/async",
                "p1", "v1",
                "p2", "v2",
                (code, contentType, body) -> {
                    puts("Async text with lambda 2 params\n", body);
                });

        Sys.sleep(100);




        /* Using Async support with lambda. */
        httpClient.postFormAsyncWith3Params("/hi/async",
                "p1", "v1",
                "p2", "v2",
                "p3", "v3",
                (code, contentType, body) -> {
                    puts("Async text with lambda 3 params\n", body);
                });

        Sys.sleep(100);


        /* Using Async support with lambda. */
        httpClient.postFormAsyncWith4Params("/hi/async",
                "p1", "v1",
                "p2", "v2",
                "p3", "v3",
                "p4", "v4",
                (code, contentType, body) -> {
                    puts("Async text with lambda 4 params\n", body);
                });

        Sys.sleep(100);


        /* Using Async support with lambda. */
        httpClient.postFormAsyncWith5Params("/hi/async",
                "p1", "v1",
                "p2", "v2",
                "p3", "v3",
                "p4", "v4",
                "p5", "v5",
                (code, contentType, body) -> {
                    puts("Async text with lambda 5 params\n", body);
                });

        Sys.sleep(100);

```

Remember if give params is not enough, then you can use the HttpBuilder.


#### Use the builder if you need more params or something special
```java

        httpRequest = httpRequestBuilder().addParam("hi", "mom")
                .addParam("hello", "dad")
                .addParam("greetings", "kids")
                .addParam("yo", "pets")
                .addParam("hola", "pets")
                .addParam("salutations", "all")
                .setTextReceiver((code, contentType, body)
                                      -> puts(code, contentType, body))
                .build();

        httpClient.sendHttpRequest(httpRequest);
        puts("6 async params", httpResponse );

```

## HTTP PUT Sync

#### PUT Sync
```java

        /* Send no param post. */
        HttpResponse httpResponse = httpClient.put( "/hello/mom" );
        puts( httpResponse );


        /* Send one param post. */
        httpResponse = httpClient.putWith1Param("/hello/singleParam",
                                     "hi", "mom");
        puts("single param", httpResponse );


        /* Send two param post. */
        httpResponse = httpClient.putWith2Params("/hello/twoParams",
                "hi", "mom", "hello", "dad");
        puts("two params", httpResponse );


        /* Send two param post. */
        httpResponse = httpClient.putWith3Params("/hello/3params",
                "hi", "mom",
                "hello", "dad",
                "greetings", "kids");
        puts("three params", httpResponse );


        /* Send four param post. */
        httpResponse = httpClient.putWith4Params("/hello/4params",
                "hi", "mom",
                "hello", "dad",
                "greetings", "kids",
                "yo", "pets");
        puts("4 params", httpResponse );

        /* Send five param post. */
        httpResponse = httpClient.putWith5Params("/hello/5params",
                "hi", "mom",
                "hello", "dad",
                "greetings", "kids",
                "yo", "pets",
                "hola", "neighbors");
        puts("5 params", httpResponse );


        /* Send six params with post. */

        final HttpRequest httpRequest = httpRequestBuilder()
                .setUri("/sixPost")
                .setMethod("PUT")
                .addParam("hi", "mom")
                .addParam("hello", "dad")
                .addParam("greetings", "kids")
                .addParam("yo", "pets")
                .addParam("hola", "pets")
                .addParam("salutations", "all").build();


        httpResponse = httpClient.sendRequestAndWait(httpRequest);
        puts("6 params", httpResponse );

```

## HTTP PUT Async


#### PUT Async with Java 8 Lambda
```java



        /* Using Async support with lambda. */
        httpClient.putAsync("/hi/async", (code, contentType, body) -> {
            puts("Async text with lambda", body);
        });

        Sys.sleep(100);


        /* Using Async support with lambda. */
        httpClient.putFormAsyncWith1Param("/hi/async", "hi", "mom",
            (code, contentType, body) -> {
            puts("Async text with lambda 1 param\n", body);
        });

        Sys.sleep(100);



        /* Using Async support with lambda. */
        httpClient.putFormAsyncWith2Params("/hi/async",
                "p1", "v1",
                "p2", "v2",
                (code, contentType, body) -> {
                    puts("Async text with lambda 2 params\n", body);
                });

        Sys.sleep(100);




        /* Using Async support with lambda. */
        httpClient.putFormAsyncWith3Params("/hi/async",
                "p1", "v1",
                "p2", "v2",
                "p3", "v3",
                (code, contentType, body) -> {
                    puts("Async text with lambda 3 params\n", body);
                });

        Sys.sleep(100);


        /* Using Async support with lambda. */
        httpClient.putFormAsyncWith4Params("/hi/async",
                "p1", "v1",
                "p2", "v2",
                "p3", "v3",
                "p4", "v4",
                (code, contentType, body) -> {
                    puts("Async text with lambda 4 params\n", body);
                });

        Sys.sleep(100);


        /* Using Async support with lambda. */
        httpClient.putFormAsyncWith5Params("/hi/async",
                "p1", "v1",
                "p2", "v2",
                "p3", "v3",
                "p4", "v4",
                "p5", "v5",
                (code, contentType, body) -> {
                    puts("Async text with lambda 5 params\n", body);
                });

        Sys.sleep(100);

```

## JSON POST


#### Async POST JSON no response
```java
        httpClient.sendJsonPost("/foo/json/",
               toJson(new Employee("Rick", "Smith")));


```

#### Async POST JSON with Async response Java 8 Lambda
```java
        httpClient.sendJsonPostAsync("/foo/json/",
                toJson(new Employee("Rick", "Smith")),
                (code, contentType, body) ->
                    puts ("ASYNC POST", code, contentType, fromJson(body)));
```
#### Async POST JSON with Async response using anonymous inner class
```java
        httpClient.sendJsonPostAsync("/foo/json/", toJson(new Employee("Rick", "Smith")),
                new HttpTextReceiver() {
                    @Override
                    public void response(int code, String contentType, String body) {
                        puts(code, contentType, body);
                    }
                });
```


#### Sync POST JSON with Sync response
```java
        HttpResponse httpResponse = httpClient.postJson("/foo/json/sync",
                toJson(new Employee("Rick", "Smith")));

        puts("POST JSON RESPONSE", httpResponse);
```

## HTTP Put JSON

#### PUT JSON Support like POST but PUT
```java
        httpClient.sendJsonPut("/foo/json/",
                      toJson(new Employee("Rick", "Smith")));


        httpClient.sendJsonPutAsync("/foo/json/",
                toJson(new Employee("Rick", "Smith")),
                (code, contentType, body)
                  -> puts("ASYNC PUT", code, contentType, fromJson(body)));


        httpResponse = httpClient.putJson("/foo/json/sync",
                               toJson(new Employee("Rick", "Smith")));

        puts("PUT JSON RESPONSE", httpResponse);

```
