# [Detailed Tutorial] Working with inproc MicroServices within QBit.

##Overview

What is a process? A process provides the resources needed to execute a program. It is a collection interrelated work tasks initiated in response to an event. In-process microservices happen in-memory, locally on your machine no need for JSON/WebSocket or JSON/REST.

## Working with Inproc MicroServices within QBit
This wiki will walk you through the process of creating high speed in-process services with QBit


## What you will build
You will build an example of qbit handling and executing very fast async services; in this case a a todo service.
Will execute 2 methods `example1(args)` and `example2(args)` each method will add the following todo item:

```javascript
Todo{name='Call Mom', description='Give Mom a call'}
```
When this example is executed it will show you the following:
```javascript
This is an async call
This is an async call to count


This lambda expression is the callback 1
This is the count back from the server 1
END EXAMPLE 1
END EXAMPLE 1
END EXAMPLE 1
END EXAMPLE 1

This is an async call
This is an async call to count

This lambda expression is the callback 1
Todo{name='Call Mom', description='Give Mom a call'}
```
I cleaned up the output to make it visible, will post the entire output at the end after testing.
This is basically telling you when the first example started and ended and returning the callback count, same thing for the second example, also the todo item is printed.

## How to complete this guide
In order to complete this example successfully you will need the following installed on your machine:

