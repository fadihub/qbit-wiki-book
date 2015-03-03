# [Rough Cut] QBit Microservice Lib Working With CallBacks


##CallBacks

To really grasp QBit, one must grasp the concepts of a CallBack.

A CallBack is a way to get an async response in QBit.

You call a service method and it calls you back.

There is really no magic. Ok there is some, but we will start magic
free and explain things as they come.

##Imagining an app - CPU Bound and IO Bound

For this example, we are going to imagine a larger app.
This app will be a recommendation engine.
Think of Cupid.com or DatesRUs.com or iTunes match or NetFlix suggestions or Amazon.com
"Customers who bought this also bought these other fine products".

Now we are not building a real recommendation engine although QBit
has been used for similar things.

The trick with an example is to keep the concepts clear enough without
getting too much clutter with a real world implementation so it can be followed.

`RecommendationService` is our CPU intensive service.
will be the recommendation engine. It our hypothetical example
`RecommendationService` is very CPU intensive. It has to iterate through products and user data
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

This is what I want you to think about when I am talking about
`UserDataService` and `RecommendationService`.

`UserDataService` IO bound.
`RecommendationService` CPU bound.

In this tutorial we will cover CallBacks. In a later tutorial we will cover sharding CPU bound services.
In another tutorial we will cover using Service pools to handle IO bound services.

QBit provides some magic to make some of this stuff easier, but unless you learn why/what/how things are being done
the magic will not help you. It will just confuse you. Thus, we will walk through it in baby steps.



##Starting out with a very simple Recommendation service

A recommendation service has an interface, where others can call methods, and an implementation.

To make this clear in this tutorial we will name things `XXXServiceClient` for the client interface
and we will name things `XXXService` for the implementation but this is in no way a recommendation for
naming but a device for clarity of thought for the tutorial. A more normative approach will clearly be
`XXXService` for the client and `XXXServiceImpl` for the implementation or whatever naming convention
that you prefer.

#### RecommendationService client interface
```java

public interface RecommendationServiceClient {


    void recommend(final Callback<List<Recommendation>> recommendationsCallback,
                          final String userName);
}

```

Client interfaces always return void. If you want to be called back, you specify a Callback as the first argument.

A callback is just a Java 8 consumer with an additional method `onError` to pass exceptions back to the client
code that invoked the service.

#### Callback

```java


public interface Callback <T> extends java.util.function.Consumer<T> {
    default void onError(java.lang.Throwable error) { /* compiled code */ }
}

```

A Java 8 Consumer class has an accept(<T>) method so a Callback adds an onError which uses the new
Java 8 syntax to create a default implementation that just logs the exception so if the exception
is truly exceptional, and all you plan on doing is logging it, then you don't have to do anything.


The client interface is your interface to calling the service. Calling methods on the client interface
enqueues those method calls onto the service queue for the service. You create a client interface
from a ServiceQueue as follows:



#### Creating a client proxy from a service queue

```java


        ServiceQueue recommendationServiceQueue = ...

        /** Create the client interface from the recommendationServiceQueue. */
        RecommendationServiceClient recommendationServiceClient =
                recommendationServiceQueue.createProxy(RecommendationServiceClient.class);


```

The `ServiceQueue` manages threads/queues for a `Service` implementation so the service can be thread
safe and fast.

The first version of the `RecommendationService`, we are going to keep really simple and some services can
always be this simple. At this point `RecommendationService` is just a POJO. Later we will stretch the definition
of POJO and annotated methods and use lambdas, and more, but for now, let's keep it simple.



#### Simple implementation of RecommendationService
```java

public class RecommendationService {


    private final SimpleCache<String, User> users =
            new SimpleCache<>(10_000);

    public List<Recommendation> recommend(final String userName) {
        User user = users.get(userName);
        if (user == null) {
            user = loadUser(userName);
        }
        return runRulesEngineAgainstUser(user);
    }

    private User loadUser(String userName) {
        ...
    }

    private List<Recommendation> runRulesEngineAgainstUser(final User user) {
        ...
    }

```

Remember this is an example so `loadUser` and `runRulesEngineAgainstUser` are stubbed with simple
implementations but pretend they are not.

