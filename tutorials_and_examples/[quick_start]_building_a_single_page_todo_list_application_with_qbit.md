# [Quick Start] Building a single page; Todo List Application with QBit

##overview

QBit enables you to build applications hassle free, it can server up non-JSON resources. The QBit lib allows you to bind objects to HTTP ports so that they can be called by REST and WebSocket clients. This will be demonstrated in this example.


## Building a single page; Todo List Application with QBit
This wiki will walk you through the process of building a Todo List application with QBit.


## What you will build
You will build a Todo List application with QBit, you be able to access the aplication at the following address:

```bash
http://localhost:9999/ui/index.html
```
The app has a text box that will let you add todo items after typing it in and hitting ENTER, after the item is added you can delete it simply by double clicking it, or you can change its status (draw a line though it) simply by checking a box. here is a picture of the app in action:

![todo list app](http://3.bp.blogspot.com/-TbWkqfGFb2g/VO0yCR6xtKI/AAAAAAAAAAg/D1IJAhivVg0/s1600/Screen%2BShot%2B2015-02-24%2Bat%2B6.22.02%2BPM.png)


## How to complete this guide
In order to complete this example successfully you will need the following installed on your machine:

- Gradle; if you need help installing it, visit [Installing Gradle. ](http://www.gradle.org/docs/current/userguide/installation.html)
- Your favorite IDE or text editor (we recommend [Intellig IDEA ](https://www.jetbrains.com/idea/) latest version).
- [JDK ](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) 1.8 or later.
- Build and install QBit on your machine click [Building QBit ](https://github.com/advantageous/qbit/wiki/%5BQuick-Start%5D-Building-QBit-the-microservice-lib-for-Java) for instrutions.


Now that your machine is all ready let's get started:
- [Download ](https://github.com/fadihub/todo-list-app/archive/master.zip) and unzip the source repository for this guide, or clone it using Git:

```bash
git clone https://github.com/fadihub/todo-list-app.git
```

Once this is done you can __test the service__, let's first explain the process:

The process will be explained in more detail under [[Detailed Tutorial] Building a single page; Todo List Application with QBit ](https://github.com/advantageous/qbit/wiki/%5BDetailed-Tutorial%5D-Building-a-single-page;-Todo-List-Application-with-QBit) or in the next section.

#### index.html Listing

`src/main/java/io.advantageous.qbit.example.todolistapp/TodoService.java`

```html
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8">
    <title>TODO list example with Qbit</title>

    <style>

        body {
        background-color: #D8D8D8;
        }
        h1 {
        text-align: center;
        }

        ul {
        list-style-type: none; padding: 20px; margin: 0px;
        }
        li {
        background: #CCFF00; font-size: 30px; border: 2px solid #000; padding: 10px 20px; color: #000; cursor: cell;
        }


        li span {
        padding: 10px; cursor: cell;
        }

        .check {
        text-decoration: line-through; font-weight: bold; color:#000;

        }

        #enteritem {
        font-size: 30px;
        border: 2px solid #000;
        padding: 0; margin: 0;

    </style>

</head>
<body>
<p><h1>TODO LIST APP EXAMPLE WITH QBIT</h1></p>


<p><label for="enteritem"><h3>Enter your item: </h3></label>   <input type="text" id="enteritem" name="enteritem"/></p>

<!--<button id="add">NEW ITEM</button>-->

<ul id="todolist"></ul>

<p id="demo"></p>
<br>
<br>
<br>
<p>**To add an item type in the box and hit enter</p>
<p>**to delete an item double click it</p>
<p>**To check off an item, check the box</p>
<script>

function httpGet(theUrl)
    {
        var xmlHttp = null;

        xmlHttp = new XMLHttpRequest();
        xmlHttp.open( "GET", theUrl, false );
        xmlHttp.send( null );
        return xmlHttp.responseText;
    }

//submit text box
var enterItem = document.getElementById("enteritem");
enterItem.focus();

enterItem.onkeyup = function(event) {

// 13 represents the enter key.
    if (event.which == 13) {

    var itemContent = enterItem.value;

    if (!itemContent || itemContent == "" || itemContent == " " || itemContent == "  ") {
        return false;
    }
    addNewItem(document.getElementById("todolist"), itemContent);

    enterItem.focus();
    enterItem.select();
        }
}

function removeItem() {
        //var listId = this.id.replace("li_", "");
        document.getElementById(this.id).style.display = "none";
    }

function addNewItem(list, itemContent) {


    // Getiing the id from TodoService
    var json = httpGet("/services/todoservice/todoo");
    var date = JSON.parse(json);
    var id = date.time;


    var listItem = document.createElement("li");
    listItem.id = "li_" + id;
    var checkBox = document.createElement("input");
    checkBox.type = "checkBox";
    checkBox.id = "idc_" + id;


    var span = document.createElement("span");
    span.id = "ids_" + id;

    checkBox.onclick = itemStatus;

    span.innerText = itemContent;
    listItem.ondblclick = removeItem;

    listItem.appendChild(checkBox);
    listItem.appendChild(span);


    //listItem.textContent = itemContent;
    list.appendChild(listItem);

}

function itemStatus() {
    var checkboxId = this.id.replace("idc_", "");
    var itemContent = document.getElementById("ids_" + checkboxId);

    if (this.checked){

        itemContent.className = "check";

    }else {
         itemContent.className = "";

    }

}
</script>


</body>
</html>
```
This html file is made of html, css, and javasript. in order to deliver the todo app seen in the picture above.



#### TodoObject.java Listing

`src/main/java/io.advantageous.qbit.example.todolistapp/TodoObject.java`

```java
package io.advantageous.qbit.example.todolistapp;

/**
 * Created by rhightower on 2/10/15.
 */


public class TodoObject {
    private final long time = System.currentTimeMillis();

}
```
TodoObject is an object that holds the id number used for the items that will be listed, deleted, and status changed in the app.

####TodoService.java Listing

`src/main/java/io.advantageous.qbit.example.todolistapp/TodoService.java`

```java
package io.advantageous.qbit.example.todolistapp;

import io.advantageous.qbit.annotation.*;


/**
 * Created by rhightower on 2/10/15.
 */

@RequestMapping("/todoservice")
public class TodoService {


    @RequestMapping("/todoo")
    public TodoObject list() {
        return new TodoObject();
    }

}
```

In this app we will retrieve the id number mentioned above, by calling TodoService from the html file.

#### TodoRestServer.java Listing

`src/main/java/io.advantageous.qbit.example.todolistapp/TodoRestServer.java`

```java
package io.advantageous.qbit.example.todolistapp;


import io.advantageous.qbit.http.server.HttpServer;
import io.advantageous.qbit.server.ServiceServer;
import io.advantageous.qbit.system.QBitSystemManager;

import static io.advantageous.qbit.http.server.HttpServerBuilder.httpServerBuilder;
import static io.advantageous.qbit.server.ServiceServerBuilder.serviceServerBuilder;
import static org.boon.Boon.resource;

/**
 * Created by rhightower on 2/9/15.
 */
public class TodoRestServer {


    public static final String HTML_PAGE = "/ui/index.html";


    public static void main(String... args) {

        /* Create the system manager to manage the shutdown. */
        QBitSystemManager systemManager = new QBitSystemManager();

        HttpServer httpServer = httpServerBuilder()
                .setPort(9999).build();

        /* Register the Predicate using a Java 8 lambda expression. */
        httpServer.setShouldContinueHttpRequest(httpRequest -> {
            /* If not the page uri we want to then just continue by returning true. */
            if (!httpRequest.getUri().equals(HTML_PAGE)) {
                return true;
            }
            /* read the page from the file system or classpath. */
            final String todoWebPage = resource(HTML_PAGE);
            /* Send the HTML file out to the browser. */
            httpRequest.getResponse().response(200, "text/html", todoWebPage);
            return false;
        });


        /* Start the service. */
        final ServiceServer serviceServer = serviceServerBuilder().setSystemManager(systemManager)
                .setHttpServer(httpServer).build().initServices(new TodoService()).startServer();

        /* Wait for the service to shutdown. */
        systemManager.waitForShutdown();

    }

}
```

Once you run this you will be able to access the Todo List application at __http://localhost:9999/ui/index.html__

##Test The Service

Using your terminal `cd todo-list-app`
then `gradle clean build`
then `gradle run`
then open up your favorite browser and visit this address __http://localhost:9999/ui/index.html__, now you should have access to a working todo list applications that looks like this:

![todo list app](http://3.bp.blogspot.com/-TbWkqfGFb2g/VO0yCR6xtKI/AAAAAAAAAAg/D1IJAhivVg0/s1600/Screen%2BShot%2B2015-02-24%2Bat%2B6.22.02%2BPM.png)

##Summary

You have just built and tested a Todo List Application with QBit.