- Gradle; if you need help installing it, visit [Installing Gradle. ](http://www.gradle.org/docs/current/userguide/installation.html)
- Your favorite IDE or text editor (we recommend [Intellij IDEA ](https://www.jetbrains.com/idea/) latest version).
- [JDK ](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) 1.8 or later.
- Build and install QBit on your machine click [Building QBit ](https://github.com/advantageous/qbit/wiki/%5BQuick-Start%5D-Building-QBit-the-microservice-lib-for-Java) for instructions.


Now that your machine is all ready let's get started:
- [Download ](https://github.com/fadihub/inproc-microservice-qbit/archive/master.zip) and unzip the source repository for this guide, or clone it using Git:

```bash
git clone https://github.com/fadihub/inproc-microservice-qbit.git
```

Once this is done you can __test the service__, let's first explain the process:

We have a `Todo` service, this sets your todo item with 2 parameters `name` and `description`, also enables you to get the name and description of the item:

```java
        public static class Todo {
        private final String name;
        private final String description;


        public Todo(String name, String description) {

            this.name = name;
            this.description = description;
        }

        public String getName() {
            return name;
        }

        public String getDescription() {
            return description;
        }

        @Override
        public String toString() {
            return "Todo{" +
                    "name='" + name + '\'' +
                    ", description='" + description + '\'' +
                    '}';
        }
    }
```


`TodoManager` service and the `TodoManagerClientInterface`, enables the user to add(), list(), count() the todo items:

```java
 interface TodoManagerClientInterface {

        void add(Todo todo);

        void list(Callback<List<Todo>> list);

        void count(Callback<Integer> count);

        void clientProxyFlush();

    }

    /**
     * Example service class
     */
    public static class TodoManager {

        private List<Todo> list = new ArrayList<>();

        public void add(Todo todo) {
            System.out.println("Add Todo");
            list.add(todo);
        }

        public List<Todo> list() {

            System.out.println("List Todo");
            return new ArrayList<>(list);
        }

        public int count() {
            System.out.println("Count Todo");
            return list.size();
        }
    }
```

So we have a `TodoManager` class, how can we make it a QBit service? Very simple you just create an object of the class and wire it in as follows:

```java
/* Synchronous service. */
        final TodoManager todoManagerImpl = new TodoManager();

        /*
        Create the service which manages async calls to todoManagerImpl.
         */
                final ServiceQueue service = serviceBuilder()
                .setQueueBuilder(queueBuilder().setBatchSize(1))
                .setServiceObject(todoManagerImpl).setInvokeDynamic(false)
                .build().start();
```
Very important to note that you can set the batch size of methods this allows threads to handle many method calls with an access of the queue.

Now you have a QBit service all you need is a proxy to invoke a method on a QBit service, also to send messages to a service. The service will get those messages as method calls.

Every call is sent over a high-speed internal in-process queue (inproc queue); to make a `todoManager` proxy do the following:

```java
/* Create Asynchronous proxy over Synchronous service. */
        final TodoManagerClientInterface todoManager = service.createProxy(TodoManagerClientInterface.class);
```

To be able to count the call backs later you have to start the call back handler:

```java
service.startCallBackHandler();
```

Since the proxy is created now, we can send messages to it. Here the client proxy is used to add __"Call Mom", "Give Mom a call"__ as a todo item:

```java
todoManager.add(new Todo("Call Mom", "Give Mom a call"));
```

The `countTracker` variable is initialized it holds the count value from call to service, `todoManager.count` return the count on the todo items listed then the lambda expression sets `count` to the value of listed items then countTracker is set to count:

```java
AtomicInteger countTracker = new AtomicInteger(); //Hold count from async call to service

System.out.println("This is an async call to count");

            todoManager.count(count -> {
            System.out.println("This lambda expression is the callback " + count);

            countTracker.set(count);
```

Every so often, we have to flush calls to the proxy. The proxy will flush calls every time the queue batch size is met. Here the queue batch size was set to 1, so it would flush every call. But no matter what, when you are done making calls, you should flush the calls. QBit employes micro-batching to reduce thread hand off into the queues. It is done as follows:

```java
todoManager.clientProxyFlush(); //Flush all methods. It batches calls.
```
There is also a utility method for flushing proxies so you do not have to add the clientProxyFlush to every proxy.

The following statements are printed to show you when the first example has ended:

```java
        System.out.printf("END EXAMPLE 1\n", countTracker.get());
        System.out.printf("END EXAMPLE 1\n", countTracker.get());
        System.out.printf("END EXAMPLE 1\n", countTracker.get());
        System.out.printf("END EXAMPLE 1\n", countTracker.get());
```

the second example `example2(args) is exactly the same; except for the todo item print at the end:
```java
            todoManager.list(todos -> {
            todos.forEach(item -> System.out.println(item));
```

it just prints the todo item posted:

```javascript
{name='Call Mom', description='Give Mom a call'}
```


#### InProcExample.java Listing (full example)

`src/main/java/io.advantageous.qbit.example.inproc/InProcExample.java`

```java
import io.advantageous.qbit.service.Callback;
import io.advantageous.qbit.service.Service;
import org.boon.core.Sys;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

import static io.advantageous.qbit.queue.QueueBuilder.queueBuilder;
import static io.advantageous.qbit.service.ServiceBuilder.serviceBuilder;

/**
 * Created by rhightower on 1/30/15.
 */
public class InProcExample {


    public static void main(String... args) throws Exception {

        example1(args);

        example2(args);

    }

    /**
     * You can use services in process
     */
    public static void example1(String... args) throws Exception {

        /* Synchronous service. */
        final TodoManager todoManagerImpl = new TodoManager();

        /*
        Create the service which manages async calls to todoManagerImpl.
         */
        final ServiceQueue service = serviceBuilder()
                .setServiceObject(todoManagerImpl)
                .build().start();


        /* Create Asynchronous proxy over Synchronous service. */
        final TodoManagerClientInterface todoManager =
                  service.createProxy(TodoManagerClientInterface.class);

        service.startCallBackHandler();


        System.out.println("This is an async call");
        /* Asynchronous method call. */
        todoManager.add(new Todo("Call Mom", "Give Mom a call"));


        AtomicInteger countTracker = new AtomicInteger(); //Hold count from async call to service

        System.out.println("This is an async call to count");

        todoManager.count(count -> {
            System.out.println("This lambda expression is the callback " + count);

            countTracker.set(count);
        });


        todoManager.clientProxyFlush(); //Flush all methods. It batches calls.

        Sys.sleep(100);

        System.out.printf("This is the count back from the server %d\n", countTracker.get());


        System.out.printf("END EXAMPLE 1\n", countTracker.get());
        System.out.printf("END EXAMPLE 1\n", countTracker.get());
        System.out.printf("END EXAMPLE 1\n", countTracker.get());
        System.out.printf("END EXAMPLE 1\n", countTracker.get());

    }


    /**
     * You can use services in process
     */
    public static void example2(String... args) throws Exception {

        /* Synchronous service. */
        final TodoManager todoManagerImpl = new TodoManager();

        /*
        Create the service which manages async calls to todoManagerImpl.

        You can control the batch size of methods.
        After it hits so many, it sends those methods to the service.
        This allows threads to handle many method calls with on access of the queue.
        Here we set it to 1 so it will flush with every call.
        Setting invoke dynamic false turns off auto type conversion which is mainly for JSON REST calls
        and WebSocket calls.
        This means that you can execute the service more efficiently when it is in proc.
         */
        final ServiceQueue service = serviceBuilder()
                .setQueueBuilder(queueBuilder().setBatchSize(1))
                .setServiceObject(todoManagerImpl).setInvokeDynamic(false)
                .build().start();


        /* Create Asynchronous proxy over Synchronous service. */
        final TodoManagerClientInterface todoManager =
                    service.createProxy(TodoManagerClientInterface.class);

        service.startCallBackHandler();


        System.out.println("This is an async call");
        /* Asynchronous method call. */
        todoManager.add(new Todo("Call Mom", "Give Mom a call"));


        AtomicInteger countTracker = new AtomicInteger();
            //Hold count from async call to service

        System.out.println("This is an async call to count");

        todoManager.count(count -> {
            System.out.println("This lambda expression is the callback " + count);

            countTracker.set(count);
        });


        /*
        We don't need this now.
         */
        //todoManager.clientProxyFlush(); //Flush all methods. It batches calls.

        Sys.sleep(100);

        System.out.printf("This is the count back from the service %d\n", countTracker.get());

        todoManager.list(todos -> {
            todos.forEach(item -> System.out.println(item));
        });

    }

    interface TodoManagerClientInterface {

        void add(Todo todo);

        void list(Callback<List<Todo>> list);

        void count(Callback<Integer> count);

        void clientProxyFlush();

    }

    /**
     * Example service class
     */
    public static class TodoManager {

        private List<Todo> list = new ArrayList<>();

        public void add(Todo todo) {
            System.out.println("Add Todo");
            list.add(todo);
        }

        public List<Todo> list() {

            System.out.println("List Todo");
            return new ArrayList<>(list);
        }

        public int count() {
            System.out.println("Count Todo");
            return list.size();
        }
    }

    public static class Todo {
        private final String name;
        private final String description;


        public Todo(String name, String description) {

            this.name = name;
            this.description = description;
        }

        public String getName() {
            return name;
        }

        public String getDescription() {
            return description;
        }

        @Override
        public String toString() {
            return "Todo{" +
                    "name='" + name + '\'' +
                    ", description='" + description + '\'' +
                    '}';
        }
    }
}
```

This is just an example to show you how to use an in-proc service.

The AtomicInteger was to demonstrate that this example is working, you would avoid AtomicInteger in your services.


#### build.gradle Listing


Here is the build

```gradle
group = 'io.advantageous.qbit'
apply plugin: 'idea'
apply plugin: 'java'
apply plugin: 'maven'
apply plugin: 'application'

sourceCompatibility = 1.8
version = '1.0'


repositories {
    mavenLocal()
    mavenCentral()
}

sourceCompatibility = JavaVersion.VERSION_1_8
targetCompatibility = JavaVersion.VERSION_1_8

mainClassName = "io.advantageous.qbit.example.inproc.InProcExample"


dependencies {

    compile group: 'io.advantageous.qbit', name: 'qbit-vertx', version: '0.5.2-SNAPSHOT'
    compile group: 'io.advantageous.qbit', name: 'qbit-boon', version: '0.5.2-SNAPSHOT'
    compile group: 'io.advantageous.qbit', name: 'qbit-core', version: '0.5.2-SNAPSHOT'

    compile "org.slf4j:slf4j-api:[1.7,1.8)"
    compile 'ch.qos.logback:logback-classic:1.1.2'

    testCompile group: 'junit', name: 'junit', version: '4.10'
}

idea {
    project {
        jdkName = '1.8'
        languageLevel = '1.8'
    }
}
```

This is how you create very fast inproc services with QBit.

## Test The In-Process Example

With your terminal cd into `inproc-microservice-qbit` then
`gradle clean build` then
`gradle run`
You should get the following output:
```javascript
Factory was null
22:46:04.525 [main] DEBUG io.advantageous.qbit.util.Timer - timer started
22:46:04.529 [main] DEBUG i.a.q.c.ScheduledThreadContext - Started: <NULL>

22:46:04.560 [main] DEBUG i.a.q.c.ScheduledThreadContext - Started: <NULL>

22:46:04.565 [main] DEBUG i.a.q.c.ScheduledThreadContext - Started: <NULL>

22:46:04.570 [main] DEBUG i.a.q.c.ScheduledThreadContext - Started: <NULL>

This is an async call
This is an async call to count
22:46:04.611 [QueueListener Send Queue  booneventmanager] DEBUG i.a.qbit.service.impl.ServiceImpl - ServiceImpl::doHandleMethodCall() METHOD CALLio.advantageous.qbit.service.impl.ServiceImpl$MethodCallLocal@69d7ae6d
22:46:04.612 [QueueListener Send Queue  booneventmanager] DEBUG i.a.qbit.service.impl.ServiceImpl - ServiceImpl::receive()
RESPONSE
ResponseImpl{address='', params=null, body={Error=Must be called from inside of a Service, Details=java.lang.IllegalStateException: Must be called from inside of a Service, Message=Problem while calling method joinService, Cause=null}}
FROM CALL
io.advantageous.qbit.service.impl.ServiceImpl$MethodCallLocal@69d7ae6d


22:46:04.616 [QueueListener Send Queue  todomanager] DEBUG i.a.qbit.service.impl.ServiceImpl - ServiceImpl::doHandleMethodCall() METHOD CALLio.advantageous.qbit.service.impl.ServiceImpl$MethodCallLocal@31d7f68
Add Todo
22:46:04.616 [QueueListener Send Queue  todomanager] DEBUG i.a.qbit.service.impl.ServiceImpl - ServiceImpl::receive()
RESPONSE
io.advantageous.qbit.service.impl.ServiceConstants$1@dda817b
FROM CALL
io.advantageous.qbit.service.impl.ServiceImpl$MethodCallLocal@31d7f68


22:46:04.616 [QueueListener Send Queue  todomanager] DEBUG i.a.qbit.service.impl.ServiceImpl - ServiceImpl::doHandleMethodCall() METHOD CALLio.advantageous.qbit.service.impl.ServiceImpl$MethodCallLocal@9e51418
Count Todo
22:46:04.617 [QueueListener Send Queue  todomanager] DEBUG i.a.qbit.service.impl.ServiceImpl - ServiceImpl::receive()
RESPONSE
ResponseImpl{address='count', params=null, body=1}
FROM CALL
io.advantageous.qbit.service.impl.ServiceImpl$MethodCallLocal@9e51418


This lambda expression is the callback 1
This is the count back from the server 1
END EXAMPLE 1
END EXAMPLE 1
END EXAMPLE 1
END EXAMPLE 1
22:46:04.682 [main] DEBUG i.a.q.c.ScheduledThreadContext - Started: <NULL>

22:46:04.682 [QueueListener Send Queue  booneventmanager] DEBUG i.a.qbit.service.impl.ServiceImpl - ServiceImpl::doHandleMethodCall() METHOD CALLio.advantageous.qbit.service.impl.ServiceImpl$MethodCallLocal@725d2172
22:46:04.683 [QueueListener Send Queue  booneventmanager] DEBUG i.a.qbit.service.impl.ServiceImpl - ServiceImpl::receive()
RESPONSE
io.advantageous.qbit.service.impl.ServiceConstants$1@dda817b
FROM CALL
io.advantageous.qbit.service.impl.ServiceImpl$MethodCallLocal@725d2172


22:46:04.683 [main] DEBUG i.a.q.c.ScheduledThreadContext - Started: <NULL>

This is an async call
This is an async call to count
22:46:04.734 [QueueListener Send Queue  todomanager] DEBUG i.a.qbit.service.impl.ServiceImpl - ServiceImpl::doHandleMethodCall() METHOD CALLio.advantageous.qbit.service.impl.ServiceImpl$MethodCallLocal@5710420b
Add Todo
22:46:04.734 [QueueListener Send Queue  todomanager] DEBUG i.a.qbit.service.impl.ServiceImpl - ServiceImpl::receive()
RESPONSE
io.advantageous.qbit.service.impl.ServiceConstants$1@dda817b
FROM CALL
io.advantageous.qbit.service.impl.ServiceImpl$MethodCallLocal@5710420b


22:46:04.734 [QueueListener Send Queue  todomanager] DEBUG i.a.qbit.service.impl.ServiceImpl - ServiceImpl::doHandleMethodCall() METHOD CALLio.advantageous.qbit.service.impl.ServiceImpl$MethodCallLocal@59d7a699
Count Todo
22:46:04.734 [QueueListener Send Queue  todomanager] DEBUG i.a.qbit.service.impl.ServiceImpl - ServiceImpl::receive()
RESPONSE
ResponseImpl{address='count', params=null, body=1}
FROM CALL
io.advantageous.qbit.service.impl.ServiceImpl$MethodCallLocal@59d7a699


This is the count back from the service 0
22:46:04.786 [QueueListener Send Queue  todomanager] DEBUG i.a.qbit.service.impl.ServiceImpl - ServiceImpl::doHandleMethodCall() METHOD CALLio.advantageous.qbit.service.impl.ServiceImpl$MethodCallLocal@5de8297b
List Todo
22:46:04.786 [QueueListener Send Queue  todomanager] DEBUG i.a.qbit.service.impl.ServiceImpl - ServiceImpl::receive()
RESPONSE
ResponseImpl{address='list', params=null, body=[Todo{name='Call Mom', description='Give Mom a call'}]}
FROM CALL
io.advantageous.qbit.service.impl.ServiceImpl$MethodCallLocal@5de8297b


This lambda expression is the callback 1
Todo{name='Call Mom', description='Give Mom a call'}
```

##Summary
You have just built an inproc service within QBit and tested it.