Let's pretend `loadUser` has to look in a local cache, and if the user is not found, look in an off-heap cache
and if not found it must ask for the user from the UserService which must check its caches and perhaps fallback
to loading the user data from a database or from other services.

In other words, `loadUser` can potentially block on IO.

The way we have written this now, we are blocking on user load. This is bad for most apps, but worse for
Actor, Active Object and Reactive libs and frameworks. When you block, you are blocking the thread that
is handling all of the messages, events, method calls for this service (and if using a ServiceBundle
described later you are even blocking others service.)

Is this always bad? Well it is a big red flag. If you were getting a lot of cache hits and your traffic
was bursty but consistent and you cache did not time out often then this would not be the end of the world.
But for almost all applications of microservice this is a bad thing.

Before we fix it let's finish showing using our service client proxy.


#### Client using a callback to call service - Creating the callback
```java


        Callback<List<Recommendation>> callback = new Callback<List<Recommendation>>() {
            @Override
            public void accept(List<Recommendation> recommendations) {
                System.out.println("recommendations " + recommendations);
            }

            @Override
            public void onError(Throwable error) {
                error.printStackTrace();
            }
        };

```

The above uses the more verbose pre Java 8 style of inner classes. We create an
anonymous inner class which implements the Callback interface, and then we use this
to call the


#### Client using a callback to call service - Using the callback

```java

        recommendationServiceClient.recommend(callback, "Rick");

```


Now when we call the `recommendationServiceClient` if the callback succeeds we get a
an `accept` call, and if it fails we get an `onError` call. We call it on one thread, and
it calls us back on another thread when it finishes.


```output
recommendations
       [Recommendation{recommendation='Take a walk'},
        Recommendation{recommendation='Read a book'},
        Recommendation{recommendation='Love more, complain less'}]

```

Well we can't do an example using Java 8 and not use the lambda expression.


#### Client using a callback to call service - Using a lambda that creates the callback

```java

        recommendationServiceClient.recommend(recommendations -> {
            System.out.println("recommendations " + recommendations);
        }, "Rick");

```

Now since it is a one liner, we don't really need the brackets in the lambda.


#### Client using a callback to call service - lambda no brackets
```java

        recommendationServiceClient.recommend(recommendations ->
            System.out.println("recommendations " + recommendations), "Rick");

```


Getting jiggy with it:

#### Client using a callback to call a service - using a Java method expression just because.

```java

        import static java.lang.System.out;

        recommendationServiceClient.recommend(out::println, "Rick");

```

So what was the point of the last few examples. I am trying to show that QBit does in fact support Java 8
style lambda / method expression syntax, but with a warning lambda expressions are not always the
easiest way to write code that is easy to comprehend. I will often show the non-lambda approach because it
is easier to understand and explain. The other caveat, lambda nested calling other lambdas using no bodies,
 method expressions that then call other lambdas using no bodies and method expression that are deeply nested
 are not very readable.
Until Josh Bloch gets off his tush and writes a Java 8 lambda guy that Java programmers will put under their
pillow and worship like the missing star wars trilogy books, tread lightly with lambdas. Use them.

Don't deeply nest lambdas 8 levels. Just don't.
You can easily write code that no one including yourself understands.
Break things out into methods that describe what you are doing at least every three levels. :)

That said, let's get a bunch of user recommendations at once using Java 8 lambdas (do as I say not as I do).

#### Client using a callback to call a service - Lambdas gone wild
```java
        List<String> userNames = Lists.list("Bob", "Joe", "Scott", "William");

        userNames.forEach( userName->
                recommendationServiceClient.recommend(recommendations -> {
                    System.out.println("Recommendations for: " + userName);
                    recommendations.forEach(recommendation->
                            System.out.println("\t" + recommendation));
                }, userName)
        );

```

Now when we run our program we get:

```output

Recommendations for: Bob
	Recommendation{recommendation='Take a walk'}
	Recommendation{recommendation='Read a book'}
	Recommendation{recommendation='Love more, complain less'}
Recommendations for: Joe
	Recommendation{recommendation='Take a walk'}
	Recommendation{recommendation='Read a book'}
	Recommendation{recommendation='Love more, complain less'}
Recommendations for: Scott
	Recommendation{recommendation='Take a walk'}
	Recommendation{recommendation='Read a book'}
	Recommendation{recommendation='Love more, complain less'}
Recommendations for: William
	Recommendation{recommendation='Take a walk'}
	Recommendation{recommendation='Read a book'}
	Recommendation{recommendation='Love more, complain less'}


```



