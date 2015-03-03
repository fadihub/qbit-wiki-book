# [Doc] Queue Callbacks for QBit queue based services

QBit supports QueueCallback methods to optimize queue throughput and IO throughput.

```java

/**
 * Created by rhightower on 2/10/15.
 */
public interface QueueCallBackHandler {

    /**
     * Queue has reached its limit, can be the same as batch size for queue.
     * This is for periodic flushing to IO or CPU intensive services to improve throughput.
     * Larger batches can equate to a lot less thread sync for the hand-off.
     */
    void queueLimit();

    /**
     * Notification that there is nothing else in the queue.
     */
    void queueEmpty();


    /** Callback for when the queue has started. */
    default void queueInit() {}

    /** Callback for when the queue is idle. */
    default void queueIdle() {}

    /** Callback for when the queue is shutdown. */
    default void queueShutdown(){}

    /** Callback for when the queue has just received some message.
     * idle can mean you are asleep with nothing to do.
     * startBatch can mean you just woke up.
     **/
    default void queueStartBatch() {}

}

```

If you implement these methods in your server, QBit will pick them up.
It can do this automatically of you can implement this interface, or you can mark your methods with an annotation. You cannot mix. Pick one style. Implement the interface, which ties you to QBit.
Or, use an annotation (you can define your own annotation and enum as long as you match the names so you are not tied to QBit) and a enum. Or use the naming convention.


Here is the annotation.

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;


@Target({ ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
public @interface QueueCallback {

    /* The type of callback. */;
    QueueCallbackType value() default QueueCallbackType.DYNAMIC;

}
```

Example usage:


##Annotation

```java
public class EventManagerImpl implements EventManager {

    @QueueCallback(QueueCallbackType.LIMIT)
    private void queueLimit() {



        if (messageCountSinceLastFlush > 100_000) {

            now = Timer.timer().now();
            sendMessages();
            return;
        }

        now = Timer.timer().now();
        long duration = now - lastFlushTime;

        if (duration > 50 && messageCountSinceLastFlush > 0) {
            sendMessages();
        }

    }



    @QueueCallback(QueueCallbackType.EMPTY)
    private void queueEmpty() {


        if (messageCountSinceLastFlush > 100) {

            now = Timer.timer().now();
            sendMessages();
            return;
        }



        now = Timer.timer().now();
        long duration = now - lastFlushTime;

        if (duration > 50 && messageCountSinceLastFlush > 0) {
            sendMessages();
        }

    }

```


##Interface
```java

public class StatService  implements QueueCallBackHandler {

    public void queueLimit() {
        now = Timer.timer().now();
        process();
    }

    public void queueEmpty() {
        now = Timer.timer().now();
        process();
    }

```

##Convention
```java
@RequestMapping("/myservice")
public class MyServiceQBit {


    void queueLimit() {
        ...
    }

    void queueEmpty() {
        ...
    }


```
