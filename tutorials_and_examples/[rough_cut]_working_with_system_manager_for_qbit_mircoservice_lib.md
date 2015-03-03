# [Rough Cut] Working with System Manager for QBit Mircoservice lib

The name system event manager sounds scary.
It seems like this might be a container or something.
It is not.

A **SystemManager** is a manager that manages the start and shutdown of QBit Services, Servers, and Queues.
You register the top level areas in your module. Then you await their being shutdown.

Let's say we have a series of services.

```java

   public class EmployeeHiringService {

       ....
   }



    public class BenefitsService {

        @OnEvent(NEW_HIRE_CHANNEL)
        public void enroll(final Employee employee) {

            ...
        }

    }

    public class VolunteerService {

        @OnEvent(NEW_HIRE_CHANNEL)
        public void invite(final Employee employee) {
             ...
        }
    }


    public class PayrollService {

        @OnEvent(PAYROLL_ADJUSTMENT_CHANNEL)
        public void addEmployeeToPayroll(final Employee employee, int salary) {

            ...
        }

    }

```

Now we want to wire them up and start them. We could use builders. Wire each one. Then start them individually. But let's say we want to integrate them with Vertx or Spring or Guice or a Servlet container. Then what? We will want to listen for the lifecycle events of those frameworks and shutdown our services cleanly when we are done with them.


First we create a **system manager** and associate that **system manager** with a builder like so:
```java

        final QBitSystemManager systemManager = new QBitSystemManager(); //<-----

        ServiceBuilder serviceBuilder = serviceBuilder()
                .setSystemManager(systemManager)             //<---------
                .setInvokeDynamic(false);

```

Now we use this builder to build services that are going to be started and stopped together.

```java

        /* Create a service queue for this event bus. */
        ServiceQueue privateEventBusService =
                    serviceBuilder.build(privateEventBus); //use builder

        /** Employee hiring service. */
        ServiceQueue employeeHiringService =
                    serviceBuilder.build(employeeHiring);

        /** Payroll service */
        ServiceQueue payrollService =
                     serviceBuilder.build(payroll);

        /** Employee Benefits service. */
        ServiceQueue employeeBenefitsService =
                     serviceBuilder.build(benefits);

        /* Community outreach program. */
        ServiceQueue volunteeringService =
                     serviceBuilder.build(volunteering);
```


Now instead of calling start on each service, we can start them all in the order that they were built.

```java

        systemManager.startAll();

```

This also works for QueueBuilder, HttpServerBuilder, HttpClientBuilder, ServiceBundleBuilder, ServiceBuilder, ServiceServerBuilder.

If we were starting up in a container of some sort Spring of a Servlet engine or Vertx, then we would call
```java

        systemManager.shutDown();

```

when we handled the callback for the container to shutdown, which would exit all of our QBit threads.

If we were writing a standalone app with QBit (which you can do), and we wanted to wait until all of our services shutdown we could do this.

```java

        systemManager.waitForShutdown();

```

The waitForShutdown() method uses a countdown latch and will wait for each service that is started to shutdown.


If let's say we were writing a test or an example, and we wanted shutdown everything after 6 seconds and we wanted to see waitForShutdown work we could do this (turn log debugging on):

```java

        Sys.sleep(5_000);

        Thread thread = new Thread(systemManager::waitForShutdown);
        thread.start();

        Sys.sleep(1_000);

        systemManager.shutDown();

        puts("Shutdown complete from my sample");


```

With debug log turned on, we will get:

```java
registerService Service{debug=false, service=BoonEventManager}
registerService Service{debug=false, service=EmployeeHiringService}
registerService Service{debug=false, service=PayrollService}
registerService Service{debug=false, service=BenefitsService}
registerService Service{debug=false, service=VolunteerService}
startAll 5
startAll 5
Hired employee Employee{firstName='Rick', employeeId=1}
Employee added to payroll  Rick 1 100
Employee enrolled into benefits system employee Rick 1
Employee will be invited to the community outreach program Rick 1
serviceShutDown 4
serviceShutDown 3
serviceShutDown 2
serviceShutDown 1
serviceShutDown Shutdown complete!
0
Shutdown complete from my sample
```


Now let's say that we want to sleep for five seconds to let our example run and then waitForShutdown.
The app will run for five seconds and then shutdown as follows:

```java
        Thread thread = new Thread(
                () -> {
                    Sys.sleep(5_000);
                    puts("Calling shutdown\n\n");
                    systemManager.shutDown();
                }

        );
        thread.start();
        systemManager.waitForShutdown();
        puts("Shutdown complete from my sample");

```

With debug turned off, you will get:

```output

Hired employee Employee{firstName='Rick', employeeId=1}
Employee will be invited to the community outreach program Rick 1
Employee enrolled into benefits system employee Rick 1
Employee added to payroll  Rick 1 100
Calling shutdown


Shutdown complete from my sample

```



## Complete example

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
import io.advantageous.qbit.service.ServiceBuilder;
import io.advantageous.qbit.system.QBitSystemManager;
import org.boon.core.Sys;

import static io.advantageous.qbit.service.ServiceBuilder.serviceBuilder;
import static io.advantageous.qbit.service.ServiceProxyUtils.flushServiceProxy;
import static org.boon.Boon.puts;

/**
 * Created by rhightower on 2/11/15.
 */
public class UsingShutDown {

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


        final EventBusProxyCreator eventBusProxyCreator =
                QBit.factory().eventBusProxyCreator();

        final EmployeeEventManager employeeEventManager =
                eventBusProxyCreator.createProxy(privateEventBus, EmployeeEventManager.class);

        /*
        Create your EmployeeHiringService but this time pass the private event bus.
        Note you could easily use Spring or Guice for this wiring.
         */
        EmployeeHiringService employeeHiring = new EmployeeHiringService(employeeEventManager);



        /* Now create your other service POJOs which have no compile time dependencies on QBit. */
        PayrollService payroll = new PayrollService();
        BenefitsService benefits = new BenefitsService();
        VolunteerService volunteering = new VolunteerService();

        final QBitSystemManager systemManager = new QBitSystemManager();
        ServiceBuilder serviceBuilder = serviceBuilder()
                .setSystemManager(systemManager)
                .setInvokeDynamic(false);



        /* Create a service queue for this event bus. */
        ServiceQueue privateEventBusService = serviceBuilder.build(privateEventBus);

        /** Employee hiring service. */
        ServiceQueue employeeHiringService = serviceBuilder.build(employeeHiring);

        /** Payroll service */
        ServiceQueue payrollService = serviceBuilder.build(payroll);

        /** Employee Benefits service. */
        ServiceQueue employeeBenefitsService = serviceBuilder.build(benefits);

        /* Community outreach program. */
        ServiceQueue volunteeringService = serviceBuilder.build(volunteering);


        /* Now wire in the event bus so it can fire events into the service queues. */
        privateEventBus.joinServices(
                payrollService,
                employeeBenefitsService,
                volunteeringService);

        systemManager.startAll();



        /** Now createWithWorkers the service proxy like before. */
        EmployeeHiringServiceClient employeeHiringServiceClientProxy =
                employeeHiringService.createProxy(EmployeeHiringServiceClient.class);

        /** Call the hireEmployee method which triggers the other events. */
        employeeHiringServiceClientProxy.hireEmployee(new Employee("Rick", 1));

        flushServiceProxy(employeeHiringServiceClientProxy);

        Sys.sleep(5_000);

        Thread thread = new Thread(systemManager::waitForShutdown);
        thread.start();

        Sys.sleep(1_000);

        systemManager.shutDown();

        puts("Shutdown complete from my sample");







    }
}

```