## QBit microserivce - micro batching

QBit tries to send many method calls at one time. Periodically, you need to tell QBit to flush what you have
done so far. Every time you tell it to flush, you are sending all of the methods you called from the last time
you flushed. There are ways to get QBit to auto-flush, which we will cover later.
Imagine a REST tier calling your service and you have 20 threads handling the REST tier and the REST tier is
handling 100K requests per second, and it is a rules engine that can only handle 200K recommendations a second
in a tight loop. Instead of each thread enqueue each call everytime it gets a request, it can send method
calls in batches of 10, 100, or 1,000 or even 10K requests per batch.
The bigger the batch the less time doing thread hand-off and less synchronizing queue internals across multiple threads.

When you use a QBit queue, you can specify a batch size.
When the queue gets requests beyond this amount, it will batch them. You can also periodically, flush the methods
when you feel its appropriate like after a related unit of work.
This is microbatching and QBit supports it at its core. Among other optimization that we will cover over time.

Therefore periodically (unless you setup auto flush or a batch size of 1), you need to flush your method call
queue.


#### Client using a callback to call a service - Don't forget to flush and put the lid down

```java

        List<String> userNames = Lists.list("Bob", "Joe", "Scott", "William");

        userNames.forEach( userName->
                recommendationServiceClient.recommend(recommendations -> {
                    System.out.println("Recommendations for: " + userName);
                    recommendations.forEach(recommendation->
                            System.out.println("\t" + recommendation));
                }, userName)
        );



        flushServiceProxy(recommendationServiceClient);

```


##Code so far - Simple client call back full example


#### Domain object
```java
package io.advantageous.qbit.example;

/* Domain object. */
public class User {

    private final String userName;

    public User(String userName){
        this.userName = userName;
    }

    public String getUserName() {
        return userName;
    }

}

```


#### RecommendationService
```java
package io.advantageous.qbit.example;


import org.boon.Lists;
import org.boon.cache.SimpleCache;
import java.util.List;

public class RecommendationService {


    private final SimpleCache<String, User> users =
            new SimpleCache<>(10_000);

    public List<Recommendation> recommend(final String userName) {
        System.out.println("recommend called");
        User user = users.get(userName);
        if (user == null) {
            user = loadUser(userName);
        }
        return runRulesEngineAgainstUser(user);
    }

    private User loadUser(String userName) {
        return new User("bob"); //stubbed out... next example will use UserService
    }

    private List<Recommendation> runRulesEngineAgainstUser(final User user) {

        return Lists.list(new Recommendation("Take a walk"), new Recommendation("Read a book"),
                new Recommendation("Love more, complain less"));
    }


}

```



#### Client interface
```java
package io.advantageous.qbit.example;

import io.advantageous.qbit.service.Callback;

import java.util.List;

/**
 * @author  rhightower
 * on 2/20/15.
 */
public interface RecommendationServiceClient {


    void recommend(final Callback<List<Recommendation>> recommendationsCallback,
                          final String userName);
}

```


#### Main method to run the service
```java

package io.advantageous.qbit.example;

import io.advantageous.qbit.QBit;
import io.advantageous.qbit.service.Callback;
import io.advantageous.qbit.service.ServiceProxyUtils;
import io.advantageous.qbit.service.ServiceQueue;
import org.boon.Lists;
import org.boon.core.Sys;

import static io.advantageous.qbit.service.ServiceProxyUtils.flushServiceProxy;
import static java.lang.System.out;

import java.util.List;

import static io.advantageous.qbit.service.ServiceBuilder.serviceBuilder;
import static org.boon.Lists.list;

/**
 * Created by rhightower on 2/20/15.
 */
public class PrototypeMain {

    public static void main(String... args) {


        /* Create the service. */
        RecommendationService recommendationServiceImpl =
                new RecommendationService();

        /* Wrap the service in a queue. */
        ServiceQueue recommendationServiceQueue = serviceBuilder()
                .setServiceObject(recommendationServiceImpl)
                .build().start().startCallBackHandler();

        /* Create a proxy interface for the service. */
        RecommendationServiceClient recommendationServiceClient =
                recommendationServiceQueue.createProxy(RecommendationServiceClient.class);


        /* Call the service with the proxy. */
        recommendationServiceClient.recommend(out::println, "Rick");

        /* Flush the call. This can be automated, but is a core concept. */
        flushServiceProxy(recommendationServiceClient);
        Sys.sleep(1000);



        /* Lambdas gone wild. */

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
}

```


