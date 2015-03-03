# [Rough Cut] QBit Java Microservice Lib Working With Workers Sharded and Pooled



This tutorial continues where the callbacks tutorial leaves off see [QBit Microservice Lib Working With CallBacks](https://github.com/advantageous/qbit/wiki/%5BRough-Cut%5D-QBit-Microservice-Lib-Working-With-CallBacks) section 1.22 before you tackle this one.


##Imagining an app - CPU Bound and IO Bound

Let's recall the app from the first example on callbacks.
Remember this app will be a recommendation engine.
Think of Cupid.com or DatesRUs.com or iTunes match or NetFlix suggestions or Amazon.com
"Customers who bought this also bought these other fine products".

Now we are not building a real recommendation engine although QBit
has been used for similar things.

The trick with an example is to keep the concepts clear enough without
getting too much clutter with a real world implementation so it can be followed.

`RecommendationService` is our CPU intensive service.
will be the recommendation engine.
We are going to shard CPU instances. Any user data would get pushed into its shard. We would replicate non-user data, and shard user data to live alongside the rules to operate on users.

It our hypothetical example
`RecommendationService` is very CPU intensive. Now we can run on X CPUs without duplicating all of the user data which is a lot of data. It has to iterate through products and user data
and pick a match. It is a classic N+1. In our example, we do all of this real time based on the
latest user activity, to the last second. What page did that just view? What product did they just bookmark?
What product did they buy? What product did their friends buy? What product are people in their same demographic
buying now. This is real time analytics. This does not mean that there is not machine learning or Hadoop batch
jobs churning data some where. But the churned data is mixed with the data science, pre-chewed caches and counters,
and up to the second user activity to make a decision based on data in memory and historic data (blended).
`RecommendationService` is the tip of the ice-burg, but it is brute force, exhaustive and fast.

`UserDataService` is our heavy IO service.
As much as we would like to, we do know which `RecommendationService` node a user might land on we can't if
we are truly elastic. Are they using an XBox on the west coast, an iPhone in Hawaii, when/how will they hit our recommendation
engine who knows.  `UserDataService` manages editing, backup, and syncing user data.
`UserDataService` keeps most users in-memory and also manages replicating and storing user data.
Once users get loaded into the system, we can keep them in-memory for a while. We will just sync any changes to the user back to the `UserDataService`. Since `UserDataService` is IO bound, we will create a pool of them.

This is what I want you to think about when I am talking about
`UserDataService` and `RecommendationService`.

`UserDataService` IO bound.
`RecommendationService` CPU bound.

In this tutorial we will cover Sharded Workers and Workers pools.


##Creating a pool of IO Workers


To create a pool of IO workers, we will use `io.advantageous.qbit.service.dispatchers.ServiceWorkers.workers` method to create a pooled `ServiceWorkers` instance.

Then we will create service queues that wrap our `UserDataService` and add those queues to our `userDataServiceWorkers`.

#### Creating a worker pool of services
```java

    private static UserDataServiceClient createUserDataServiceClientProxy(
            final ServiceBundle serviceBundle,
            final int numWorkers) {
        final ServiceWorkers userDataServiceWorkers = workers();

        for (int index =0; index < numWorkers; index++) {
            ServiceQueue userDataService = serviceBuilder()
                    .setServiceObject(new UserDataService())
                    .build();
            userDataService.startCallBackHandler();
            userDataServiceWorkers.addService(userDataService);
        }

        userDataServiceWorkers.start();



        serviceBundle.addServiceConsumer("workers", userDataServiceWorkers);

        return serviceBundle.createLocalProxy(UserDataServiceClient.class, "workers");
    }
```

Now we have a pool of IO workers. Every call to the `UserDataServiceClient` will go to a different service queue instance which will go to a different userDataService. The calls are round robin.


That takes care of our heavy IO. We can create just the right amount of workers which will talk to our backend database or key/value store. Next we need to create our CPU intensive service, our recommendation engine so that we can utilize all of our CPU cores when evaluating user data. Instead of copying user data to each shard, each shard will have a portion of the users.

This is very much like message driven beans except that you have more methods than just `onMessage`
and you get the benefits of a high-speed queue system.

##Creating a CPU intensive shards to maximize CPU intensive services and use all of your cores

In this example, we will use a canned shard rule which will shard on the hashcode of the first argument to each method. We would want that first argument to be something like userName or some other object that would give us a nice consistent hashCode to use to divvy up users so we can execute the CPU intensive rules right next to the actual user data that we have in memory. We use the method `io.advantageous.qbit.service.dispatchers.ServiceWorkers.shardOnFirstArgumentWorkers`, and there are many such methods on the `ServiceWorkers` class. You can also create your own `ShardRule`s and pass that to the
`ServiceWorkers.shardedWorkers` method.


Other than that the code looks pretty similar to what we did with the IO bound workers.
We pass the service queue client proxy `userDataServiceClient` from the last creation method as
an argument to this one so that this `recommendationService` can call `userDataServiceClient` as needed.


#### Creating a sharded set of services
```java

    private static RecommendationServiceClient createRecommendationServiceClientProxy(
            final ServiceBundle serviceBundle,
            final UserDataServiceClient userDataServiceClient,
            int numWorkers) {


        final ServiceWorkers recommendationShardedWorkers = shardOnFirstArgumentWorkers();

        for (int index = 0; index < numWorkers; index++) {
            RecommendationService recommendationServiceImpl =
                    new RecommendationService(userDataServiceClient);

            ServiceQueue serviceQueue = serviceBuilder()
                    .setServiceObject(recommendationServiceImpl)
                    .build();
            serviceQueue.startCallBackHandler();
            recommendationShardedWorkers.addService(serviceQueue);
        }

        recommendationShardedWorkers.start();

        serviceBundle.addServiceConsumer("recommendation", recommendationShardedWorkers);

        return serviceBundle.createLocalProxy(RecommendationServiceClient.class, "recommendation");
    }

```

Each time the service queue client proxy is called, i.e., `RecommendationServiceClient` it will select on the N `RecommendationService` service queues to handle the method call. If we could only handle 20,000 recommendation lists per second for users, then with 5 CPU cores, we can approach 100,000 recommendation lists per second.

##Putting it altogether


####The complete example with the changes for worker pools and sharded pools
```java
package io.advantageous.qbit.example;

import io.advantageous.qbit.QBit;
import io.advantageous.qbit.service.ServiceBundle;
import io.advantageous.qbit.service.ServiceQueue;
import io.advantageous.qbit.service.dispatchers.ServiceWorkers;
import org.boon.core.Sys;

import static io.advantageous.qbit.service.ServiceBundleBuilder.serviceBundleBuilder;
import static io.advantageous.qbit.service.ServiceProxyUtils.flushServiceProxy;

import java.util.List;

import static io.advantageous.qbit.service.ServiceBuilder.serviceBuilder;
import static io.advantageous.qbit.service.dispatchers.ServiceWorkers.shardOnFirstArgumentWorkers;
import static io.advantageous.qbit.service.dispatchers.ServiceWorkers.workers;
import static org.boon.Lists.list;

/**
 * Created by rhightower on 2/20/15.
 */
public class PrototypeMain {

    public static void main(String... args) {


        QBit.factory().systemEventManager();


        final ServiceBundle serviceBundle = serviceBundleBuilder()
                .setAddress("/root").build();


        serviceBundle.start();

        final UserDataServiceClient userDataServiceClient =
                createUserDataServiceClientProxy(serviceBundle, 8);


        final RecommendationServiceClient recommendationServiceClient =
                createRecommendationServiceClientProxy(serviceBundle,
                        userDataServiceClient, 4);




        List<String> userNames = list("Bob", "Joe", "Scott", "William");

        userNames.forEach( userName->
                recommendationServiceClient.recommend(recommendations -> {
                    System.out.println("Recommendations for: " + userName);
                    recommendations.forEach(recommendation->
                            System.out.println("\t" + recommendation));
                }, userName)
        );



        flushServiceProxy(recommendationServiceClient);
        Sys.sleep(1000);

    }

    private static RecommendationServiceClient createRecommendationServiceClientProxy(
            final ServiceBundle serviceBundle,
            final UserDataServiceClient userDataServiceClient,
            int numWorkers) {


        final ServiceWorkers recommendationShardedWorkers = shardOnFirstArgumentWorkers();

        for (int index = 0; index < numWorkers; index++) {
            RecommendationService recommendationServiceImpl =
                    new RecommendationService(userDataServiceClient);

            ServiceQueue serviceQueue = serviceBuilder()
                    .setServiceObject(recommendationServiceImpl)
                    .build();
            serviceQueue.startCallBackHandler();
            recommendationShardedWorkers.addService(serviceQueue);
        }

        recommendationShardedWorkers.start();

        serviceBundle.addServiceConsumer("recomendation", recommendationShardedWorkers);

        return serviceBundle.createLocalProxy(RecommendationServiceClient.class, "recomendation");
    }

    private static UserDataServiceClient createUserDataServiceClientProxy(
            final ServiceBundle serviceBundle,
            final int numWorkers) {
        final ServiceWorkers userDataServiceWorkers = workers();

        for (int index =0; index < numWorkers; index++) {
            ServiceQueue userDataService = serviceBuilder()
                    .setServiceObject(new UserDataService())
                    .build();
            userDataService.startCallBackHandler();
            userDataServiceWorkers.addService(userDataService);
        }

        userDataServiceWorkers.start();



        serviceBundle.addServiceConsumer("workers", userDataServiceWorkers);

        return serviceBundle.createLocalProxy(UserDataServiceClient.class, "workers");
    }
}

```
