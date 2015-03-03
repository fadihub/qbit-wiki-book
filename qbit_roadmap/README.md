# Qbit Roadmap

### Servlets / WebSocket

1. Integrate with Servlet async request and WebSocket support (DONE for servlet not WebSocket)
2. ***qbit-servlet*** (DONE for REST not for standard websocket)


### Jetty support

1. ***qbit-jetty*** Build on top of ***qbit-servlet*** to provide jetty support for QBit (DONE)

### Support Spring integration

1. Work with Spring @Context listeners
2. Work with @Service, @InProcService, @RestServer
3. Nice to have work with Spring AOP / Proxy support to create cglib proxies
4. ***qbit-spring*** sub project
5. Still brain storming

### Metrics engine, time series database

1. ***qbit-core*** improvement
2. Start with in-memory version
3. Use it and the event bus to explain QBit (IO, replication, etc.)
4. Started but no where close to done.

These are not minor importance overall. Just minor for the next few releases of QBit.


### Refactor
1. B9 BasicQueue needs to be three queues maybe more. Too much code in once class has already caused some bugs.
2. B10 BoonMethodCall handler has flags for handling dynamic methods (JSON, Maps) and strongly typed methods (Object array with actual objects to methods already pre-chewed), it needs to be refactored.
3. B11 Handle events in the queue (for Service) as the methods
4. Create WebSocket abstraction DONE!
```
onTextMessage
onBinaryMessage
onClose
onError
isOpen
isClosed
sendText
sendBinary
Plus consumers.
```

### Features
1. B1 Have a QBit sessions for disconnected REST and WebSocket clients so on reconnect they pick up where they left off. This would be a separate system on top of what we already offer that provides a logical connection called a session. This is more to support Mobile clients than Service to Service calls
2. A1 Handle callbacks on the same thread as the method calls, this is an IMPORTANT feature and critical to the QBit programming model. DONE!


## Minor
1. C kafka, camel, jms, and 0mq are great examples of things to integrate with
1. they will aid in positioning QBit as a microservice framework
1. ***qbit-kafka*** Integrate QBit event bus with kafka
2. ***qbit-camel*** Integrate Camel with QBit event bus
3. ***qbit-jms*** Integrate JMS with QBit event bus
4. ***qbit-0mq*** Integrate 0mq with QBit event bus
5. create new ring buffer base Queue
6. C integrate with fast Queue from psycho blog
7. C integrate with LMAX queue
8. C1 **qbit-ratpack** I like ratpack. I want to combine QBit and ratpack.

## Bug parade - Gotta have missing features

### Major

1. A uri params work in unit test but not integration server FIXED DONE!
2. A embedded mode for vertx not working (works standalone but embedded broke) PULLED OUT.. Cancelled.
3. B1 @RequestMapping only does single uri, to be more like Spring MVC, needs to work with groups

### Minor

* Event Bus

1. B Event bus no real difference between consume and publish. Not sure we can do much until we integrate with JMS or Kafka. Throwing an exception can mean we did not consume method. Needs plumbing. Might be part of the Kafka and/or JMS integration and not part of the core. This will have to wait until we integrate.
2. A @EventChannel has a flag to use method name as part of event dispatch, we don't use it. Yet.
3. Add EventConnector and Event replication (DONE!)


## Documentation roadmap

1. Document how callbacks work (DONE)
3. Document how you can send non-REST / WebSocket RPC calls with a ServiceServer. ???
4. Document what a ServiceServer is
5. Create a TOC for main wiki and man github page DONE
6. Create a website for the project
7. Use EventManager impl to explain QBit
8. Use Metrics engine to explain QBit
DONE
2. Document how SystemManager works DONE