#### build file
```

apply plugin: 'java'
apply plugin: 'application'


sourceCompatibility = 1.8
version = '1.0'

repositories {
    mavenLocal()
    mavenCentral()
}


dependencies {
    compile group: 'io.advantageous.qbit', name: 'qbit-jetty', version: '0.6.3'
    testCompile "junit:junit:4.11"
    testCompile "org.slf4j:slf4j-simple:[1.7,1.8)"
}

```


You are done. Stretch your legs.
Smoke 'em if you got them.
Get some coffee or tea.
Do some yoga. Pet your dog.
Ask your kids how their day went.
Get ready for Round 2.

## The first rule of Queue Club - don't block


Our client does not block, but our service does. Going back to our `RecommendationService`.
If we get a lot of cache hits for user loads, perhaps the
block will not be that long, but it will be there and every time we have to fault in a user, the whole system
is gummed up. What we want to be able to do is if we can't handle the recommendation request,
we go ahead and make an async call to the `UserDataService`. When that async callback comes back, then we
handle that request. In the mean time, we handle recommendation lists requests as quickly as we can.
We never block.

So let's revisit the service. The first thing we are going to do is make the service method take
a callback.

#### The first rule of queue club don't block.
#### The second rule of queue club if you are not ready, use a callback and continue handling stuff you are ready for

Let's implement the second rule of queue club.


#### Adding a CallBack to the RecommendationService inproc microservice

```java
public class RecommendationService {


    public void recommend(final Callback<List<Recommendation>> recommendationsCallback,
                          final String userName) {

```

Now we are taking a callback and we can decide when we want to handle this recommendation generation request.
We can do it right away if there user data we need is in-memory or we can delay it.


#### If the user is found, call the callback right away for RecommendationService inproc microservice

```java

    public void recommend(final Callback<List<Recommendation>> recommendationsCallback,
                          final String userName) {

        /** Look for user in user cache. */
        User user = users.get(userName);

        /** If the user not found, load the user from the user service. */
        if (user == null) {
             ...
        } else {
             /* Call the callback now because we can handle the callback now. */
            recommendationsCallback.accept(runRulesEngineAgainstUser(user));
        }

    }

```


Notice, if the user is found in the cache, we run our recommendation rules in-memory and call the callback right away
`recommendationsCallback.accept(runRulesEngineAgainstUser(user))`.

The interesting part is what do we do if don't have the user loaded.


#### If the user was not found, load him from the user microservice, but still don't block

```java


    public void recommend(final Callback<List<Recommendation>> recommendationsCallback,
                          final String userName) {


        /** Look for user in users cache. */
        User user = users.get(userName);

        /** If the user not found, load the user from the user service. */
        if (user == null) {

            /* Load user using Callback. */
            userDataService.loadUser(new Callback<User>() {
                @Override
                public void accept(final User loadedUser) {
                        handleLoadFromUserDataService(loadedUser,
                                recommendationsCallback);
                }
            }, userName);

        }
        ...

```

Here we use a CallBack to load the user, and when the user is loaded, we call `handleLoadFromUserDataService`
which adds some management about handling the callback so we can still handle this call, just not now.


We can replace the last example using a lambda expression as follows:

#### If the user was not found, load him from the user microservice, but still don't block, open lambda style

```java


    public void recommend(final Callback<List<Recommendation>> recommendationsCallback,
                          final String userName) {


        /** Look for user in users cache. */
        User user = users.get(userName);

        /** If the user not found, load the user from the user service. */
        if (user == null) {

            /* Load user using lambda expression. */
         userDataService.loadUser(
                    loadedUser -> {
                        handleLoadFromUserDataService(loadedUser,
                        recommendationsCallback);
                    }, userName);

        }
        ...

```

