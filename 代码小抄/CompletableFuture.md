# CompletableFuture



```java
/**
 * 将抛出异常的方法适配CompletableFuture，并快速组装，实现同步转异步
 */
public class Main {
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        try {
            createAllOfCompletableFuture(executorService, new NamedExceptionRunnable("task1", () -> {
                System.out.println("task1");
                xxx();
            }), new NamedExceptionRunnable("task2", () -> {
                System.out.println("task2");
                xxx();
            }), new NamedExceptionRunnable("task3", () -> {
                System.out.println("task3");
                xxx();
            }), new NamedExceptionRunnable("task4", () -> {
                System.out.println("task4");
                xxx();
            }), new NamedExceptionRunnable("task5", () -> {
                System.out.println("task5");
                xxx();
            }), new NamedExceptionRunnable("task6", () -> {
                System.out.println("task6");
                xxx();
            }), new NamedExceptionRunnable("task7", () -> {
                System.out.println("task7");
                xxx();
            }), new NamedExceptionRunnable("task8", () -> {
                System.out.println("task8");
                xxx();
            }), new NamedExceptionRunnable("task9", () -> {
                System.out.println("task9");
                xxx();
            })).join();
        } finally {
            executorService.shutdown();
        }
    }
    
    private static void xxx() throws Exception {
        TimeUnit.SECONDS.sleep(1);
        System.out.println("xxx");
        TimeUnit.SECONDS.sleep(1);
    }

    private static class NamedExceptionRunnable {
        private final String taskName;
        private final ExceptionRunnable runnable;
        
        public NamedExceptionRunnable(String taskName, ExceptionRunnable runnable) {
            this.taskName = taskName;
            this.runnable = runnable;
        }
        
        public String getTaskName() { return taskName; }
        
        public ExceptionRunnable getRunnable() { return runnable; }
    }
    
    @FunctionalInterface
    private interface ExceptionRunnable {
        void run() throws Exception;
    }

    private static CompletableFuture<Void> createAllOfCompletableFuture(Executor executor, NamedExceptionRunnable... nrs) {
        return CompletableFuture.allOf(
                Arrays.stream(nrs)
                        .map(nr -> createCompletableFuture(nr.getTaskName(), nr.getRunnable(), executor))
                        .toArray(CompletableFuture[]::new)
        );
    }
    
    private static CompletableFuture<Void> createCompletableFuture(String taskName, ExceptionRunnable runnable, Executor executor) {
        return CompletableFuture.runAsync(() -> {
            StopWatch stopWatch = new StopWatch();
            try {
                stopWatch.start();
                runnable.run();
            } catch (Exception e) {
                throw new RuntimeException(e.getMessage());
            } finally {
                stopWatch.stop();
                System.out.println(taskName + "耗时:" + stopWatch.getTotalTimeMillis());
            }
        }, executor);
    }
    
}
```

