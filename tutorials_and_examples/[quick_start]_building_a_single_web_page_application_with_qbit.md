# [Quick Start] building a single web page application with QBit

##Overview
The QBit lib allows you to bind objects to HTTP ports so that they can be called by REST and WebSocket clients.
This example will demonstrate that.


## Building a simple Hello World app using QBit
This wiki will walk trough the process of building a simple single page app will QBit.



## What you will build
You will build a page that sends (HTTP/GET) request and gets back some JSON in which it parses and displays the Hello World message.

You will be able to view the app and test it after you run `HelloWorldRestServer.java` at the following address using your favorite browser:

```bash
 http://localhost:9999/ui/helloWorld.html
```
When you open up the app you will get an opting to press a `try it` button once you do you will get the following:
```javascript
Hello World!
```

## How to complete this guide
In order to complete this example successfully you will need the following installed on your machine:

- Gradle; if you need help installing it, visit [Installing Gradle. ](http://www.gradle.org/docs/current/userguide/installation.html)
- Your favorite IDE or text editor (we recommend [Intellig IDEA ](https://www.jetbrains.com/idea/) latest version).
- [JDK ](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) 1.8 or later.
- Build and install QBit on your machine click [Building QBit ](https://github.com/advantageous/qbit/wiki/%5BQuick-Start%5D-Building-QBit-the-microservice-lib-for-Java) for instrutions.


Now that your machine is all ready let's get started:
- [Download ](https://github.com/fadihub/hello/archive/master.zip) and unzip the source repository for this guide, or clone it using Git:

```bash
git clone https://github.com/fadihub/hello.git
```

Once this is done you can __test the service__, let's first explain the process:

#### helloWorld.html Listing

`~/hello/src/main/resources/ui/helloWorld.html`

```html
<!DOCTYPE html>
<html>
<body>

<p>click the button:</p>


<button onclick="myFunction()">Try it</button>

<p id="demo"></p>

<script>
    function httpGet(theUrl)
    {
        var xmlHttp = null;
        xmlHttp = new XMLHttpRequest();
        xmlHttp.open( "GET", theUrl, false );
        xmlHttp.send( null );
        return xmlHttp.responseText;
    }
    function myFunction() {
        var json = httpGet("/services/helloservice/hello");
        var helloObject = JSON.parse(json);
        document.getElementById("demo").innerHTML = helloObject.hello;
    }
</script>

</body>
</html>
```

As you can see very simple code requesting the hello object, then parsing it into JSON, then displaying it onto the app.

#### HelloObject.java Listing

`~/hello/src/main/java/io.advantageous.qbit.example.hello/HelloObject.java`

```java
package io.advantageous.qbit.example.hello;

/**
 * Created by rhightower on 2/10/15.
 */


public class HelloObject {
    private final String hello;
    private final long time = System.currentTimeMillis();

    public HelloObject(String hello) {
        this.hello = hello;
    }
}
```

This is the hello object. This represents the Model (M) if you are comparing it to a MVC system.

#### HelloService.java Listing

`~/hello/src/main/java/io.advantageous.qbit.example.hello/HelloService.java`

```java
package io.advantageous.qbit.example.hello;

import io.advantageous.qbit.annotation.RequestMapping;

/**
 * Created by rhightower on 2/10/15.
 */

@RequestMapping("/helloservice")
public class HelloService {


    @RequestMapping("/hello")
    public HelloObject hello() {
        return new HelloObject("Hello World!");
    }

}
```
If you are comparing it to a MVC system this this like the controller (C).


#### HelloWorldRestServer.java Listing

`~/hello/src/main/java/io.advantageous.qbit.example.hello/HelloWorldRestServer.java`

```java
package io.advantageous.qbit.example.hello;

import io.advantageous.qbit.http.HttpServer;
import io.advantageous.qbit.server.ServiceServer;
import io.advantageous.qbit.system.QBitSystemManager;

import static io.advantageous.qbit.http.HttpServerBuilder.httpServerBuilder;
import static io.advantageous.qbit.server.ServiceServerBuilder.serviceServerBuilder;
import static org.boon.Boon.resource;

/**
 * Created by rhightower on 2/9/15.
 */
public class HelloWorldRestServer {


    public static final String HTML_HELLO_PAGE = "/ui/helloWorld.html";


    public static void main(String... args) {

        /* Create the system manager to manage the shutdown. */
        QBitSystemManager systemManager = new QBitSystemManager();

        HttpServer httpServer = httpServerBuilder()
                .setPort(9999).build();

        /* Register the Predicate using a Java 8 lambda expression. */
        httpServer.setShouldContinueHttpRequest(httpRequest -> {
            /* If not the page uri we want to then just continue by returning true. */
            if (!httpRequest.getUri().equals(HTML_HELLO_PAGE)) {
                return true;
            }
            /* read the page from the file system or classpath. */
            final String helloWorldWebPage = resource(HTML_HELLO_PAGE);
            /* Send the HTML file out to the browser. */
            httpRequest.getResponse().response(200, "text/html", helloWorldWebPage);
            return false;
        });


        /* Start the service. */
        final ServiceServer serviceServer = serviceServerBuilder().setSystemManager(systemManager)
                .setHttpServer(httpServer).build().initServices(new HelloService()).startServer();

        /* Wait for the service to shutdown. */
        systemManager.waitForShutdown();

    }


}
```

This is like the view (V) in a MVC system, when you run this you can see the app on __http://localhost:9999/ui/helloWorld.html__

## Test The APP

With your terminal cd into `hello` then
`gradle clean build` then
`gradle run` then
open up your favorite browser and visit __http://localhost:9999/ui/helloWorld.html__ the simple app will show up and then you can click `try it` and the message `Hello World!` will be printed.

##Summary
You have just built a simple app with QBit and tested it.
