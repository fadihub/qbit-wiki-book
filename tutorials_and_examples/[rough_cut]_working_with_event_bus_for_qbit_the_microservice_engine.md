# [Rough Cut] Working with event bus for QBit the microservice engine



QBit has an event bus. The event bus is loosely modeled after the vertx event bus.

QBit Services are sent events on the same thread/queue they use to handle method calls so the events a QBit service gets is thread safe.

The event bus is a great way to include additional services without disrupting existing services.

Except with QBit you can send object strongly typed objects, JSON, Maps, etc.
When sending strongly typed objects care must be done to ensure the objects are immutable or one must ensure that object being sent will not be written to by more than one service.
It is best just to send immutable objects or JSON.



You get the system event bus from the QBit factory:

```java

EventManager eventManager = QBit.factory().systemEventManager();


```

You can register for events like this:

```java


        String rick = "rick"; //channelName

        eventManager.register(rick, new EventConsumer<Object>() {
            @Override
            public void listen(Event<Object> event) {
                //puts(event);
                consumerMessageCount++;
            }
        });

```

In this hello world example, we are using the channel called "rick" to send a string.
A follow up example uses an Employee domain object.

The above registers for the event as a consumer.
You can only have one consumer.
The consumer gets called after all the subscribers.
If you subscribe from a QBit Service, then the event is enqueued on to the Service's queue.
QBit implements the Actor/Active Object style programming similar to Akka or Go or ErLang.


To register as a subscriber, you would do this:

```java

        eventManager.register(rick, new EventSubscriber<Object>() {
            @Override
            public void listen(Event<Object> event) {
                System.out.println(event.getBody());
            }
        });
```

Recall that rick is just a channel name. When the event happens, the listener will be notified.

The body of the event holds the object you sent.
You can send arrays or lists as the body and if you have a listener that uses an annotation show later, QBit will use the list or array to invoke the listener method.
QBit supports strongly typed listeners.

To use a CallBack instance instead of a EventListener, you would do this:

```java
        eventManager.register(rick, callbackEventListener(event -> {
            if (subscribeMessageCount < 1000) puts(event);
            subscribeMessageCount++;
        }));
```

You can also register class instance as listeners. To do this you use the @Listen annotation as shown below.

Here is an example listener using the @Listen annotation.

```java

    public static class MyEventListener {

        volatile int callCount = 0;

        @Listen("rick")
        void listen(String message) {
            callCount++;
        }
    }
```

Notice we use the @Listen annotation to denote which method will get called when the event happens.

To register the above you would use this:

```java


        MyEventListener myEventListener = new MyEventListener();

        eventManager.listen(myEventListener);
```


You send messages like this:

```java
        eventManager.send(rick, "Hello Rick");
```

If you are using the eventManager outside of a service in a QBit queue, then you have to flush the eventManager after you are done using it.

```java
flushServiceProxy(eventManager);
```

