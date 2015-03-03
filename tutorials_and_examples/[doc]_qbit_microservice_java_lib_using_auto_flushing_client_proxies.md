# [Doc] QBit microservice Java lib Using auto flushing client proxies

Mostly we show you how to flush method calls for microbatching.
At times it is convenient to periodically flush method calls just in case you are not under high loads.

Instead of having you create a lot of boiler plate code, we provide a SendQueue that is auto-flushing.
Then we also provide the ability to create service queue proxies that are also auto-flushing.

```java


        ServiceQueue serviceQueue = ...

        TodoServiceClient todoServiceClient =
                                      serviceQueue
                                          .createProxyWithAutoFlush(
                                              TodoServiceClient.class,
                                               50, TimeUnit.MILLISECONDS);

        todoServiceClient.add(new TodoItem("foo", "foo", null));

        AtomicReference<List<TodoItem>> items = new AtomicReference<>();
        todoServiceClient.list(new Callback<List<TodoItem>>() {
            @Override
            public void accept(List<TodoItem> todoItems) {
                items.set(todoItems);
            }
        });
```

The new method is createProxyWithAutoFlush. There are two versions of createProxyWithAutoFlush.
One that takes a PeriodicScheduler and one that does not take a PeriodicScheduler.

You can pass your own PeriodicScheduler or you can use the system one.

```java

public interface PeriodicScheduler extends Startable, Stoppable{

    ScheduledFuture repeat(Runnable runnable, int interval, TimeUnit timeUnit);

    default void start() {}
    default void stop() {}

}
```


