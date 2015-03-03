# [Quick Start] Building a TODO web microservice server with QBit

## Building a TODO web service with QBit

This wiki will walk you through the process for building a Hello World TODO web microservice with QBit.



## What you will build
You will build a service that will accept HTTP GET/POST requests at:

```bash
curl http://localhost:8080/services/todo-service/todo/
```
Post a TODO item "Hello World" in this case using your terminal,

and respond with a JSON representation of a greeting or whatever you post:

```javascript
[{"name":"Hello World"}
```


## How to complete this guide
In order to complete this example successfully you will need the following installed on your machine:

- Gradle; if you need help installing it, visit [Installing Gradle. ](http://www.gradle.org/docs/current/userguide/installation.html)
- Your favorite IDE or text editor (we recommend [Intellig IDEA ](https://www.jetbrains.com/idea/) latest version).
- [JDK ](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) 1.8 or later.
- Build and install QBit on your machine click [Building QBit ](https://github.com/advantageous/qbit/wiki/%5BQuick-Start%5D-Building-QBit-the-microservice-lib-for-Java) for instrutions.

Now that your machine is all ready let's get started:
- [Download ](https://github.com/fadihub/todo-service/archive/master.zip) and unzip the source repository for this guide, or clone it using Git:

```bash
git clone https://github.com/fadihub/todo-service.git
```

Once this is done you can __test the service__, let's first explain the process:

This service will handle GET/POST requests for `/TodoService`. The POST request
```bash
curl -X POST -H "Content-Type: application/json" \
-d '{"name":"Hello World","description":"Greeting"}' \
http://localhost:8080/services/todo-service/todo
```
The GET request will return the following:
```javascript
{"description":"greeting","name":"Hello World"}
```
The `description` field is a unique identifier for the greeting in this case, and `name` is the textual content representation of the greeting.

To model the greeting representation, provide a plain old java object with fields, constructors, and accessors for the `description` and `name` data:

#### TodoItem Listing
`/src/main/java/io.advantageous.qbit.examples/TodoItem.java`

```java
package io.advantageous.qbit.examples;

import java.util.Date;


public class TodoItem {


    private final String description;
    private final String name;
    private final Date due;

    public TodoItem(final String description, final String name, final Date due) {
        this.description = description;
        this.name = name;
        this.due = due;
    }

    public String getDescription() {
        return description;
    }

    public String getName() {
        return name;
    }

    public Date getDue() {
        return due;
    }
}
```

#### TodoService Listing

The `/TodoService` will handle GET/POST requests for `/TodoItem`:

`/src/main/java/io.advantageous.qbit.examples/TodoService.java`

```java
package io.advantageous.qbit.examples;


import io.advantageous.qbit.annotation.RequestMapping;
import io.advantageous.qbit.annotation.RequestMethod;

import java.util.ArrayList;
import java.util.List;

@RequestMapping("/todo-service")
public class TodoService {

    List<TodoItem> items = new ArrayList<>();


    @RequestMapping("/todo/count")
    public int size() {

        return items.size();
    }

    @RequestMapping("/todo/")
    public List<TodoItem> list() {
        return items;
    }

    @RequestMapping(value = "/todo", method = RequestMethod.POST)
    public void add(TodoItem todoItem) {
        items.add(todoItem);
    }

}
```

## Running your local server

To make and run your localhost server:

#### TodoMain Listing

`/src/main/java/io.advantageous.qbit.examples/TodoMain.java`

```java
package io.advantageous.qbit.examples;

import io.advantageous.qbit.server.ServiceServer;
import io.advantageous.qbit.server.ServiceServerBuilder;

public class TodoMain {

    public static void main(String... args) {
        ServiceServer server = new ServiceServerBuilder().setHost("localhost").setPort(8080).build();
        server.initServices(new TodoService());
        server.start();
    }

}
```
now we are ready to test the service.

## Test the service

With your terminal cd into `/todo-service`

run `gradle clean` then `gradle run`
open up another terminal, then

```bash
curl http://localhost:8080/services/todo-service/todo/
```
you should get the following:

```javascript
[]
```
because you haven't posted anything yet, now post a greeting:

```bash
curl -X POST -H "Content-Type: application/json" \
-d '{"name":"Hello World","description":"greeting"}' \
http://localhost:8080/services/todo-service/todo
```

then visit __http://localhost:8080/services/todo-service/todo__ with your browser or:

```bash
curl localhost:8080/services/todo-service/todo
```
you should get the following:

```javascript
[{"description":"greeting","name":"Hello World"}]
```

You can post as many todo items as you want.

To query the size of the todo list run the following command:

```bash
curl localhost:8080/services/todo-service/todo/count
```


##Summary
You have just built and tested a TODO web service with QBit.
