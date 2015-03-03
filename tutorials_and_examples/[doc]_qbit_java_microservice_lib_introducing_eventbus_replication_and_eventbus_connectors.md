# [Doc] QBit Java microservice lib introducing EventBus replication and EventBus connectors

QBit has an event bus.

It is useful for services to listen to events.
Events are treated like method calls and that they can be handled by the same thread that handles the service method calls so modification of the state of the service is handled in one thread negating the need for complex thread management and synchronization.

## Remote events and event connectors
QBit now has the ability to send events over a remote connection.

Every event manager `EventManager` talks to an `EventBus` which talks to a series of `Channels`. `Channels` are like topics and they manage a collection of listeners. There are adapters that can move an event from a `Channel` to a `ServiceQueue`.

To support integration with other event busses and remote event busses we added the concept of an `EventConnector`. The `EventBus` sends all events to its `EventConnector`. The default `EventConnector` is a no-op `EventConnector`. You can plug in additional event connection with the `EventConnector`.

The `EventConnector` interface is a simple one.


```java
package io.advantageous.qbit.events.spi;

import io.advantageous.qbit.service.ServiceFlushable;

public interface EventConnector extends ServiceFlushable{


    /** Forwards the event.
     * @param event   event
     */
    void forwardEvent(EventTransferObject<Object> event);

    default void flush() {

    }
}

```

##EventManagerBuilder

Since constructing an event manager is more complicated now, we added an EventManager builder which allows you to specify Predicates which then determine if an event should be replicated by the event connector.

####Using Event manager builder

```java



        EventManagerBuilder eventManagerBuilder = eventManagerBuilder();

        eventManagerBuilder.setEventConnector(eventConnector);

        eventManagerBuilder.addEventConnectorPredicate(objectEvent -> apples.isColor("Red") );

        eventManagerBuilder.addEventConnectorPredicate(objectEvent -> day.is("Tuesday") );

        EventManager build = eventManagerBuilder.build();
```

We also added another SPI class for events that implements the Event interface.

####Event Transfer Object for sending events

```java


public class EventTransferObject<T> implements Event<T> {

```

This is a concrete class that gets passed for connecting events.

We also added a utility class for connecting event managers easily.


####SimpleEventConnector for connecting EventManagers.

```java
public class SimpleEventConnector implements EventConnector {

    private final EventManager eventManager;

    public SimpleEventConnector(EventManager eventManager) {
        this.eventManager = eventManager;
    }

    @Override
    public void forwardEvent(EventTransferObject<Object> event) {

        this.eventManager.forwardEvent(event);

    }

    @Override
    public void flush() {

        flushServiceProxy(eventManager);
    }
}

```

One can imagine using the predicate of the builder and SimpleEventConnector to channel events to certain event managers.


##Remote Event Bus replication and connection

None of this is required for the simple cases.

There is now a subproject that provides remote event replication support: `qbit-eventbus-replicator`.

You can use this to replicate events:

Let's say that you have two event busses:


####EventManager A
```java

        /** Two event managers A and B. Event on A gets replicated to B. */

        EventManager eventManagerA;
```


####EventManager B who will listen remotely to A
```java

        EventManager eventManagerB;
```

You want events that happen on A to be transmitted to B. Now A and B are on two different server instances on two different JVMs.


Let's setup our EventB service bus.


####EventManager B Start up
```java

        EventManagerBuilder eventManagerBuilderB = new EventManagerBuilder();
        /** Build B. */
        EventManager eventManagerBImpl = eventManagerBuilderB.build();

```

Create a service bundle and add event manager B to it.

####EventManager B Start up 2
```java
        serviceBundle = serviceBundleBuilder().build(); //build service bundle
        serviceBundle.addServiceObject("eventManagerB", eventManagerBImpl);

```

Create a service queue client proxy to B and register for events.


####EventManager B create service queue proxy and listen for an event foo.bar
```java
        eventManagerB = serviceBundle.createLocalProxy(EventManager.class, "eventManagerB");
        eventManagerB.register("foo.bar", event ->  body.set(event.body()));

```

Now we want B to listen to calls from a remote event bus. To do this we use a `EventRemoteReplicatorService`
which is a service that is easy to expose via WebSockets/REST for handing remote event calls.

To make `EventRemoteReplicatorService` easy to use, we added a `EventBusRemoteReplicatorBuilder`.



####Using EventRemoteReplicatorService via EventBusRemoteReplicatorBuilder
```java

        EventBusRemoteReplicatorBuilder replicatorBuilder = eventBusRemoteReplicatorBuilder();
        replicatorBuilder.serviceServerBuilder().setPort(9097);
        replicatorBuilder.setEventManager(eventManagerB);
        ServiceServer serviceServer = replicatorBuilder.build();
```

You can pass `EventBusRemoteReplicatorBuilder` a serviceServerBuilder or it will construct one.
Notice above we call


####Telling EventRemoteReplicatorService which event bus to replicate to
```java
        replicatorBuilder.setEventManager(eventManagerB);

```

This connects the `eventManagerB` to the `EventRemoteReplicatorService` which is exposed via WebSockets to handle remote calls.

`EventRemoteReplicatorService` is a `EventConnector`. This means when we start up `eventManagerA` on the other server, all we have to do is pass the proxy client to `EventRemoteReplicatorService` into `eventManagerA` and all the events from A will be replicated to `eventManagerB`.


