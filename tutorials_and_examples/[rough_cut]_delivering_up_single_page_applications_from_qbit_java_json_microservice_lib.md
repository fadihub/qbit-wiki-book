# [Rough Cut] Delivering up Single Page Applications from QBit Java JSON Microservice lib


QBit can server up non-JSON resources. The QBit lib allows you to bind objects to HTTP ports so that they can be called by REST and WebSocket clients.

```java

/* The service. */
@RequestMapping("/todo-manager")
public class TodoService  {


    private final TodoRepository todoRepository = new ListTodoRepository();


    @RequestMapping("/todo/size")
    public int size() {

        return todoRepository.size();
    }


    @RequestMapping("/todo/list")
    public List<TodoItem> list() {

        return todoRepository.list();
    }
...

/* Then to start up. */
public class TodoServerMain {

    public static void main(String... args) {
        ServiceServer server = new ServiceServerBuilder().setRequestBatchSize(100).build();
        server.initServices(new TodoService());
        server.start();

    }
}
```

This is a smaller version of the wiring that starts things up and registers a system manager.
And waits for the server to shutdown.

```java

        QBitSystemManager systemManager = new QBitSystemManager();

        ServiceServer server = serviceServerBuilder()
                .setSystemManager(systemManager).build()
                .initServices(new TodoService()).startServer();

        systemManager.waitForShutdown();
```


Now let's see, how do we use a ServiceServer to host our REST and WebSocket and still deliver up a single page app.

Ok.. let's walk you through another example....

This is a simple HelloWorld type of an example.

Here is our HTML page. Small and to the point. Just example ware ok.

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

The above page sends request (REST/HTTP/GET), and gets back some JSON which is parses and displays the hello property. It is simple and small on purpose.


To deliver up this page, instead of having the ServiceServerBuilder create an HttpServer, we will create one for it, and pass that to the builder. HttpServer uses Java 8 functional interfaces to register event handlers for WebSocket events and HttpRequest events. The special sauce in Java 8 is called a Consumer.
The HttpServer also uses a special sauce Java 8 sauce Predicate. With the Predicate callback, you can specify if you want the request to be handled by the consumer or not. This means all we have to do is create a Predicate that handles the request and then return false so the consumer which is wired into the ServiceServerBuilder is never called. Then we load the resource "/ui/helloWorld.html". And we deliver that up. All other requests we let go through.

Let's show the code. First the HelloWorld object.

```java
public class HelloObject {
    private final String hello;
    private final long time = System.currentTimeMillis();

    public HelloObject(String hello) {
        this.hello = hello;
    }
}
```

That is your M in the MVC world.  (M Model) :)

Next the C in the MVC world. (C controller)

```java

@RequestMapping("/helloservice")
public class HelloService {


    @RequestMapping("/hello")
    public HelloObject hello() {
        return new HelloObject("Hello World!");
    }

}
```

Now we need to deliver up the V. (View).

When you run this one, you can see it here.

http://localhost:9999/ui/helloWorld.html

```java
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
            if ( ! httpRequest.getUri().equals(HTML_HELLO_PAGE) ) {
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

You can find the full example here:

[Hello World Full Example Showing Single Page App With QBit Microservice lib](https://github.com/advantageous/qbit/tree/master/qbit-examples/qbit-examples-standalone/src/main/java/io/advantageous/qbit/example/hello)
