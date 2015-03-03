# [Doc] QBit Java microservice lib auto flushing service queue proxies

QBit is a queue batching system.
QBit has the ability to batch many queue items and send them in batches.
When you create a QBit queue you can ask how large you would like that batch to be.
QBit also has the ability to check to see if the other side is busy. By other side we mean the consumer or consumers of the queue items is busy.

This is all fine and dandy, until you get a message waiting to be sent.
Let's say that you have a batch size of 20,000 but you only get 100 messages. It is a slow day, and the messages are just waiting there to be sent. We do not flush to the other side or check to see if the other side is not busy (bored) until we get a new message. Then you have 100 messages just waiting until that one message comes through or the 19,900 message come through depending on how you have the periodic busy check configured.

This means that you as the developer must remember to flush the queue messages periodically.
It is worse than that because if you use a ServiceQueue proxy, which is basically just a proxy on top of a series of QBit queues that talk to one or more services or an event bus, and you do not flush the proxy, the method calls never get sent.

In practice this is not too difficult, because when you develop services in QBit there are some natural places to flush messages to queues and client proxies, i.e., you get notified that you are empty, idle or your limit has been reached then you might as well flush calls to your collaborators.

One thing we added to QBIt is auto-flushing Queue senders. QBit divides up a `Queue` into `SendQueue` and `ReceiveQueue` and the uses this division to add batching of messages on to a queue. You use `Queue` as a factory object to create `SendQueue`s and `ReceiveQueue`s.

##Example Basic SendQueue no auto flush

####Basic Send Queue example


Here is the full code for a basis send queue example where we send items on one thread
and read those times from another thread.

####Complete code example of basic send queue.
```java

        /** Build our queue. */
        final QueueBuilder builder = new QueueBuilder().setName("test").setPollWait(1000).setBatchSize(10);
        Queue<String> queue = builder.build();


        final AtomicInteger count = new AtomicInteger();

        /* Create a sender queue. */
        final SendQueue<String> sendQueue = queue.sendQueue();


        /* Create a receiver queue. */
        final ReceiveQueue<String> receiveQueue = queue.receiveQueue();


        /* Create a writer thread that uses the send queue. */
        Thread writerThread = new Thread(() -> {


            for (int index = 0; index < 1000; index++) {
                sendQueue.send("item" + index); //It will flush every 10 or so
            }
            sendQueue.flushSends(); //We can also call flushSends so it sends what remains.
        });



        /* Create a reader thread that consumes queue items. */
        Thread readerThread = new Thread(() -> {
            String item = receiveQueue.pollWait();

            while (item != null) {
                count.incrementAndGet();
                item = receiveQueue.pollWait();

            }
        });

        /* Starts the threads and wait for them to end. */
        writerThread.start();
        readerThread.start();

        /* Wait for them to end. */
        writerThread.join();
        readerThread.join();

        puts(count);

        ok = count.get() == 1000 || die("count should be 1000", count.get());

```

Let's break this down:



####Build the queue with QueueBuilder
```java
        /** Build our queue. */
        final QueueBuilder builder = queueBuilder().setName("test")
                                      .setPollWait(1000).setBatchSize(10);
        Queue<String> queue = builder.build();

```


Create a counter so we can track how many items the reader thread gets.

####Counter for testing
```java

        final AtomicInteger count = new AtomicInteger();


```

Use the queue to create a sender queue.


####Create SendQueue from queue to send messages
```java
     /* Create a sender queue. */
        final SendQueue<String> sendQueue = queue.sendQueue();

```

Sender queues are not thread safe. They are optimized to be accessed by on thread so they can buffer enqueues and so that can check to see if the other side is busy.



Use the queue to create a receiver queue.


####Create ReceiveQueue from queue to receive messages
```java
        /* Create a receiver queue. */
        final ReceiveQueue<String> receiveQueue = queue.receiveQueue();

```

Now create a writer thread that sends 1000 queue messages to the receiver.


####Writer Thread
``java
        /* Create a writer thread that uses the send queue. */
        Thread writerThread = new Thread(() -> {


            for (int index = 0; index < 1000; index++) {
                sendQueue.send("item" + index); //It will flush every 10 or so
            }
            sendQueue.flushSends(); //We can also call flushSends so it sends what remains.
        });

```

Notice that we call `sendQueue.flushSends()` after the loop finishes. We do this so we know the queue
sends all of its items to the other side.

Now create a reader that reads the messages from the writer thread.


####Reader Thread
```java


        /* Create a reader thread that consumes queue items. */
        Thread readerThread = new Thread(() -> {
            String item = receiveQueue.pollWait();

            while (item != null) {
                count.incrementAndGet();
                item = receiveQueue.pollWait();

            }
        });
```

Then start the threads and wait for them to end.


####Close shop and assert we got the right count
```java

        /* Starts the threads and wait for them to end. */
        writerThread.start();
        readerThread.start();

        /* Wait for them to end. */
        writerThread.join();
        readerThread.join();