To connect A to B, we need a WebSocket QBit proxy client to the replication running in the B server. To do this we provided a utility builder class called `EventBusReplicationClientBuilder` as follows:


####Creating an EventConnector that talks to the EventRemoteReplicatorService running on B
```java
        EventConnector replicatorClient;
        ...

        EventBusReplicationClientBuilder clientReplicatorBuilder = eventBusReplicationClientBuilder();
        clientReplicatorBuilder.clientBuilder().setPort(9097);
        Client client = clientReplicatorBuilder.build();

        replicatorClient = clientReplicatorBuilder.build(client);
```

Calls to the `replicatorClient` get sent to the remote replicator running in server B.

Now we just wire the A event bus to call the replicatorClient.


####Wiring up event bus A

```java

        /* Create A that connects to the replicator client. */
        EventManager eventManagerAImpl =
              eventManagerBuilderA.setEventConnector(replicatorClient).build();
        serviceBundleA = serviceBundleBuilder().build(); //build service bundle
         /* Wire in the bus. */
        serviceBundleA.addServiceObject("eventManagerA", eventManagerAImpl);
        /* Create a proxy client so we can use it over the service queue. */
        eventManagerA = serviceBundleA.createLocalProxy(EventManager.class, "eventManagerA");

```

Now we can listen on B running on another machine:


####Listening on B
```java

        eventManagerB.register("foo.bar", event ->  handleEvent(event.body()));
```

And when events get sent to A, they are also sent to B.


####Sending on A
```java


        eventManagerA.send("foo.bar", "hello");
```

We just created a listener running on a remote server.


Here is a complete example:

```java
package io.advantageous.qbit.events.impl;

import io.advantageous.qbit.client.Client;
import io.advantageous.qbit.events.EventManager;
import io.advantageous.qbit.events.EventManagerBuilder;
import io.advantageous.qbit.events.spi.EventConnector;
import io.advantageous.qbit.server.ServiceServer;
import io.advantageous.qbit.service.ServiceBundle;
import io.advantageous.qbit.service.ServiceProxyUtils;
import io.advantageous.qbit.test.TimedTesting;
import org.boon.core.Sys;
import org.junit.Test;

import java.util.concurrent.atomic.AtomicReference;

import static io.advantageous.qbit.events.impl.EventBusRemoteReplicatorBuilder.eventBusRemoteReplicatorBuilder;
import static io.advantageous.qbit.events.impl.EventBusReplicationClientBuilder.eventBusReplicationClientBuilder;
import static io.advantageous.qbit.service.ServiceBundleBuilder.serviceBundleBuilder;
import static org.junit.Assert.assertEquals;

/**
 * @author rhightower
 */
public class EventManagerReplicationOverWebSocket extends TimedTesting {

    EventConnector replicatorClient;
    ServiceBundle serviceBundleB;
    ServiceBundle serviceBundleA;


    @Test
    public void test() {


        /** Two event managers A and B. Event on A gets replicated to B. */

        EventManager eventManagerA;
        EventManager eventManagerB;

        EventManagerBuilder eventManagerBuilderA = new EventManagerBuilder();
        EventManagerBuilder eventManagerBuilderB = new EventManagerBuilder();


        /** Build B. */
        EventManager eventManagerBImpl = eventManagerBuilderB.build();
        serviceBundleB = serviceBundleBuilder().build(); //build service bundle
        serviceBundleB.addServiceObject("eventManagerB", eventManagerBImpl);
        eventManagerB = serviceBundleB.createLocalProxy(EventManager.class, "eventManagerB"); //wire B to Service Bundle

        EventBusRemoteReplicatorBuilder replicatorBuilder = eventBusRemoteReplicatorBuilder();
        replicatorBuilder.serviceServerBuilder().setPort(9097);
        replicatorBuilder.setEventManager(eventManagerB);
        ServiceServer serviceServer = replicatorBuilder.build();

        EventBusReplicationClientBuilder clientReplicatorBuilder = eventBusReplicationClientBuilder();
        clientReplicatorBuilder.clientBuilder().setPort(9097);
        Client client = clientReplicatorBuilder.build();
        replicatorClient = clientReplicatorBuilder.build(client);

        serviceServer.start();
        client.start();

        /* Create A that connects to the replicator client. */
        EventManager eventManagerAImpl = eventManagerBuilderA.setEventConnector(replicatorClient).build();
        serviceBundleA = serviceBundleBuilder().build(); //build service bundle
        serviceBundleA.addServiceObject("eventManagerA", eventManagerAImpl);
        eventManagerA = serviceBundleA.createLocalProxy(EventManager.class, "eventManagerA"); //wire A to Service Bundle


        serviceBundleA.start();

        serviceBundleB.start();


        final AtomicReference<Object> body = new AtomicReference<>();

        eventManagerB.register("foo.bar", event ->  body.set(event.body()));

        eventManagerA.send("foo.bar", "hello");
        ServiceProxyUtils.flushServiceProxy(eventManagerA);

        waitForTrigger(20, o -> body.get()!=null);


        assertEquals("hello", body.get());

        serviceBundleA.stop();
        serviceBundleB.stop();
        client.stop();
        Sys.sleep(100);
        serviceServer.stop();
    }


}

```
