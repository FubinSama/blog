# Executor线程执行体

## Executor接口

```java
public interface Executor {
    /**
     * @param command the runnable task
     * @throws RejectedExecutionException if this task cannot be accepted for execution
     * @throws NullPointerException if command is null
     */
    void execute(Runnable command);
}
```

Executor线程执行体可以管理和执行通过execute方法提交的Runnable任务

这些任务可能直接在当前线程执行

```java
class DirectExecutor implements Executor {
    public void execute(Runnable r) {
        r.run();
    }
}
```

也可能交给了新的线程执行

```java
class ThreadPerTaskExecutor implements Executor {
    public void execute(Runnable r) {
        new Thread(r).start();
    }
}
```

也可能对任务的调度方式和时间施加了某种限制。如下的代码将提交的请求按顺序序列化给了另一个执行器执行

```java
class SerialExecutor implements Executor {
    final Queue<Runnable> tasks = new ArrayDeque<>();
    final Executor executor;
    Runnable active;
    SerialExecutor(Executor executor) {
        this.executor = executor;
    }
    public synchronized void execute(Runnable r) {
        tasks.add(() -> {
            try {
                r.run();
            }
            finally {
                scheduleNext();
            }
        }
        );
        if (active == null) {
            scheduleNext();
        }
    }
    protected synchronized void scheduleNext() {
        if ((active = tasks.poll()) != null) {
            executor.execute(active);
        }
    }
}
```

## ExecutorService接口

```java
public interface ExecutorService extends Executor {

    void shutdown(); // 不会立即销毁 ExecutorService 实例，而是首先让 ExecutorService 停止接受新任务，并在所有正在运行的线程完成当前工作后关闭。

    List<Runnable> shutdownNow(); // 尝试立即销毁 ExecutorService 实例，但并不能保证所有正在运行的线程将同时停止。该方法会返回等待处理的任务列表，由开发人员自行决定如何处理这些任务。

    boolean isShutdown(); // Returns true if this executor has been shut down.

    boolean isTerminated(); // Returns true if all tasks have completed following shut down. 

    boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException; // 在shutdown请求发起后，阻塞知道所有任务执行完成或者超时或者当前线程被中断

    <T> Future<T> submit(Callable<T> task);

    <T> Future<T> submit(Runnable task, T result);

    Future<?> submit(Runnable task);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException;

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)throws InterruptedException, ExecutionException, TimeoutException;
}
```

ExecutorService接口扩展了Executor接口，除了最基本的执行Runnable任务，它还提供了执行器生命周期的管理，并对`void execute(Runnable command);`进行了扩展，提供了返回`Future<T>`类型结果的`submit`方法和等待任何一个任务执行完成就返回的`invokeAny`方法和等待全部任务执行完成的`invokeAll`方法。

一个简单的tcp服务器实现：

```java
class NetworkService implements Runnable {
    private final ServerSocket serverSocket;
    private final ExecutorService pool;
    public NetworkService(int port, int poolSize) throws IOException {
        serverSocket = new ServerSocket(port);
        pool = Executors.newFixedThreadPool(poolSize);
    }
    public void run() { // run the service
        try {
            for (;;) {
                pool.execute(new Handler(serverSocket.accept()));
            }
        }
        catch (IOException ex) {
            pool.shutdown();
        }
    }
}
class Handler implements Runnable {
    private final Socket socket;
    Handler(Socket socket) {
        this.socket = socket;
    }
    public void run() {
        // read and service request on socket
    }
}
```

ExecutorService的shutdown最佳实践：

```java
void shutdownAndAwaitTermination(ExecutorService pool) {
    pool.shutdown(); // Disable new tasks from being submitted    
    try {
        // Wait a while for existing tasks to terminate     
        if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
            pool.shutdownNow(); // Cancel currently executing tasks        
            // Wait a while for tasks to respond to being cancelled       
            if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
                System.err.println("Pool did not terminate");
            }
        }
    } catch (InterruptedException ie) {
        // (Re-)Cancel if current thread also interrupted      
        pool.shutdownNow();
        // Preserve interrupt status     
        Thread.currentThread().interrupt();
    }
}
```