All of the event listeners we showed so fall are for events that come on another thread. Services are special. They can register for events that come back on their own thread. Side note: you can define your own event bus, as all services monitor a method queue and an event queue in the same thread. See (QBit Microservice lib private event bus)[https://github.com/advantageous/qbit/wiki/%5BRough-Cut%5D-Working-with-private-event-bus-for-inproc-microservices]. The event bus is a good integration point for Kafka, ActiveMQ, Tibco, STOMP, Camel, 0MQ, etc.  This was "call" can come from REST, WebSocket or other means.

##Event Manager and QBit Services

You can use the EventManager with QBit Services.

Here are two example simple services.

```java

    public static class MyService {

        public void sendHi(String hi) {
            serviceContext().send("rick", "hello rick " + hi);
        }
    }

    public static class MyServiceConsumer {

        int callCount = 0;


        @Listen("rick")
        private void listen(String message) {

            puts(message);
            callCount++;
        }


        private int callCount() {
            return callCount;
        }


    }

```

The one service, sends a message to the other service using the event bus.

If you wanted the listener service to be a consumer instead of a subscriber, then you would do this:
```java


    public static class MyServiceConsumer {
        @Listen(value = "rick", consume = true)
        private void listen(String message) {

            puts(message);
            callCount++;
        }
    }

```

To use the services within QBit you could do this:

```java

    public static interface MyServiceClient  {

        void sendHi(String hi);
    }
...
        final MyServiceConsumer myServiceConsumer = new MyServiceConsumer();

        final MyService myService = new MyService();


        final ServiceQueue consumerService = serviceBuilder()
                .setServiceObject(myServiceConsumer)
                .setInvokeDynamic(false).build();

        final ServiceQueue senderService = serviceBuilder()
                .setServiceObject(myService)
                .setInvokeDynamic(false).build();



        final MyServiceClient clientProxy =
              senderService.createProxy(MyServiceClient.class);

        clientProxy.sendHi("Hello");
        flushServiceProxy(clientProxy);



```

The advantage of using the event bus with QBit services is that the events come into the same
queue that handles the method calls so the events method calls are thread safe.
Everything comes in on the same thread, events and methods. Responses can either go out on the same thread, or you can have a separate response thread or if you use a ServiceBundle the bundle will handle all responses. But this is off topic.


The event bus is very fast. Expect speeds up to 10M to 100M messages a second.

Let's try another example. First I will show all of the code and then I will break it down step by step.

Let's build a set of services that handles when a new employees is hired. We want to add the employee to the payroll system, enrolled the employee into the benefits system and we want to invite them to our community outreach program.

We will define four services but the first service will not know about the other service.
And we can add more services in the future which can listen to events and participate in the new employee being hired.

```java
package io.advantageous.qbit.example.events;

import io.advantageous.qbit.annotation.OnEvent;
import io.advantageous.qbit.events.EventManager;
import io.advantageous.qbit.service.Service;
import io.advantageous.qbit.service.ServiceProxyUtils;
import org.boon.Lists;
import org.boon.core.Sys;

import static io.advantageous.qbit.service.ServiceBuilder.serviceBuilder;
import static io.advantageous.qbit.service.ServiceContext.serviceContext;
import static io.advantageous.qbit.service.ServiceProxyUtils.flushServiceProxy;

/**
 * Created by rhightower on 2/4/15.
 */
public class EmployeeEventExample {

    public static final String NEW_HIRE_CHANNEL = "com.mycompnay.employee.new";

    public static final String PAYROLL_ADJUSTMENT_CHANNEL = "com.mycompnay.employee.payroll";

    public class Employee {
        final String firstName;
        final int employeeId;

        public Employee(String firstName, int employeeId) {
            this.firstName = firstName;
            this.employeeId = employeeId;
        }

        public String getFirstName() {
            return firstName;
        }

        public int getEmployeeId() {
            return employeeId;
        }

        @Override
        public String toString() {
            return "Employee{" +
                    "firstName='" + firstName + '\'' +
                    ", employeeId=" + employeeId +
                    '}';
        }
    }

    interface EmployeeHiringServiceClient {
        void hireEmployee(final Employee employee);

    }

    public class EmployeeHiringService {



        public void hireEmployee(final Employee employee) {

            int salary = 100;
            System.out.printf("Hired employee %s\n", employee);

            //Does stuff to hire employee

            //Sends events
            final EventManager eventManager =
                              serviceContext().eventManager();

            eventManager.send(NEW_HIRE_CHANNEL, employee);

            eventManager.sendArray(PAYROLL_ADJUSTMENT_CHANNEL,
                                                employee, salary);


        }

    }



    public class BenefitsService {

        @OnEvent(NEW_HIRE_CHANNEL)
        public void enroll(final Employee employee) {

            System.out.printf("Employee enrolled into benefits system employee %s %d\n",
                    employee.getFirstName(), employee.getEmployeeId());

        }

    }

    public class VolunteerService {

        @OnEvent(NEW_HIRE_CHANNEL)
        public void invite(final Employee employee) {

            System.out.printf("Employee will be invited to the community outreach program %s %d\n",
                    employee.getFirstName(), employee.getEmployeeId());

        }

    }



    public class PayrollService {

        @OnEvent(PAYROLL_ADJUSTMENT_CHANNEL)
        public void addEmployeeToPayroll(final Employee employee, int salary) {

            System.out.printf("Employee added to payroll  %s %d %d\n",
                    employee.getFirstName(), employee.getEmployeeId(), salary);

        }

    }

    public static void main(String... args) {


        EmployeeHiringService employeeHiring = new EmployeeHiringService();
        PayrollService payroll = new PayrollService();
        BenefitsService benefits = new BenefitsService();
        VolunteerService volunteering = new VolunteerService();


        ServiceQueue employeeHiringService = serviceBuilder()
                .setServiceObject(employeeHiring)
                .setInvokeDynamic(false).build();

        Sys.sleep(100);


        ServiceQueue payrollService = serviceBuilder()
                .setServiceObject(payroll)
                .setInvokeDynamic(false).build();

        Sys.sleep(100);

        ServiceQueue employeeBenefitsService = serviceBuilder()
                .setServiceObject(benefits)
                .setInvokeDynamic(false).build();


        ServiceQueue volunteeringService = serviceBuilder()
                .setServiceObject(volunteering)
                .setInvokeDynamic(false).build();

        EmployeeHiringServiceClient employeeHiringServiceClientProxy =

                   employeeHiringService.createProxy(EmployeeHiringServiceClient.class);

        employeeHiringServiceClientProxy.hireEmployee(new Employee("Rick", 1));

        flushServiceProxy(employeeHiringServiceClientProxy);

        Sys.sleep(5_000);

    }
}

```


I added another example because the first one just used strings and I wanted to show how to use strongly typed objects.

This one uses two channels.

```
public static final String NEW_HIRE_CHANNEL = "com.mycompnay.employee.new";

public static final String PAYROLL_ADJUSTMENT_CHANNEL = "com.mycompnay.employee.payroll";
```

The first channel NEW_HIRE_CHANNEL is where we send new employee objects when they are hired.
A whole slew of services could be listening to this channel.


An employee object looks like this:

```java

public static class Employee {
       final String firstName;
       final int employeeId;
```

Imagine getter and setter methods and a constructor or go look at the full listing.


This example has three services: EmployeeHiringService, BenefitsService, and PayrollService.

These services are inproc services. QBit supports WebSocket, HTTP and REST remote services as well, but for now, let's focus on inproc services. If you understand inproc then you will understand remote.

The EmployeeHiringService actually fires off the events to other two services.

```java
public class EmployeeHiringService {


    public void hireEmployee(final Employee employee) {

           int salary = 100;
           System.out.printf("Hired employee %s\n", employee);

           //Does stuff to hire employee

           //Sends events
           final EventManager eventManager =
                               serviceContext().eventManager();
           eventManager.send(NEW_HIRE_CHANNEL, employee);

           eventManager.sendArray(PAYROLL_ADJUSTMENT_CHANNEL,
                                     employee, salary);


    }

   }
```

When working inside of a QBit Service, you can access the event manager using serviceContext().eventManager(). If you access it this way, the the flushing is taken care of for you.
Flushing messages to other services in batches helps with the performance. You have to flush after
you use a client proxy. The eventManager() method returns a client proxy.
When running inside of QBit, you do not have to flush, it is done for you at the time when/where
you will get the most performance out of the system. This is what allows the event manager to send so many messages in such a short period of time. Not only send the messages but enqueue them on other
service queues.

Notice that we call sendArray so we can send the employee and their salary. The listener for PAYROLL_ADJUSTMENT_CHANNEL will have to handle both an employee and an int that represents the new employees salary.

The BenefitsService listens for new employees being hired so it can enroll them into the benefits system.

```java
public static class BenefitsService {

       @OnEvent(NEW_HIRE_CHANNEL)
       public void enroll(final Employee employee) {

           System.out.printf("Employee enrolled into benefits system employee %s %d\n",
                   employee.getFirstName(), employee.getEmployeeId());

       }

```

```java
    public static class PayrollService {

        @OnEvent(PAYROLL_ADJUSTMENT_CHANNEL)
        public void addEmployeeToPayroll(final Employee employee, int salary) {

            System.out.printf("Employee added to payroll  %s %d %d\n",
                    employee.getFirstName(), employee.getEmployeeId(), salary);

        }

    }

```

The employee is the employee object from the EmployeeHiringService.

so you can get your benefits...


To start things off, you need to get a client proxy to the EmployeeHiringService using the employeeHiringService.

```java
EmployeeHiringServiceClient employeeHiringServiceClientProxy =
         employeeHiringService.createProxy(EmployeeHiringServiceClient.class);

employeeHiringServiceClientProxy.hireEmployee(new Employee("Rick", 1));
```

The full wiring is done like this for all of the services.

Create the POJOs
```java

        EmployeeHiringService employeeHiring = new EmployeeHiringService();
        PayrollService payroll = new PayrollService();
        BenefitsService benefits = new BenefitsService();

```

Wire in employeeHiring, payroll and benefits into the QBit queuing apparatus.

```
        ServiceQueue employeeHiringService = serviceBuilder()
                .setServiceObject(employeeHiring)
                .setInvokeDynamic(false).build();

        ServiceQueue payrollService = serviceBuilder()
                .setServiceObject(payroll)
                .setInvokeDynamic(false).build();


        ServiceQueue employeeBenefitsService = serviceBuilder()
                .setServiceObject(benefits)
                .setInvokeDynamic(false).build();
```

The objects employeeHiringService, payrollService, employeeBenefitsService are QBit services.

To invoke a method on a QBit service, you want to get a client proxy.
A client proxy will send messages to a service. The service will get those messages as method calls.

Every call is sent over a high-speed internal inproc queue.
You can also use a client proxy to talk to QBit over WebSockets.

To create a proxy you use the createProxy method of Service.
```java
        EmployeeHiringServiceClient employeeHiringServiceClientProxy =
                      employeeHiringService.createProxy(EmployeeHiringServiceClient.class);
```

Now that we created our proxy, we can send messages to it.
```java

        employeeHiringServiceClientProxy.hireEmployee(new Employee("Rick", 1));
```

Every so often, we have to flush calls to the client proxy.

The client proxy will flush calls every time the queue batch size is met.
So if the queue batch size was set to 5, then it would flush every five calls.
But no matter, when you are done making calls, you should flush the calls as follows:

```java
        flushServiceProxy(employeeHiringServiceClientProxy);
```

If you were making calls to a service in a tight loop, you may want to flush every ten calls or every 100 calls. Or you may want to flush related calls.

If you set the batch size to 1, then every method calls is flushed, but this hampers performance.

If you use the event manager service, it will get auto flushed for you but in an extremely performant way. We may provide similar support for injected client proxies into a service.


Now we are all done. But you know how apps are. Later someone decides to create a community outreach program. Now what?

Simple enough, we just add another service to listen to the new hire event channel.

First we define our POJO:

```java
    public class VolunteerService {

        @OnEvent(NEW_HIRE_CHANNEL)
        public void invite(final Employee employee) {

            System.out.printf(
             "Employee will be invited to the community " +
                "outreach program %s %d\n",
                 employee.getFirstName(), employee.getEmployeeId());

        }

    }


```

The OnEvent annotation is an alternative to @Listen. We also have @Consume, @Subscribe, @Hear.
You do not have to use our annotation. So if you wanted to write a service that was not tied to
QBit at all, i.e., no compile time dependencies, then you would just define your own annotation called
OnEvent, or Listen, or Consume or Subscribe or Hear. We believe in no compile time dependencies for your services. And no class-loader discovery magic that would tie you to Boon or QBit. There is an API.
Your implementation can be divorced from QBit as much as possible.

```java

@Target({ ElementType.METHOD, ElementType.FIELD })
@Retention(RetentionPolicy.RUNTIME)
public @interface OnEvent {


    /* The channel you want to listen to. */;
    String value() default "";

    /* The consume is the last object listening to this event.
       An event channel can have many subscribers but only one consume.
     */
    boolean consume() default false;


}

```

So now we have our new service and we are set to do good in the world.

Let's wire it in!

```java

    public static void main(String... args) {

        //Create the POJO which is not tied to QBit at all
        VolunteerService volunteering = new VolunteerService();

        //Create a service to wrap the POJO in a class
        ServiceQueue volunteeringService = serviceBuilder()
                .setServiceObject(volunteering)
                .setInvokeDynamic(false).build();


        //Now send our message!
        EmployeeHiringServiceClient employeeHiringServiceClientProxy =
               employeeHiringService
                  .createProxy(EmployeeHiringServiceClient.class);

        employeeHiringServiceClientProxy.hireEmployee(new Employee("Rick", 1));

        flushServiceProxy(employeeHiringServiceClientProxy);

        Sys.sleep(5_000);

    }
```

This is the output from this app.

```
Hired employee Employee{firstName='Rick', employeeId=1}
Employee added to payroll  Rick 1 100
Employee enrolled into benefits system employee Rick 1
Employee will be invited to the community outreach program Rick 1
```

Now our new employees are hired, added to the payroll system, enrolled into benefits and are invited to our community outreach program.


Here is the full wiring. Since we use Java properties and builders, it would be easy to use QBit from Guice or Spring.

```java

    public static void main(String... args) {


        EmployeeHiringService employeeHiring = new EmployeeHiringService();
        PayrollService payroll = new PayrollService();
        BenefitsService benefits = new BenefitsService();
        VolunteerService volunteering = new VolunteerService();


        ServiceQueue employeeHiringService = serviceBuilder()
                .setServiceObject(employeeHiring)
                .setInvokeDynamic(false).build();



        ServiceQueue payrollService = serviceBuilder()
                .setServiceObject(payroll)
                .setInvokeDynamic(false).build();


        ServiceQueue employeeBenefitsService = serviceBuilder()
                .setServiceObject(benefits)
                .setInvokeDynamic(false).build();


        ServiceQueue volunteeringService = serviceBuilder()
                .setServiceObject(volunteering)
                .setInvokeDynamic(false).build();

        EmployeeHiringServiceClient employeeHiringServiceClientProxy =
                 employeeHiringService.createProxy(EmployeeHiringServiceClient.class);

        employeeHiringServiceClientProxy.hireEmployee(new Employee("Rick", 1));

        flushServiceProxy(employeeHiringServiceClientProxy);

        Sys.sleep(5_000);

    }
```

Here is the code from our first example which came from a Unit Test which uses just strings instead of domain objects.

```java
package io.advantageous.qbit.events.impl;

import io.advantageous.qbit.QBit;
import io.advantageous.qbit.annotation.Listen;
import io.advantageous.qbit.client.ClientProxy;
import io.advantageous.qbit.events.EventConsumer;
import io.advantageous.qbit.events.EventManager;
import io.advantageous.qbit.events.EventSubscriber;
import io.advantageous.qbit.message.Event;
import io.advantageous.qbit.service.Service;
import io.advantageous.qbit.service.ServiceBuilder;
import io.advantageous.qbit.service.ServiceContext;
import io.advantageous.qbit.service.ServiceProxyUtils;
import org.boon.core.Sys;
import org.junit.Before;
import org.junit.Test;

import static io.advantageous.qbit.events.EventUtils.callbackEventListener;
import static io.advantageous.qbit.service.ServiceBuilder.serviceBuilder;
import static io.advantageous.qbit.service.ServiceContext.serviceContext;
import static org.boon.Boon.puts;
import static org.boon.Exceptions.die;

public class BoonEventManagerTest {

    EventManager eventManager;

    ClientProxy clientProxy;

    volatile int subscribeMessageCount = 0;

    volatile int consumerMessageCount = 0;

    boolean ok;

    @Before
    public void setup() {
        eventManager = QBit.factory().systemEventManager();
        clientProxy = (ClientProxy) eventManager;
        subscribeMessageCount = 0;
        consumerMessageCount = 0;

    }


    @Test
    public void test() throws Exception {


        String rick = "rick";

        MyEventListener myEventListener = new MyEventListener();

        eventManager.listen(myEventListener);
        clientProxy.clientProxyFlush();

        eventManager.register(rick, new EventConsumer<Object>() {
            @Override
            public void listen(Event<Object> event) {
                //puts(event);
                consumerMessageCount++;
            }
        });


        eventManager.register(rick, new EventSubscriber<Object>() {
            @Override
            public void listen(Event<Object> event) {
                //puts(event);
                subscribeMessageCount++;
            }
        });

        eventManager.register(rick, callbackEventListener(event -> {
            if (subscribeMessageCount < 1000) puts(event);
            subscribeMessageCount++;
        }));


        final MyServiceConsumer myServiceConsumer = new MyServiceConsumer();

        final MyService myService = new MyService();

        ServiceQueue consumerService = serviceBuilder()
                .setServiceObject(myServiceConsumer)
                .setInvokeDynamic(false).build();

        clientProxy.clientProxyFlush();

        Sys.sleep(100);


        eventManager.send(rick, "Hello Rick");
        clientProxy.clientProxyFlush();

        Sys.sleep(100);

        ok = subscribeMessageCount == 2 || die(subscribeMessageCount);

        ok = consumerMessageCount == 1 || die();


        ok = myEventListener.callCount == 1 || die();

        ok = myServiceConsumer.callCount() == 1 || die();


        Sys.sleep(100);
        ServiceQueue senderService = serviceBuilder()
                .setServiceObject(myService)
                .setInvokeDynamic(false).build();

        final MyServiceClient clientProxy = senderService.createProxy(MyServiceClient.class);

        clientProxy.sendHi("Hello");
        ServiceProxyUtils.flushServiceProxy(clientProxy);

        Sys.sleep(100);

        ok = subscribeMessageCount == 4 || die(subscribeMessageCount);

        ok = consumerMessageCount == 2 || die();


        ok = myEventListener.callCount == 2 || die();

        ok = myServiceConsumer.callCount() == 2 || die();

    }


    //@Test This takes a long time to run. I only need it for perf tuning.
    public void testPerfMultiple() throws Exception {

        for (int index = 0; index < 5; index++) {
            testPerf();
            Sys.sleep(5_000);
        }
    }

    @Test
    public void testPerf() throws Exception {


        eventManager = QBit.factory().systemEventManager();
        consumerMessageCount = 0;
        Sys.sleep(100);
        subscribeMessageCount = 0;
        Sys.sleep(100);


        String rick = "rick";

        eventManager.register(rick, new EventConsumer<Object>() {
            @Override
            public void listen(Event<Object> event) {
                consumerMessageCount++;
            }
        });


        eventManager.register(rick, callbackEventListener(event -> {
            subscribeMessageCount++;
        }));

        clientProxy.clientProxyFlush();
        Sys.sleep(100);


        long start = System.currentTimeMillis();

        for (int index = 0; index < 1_000_000; index++) {
            eventManager.send(rick, "PERF");

        }


        clientProxy.clientProxyFlush();
        Sys.sleep(100);


        while (true) {

            Sys.sleep(10);


            if (consumerMessageCount >= 900_000) {
                break;
            }

            if (start - System.currentTimeMillis() > 3_000) {
                break;
            }
        }


        long stop = System.currentTimeMillis();
        Sys.sleep(100);


        long duration = (stop - start);

        if (duration > 10_000) {
            die("duration", duration);
        }


        if (consumerMessageCount < 900_000) {
            die("consumerMessageCount", consumerMessageCount);
        }

        puts("Duration to send messages", duration,
                "ms. \nconsume message count", consumerMessageCount,
                "\ntotal message count", consumerMessageCount + subscribeMessageCount);


    }


    public static class MyEventListener {

        volatile int callCount = 0;

        @Listen("rick")
        void listen(String message) {
            callCount++;
        }
    }


    public static interface MyServiceClient  {

        void sendHi(String hi);
    }

    public static class MyService {

        private void queueInit() {
            puts("QUEUE START MyService");
        }

        public void sendHi(String hi) {
            serviceContext().send("rick", "hello rick " + hi);
        }
    }

    public static class MyServiceConsumer {

        int callCount = 0;

        public MyServiceConsumer() {
            puts("MyService created");
        }



        @Listen("rick")
        private void listen(String message) {

            puts(message);
            callCount++;
        }


        private int callCount() {
            return callCount;
        }


    }


}
```


## Random thoughts

Vetx and qbit have the concept of high-speed inproc services that may not know anything about other high-speed in services. Vertx calls this Verticles and they are loaded in different Verticles. QBit just has Java objects that might not know anything about the other services running in the same JVM but still be able to communicate via the event bus. QBit has Services where Vertx has Verticles.

Messaging like this has to be somewhat untyped. Or you have to agree on what objects you have in common. I can see sending common objects, but I also see sending maps, and JSON.

But if we agree on some common objects and events. Then we never have to know about the other systems that we add later is why we have the event bus concept.

We can keep adding stuff that keeps listening.. but it is loose... some things like to be tight and should be tight.. some things need to be loose. You can do tight or loose.

You can create an event bus and make it work with JSON and Maps (it is a flag)... so literally you can implement micro-service tolerant readers and writers in the same JVM.

You can have more than one event bus btw... An event bus is just another QBit service... There is a system one but it is easy to create more. Really easy. It takes three lines of code to create a high-speed event bus. So each module could have its own event bus.. And then use the system event bus to send messages between modules.

So it is very different than
https://code.google.com/p/guava-libraries/wiki/EventBusExplained, but more like http://vertx.io/core_manual_java.html#the-event-bus. This does not mean that Guava is wrong. It just means that this is different.
