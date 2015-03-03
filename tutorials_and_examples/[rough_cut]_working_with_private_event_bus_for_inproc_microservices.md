# [Rough Cut] Working with private event bus for inproc microservices

This is the second part of this [Rough Cut on event bus for microservices](https://github.com/advantageous/qbit/wiki/%5BRough-Cut%5D-Working-with-event-bus) (previous section).


So I said this before and I will say this again.
QBit is a lib not a framework.
You can use it with Spring, Vertx, Jetty, etc.
It does not want to take over your world.

QBit services and busses were designed to be composable. You can wire in QBit Services into many event bus systems. This makes it easy to wire a service to listen to events coming from Kafka, RabbitMQ or something else.

But if you look at the last example, with the system event bus, it is starting to look a lot like EJB. WHICH IS NOT A GOOD THING FOR QBIT. Well, you don't have to use the system event bus. QBit aint EJB.

So if you want to use the system event bus, we make it easy. But you can use **a QBit event bus** and still have no compile time dependencies to QBit. You would do this by using a **private event bus**.

Remember that even Annotations are not compile time dependency because you can define your own and if they have the same name and params at QBit, then QBit will happily use them (same for Boon, this is a theme).

QBit is interface by convention like Go and Rust. It is not compile time dependent interfaces.
You can develop QBit services in standalone jar files that do not have any compile time dependencies on QBit.


Ok.. Enough monologue, let's get cracking.

Continuing our last [last example of the event bus](https://github.com/advantageous/qbit/wiki/%5BRough-Cut%5D-Working-with-event-bus).

First step define your own interface to QBit event bus


```java
    interface EventManagerClient {

        <T> void send(String channel, T event);

        <T> void sendArray(String channel, T... event);

        void clientProxyFlush();
    }
```

Proxy client interfaces are by convention. If the interface matches, then it works. The method **clientProxyFlush** is a special method that will flush the events. When you do things by convention instead of my interface, then it is easier to mix and match different versions of jars. I learned this trick from Go. Go is strongly typed but supports interfaces. QBit and Boon do the same thing but for Java.

To get this example to work, we only have to change one class and then we have to do some wiring to let the private bus know about the other services. QBit services and busses were designed to be composable. You can wire in QBit Services into many event bus systems. This makes it easy to wire a service to listen to events coming from Kafka, RabbitMQ or something else.


```java

    public static class EmployeeHiringService {

        private final EventManagerClient eventManager;

        public EmployeeHiringService (final EventManagerClient eventManager) {
            this.eventManager = eventManager;
        }

        void queueEmpty() {
            eventManager.clientProxyFlush();
        }

        void queueLimit() {
            eventManager.clientProxyFlush();
        }



        public void hireEmployee(final Employee employee) {

            int salary = 100;
            System.out.printf("Hired employee %s\n", employee);

            //Does stuff to hire employee

            eventManager.send(NEW_HIRE_CHANNEL, employee);
            eventManager.sendArray(PAYROLL_ADJUSTMENT_CHANNEL, employee, salary);


        }

    }

```

We added an **EventManagerClient** and we use it to send messages. Your code it not tied to QBit at all.
This code could work somewhere else.


QBit has callbacks for Queue events like queue is idle, queue just received a bunch of messages, queue is busy, queue reached its batch limit, queue is shutdown, queue ate my cat (just seeing if you are paying attention).

To handle when the queue has reached a batch limit, we use queueLimit.
To handle when the queue is empty, we use queueEmpty.

```java

        void queueEmpty() {
            eventManager.clientProxyFlush();
        }

        void queueLimit() {
            eventManager.clientProxyFlush();
        }


```

The method names like the annotations are by convention not by interface. This allows us to add more callbacks in QBit without breaking your code. This is the Go programming concept of yeah go ahead and use interfaces but make them a convention not a hard fast rule so you can mix and match versions and evolve the software. QBit adopts this philosophy as does Boon. It is not the right fit for everything. A domain model could and should have strong interfaces. A lib like QBit should not.

To add to the "Don't add compile time dependencies to QBit", I added an annotation for OnEvent to the example:

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

The last example used the OnEvent from QBit, this example defines its own. So now our services truly are QBit agnostic. They don't care if QBit adds more callbacks or more annotations. It all works by convention.

Next we create our own event bus:

```java


        /* Create you own private event bus. */
        EventManager privateEventBus = QBit.factory().createEventManager();

```

Then we wrap that bad boy into a service queue so we can call it async.

```java

        /* Create a service queue for this event bus. */
        ServiceQueue privateEventBusService = serviceBuilder()
                .setServiceObject(privateEventBus)
                .setInvokeDynamic(false).build();

```

Now we need a way to call the event bus.

```java
        /*
         Create a proxy client for the queued event bus service.
         */
        EventManagerClient eventBus = privateEventBusService.createProxy(EventManagerClient.class);

```

The client proxy will create MethodCall objects and enqueue those objects onto the queue for the EventManager. This way the calls are async. And the EventManager does not need to do a thread sync to send messages.

Next we need to create the EmployeeHiringService and give it this client proxy to the private event manager.

```java
        /*
        Create your EmployeeHiringService but this time pass the private event bus.
        Note you could easily use Spring or Guice for this wiring.
         */
        EmployeeHiringService employeeHiring = new EmployeeHiringService(eventBus);
```

Ok.. Now we are in for the home stretch. Just create your services like you did before.
These are just POJOs that are QBit agnostic except for some coding conventions.

```java

        /* Now create your other service POJOs which have no compile time dependencies on QBit. */
        PayrollService payroll = new PayrollService();
        BenefitsService benefits = new BenefitsService();
        VolunteerService volunteering = new VolunteerService();
```

This should be familiar by now. Wire your POJOs up to make them high-speed services.

```java

        /** Employee hiring service. */
        ServiceQueue employeeHiringService = serviceBuilder()
                .setServiceObject(employeeHiring)
                .setInvokeDynamic(false).build();
        /** Payroll service */
        ServiceQueue payrollService = serviceBuilder()
                .setServiceObject(payroll)
                .setInvokeDynamic(false).build();
        /** Employee Benefits service. */
        ServiceQueue employeeBenefitsService = serviceBuilder()
                .setServiceObject(benefits)
                .setInvokeDynamic(false).build();
        /* Community outreach program service. */
        ServiceQueue volunteeringService = serviceBuilder()
                .setServiceObject(volunteering)
                .setInvokeDynamic(false).build();
```

Now this is NEW. Pay attention. This is an important step.

Wire the private event bus to the services that are consuming the messages.


```java
        /* Now wire in the event bus so it can fire events into the service queues. */
        privateEventBus.joinService(payrollService);
        privateEventBus.joinService(employeeBenefitsService);
        privateEventBus.joinService(volunteeringService);
```

SIDE NOTE: The reason we can wire in another event bus system into services is because services publishes a event queue as part of its public interface so anyone or anything can create **Event** objects and publish them to a service. (Look at the interface for service and then look at the **eventReceiveQueue** method. )


That is it. The rest is the same.

```java

        /** Now create the service proxy like before. */
        EmployeeHiringServiceClient employeeHiringServiceClientProxy =
                employeeHiringService.createProxy(EmployeeHiringServiceClient.class);

        /** Call the hireEmployee method which triggers the other events. */
        employeeHiringServiceClientProxy.hireEmployee(new Employee("Rick", 1));

        flushServiceProxy(employeeHiringServiceClientProxy);

        Sys.sleep(5_000);

```

output
```
Hired employee Employee{firstName='Rick', employeeId=1}
Employee will be invited to the community outreach program Rick 1
Employee added to payroll  Rick 1 100
Employee enrolled into benefits system employee Rick 1
```

##Full code listing

```java
package io.advantageous.qbit.example.events;

import io.advantageous.qbit.QBit;
import io.advantageous.qbit.events.EventManager;
import io.advantageous.qbit.service.Service;
import org.boon.core.Sys;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import static io.advantageous.qbit.service.ServiceBuilder.serviceBuilder;
import static io.advantageous.qbit.service.ServiceContext.serviceContext;
import static io.advantageous.qbit.service.ServiceProxyUtils.flushServiceProxy;

/**
 * Created by rhightower on 2/5/15.
 */
public class EmployeeEventExampleUsingStandaloneEventBus {

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


    public static final String NEW_HIRE_CHANNEL = "com.mycompnay.employee.new";

    public static final String PAYROLL_ADJUSTMENT_CHANNEL = "com.mycompnay.employee.payroll";

    public static class Employee {
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

    interface EventManagerClient {

        <T> void send(String channel, T event);

        <T> void sendArray(String channel, T... event);

        void clientProxyFlush();
    }


    public static class EmployeeHiringService {

        final EventManagerClient eventManager;

        public EmployeeHiringService (final EventManagerClient eventManager) {
            this.eventManager = eventManager;
        }

        void queueEmpty() {
            eventManager.clientProxyFlush();
        }

        void queueLimit() {
            eventManager.clientProxyFlush();
        }



        public void hireEmployee(final Employee employee) {

            int salary = 100;
            System.out.printf("Hired employee %s\n", employee);

            //Does stuff to hire employee

            //Sends events
            eventManager.send(NEW_HIRE_CHANNEL, employee);
            eventManager.sendArray(PAYROLL_ADJUSTMENT_CHANNEL, employee, salary);


        }

    }



    public static class BenefitsService {

        @OnEvent(NEW_HIRE_CHANNEL)
        public void enroll(final Employee employee) {

            System.out.printf("Employee enrolled into benefits system employee %s %d\n",
                    employee.getFirstName(), employee.getEmployeeId());

        }

    }

    public static class VolunteerService {

        @OnEvent(NEW_HIRE_CHANNEL)
        public void invite(final Employee employee) {

            System.out.printf("Employee will be invited to the community outreach program %s %d\n",
                    employee.getFirstName(), employee.getEmployeeId());

        }

    }



    public static class PayrollService {

        @OnEvent(PAYROLL_ADJUSTMENT_CHANNEL)
        public void addEmployeeToPayroll(final Employee employee, int salary) {

            System.out.printf("Employee added to payroll  %s %d %d\n",
                    employee.getFirstName(), employee.getEmployeeId(), salary);

        }

    }

    public static void main(String... args) {


        /* Create you own private event bus. */
        EventManager privateEventBus = QBit.factory().createEventManager();

        /* Create a service queue for this event bus. */
        ServiceQueue privateEventBusService = serviceBuilder()
                .setServiceObject(privateEventBus)
                .setInvokeDynamic(false).build();

        /*
         Create a proxy client for the queued event bus service.
         */
        EventManagerClient eventBus = privateEventBusService.createProxy(EventManagerClient.class);


        /*
        Create your EmployeeHiringService but this time pass the private event bus.
        Note you could easily use Spring or Guice for this wiring.
         */
        EmployeeHiringService employeeHiring = new EmployeeHiringService(eventBus);



        /* Now create your other service POJOs which have no compile time dependencies on QBit. */
        PayrollService payroll = new PayrollService();
        BenefitsService benefits = new BenefitsService();
        VolunteerService volunteering = new VolunteerService();



        /** Employee hiring service. */
        ServiceQueue employeeHiringService = serviceBuilder()
                .setServiceObject(employeeHiring)
                .setInvokeDynamic(false).build();
        /** Payroll service */
        ServiceQueue payrollService = serviceBuilder()
                .setServiceObject(payroll)
                .setInvokeDynamic(false).build();
        /** Employee Benefits service. */
        ServiceQueue employeeBenefitsService = serviceBuilder()
                .setServiceObject(benefits)
                .setInvokeDynamic(false).build();
        /* Community outreach program. */
        ServiceQueue volunteeringService = serviceBuilder()
                .setServiceObject(volunteering)
                .setInvokeDynamic(false).build();


        /* Now wire in the event bus so it can fire events into the service queues. */
        privateEventBus.joinService(payrollService);
        privateEventBus.joinService(employeeBenefitsService);
        privateEventBus.joinService(volunteeringService);

        /** Now create the service proxy like before. */
        EmployeeHiringServiceClient employeeHiringServiceClientProxy =
                employeeHiringService.createProxy(EmployeeHiringServiceClient.class);

        /** Call the hireEmployee method which triggers the other events. */
        employeeHiringServiceClientProxy.hireEmployee(new Employee("Rick", 1));

        flushServiceProxy(employeeHiringServiceClientProxy);

        Sys.sleep(5_000);

    }
}

```
