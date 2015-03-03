# [Rough Cut] QBit Microservices using Service Workers and sharded service workers

This covers using QBit microservice lib to create worker services and sharded worker services to maximize IO and CPU usage.

Some services can be single thread services through an async interface of course. Some services are CPU intensive ones and you tend to want to shard them to utilize all of your CPU cores. Some services need to talk to other services on the other side of a bus (Kafka, WebSocket) or REST, or perhaps talk to a database, these IO services tend to want to be pooled. QBit supports all of these models and in fact provides an interface so you can decide how to dispatch to a service.

Previous sections covered in-proc QBit services, event bus, etc. This area of the documentation will focus on pooled workers and sharded workers.

QBit has the following helper methods to create worker pools and sharded workers.


```java
public class ServiceWorkers {

    public static RoundRobinServiceDispatcher workers() {...

    public static ShardedMethodDispatcher shardedWorkers(final ShardRule shardRule) {...
```

You can compose sharded workers (for in-memory, thread safe, CPU intensive services), or workers for IO or talking to foreign services or foreign buses.

## Worker pool

Here is an example that uses a worker pool with three service workers in it:

Let's say you have a service that does something:

```java

    //Your POJO
    public  class MultiWorker {

        void doSomeWork(...) {
           ...
        }

    }

```

Now this does some sort of IO and you want to have a bank of these running not just one so you can do IO in parallel. After some performance testing, you found out that three is the magic number.

You want to use your API for accessing this service:

```java
    public  interface MultiWorkerClient {
        void doSomeWork(...);
    }

```

Now let's create a bank of these and use it.

First create the QBit services which add the thread/queue/micro-batch.

```java

        /* Create a service builder. */
        final ServiceBuilder serviceBuilder = serviceBuilder();

        /* Create some qbit services. */
        final ServiceQueue service1 = serviceBuilder
                .setServiceObject(new MultiWorker()).build();
        final ServiceQueue service2 = serviceBuilder
                .setServiceObject(new MultiWorker()).build();
        final ServiceQueue service3 = serviceBuilder
               .setServiceObject(new MultiWorker()).build();
```

Now add them to a ServiceWorkers object which sets up the worker pool dispatching.

```java

        ServiceWorkers dispatcher;
        dispatcher = workers(); //Create a round robin service dispatcher
        dispatcher.addServices(service1, service2, service3);
        dispatcher.start(); // start up the workers

```

You can add services, POJOs (which become services) and method consumers, method dispatchers to a service bundle. This allows you to compose the flow through of your services and how they talk to each other.

The service bundle is an integration point into QBit.

Let's add our new Service workers. ServiceWorkers is a ServiceMethodDispatcher which is a Consumer<MethodCall> of method calls. I love Java 8.

```Java
        /* Add the dispatcher to a service bundle. */
        bundle = serviceBundleBuilder().setAddress("/root").build();
        bundle.addServiceConsumer("/workers", dispatcher);
        bundle.start();
```

We are probably going to add a helper method to the service bundle so most of this can happen in a single call. But this is the way to do it now.

Now you can start using your workers.

```java

        /* Start using the workers. */
        final MultiWorkerClient worker =
              bundle.createLocalProxy(MultiWorkerClient.class, "/workers");

```

Now you could instead use Spring or Guice to configure the builders and the service bundle.
But you can just do it like the above which is good for testing and understanding QBit internals.

## Sharded Workers
QBit also supports the concept of sharded services which is good for sharding resources like CPU (run a rules engine on each CPU core for a user recommendation engine).

QBit does not know how to shard your services, you have to give it a hint.
You do this through a shard rule.

```java
public interface ShardRule {
    int shard(String methodName, Object[] args, int numWorkers);
}
```
We worked on an app where the first argument to the services was the username, and then we used that to shard calls to a CPU intensive in-memory rules engine. This technique works and allowed us to scale like mad.
:)

The ServiceWorkers class has a method for creating a sharded worker pool.

```java

    public static ShardedMethodDispatcher shardedWorkers(final ShardRule shardRule) {
        ...
    }

```

To use you just pass a shard key when you create the service workers.

```java


        dispatcher = shardedWorkers((methodName, methodArgs, numWorkers) -> {
            String userName = methodArgs[0].toString();
            int shardKey =  userName.hashCode() % numWorkers;
            return shardKey;
        });

```

Then add your services to the ServiceWorkers composition.
```java
        int workerCount = Runtime.getRuntime().availableProcessors();

        for (int index = 0; index < workerCount; index++) {
            final ServiceQueue service = serviceBuilder
                    .setServiceObject(new ContentRulesEngine()).build();
            dispatcher.addServices(service);

        }
```

Then add it to the service bundle as before.
```java

        dispatcher.start();

        bundle = serviceBundleBuilder().setAddress("/root").build();

        bundle.addServiceConsumer("/workers", dispatcher);
        bundle.start();
```

Then just use it:

```java
        final MultiWorkerClient worker = bundle.createLocalProxy(MultiWorkerClient.class, "/workers");

        for (int index = 0; index < 100; index++) {
            String userName = "rickhigh" + index;
            worker.pickSuggestions(userName);
        }

```


## Built in shard rules

We have some built in shard rules for you to use.


```java


public class ServiceWorkers {
...
    public static ShardedMethodDispatcher shardOnFirstArgumentWorkers() {
       ...
    }


    public static ShardedMethodDispatcher shardOnSecondArgumentWorkers() {
        ...
    }


    public static ShardedMethodDispatcher shardOnThirdArgumentWorkers() {
        ...
    }


    public static ShardedMethodDispatcher shardOnFourthArgumentWorkers() {
         ...
    }


    public static ShardedMethodDispatcher shardOnFifthArgumentWorkers() {
         ...
    }


    public static ShardedMethodDispatcher shardOnBeanPath(final String beanPath) {
        ...
    }

```

The shardOnBeanPath allows you to create a complex bean path navigation call.

```java

     /* shard on 2nd arg which is an employee
       Use the employees department's id property. */
     dispatcher = shardOnBeanPath("[1].department.id");

     /* Same as above. */
     dispatcher = shardOnBeanPath("1/department/id");

```


