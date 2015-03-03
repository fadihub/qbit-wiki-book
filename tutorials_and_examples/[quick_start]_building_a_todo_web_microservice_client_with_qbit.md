# [Quick Start] Building a TODO web microservice client with QBit

## Building a TODO web microservice client with QBit

This wiki will walk you through the process for building a websocket client for the Todo web service we built in [the previous wiki. ](https://github.com/advantageous/qbit/wiki/%5BQuick-Start%5D-Building-a-TODO-web-microservice-server-with-QBit)

## What you will build
 You will build a websocket client service that will POST/GET the following:

```javascript
Greeting Hello World Sun Jan 25 13:10:47 PST 2015
Tutorial I hope you got this to work Sun Jan 25 13:10:47 PST 2015
```
to the `todo-service` you have built in the [previous tutorial. ](https://github.com/advantageous/qbit/wiki/Building-a-TODO-web-service-server-with-QBit)

## How to complete this guide
In order to complete this example successfully you will need the following installed on your machine:

- Gradle; if you need help installing it, visit [Installing Gradle. ](http://www.gradle.org/docs/current/userguide/installation.html)
- Your favorite IDE or text editor (we recommend [Intellig IDEA ](https://www.jetbrains.com/idea/) latest version).
- [JDK ](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) 1.8 or later.
- Build and install QBit on your machine click [Building QBit ](https://github.com/advantageous/qbit/wiki/%5BQuick-Start%5D-Building-QBit-the-microservice-lib-for-Java) for instrutions.


Now that your machine is all ready let's get started:

-[Download](https://github.com/fadihub/todo-service-websocket-client/archive/master.zip) and unzip the source repository for this guide, or clone it using Git:

```bash
git clone https://github.com/fadihub/todo-service-websocket-client.git
```
Once this is done you can __test the service__, let's first explain the process:

This is the `/TodoItem` that the client is GETTING/POSTING, provides a plain old java object with fields, constructors, and accessors for the `description`, `name`, and date data:

#### TodoItem Listing
`src/main/java/io.advantageous.qbit.examples.client/TodoItem.java`
```java
package io.advantageous.qbit.examples.client;

import java.util.Date;


public class TodoItem {


    private String description;
    private String name;
    private Date due;

    public TodoItem(final String description, final String name, final Date due) {
        this.description = description;
        this.name = name;
        this.due = due;
    }

    public TodoItem(String description, String name) {

        this.description = description;
        this.name = name;
        due = new Date();
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


    public void setDescription(String description) {
        this.description = description;
    }

    public void setName(String name) {
        this.name = name;
    }

    public void setDue(Date due) {
        this.due = due;
    }
}
```

#### TodoServiceClientInterface Listing
This is the `/TodoServiceClientInterface`:

`src/main/java/io.advantageous.qbit.examples.client/TodoServiceClientInterface`
```java
package io.advantageous.qbit.examples.client;

import io.advantageous.qbit.service.Callback;

import java.util.List;

public interface TodoServiceClientInterface {

    void list(Callback<List<TodoItem>> handler);

    void add(TodoItem todoItem);


}
```

#### TodoClientMain listing
To run the client and POST the items `Greeting` and `Tutorial`:

`src/main/java/io.advantageous.qbit.examples.client/TodoClientMain.java`
```java
package io.advantageous.qbit.examples.client;


import io.advantageous.qbit.client.Client;
import io.advantageous.qbit.client.ClientBuilder;
import org.boon.core.Sys;

/**
 * Created by fadi on 1/9/15.
 */
public class TodoClientMain {

    public static void main(String... args) {

        String host = "localhost";

        int port = 8080;


        Client client = new ClientBuilder().setPort(port).setHost(host).setPollTime(10)
                .setAutoFlush(true).setFlushInterval(50).setRequestBatchSize(50)
                .setProtocolBatchSize(50).build();

        TodoServiceClientInterface todoService =
                client.createProxy(TodoServiceClientInterface.class, "todoService");

        client.start();


        todoService.add(new TodoItem("Greeting", "Hello World"));
        todoService.add(new TodoItem("Tutorial","I hope you got this to work"));

        client.flush();


        todoService.list(todoItems -> { //LAMBDA EXPRESSION Java 8

            for (TodoItem item : todoItems) {
                System.out.println("TODO ITEM " + item.getDescription() + " " + item.getName() + " " + item.getDue());
            }
        });

        client.flush();

        Sys.sleep(1000);



    }
}
```
Now we are ready to test this client service.



## Test the client
In order for this client to work you need to have [todo-service] (https://github.com/advantageous/qbit/wiki/Building-a-TODO-web-service-server-with-QBit) running then cd into `~/todo-service-websocket-client` and run `gradle clean` then `gradle run`
you will get the following:
```javascript
Greeting Hello World Sun Jan 25 13:10:47 PST 2015
Tutorial I hope you got this to work Sun Jan 25 13:10:47 PST 2015
```
`Greeting` and `Tutorial` represent the descriptions for the two `TodoItem`s we are POSTING, `Hello World` and `I hope you got this to work` represent the `name` for the two `TodoItem`s, also `Sun Jan 25 13:10:47 PST 2015` will be different in your case, it will represent your current time on your machine.

If you:
```bash
curl http://localhost:8080/services/todo-service/todo/
```
you will get:
```javascript
[{"description":"Greeting","name":"Hello World"},{"description":"Tutorial","name":"I hope you got this to work"}]
```
which reflects what we have mentioned before.

##Summary
You have just built and tested a TODO web service client with QBit.