```

This is great but what if we forget to call flush. It can happen. Also what if we are using the sendQueue behind a `ServiceQueue` client proxy, it becomes a leaky abstraction if we force everyone to use our flush utilities to flush client proxies. What we want is a way to auto flush the sender queue every so often.

##Auto-flushing send queue


We added two methods to `Queue` to support auto-flushing.

```java

public interface Queue<T> {

...


    default SendQueue<T> sendQueueWithAutoFlush(final int interval, final TimeUnit timeUnit) {

        PeriodicScheduler periodicScheduler = QBit.factory().periodicScheduler();

        return sendQueueWithAutoFlush(periodicScheduler, interval, timeUnit);
    }

    default SendQueue<T> sendQueueWithAutoFlush(final PeriodicScheduler periodicScheduler,
                                                final int interval, final TimeUnit timeUnit) {

        SendQueue<T> sendQueue = sendQueue();
        return new AutoFlushingSendQueue<>(sendQueue, periodicScheduler, interval, timeUnit);
    }

```

We have the `sendQueueWithAutoFlush` that takes a `PeriodicScheduler` and one that does not and just uses the system `PeriodicScheduler`. The one that takes a `PeriodicScheduler` will probably never be needed and if it is, you are probably missing the whole concept of what a MircoSerivce is. That said, it is there. The provided `PeriodicScheduler` should meet most needs. The `PeriodicScheduler` is a simple interface as follows:


####PeriodicScheduler
```java

public interface PeriodicScheduler extends Startable, Stoppable{

    ScheduledFuture repeat(Runnable runnable, int interval, TimeUnit timeUnit);

    default void start() {}
    default void stop() {}

}

```

You can provide your own or use the Factory create method for `PeriodicScheduler` if your system needs more than one (unlikely).

The `sendQueueWithAutoFlush` method passes a `PeriodicScheduler` and then constructs a `AutoFlushingSendQueue`. A `AutoFlushingSendQueue` implements the `SendQueue` and wraps a `SendQueue` which it uses the `PeriodicScheduler` to flush periodically. Recall that 50 ms is like years and years to a CPU.

Let's redo our last example with the periodic flush.

####Complete listing showing using a auto-flushing send queue
```java
    @Test
    public void testUsingAutoFlush() throws Exception {


        final QueueBuilder builder = new QueueBuilder().setName("test").setPollWait(1000).setBatchSize(20_000);
        final Queue<String> queue = builder.build();

        final AtomicInteger count = new AtomicInteger();

        final SendQueue<String> sendQueue = queue.sendQueueWithAutoFlush(50, TimeUnit.MILLISECONDS);
        final ReceiveQueue<String> receiveQueue = queue.receiveQueue();

        sendQueue.start();

        Thread writerThread = new Thread(() -> {
            for (int index = 0; index < 1000; index++) {
                sendQueue.send("item" + index);
            }
        });


        Thread readerThread = new Thread(() -> {
            while (receiveQueue.pollWait() != null) {
                count.incrementAndGet();
            }
        });

        writerThread.start();
        readerThread.start();
        writerThread.join();
        readerThread.join();
        Sys.sleep(1000); //simulate a long sleep
        sendQueue.stop();

        puts(count);

        ok = count.get() == 1000 || die("count should be 1000", count);

    }

```


The big change from before is this:

####Using the autoFlush creation method.
```java

        final SendQueue<String> sendQueue =
              queue.sendQueueWithAutoFlush(50, TimeUnit.MILLISECONDS);
```

The above creates a `sendQueue`, which will be auto-flushed every 50 milliseconds.

This means we do not have to explicitly flush like we did in the last example.

##Using auto flushing service queue proxies

To create a service queue proxy client that auto flushes, you use the `createProxyWithAutoFlush` method
of the ServiceQueue.


####Using createProxyWithAutoFlush method
```java

        TodoServiceClient todoServiceClient =
                serviceQueue.createProxyWithAutoFlush(TodoServiceClient.class,
                                                      50, TimeUnit.MILLISECONDS);

```

The above method creates a proxy that will be flushed every 50 ms. The `createProxyWithAutoFlush` creates
a proxy that in turn uses `queue.sendQueueWithAutoFlush` for the method sender queue.


####Complete example using ServiceQueue createProxyWithAutoFlush
```java

    @Test
    public void testUsingProxyWithAutoFlush() {


        /* Create a service that lives behind a ServiceQueue. */
        ServiceQueue serviceQueue = serviceBuilder()
                                    .setServiceAddress("/todo-service")
                                    .setServiceObject(new TodoService())
                                    .build();

        serviceQueue.start().startCallBackHandler();

        TodoServiceClient todoServiceClient =
                serviceQueue.createProxyWithAutoFlush(TodoServiceClient.class, 50, TimeUnit.MILLISECONDS);

        todoServiceClient.add(new TodoItem("foo", "foo", null));

        AtomicReference<List<TodoItem>> items = new AtomicReference<>();
        todoServiceClient.list(todoItems -> items.set(todoItems));

        Sys.sleep(200);

        ok = items.get()!=null || die();
        ok = items.get().size() > 0 || die();
        ok = items.get().get(0).getDescription().equals("foo") || die();

    }

```




