# [Quick Start] Working with inproc MicroServices within QBit

##overview

What is a process? A process provides the resources needed to execute a program. It is a collection interrelated work tasks initiated in response to an event. In-process microservices happen in-memory, locally on your machine no need for JSON/websocket or JSON/REST.

## Working with inproc MicroServices within QBit
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
- Your favorite IDE or text editor (we recommend [Intellig IDEA ](https://www.jetbrains.com/idea/) latest version).
- [JDK ](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) 1.8 or later.
- Build and install QBit on your machine click [Building QBit ](https://github.com/advantageous/qbit/wiki/%5BQuick-Start%5D-Building-QBit-the-microservice-lib-for-Java) for instrutions.


Now that your machine is all ready let's get started:
- [Download ](https://github.com/fadihub/inproc-microservice-qbit/archive/master.zip) and unzip the source repository for this guide, or clone it using Git:

```bash
https://github.com/fadihub/inproc-microservice-qbit.git
```

Once this is done you can __test the service__, let's first explain the process:

## Full Example Listing
#### -InProcExample.java
#### -TodoManager.java
#### -Todo.java

`~/src/main/java/io.advantageous.qbit.example.iproc/InProcExample.java`

```java
/*******************************************************************************

 * Copyright (c) 2015. Rick Hightower, Geoff Chandler
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *  		http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 *  ________ __________.______________
 *  \_____  \\______   \   \__    ___/
 *   /  / \  \|    |  _/   | |    |  ______
 *  /   \_/.  \    |   \   | |    | /_____/
 *  \_____\ \_/______  /___| |____|
 *         \__>      \/
 *  ___________.__                  ____.                        _____  .__                                             .__
 *  \__    ___/|  |__   ____       |    |____ ___  _______      /     \ |__| ___________  ____  ______ ______________  _|__| ____  ____
 *    |    |   |  |  \_/ __ \      |    \__  \\  \/ /\__  \    /  \ /  \|  |/ ___\_  __ \/  _ \/  ___// __ \_  __ \  \/ /  |/ ___\/ __ \
 *    |    |   |   Y  \  ___/  /\__|    |/ __ \\   /  / __ \_ /    Y    \  \  \___|  | \(  <_> )___ \\  ___/|  | \/\   /|  \  \__\  ___/
 *    |____|   |___|  /\___  > \________(____  /\_/  (____  / \____|__  /__|\___  >__|   \____/____  >\___  >__|    \_/ |__|\___  >___  >
 *                  \/     \/                \/           \/          \/        \/                 \/     \/                    \/    \/
 *  .____    ._____.
 *  |    |   |__\_ |__
 *  |    |   |  || __ \
 *  |    |___|  || \_\ \
 *  |_______ \__||___  /
 *          \/       \/
 *       ____. _________________    _______         __      __      ___.     _________              __           __      _____________________ ____________________
 *      |    |/   _____/\_____  \   \      \       /  \    /  \ ____\_ |__  /   _____/ ____   ____ |  | __ _____/  |_    \______   \_   _____//   _____/\__    ___/
 *      |    |\_____  \  /   |   \  /   |   \      \   \/\/   // __ \| __ \ \_____  \ /  _ \_/ ___\|  |/ // __ \   __\    |       _/|    __)_ \_____  \   |    |
 *  /\__|    |/        \/    |    \/    |    \      \        /\  ___/| \_\ \/        (  <_> )  \___|    <\  ___/|  |      |    |   \|        \/        \  |    |
 *  \________/_______  /\_______  /\____|__  / /\    \__/\  /  \___  >___  /_______  /\____/ \___  >__|_ \\___  >__| /\   |____|_  /_______  /_______  /  |____|
 *                   \/         \/         \/  )/         \/       \/    \/        \/            \/     \/    \/     )/          \/        \/        \/
 *  __________           __  .__              __      __      ___.
 *  \______   \ ____   _/  |_|  |__   ____   /  \    /  \ ____\_ |__
 *  |    |  _// __ \  \   __\  |  \_/ __ \  \   \/\/   // __ \| __ \
 *   |    |   \  ___/   |  | |   Y  \  ___/   \        /\  ___/| \_\ \
 *   |______  /\___  >  |__| |___|  /\___  >   \__/\  /  \___  >___  /
 *          \/     \/             \/     \/         \/       \/    \/
 *
 * QBit - The Microservice lib for Java : JSON, WebSocket, REST. Be The Web!
 *  http://rick-hightower.blogspot.com/2014/12/rise-of-machines-writing-high-speed.html
 *  http://rick-hightower.blogspot.com/2014/12/quick-guide-to-programming-services-in.html
 *  http://rick-hightower.blogspot.com/2015/01/quick-start-qbit-programming.html
 *  http://rick-hightower.blogspot.com/2015/01/high-speed-soa.html
 *  http://rick-hightower.blogspot.com/2015/02/qbit-event-bus.html

 ******************************************************************************/

package io.advantageous.qbit.example.inproc;

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
        final TodoManagerClientInterface todoManager = service.createProxy(TodoManagerClientInterface.class);

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
        final TodoManagerClientInterface todoManager = service.createProxy(TodoManagerClientInterface.class);

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
This code will be explained in detail under [[Detailed Tutorial] Working with inproc MicroServices within QBit ](https://github.com/advantageous/qbit/wiki/%5BDetailed-Tutorial%5D-Working-with-inproc-MicroServices-within-QBit.) or in the next section




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
