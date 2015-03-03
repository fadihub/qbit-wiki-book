# [Doc] Using QBit microservice lib's WebSocket support


QBit microservice lib comes with a WebSocket lib that is geared towards JSON.

It is very easy to use.

#### Create an HTTP server
```java

        /* Create an HTTP server. */
        HttpServer httpServer = httpServerBuilder()
                .setPort(8080).build();

```

#### Setup server WebSocket support
```java
        /* Setup WebSocket Server support. */
        httpServer.setWebSocketOnOpenConsumer(webSocket -> {
            webSocket.setTextMessageConsumer(message -> {
                webSocket.sendText("ECHO " + message);
            });
        });

```

#### Start the server
```java

        /* Start the server. */
        httpServer.start();
```

#### Setup the WebSocket client
```java

        /** CLIENT. */

        /* Setup an httpClient. */
        HttpClient httpClient = httpClientBuilder()
                .setHost("localhost").setPort(8080).build();
        httpClient.start();
```

#### Client WebSocket

```java

        /* Setup the client websocket. */
        WebSocket webSocket = httpClient
                .createWebSocket("/websocket/rocket");

        /* Setup the text consumer. */
        webSocket.setTextMessageConsumer(message -> {
            System.out.println(message);
        });
        webSocket.openAndWait();

        /* Send some messages. */
        webSocket.sendText("Hi mom");
        webSocket.sendText("Hello World!");

```

#### Output
```output

ECHO Hi mom
ECHO Hello World!

```

Now stop the server and client. Pretty easy eh?

```java

        Sys.sleep(1000);
        webSocket.close();
        httpClient.stop();
        httpServer.stop();
```


#### Full example using WebSocket client and server

```java


import io.advantageous.qbit.http.client.HttpClient;
import io.advantageous.qbit.http.server.HttpServer;
import io.advantageous.qbit.http.websocket.WebSocket;
import org.boon.core.Sys;

import static io.advantageous.qbit.http.client.HttpClientBuilder.httpClientBuilder;
import static io.advantageous.qbit.http.server.HttpServerBuilder.httpServerBuilder;

public class EchoWebSocket {


    public static void main(String... args) {


        /* Create an HTTP server. */
        HttpServer httpServer = httpServerBuilder()
                .setPort(8080).build();

        /* Setup WebSocket Server support. */
        httpServer.setWebSocketOnOpenConsumer(webSocket -> {
            webSocket.setTextMessageConsumer(message -> {
                webSocket.sendText("ECHO " + message);
            });
        });

        /* Start the server. */
        httpServer.start();

        /** CLIENT. */

        /* Setup an httpClient. */
        HttpClient httpClient = httpClientBuilder()
                .setHost("localhost").setPort(8080).build();
        httpClient.start();

        /* Setup the client websocket. */
        WebSocket webSocket = httpClient
                .createWebSocket("/websocket/rocket");

        webSocket.setTextMessageConsumer(message -> {
            System.out.println(message);
        });
        webSocket.openAndWait();

        /* Send some messages. */
        webSocket.sendText("Hi mom");
        webSocket.sendText("Hello World!");


        Sys.sleep(1000);
        webSocket.close();
        httpClient.stop();
        httpServer.stop();
    }

}
```


## A more involved example showing onClose, onOpen, onError, and onMessage

First lets show the server side setup.

#### Server side Java WebSocket event handling with Java 8 Lambda
```java

        /* Create an HTTP server. */
        HttpServer httpServer = httpServerBuilder()
                .setPort(8080).build();

        /* Setup WebSocket Server support. */
        httpServer.setWebSocketOnOpenConsumer(webSocket -> {

            /** Set up onMessage. */
            webSocket.setTextMessageConsumer(message -> {
                webSocket.sendText("ECHO " + message);
            });

            /** Set up onClose. */
            webSocket.setCloseConsumer(obj -> {
                puts("SERVER CLOSE ");
            });

            /** Set up onError. */
            webSocket.setErrorConsumer(error -> {
                puts("SERVER ERROR", error);
            });

        });

```

Now here is the client side setup.

#### Client side Java WebSocket event handling with Java 8 Lambda

```java

        /* Setup an httpClient. */
        HttpClient httpClient = httpClientBuilder()
                .setHost("localhost").setPort(8080).build();
        httpClient.start();

        /* Setup the client websocket. */
        WebSocket webSocket = httpClient
                .createWebSocket("/websocket/rocket");

        webSocket.setTextMessageConsumer(message -> {
            System.out.println("CLIENT ON MESSAGE \n" + message);
        });

        /** Set up onClose. */
        webSocket.setCloseConsumer(obj -> {
            puts("CLIENT CLOSE");
        });


        /** Set up onError. */
        webSocket.setErrorConsumer(error -> {
            puts("CLIENT ERROR", error);
        });

```

Now the complete example

#### Complete Java 8 Lambda WebSocket example

```java
package io.advantageous.qbit.http.jetty;

import io.advantageous.qbit.http.client.HttpClient;
import io.advantageous.qbit.http.server.HttpServer;
import io.advantageous.qbit.http.websocket.WebSocket;
import org.boon.core.Sys;

import static io.advantageous.qbit.http.client.HttpClientBuilder.httpClientBuilder;
import static io.advantageous.qbit.http.server.HttpServerBuilder.httpServerBuilder;
import static org.boon.Boon.puts;

/**
 * Created by rhightower on 2/16/15.
 */
public class EchoWebSocketMoreComplex {

    public static void main(String... args) {


        /* Create an HTTP server. */
        HttpServer httpServer = httpServerBuilder()
                .setPort(8080).build();

        /* Setup WebSocket Server support. */
        httpServer.setWebSocketOnOpenConsumer(webSocket -> {

            /** Set up onMessage. */
            webSocket.setTextMessageConsumer(message -> {
                webSocket.sendText("ECHO " + message);
            });

            /** Set up onClose. */
            webSocket.setCloseConsumer(obj -> {
                puts("SERVER CLOSE ");
            });

            /** Set up onError. */
            webSocket.setErrorConsumer(error -> {
                puts("SERVER ERROR", error);
            });

        });

        /* Start the server. */
        httpServer.start();

        /** CLIENT. */

        /* Setup an httpClient. */
        HttpClient httpClient = httpClientBuilder()
                .setHost("localhost").setPort(8080).build();
        httpClient.start();

        /* Setup the client websocket. */
        WebSocket webSocket = httpClient
                .createWebSocket("/websocket/rocket");

        webSocket.setTextMessageConsumer(message -> {
            System.out.println("CLIENT ON MESSAGE \n" + message);
        });

        /** Set up onClose. */
        webSocket.setCloseConsumer(obj -> {
            puts("CLIENT CLOSE");
        });


        /** Set up onError. */
        webSocket.setErrorConsumer(error -> {
            puts("CLIENT ERROR", error);
        });

        webSocket.openAndWait();

        /* Send some messages. */
        webSocket.sendText("Hi mom");
        webSocket.sendText("Hello World!");


        Sys.sleep(1000);
        puts("----------- SHUTDOWN --------------");
        webSocket.close();
        Sys.sleep(100);
        httpClient.stop();
        httpServer.stop();
    }

}

```


#### Output from last example
```output
CLIENT ON MESSAGE
ECHO Hi mom
CLIENT ON MESSAGE
ECHO Hello World!
----------- SHUTDOWN --------------
SERVER CLOSE
CLIENT CLOSE

```
