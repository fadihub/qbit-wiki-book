# [Rough Cut] Using QBit microservice lib's REST support with URI Params

QBit supports JSON, HTTP, REST and WebSocket. This examples shows how to access a service via an HTTP client API, a high-speed WebSocket proxy and curl. The REST part of the example shows how to use URI params in your service.

As part of its REST support, it supports arguments to service methods which are URI params.
You can also use request params.

Let's demonstrated this.

First a service class.

```java

    @RequestMapping("/adder-service")
    public class AdderService {


        @RequestMapping("/add/{0}/{1}")
        public int add(@PathVariable int a, @PathVariable int b) {

            return a + b;
        }
    }

```

QBit uses the same style annotations as Spring MVC. We figure these are the ones that most people are familiar with. Since QBit focuses just on Microservice, it just supports JSON. No XML. No SOAP. No way!



## WebSocket
You can always invoke QBit services via a WebSocket proxy.
The advantage of a WebSocket proxy is it allows you execute 1M RPC+ a second (1 million remote calls every second).


#### Using a microservice remotely with WebSocket
```java
       /* Start QBit client for WebSocket calls. */
        final Client client = clientBuilder()
                   .setPort(7000).setRequestBatchSize(1).build();


       /* Create a proxy to the service. */
        final AdderServiceClientInterface adderService =
                client.createProxy(AdderServiceClientInterface.class,
                "adder-service");

        client.start();



       /* Call the service */
        adderService.add(System.out::println, 1, 2);

```

The output is 3.

```output
3
```


The above uses a WebSocket proxy interface to call the service async.

```java

    interface AdderServiceClientInterface {

        void add(Callback<Integer> callback, int a, int b);
    }
```

## REST call with URI params

The last example uses WebSocket. You could also just use REST.
REST is nice but it is going to be slower than WebSocket support.

QBit ships with a nice little HTTP client.
We can use it.

You can use it to send async calls and websocket messages with the HTTP client.


Here we will use the http client to invoke our remote method:

#### Using a microservice remotely with WebSocket
```java


        HttpClient httpClient = httpClientBuilder()
                .setHost("localhost")
                .setPort(7000).build();

        httpClient.start();
        String results = httpClient
                   .get("/services/adder-service/add/2/2").body();
        System.out.println(results);

```


The output is 4.

```output
4
```

## Accessing our microservice from curl

You can also access the service from curl.

```bash
$ curl http://localhost:7000/services/adder-service/add/2/2
```

##Full example

```java
/*
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
 *
 * QBit - The Microservice lib for Java : JSON, WebSocket, REST. Be The Web!
 */

package io.advantageous.qbit.example.servers;

import io.advantageous.qbit.annotation.PathVariable;
import io.advantageous.qbit.annotation.RequestMapping;
import io.advantageous.qbit.client.Client;
import io.advantageous.qbit.http.client.HttpClient;
import io.advantageous.qbit.http.request.HttpResponse;
import io.advantageous.qbit.server.ServiceServer;
import io.advantageous.qbit.service.Callback;
import io.advantageous.qbit.system.QBitSystemManager;
import org.boon.core.Sys;

import static io.advantageous.qbit.client.ClientBuilder.clientBuilder;
import static io.advantageous.qbit.http.client.HttpClientBuilder.httpClientBuilder;
import static io.advantageous.qbit.server.ServiceServerBuilder.serviceServerBuilder;
import static io.advantageous.qbit.service.ServiceProxyUtils.flushServiceProxy;

/**
 * @author rhightower
 *         on 2/17/15.
 */
public class SimpleRestServerWithURIParamsMain {


    public static void main(String... args) throws Exception {

        QBitSystemManager systemManager = new QBitSystemManager();

       /* Start Service server. */
        final ServiceServer server = serviceServerBuilder()
                .setSystemManager(systemManager)
                .setPort(7000).build();

        server.initServices(new AdderService());
        server.start();

       /* Start QBit client for WebSocket calls. */
        final Client client = clientBuilder().setPort(7000).setRequestBatchSize(1).build();


       /* Create a proxy to the service. */
        final AdderServiceClientInterface adderService =
                client.createProxy(AdderServiceClientInterface.class, "adder-service");

        client.start();



       /* Call the service */
        adderService.add(System.out::println, 1, 2);



        HttpClient httpClient = httpClientBuilder()
                .setHost("localhost")
                .setPort(7000).build();

        httpClient.start();
        String results = httpClient
                .get("/services/adder-service/add/2/2")
                .body();
        System.out.println(results);


        Sys.sleep(100);

        client.stop();
        systemManager.shutDown();


    }

    interface AdderServiceClientInterface {

        void add(Callback<Integer> callback, int a, int b);
    }

    @RequestMapping("/adder-service")
    public static class AdderService {


        @RequestMapping("/add/{0}/{1}")
        public int add(@PathVariable int a, @PathVariable int b) {

            return a + b;
        }
    }


}


```
