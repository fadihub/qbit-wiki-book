# [Rough Cut] Working with strongly typed event bus proxies for QBit Java Microservice lib

This Rough Cut extends these example on the event bus: [Rough Cut: Working with event bus for QBit the microservice engine](https://github.com/advantageous/qbit/wiki/%5BRough-Cut%5D-Working-with-event-bus-for-QBit-the-microservice-engine), [Rough Cut: Working with private event bus for inproc microservices](https://github.com/advantageous/qbit/wiki/%5BRough-Cut%5D-Working-with-private-event-bus-for-inproc-microservices) (the last 2 sections).

In the last example we used the event bus to "call" OnEvent methods on other services - sort of. We had to employ an event manager, and even though it was our event manager interface, we had weird method names to use like send and sendArray. Also we had to pass around event channel names which felt icky. It all worked fine, but wouldn't it be nice to not have our code tied to an event manager per se. We can still have callbacks/events, which just don't need to know that we are using an event manager.

In this example, we are going to do things a bit different. We are going to define a strong typed, abstraction over our use of the event manager.


#### Strongly typed interface to event manager

```java
    interface EmployeeEventManager {

        @EventChannel(NEW_HIRE_CHANNEL)
        void sendNewEmployee(Employee employee);


        @EventChannel(NEW_HIRE_CHANNEL)
        void sendSalaryChange(Employee employee, int newSalary);

    }
```

Now we are sending out events with methods that abstract our use of an event manager.


#### Using Strongly typed interface to event manager

```java
    public static class EmployeeHiringService {

        final EmployeeEventManager eventManager;// <----------- Notice we use
                                                // EmployeeEventManager which is strongly typed

        public EmployeeHiringService (
                       final EmployeeEventManager employeeEventManager) {// <-----------
            this.eventManager = employeeEventManager;
        }

        @QueueCallback(QueueCallbackType.EMPTY)
        private void noMoreRequests() {
            flushServiceProxy(eventManager);
        }

        @QueueCallback(QueueCallbackType.LIMIT)
        private void hitLimitOfRequests() {
            flushServiceProxy(eventManager);
        }

        public void hireEmployee(final Employee employee) {

            int salary = 100;
            System.out.printf("Hired employee %s\n", employee);

            //Does stuff to hire employee


            eventManager.sendNewEmployee( employee); // <------------------
                                                     //  Strongly typed method call
            eventManager.sendSalaryChange( employee, salary );// <------------------
                                                              // Strongly typed method call

        }

    }
```

#### To wire this in do this

```java

        /* Get an eventBusProxyCreator which can create strongly typed event interface. */
        final EventBusProxyCreator eventBusProxyCreator = QBit.factory().eventBusProxyCreator();

        /* Create a strongly typed proxy to the event manager. */
        final EmployeeEventManager employeeEventManager =
                eventBusProxyCreator.createProxy(privateEventBus, EmployeeEventManager.class);


        /*
        Create your EmployeeHiringService but this time pass strongly typed,  private event bus.
         */
        EmployeeHiringService employeeHiring = new EmployeeHiringService(employeeEventManager);


        /** Employee hiring service. */
        ServiceQueue employeeHiringService = serviceBuilder()
                .setServiceObject(employeeHiring)
                .setInvokeDynamic(false).build();


        /* Now wire in the event bus so it can fire events into the service queues. */
        privateEventBus.joinService(payrollService);
        privateEventBus.joinService(employeeBenefitsService);
        privateEventBus.joinService(volunteeringService);


        privateEventBusService.start();
        employeeHiringService.start();

        /** Now the service proxy like before. */
        EmployeeHiringServiceClient employeeHiringServiceClientProxy =
                employeeHiringService.createProxy(EmployeeHiringServiceClient.class);

        /** Call the hireEmployee method which triggers the other events. */
        employeeHiringServiceClientProxy.hireEmployee(new Employee("Rick", 1));

```


I also included strongly typed (no magic method) queue listener callbacks.

```java
        @QueueCallback(QueueCallbackType.EMPTY)
        private void noMoreRequests() {
            flushServiceProxy(eventManager);
        }

        @QueueCallback(QueueCallbackType.LIMIT)
        private void hitLimitOfRequests() {
            flushServiceProxy(eventManager);
        }

```

As before and always, you can use your own annotations and enums or use the ones that ship with QBit.


Now we can have a service that can have 0 tie to QBit. You could with some elbow grease use your implementation in Akka and QBit. I hate framework tie-in which is why QBit is a lib not a framework. Use it how, when you want.

#### Complete Example

```java
package io.advantageous.qbit.example.events;

import io.advantageous.qbit.QBit;
import io.advantageous.qbit.annotation.EventChannel;
import io.advantageous.qbit.annotation.OnEvent;
import io.advantageous.qbit.annotation.QueueCallback;
import io.advantageous.qbit.annotation.QueueCallbackType;
import io.advantageous.qbit.events.EventBusProxyCreator;
import io.advantageous.qbit.events.EventManager;
import io.advantageous.qbit.service.Service;
import io.advantageous.qbit.service.ServiceProxyUtils;
import org.boon.core.Sys;


import static io.advantageous.qbit.service.ServiceBuilder.serviceBuilder;
import static io.advantageous.qbit.service.ServiceContext.serviceContext;
import static io.advantageous.qbit.service.ServiceProxyUtils.flushServiceProxy;

/**
 * Created by rhightower on 2/11/15.
 */
public class EmployeeEventExampleUsingEventProxyToSendEvents {



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


    interface EmployeeEventManager {

        @EventChannel(NEW_HIRE_CHANNEL)
        void sendNewEmployee(Employee employee);


        @EventChannel(PAYROLL_ADJUSTMENT_CHANNEL)
        void sendSalaryChangeEvent(Employee employee, int newSalary);

    }

    public static class EmployeeHiringService {

        final EmployeeEventManager eventManager;

        public EmployeeHiringService (final EmployeeEventManager employeeEventManager) {
            this.eventManager = employeeEventManager;
        }



        @QueueCallback(QueueCallbackType.EMPTY)
        private void noMoreRequests() {
            flushServiceProxy(eventManager);
        }


        @QueueCallback(QueueCallbackType.LIMIT)
        private void hitLimitOfRequests() {
            flushServiceProxy(eventManager);
        }



        public void hireEmployee(final Employee employee) {

            int salary = 100;
            System.out.printf("Hired employee %s\n", employee);

            //Does stuff to hire employee


            eventManager.sendNewEmployee( employee);
            eventManager.sendSalaryChangeEvent( employee, salary );


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


        final EventBusProxyCreator eventBusProxyCreator =
                           QBit.factory().eventBusProxyCreator();

        final EmployeeEventManager employeeEventManager =
                       eventBusProxyCreator.createProxy(privateEventBus, EmployeeEventManager.class);

        /*
        Create your EmployeeHiringService but this time pass the private event bus.
        Note you could easily use Spring or Guice for this wiring.
         */
        EmployeeHiringService employeeHiring = new EmployeeHiringService(employeeEventManager);



        /* Now createWithWorkers your other service POJOs which have no compile time dependencies on QBit. */
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


        privateEventBusService.start();
        employeeHiringService.start();
        volunteeringService.start();
        payrollService.start();
        employeeBenefitsService.start();


        /** Now createWithWorkers the service proxy like before. */
        EmployeeHiringServiceClient employeeHiringServiceClientProxy =
                employeeHiringService.createProxy(EmployeeHiringServiceClient.class);

        /** Call the hireEmployee method which triggers the other events. */
        employeeHiringServiceClientProxy.hireEmployee(new Employee("Rick", 1));

        flushServiceProxy(employeeHiringServiceClientProxy);

        Sys.sleep(5_000);

    }
}

```
