# [Quick Start] Building a simple Rest web microservice server with QBit

##overview
WHAT IS REST? REST stands for Representational State Transfer. It relies on a stateless, client-server, cacheable communications protocol. In virtually all cases, the HTTP protocol is used.
Restful applications use HTTP requests to post data (create and/or update), read data, and delete data. Thus, REST uses HTTP for all four CRUD (Create/Read/Update/Delete) operations.
The World Wide Web itself is based on HTTP, can be viewed as a REST-based architecture.

## Building a simple Rest web service server with QBit

This wiki will walk you through the process for building a simple Rest web server with QBit.



## What you will build
You will build a service that will accept HTTP GET/POST (CREATE/READ) requests at:

```bash
curl http://localhost:6060/services/myservice/ping
```
when a ping is sent by a client, a response of a greeting and a confirmation that the server is working will be posted; the JSON greeting representation is as follows:
```javascript
["Hello; My REST server is working"]
```


## How to complete this guide
In order to complete this example successfully you will need the following installed on your machine:

- Gradle; if you need help installing it, visit [Installing Gradle. ](http://www.gradle.org/docs/current/userguide/installation.html)
- Your favorite IDE or text editor (we recommend [Intellig IDEA ](https://www.jetbrains.com/idea/) latest version).
- [JDK ](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) 1.8 or later.
- Build and install QBit on your machine click [Building QBit ](https://github.com/advantageous/qbit/wiki/%5BQuick-Start%5D-Building-QBit-the-microservice-lib-for-Java) for instrutions.

Now that your machine is all ready let's get started:
- [Download ](https://github.com/fadihub/simple-rest-server/archive/master.zip) and unzip the source repository for this guide, or clone it using Git:

```bash
git clone https://github.com/fadihub/simple-rest-server.git
```

Once this is done you can __test the service__, let's first explain the process:

This service will handle GET/POST requests. Once the client pings the server, a greeting  and a confirmation message will be posted at:
```bash
curl http://localhost:6060/services/myservice/ping
```
The content of the message is the following:

```javascript
["Hello: This REST server is working!"]
```
####SimpleRestServer Listing

`src/main/java/io.advantageous.qbit.examples/SimpleRestServer`

```java
package io.advantageous.qbit.examples;

import io.advantageous.qbit.annotation.RequestMapping;
import io.advantageous.qbit.queue.QueueBuilder;
import io.advantageous.qbit.server.ServiceServer;
import io.advantageous.qbit.server.ServiceServerBuilder;
import org.boon.Boon;

import java.util.Collections;
import java.util.List;

/**
 *

 $ ./wrk -c 200 -d 10s http://localhost:6060/services/myservice/ping -H "X_USER_ID: RICK"  --timeout 100000s -t 8
 Running 10s test @ http://localhost:6060/services/myservice/ping
 8 threads and 200 connections
 Thread Stats   Avg      Stdev     Max   +/- Stdev
 Latency     2.63ms  360.02us   4.41ms   68.14%
 Req/Sec    10.17k     1.32k   12.89k    55.97%
 766450 requests in 10.00s, 76.75MB read
 Requests/sec:  76651.84
 Transfer/sec:      7.68MB

 */
public class SimpleRestServer {


    public static class MyService {

        /*
        curl http://localhost:6060/services/myservice/ping -H "X_USER_ID: RICK"
         */
        @RequestMapping
        public List ping() {
            return Collections.singletonList("Hello: This REST server is working!");
        }
    }

    public static void main(String... args) throws Exception {


        final ServiceServer serviceServer = new ServiceServerBuilder().setPort(6060).setQueueBuilder(
                new QueueBuilder().setLinkTransferQueue().setTryTransfer(true).setBatchSize(10).setPollWait(10)
        ).setNumberOfOutstandingRequests(1000000).setTimeoutSeconds(40)
                .build();

        serviceServer.initServices(new MyService());
        serviceServer.start();

        Boon.gets();
    }
}
```

 As you can see the code for this simple rest server is very straight forward, we are just creating the server and setting the port to 6060, and setting a few other parameters like PollWait etc.. will talk about these in later tutorials.

####build.gradle Listing

`~/simple-rest-server/build.gradle`

```java
group = 'io.advantageous.qbit.examples'

apply plugin: 'idea'
apply plugin: 'java'
apply plugin: 'maven'
apply plugin: 'application'

version = '0.1-SNAPSHOT'


sourceCompatibility = JavaVersion.VERSION_1_8
targetCompatibility = JavaVersion.VERSION_1_8


sourceSets {
    main {
        java {
            srcDir 'src/main/java'
        }
        resources {
            srcDir 'src/main/resources'
        }
    }
}


mainClassName = "io.advantageous.qbit.examples.SimpleRestServer"

repositories {
    mavenLocal()
    mavenCentral()
}



dependencies {
    compile group: 'io.advantageous.qbit', name: 'qbit-vertx', version: '0.5.2-SNAPSHOT'
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

This is where all the dependencies are set up for the __simple-rest-server__ code.

Now we are ready to test the localhost server.

##Test The Service

With your terminal cd into `~/simple-rest-sever`

run `gradle clean build` then `gradle run` open up your favorite browser then visit __http://localhost:6060/services/myservice/ping__ or open up another terminal and type the following:

```bash
curl http://localhost:6060/services/myservice/ping
```
you should get this response:

```javascript
["Hello: This REST server is working!"]
```

##Summary
You have just built a simple REST server with QBit and tested it.

