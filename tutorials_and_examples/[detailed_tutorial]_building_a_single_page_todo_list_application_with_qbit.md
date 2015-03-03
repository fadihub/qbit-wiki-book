# [Detailed Tutorial] Building a single page; Todo List Application with QBit

##overview

QBit enables you to build applications hassle free, it can server up non-JSON resources. The QBit lib allows you to bind objects to HTTP ports so that they can be called by REST and WebSocket clients. This will be demonstrated in this example.


## Building a single page; Todo List Application with QBit
This wiki will walk you through the process of building a Todo List application with QBit.


## What you will build
You will build a Todo List application with QBit, you be able to access the application at the following address:

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
https://github.com/fadihub/todo-list-app.git
```

Once this is done you can __test the service__, let's first explain the process:

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
This `TodoObject` will only hold a variable `time` that holds the current time in milli seconds, as you will see later this variable will be used as a unique id element for the todo items listed in the app.


#### TodoService.java Listing

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

This `TodoService` will be called by the html file (as you will see later) to return a `TodoObject` that holds the time which is used as an id for the items that will be listed.

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



<ul id="todolist"></ul>


<br>
<br>
<br>
<p>**To add an item type in the box and hit enter</p>
<p>**to delete an item double click it</p>
<p>**To check off an item, check the box</p>
<script>

//This function will enable you to call TodoService
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

// 13 represents the ASCII dec number for the ENTER key.
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

//to delete a listed item from the todo list
function removeItem() {

        document.getElementById(this.id).style.display = "none";

    }

// function to add a todo item
function addNewItem(list, itemContent) {


    // Calls TodoService to retrieve an object then retrieves the id number for the item listed
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


    list.appendChild(listItem);


}

//changes the status of the item listed by drawing a line through it when the box is checked
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

The `index.html` holds some css to format the todo list app and make it somewhat pretty here is the css part:

```css
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
```

Also it holds some html to show the texts like the title of the app "TODO LIST APP EXAMPLE WITH QBIT",the text box is where you enter your item that you want to be listed, and the list is where the added items are listed. very simple really here is the html part:

```html
<p><h1>TODO LIST APP EXAMPLE WITH QBIT</h1></p>


<p><label for="enteritem"><h3>Enter your item: </h3></label>   <input type="text" id="enteritem" name="enteritem"/></p>



<ul id="todolist"></ul>


<br>
<br>
<br>
<p>**To add an item type in the box and hit enter</p>
<p>**to delete an item double click it</p>
<p>**To check off an item, check the box</p>
```
Also it holds some javascript this is where it calls `TodoServices` to get a `TodoObject` then it parses it into a JSON since the object holds the current time in milli seconds like we mentioned before, current time can be retrieved and stored in a variable `id`. Every todo item added by the user will have a unique `id` current time in milli seconds. This unique `id` will enable us to delete and change the status of a todo item.

To distinguish the parts we explained here they will be commented on the code. Here is the javascript part of the `index.html`:

```javascript
<script>

//This function will enable you to call TodoService
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

// 13 represents the ASCII dec number for the ENTER key.
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

//to delete a listed item from the todo list
function removeItem() {

        document.getElementById(this.id).style.display = "none";

    }

// function to add a todo item
function addNewItem(list, itemContent) {


    // Calls TodoService to retrieve an object then retrieves the id number for the item listed
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


    list.appendChild(listItem);


}

//changes the status of the item listed by drawing a line through it when the box is checked
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
```

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

Here we will show how to use a ServiceServer to host The REST and WebSocket and still deliver up a single page app. Instead of having the ServiceServerBuilder create an HttpServer, we will create one for it, and pass that to the builder. We create the `httpserver` and set its port to 9999, then the predicates are set using Lambda 8 java, With the Predicate callback, you can specify if you want the request to be handled by the consumer or not. This means all we have to do is create a Predicate that handles the request and then return false so the consumer which is wired into the ServiceServerBuilder is never called. Then we load the resource "/ui/index.html". And we deliver that up to the browser. All other requests we let go through.
At the end we just start the service and wait for the service to shutdown:
```java
  /* Start the service. */
        final ServiceServer serviceServer = serviceServerBuilder().setSystemManager(systemManager)
                .setHttpServer(httpServer).build().initServices(new TodoService()).startServer();

        /* Wait for the service to shutdown. */
        systemManager.waitForShutdown();
```
`systemManager` is an instance of `QBitSystemManager`
```java
/* Create the system manager to manage the shutdown. */
        QBitSystemManager systemManager = new QBitSystemManager();
```
So when you run `TodoRestServer` you will be able to access the application at __http://localhost:9999/ui/index.html__


##Test The Service

Using your terminal `cd todo-list-app`
then `gradle clean build`
then `gradle run`
then open up your favorite browser and visit this address __http://localhost:9999/ui/index.html__, now you should have access to a working todo list applications that looks like this:

![todo list app](http://3.bp.blogspot.com/-TbWkqfGFb2g/VO0yCR6xtKI/AAAAAAAAAAg/D1IJAhivVg0/s1600/Screen%2BShot%2B2015-02-24%2Bat%2B6.22.02%2BPM.png)

##Summary

You have just built and tested a Todo List Application with QBit, as you can see by going over this example we have demonstrated that QBit can server up non-JSON resources and that QBit lib allows you to bind objects to HTTP ports so that they can be called by REST and WebSocket clients.
