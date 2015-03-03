# Does it work?

We have used techniques in Boon and QBit with great success in high-end, high-performance, high-scalable apps.
We helped clients handle 10x the load with 1/10th the servers of their competitors using techniques in QBit.
QBit is us being sick of hand tuning queue access and threads.

Does QBit compete with...
====
__Spring Disruptor:__ No. You could use QBit to write plugins for Spring Disruptor I suppose, but QBit does not compete with Spring Disruptor.

__Spring Boot/Spring MVC:__ No. We use the same annotations but QBit is geared for high-speed in-memory microservices. It is more like Akka than Spring Boot. QBit has a subset of the features of Spring MVC geared only for microservices, i.e., WebSocket RPC, REST, JSON marshaling, etc.

__Akka:__ No. Well Maybe. Akka has similar concepts but they take a different approach. QBit is more focused on Java, and microservices (REST, JSON, WebSocket) than Akka.

__LMAX Disruptor:__ No. In fact, we can use disruptor as on of the queues that QBit uses underneath the covers.

