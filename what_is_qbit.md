# What is Qbit?

QBit is a queuing library for microservices. It is similar to many other projects like Akka, Spring Reactor,
etc. QBit is just a library not a platform. QBit has libraries to put a service behind a queue.
You can use QBit queues directly or you can create a service. QBit services can be exposed by WebSocket, HTTP, HTTP pipeline, and other types of remoting. A service in QBit is a Java class whose methods are executed behind service queues. QBit implements apartment model threading and is similar to the Actor model or a better description would be Active Objects. QBit does not use a disruptor.
It uses regular Java Queues. QBit can do north of 100 million ping pong calls per second which is an amazing speed (seen as high as 200M). QBit also supports calling services via REST, and WebSocket.
QBit is microservices in the pure Web sense: JSON, HTTP, WebSocket, etc. QBit uses micro batching to push messages through the pipe (queue, IO, etc.) faster to reduce thread hand-off.

# QBit lingo

QBit is a Java microservice lib supporting REST, JSON and WebSocket. It is written in Java but we could one day write a version in Rust or Go or C# (but that would require a large payday).

**Service**
POJO (plain old Java object) behind a queue that can receive method calls via proxy calls or events (May have one thread managing events, method calls, and responses or two one for method calls and events and the other for responses so response handlers do not block service. One is faster unless responses block). Services can use Spring MVC style REST annotations to expose themselves to the outside world via REST and WebSocket.

**ServiceBundle**
Many POJOs behind one response queue and many receive queues. There may be one thread for all responses or not. They also can be one receive queue.

**Queue**
A thread managing a queue. It supports batching. It has events for empty, reachedLimit, startedBatch, idle. You can listen to these events from services that sit behind a queue. You don't have to use Services.
You can use Queue's direct. In QBit, you have sender queues and receivers queues. They are separated to support micro-batching.

**ServiceServer**
ServiceBundle that is exposed to REST and WebSocket communication.

**EventBus**
EventBus is a way to send a lot of messages to services that may be loosely coupled.

**ClientProxy**
ClientProxy is a way to invoke service through async interface, service can be inproc (same process) or remoted over WebSocket.

**Non-blocking**
QBit is a non-blocking lib. You use CallBacks via Java 8 Lambdas. You can also send event messages and get replies. Messaging is built into the system so you can easily coordinate complex tasks.
QBit takes an object-oriented approach to service development so services look like normal Java services that you already write, but the services live behind a queue/thread. This is not a new concept. Microsoft did this
with DCOM/COM and called it active objects. Akka does it with actors and called them strongly typed Actors.
The important concepts is that you get the speed of reactive and actor style messaging but you develop in a natural OOP approach. QBit is not the first. QBit is not the only.


**Speed**
QBit is VERY fast. There is a of course a lot of room for improvement. But already 200M+ TPS inproc ping pong, 10M-20M+ TPS event bus, 500K TPS RPC calls over WebSocket/JSON, etc.
More work needs to be done to improve speed, but now it is fast enough where we are focusing more on
usability.
The JSON support uses Boon by default which is up to 4x faster than other JSON parsers for the
REST/JSON, WebSocket/JSON use case.