Using lambdas like this makes the code more readable and terse, but remember don't deeply next
lambda expressions or you will create a code maintenance nightmare. Use the judiciously.


##Doing something later

There is some internal plumbing to automate many things in this example, but to be honest.
If you don't know what we are doing underneath the covers, then you are in for a world of hurt.
We will show the hard way to handle callbacks and then show you the easy way later.

What we want is to handle the request for recommendations after the user service system loads
the user from its store.



#### Handling UserServiceData callback methods once we get them.

```java

public class RecommendationService {


    private final SimpleCache<String, User> users =
            new SimpleCache<>(10_000);

    private UserDataServiceClient userDataService;


    private BlockingQueue<Runnable> callbacks =
               new ArrayBlockingQueue<Runnable>(10_000);


    ...

    public void recommend(final Callback<List<Recommendation>> recommendationsCallback,
                          final String userName) {

        ...

    }

    /** Handle defered recommendations based on user loads. */
    private void handleLoadFromUserDataService(final User loadedUser,
                                               final Callback<List<Recommendation>> recommendationsCallback) {

        /** Add a runnable to the callbacks queue. */
        callbacks.add(new Runnable() {
            @Override
            public void run() {
              List<Recommendation> recommendations = runRulesEngineAgainstUser(loadedUser);
              recommendationsCallback.accept(recommendations);
            }
        });
    }



```


We can rewrite the above using lambda.


####  Handling UserServiceData callback methods once we get them using lambda


```java


public class RecommendationService {

...

    /** Handle defered recommendations based on user loads. */
    private void handleLoadFromUserDataService(final User loadedUser,
                                               final Callback<List<Recommendation>> recommendationsCallback) {

        /** Add a runnable to the callbacks list. */
        callbacks.add(() -> {
            List<Recommendation> recommendations = runRulesEngineAgainstUser(loadedUser);
            recommendationsCallback.accept(recommendations);
        });

    }

```


The important part there is that every time we get a callback call from `UserDataService`, we then
perform our CPU intensive recommendation rules and callback our caller. Well not exactly, what we
do is enqueue an runnable onto our callbacks queue, and later we will iterate through those but when?


##Handling callbacks when our receive queue is empty, a new batch started or we hit a batch limit

The `RecommendationService` can be notified when its queue is empty, it has started a new batch and when
it has reached a batch limit. These are all good times to handle callbacks from the `UserDataService`.

#### Draining our callback queue
```java

    @QueueCallback({
            QueueCallbackType.EMPTY,
            QueueCallbackType.START_BATCH,
            QueueCallbackType.LIMIT})
    private void handleCallbacks() {

        flushServiceProxy(userDataService);
        Runnable runnable = callbacks.poll();

        while (runnable != null) {
            runnable.run();
            runnable = callbacks.poll();
        }
    }

```

It is important to remember when handling callbacks from another microservice that you want to handle
callbacks from the other service before you handle more incoming requests from you clients.
Essentially you have clients that have been waiting (async waiting but still), and these clients
might represent an open TCP/IP connection like an HTTP call so it is best to close them out
before handling more requests and like we said they were already waiting around with an open connection
for users to load form the user service.


#### Full code listing for part 2

#### Domain object

```java

package io.advantageous.qbit.example;

/* Domain object. */
public class User {

    private final String userName;

    public User(String userName){
            this.userName = userName;
    }

    public String getUserName() {
        return userName;
    }
}



```


#### User Data Service Client Proxy for UserDataService Microservice

```java

package io.advantageous.qbit.example;

import io.advantageous.qbit.service.Callback;

public interface UserDataServiceClient {

    void loadUser(Callback<User> callBack, String userId);
}


```


#### User Data Service microservice

```java

package io.advantageous.qbit.example;


import io.advantageous.qbit.annotation.QueueCallback;
import io.advantageous.qbit.annotation.QueueCallbackType;
import io.advantageous.qbit.service.Callback;
import org.boon.core.Sys;

import java.util.ArrayList;
import java.util.List;

import static org.boon.Boon.puts;


public class UserDataService {


    private final List<Runnable> userLoadCallBacks = new ArrayList<>(1_000);

    public void loadUser(final Callback<User> callBack, final String userId) {

        puts("UserDataService :: loadUser called", userId);
        userLoadCallBacks.add(
                new Runnable() {
                    @Override
                    public void run() {
                        callBack.accept(new User(userId));
                    }
                });

    }


    @QueueCallback({QueueCallbackType.EMPTY, QueueCallbackType.LIMIT})
    public void pretendToDoIO() {
        Sys.sleep(100);

        if (userLoadCallBacks.size()==0) {
            return;
        }
        for (Runnable runnable : userLoadCallBacks) {
            runnable.run();
        }
        userLoadCallBacks.clear();

    }




}

```


#### Recommendation Service microservice Java listing

```java
package io.advantageous.qbit.example;


import io.advantageous.qbit.annotation.QueueCallback;
import io.advantageous.qbit.annotation.QueueCallbackType;
import io.advantageous.qbit.service.Callback;
import org.boon.Lists;
import org.boon.cache.SimpleCache;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

import static io.advantageous.qbit.service.ServiceProxyUtils.flushServiceProxy;

public class RecommendationService {


    private final SimpleCache<String, User> users =
            new SimpleCache<>(10_000);


    private BlockingQueue<Runnable> callbacks = new ArrayBlockingQueue<Runnable>(10_000);
    private UserDataServiceClient userDataService;


    public RecommendationService(UserDataServiceClient userDataService) {
        this.userDataService = userDataService;
    }


    public void recommend(final Callback<List<Recommendation>> recommendationsCallback,
                          final String userName) {


        System.out.println("recommend called");

        User user = users.get(userName);

        if (user == null) {
            userDataService.loadUser(
                    loadedUser -> {
                        handleLoadFromUserDataService(loadedUser, recommendationsCallback);
                     }, userName);
        } else {
            recommendationsCallback.accept(runRulesEngineAgainstUser(user));
        }

    }

    /**
     * Handle defered recommendations based on user loads.
     */
    private void handleLoadFromUserDataService(final User loadedUser,
                                               final Callback<List<Recommendation>> recommendationsCallback) {

        /** Add a runnable to the callbacks list. */
        callbacks.add(() -> {
            List<Recommendation> recommendations = runRulesEngineAgainstUser(loadedUser);
            recommendationsCallback.accept(recommendations);
        });
    }


    @QueueCallback({
            QueueCallbackType.EMPTY,
            QueueCallbackType.START_BATCH,
            QueueCallbackType.LIMIT})
    private void handleCallbacks() {

        flushServiceProxy(userDataService);
        Runnable runnable = callbacks.poll();

        while (runnable != null) {
            runnable.run();
            runnable = callbacks.poll();
        }
    }

    /* Fake CPU intensive operation. */
    private List<Recommendation> runRulesEngineAgainstUser(final User user) {
        return Lists.list(new Recommendation("Take a walk"), new Recommendation("Read a book"),
                new Recommendation("Love more, complain less"));
    }

}
```


#### Running part two
```java

package io.advantageous.qbit.example;

import io.advantageous.qbit.QBit;
import io.advantageous.qbit.service.ServiceQueue;
import org.boon.core.Sys;

import static io.advantageous.qbit.service.ServiceProxyUtils.flushServiceProxy;

import java.util.List;

import static io.advantageous.qbit.service.ServiceBuilder.serviceBuilder;
import static org.boon.Lists.list;

/**
 * Created by rhightower on 2/20/15.
 */
public class PrototypeMain {

    public static void main(String... args) {



        /* Create user data service and client proxy. */
        ServiceQueue userDataService = serviceBuilder()
                                    .setServiceObject(new UserDataService())
                                    .build().start();
        userDataService.startCallBackHandler();
        UserDataServiceClient userDataServiceClient = userDataService
                                .createProxy(UserDataServiceClient.class);



        /* Create recommendation service and client proxy. */
        RecommendationService recommendationServiceImpl =
                new RecommendationService(userDataServiceClient);
        ServiceQueue recommendationServiceQueue = serviceBuilder()
                .setServiceObject(recommendationServiceImpl)
                .build().start().startCallBackHandler();

        RecommendationServiceClient recommendationServiceClient =
                recommendationServiceQueue.createProxy(RecommendationServiceClient.class);


        /* Use recommendationServiceClient for 4 recommendations for
          Bob, Joe, Scott and William. */
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
}

```
