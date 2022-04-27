# Java多线程

## Java主线程结束和子线程结束之间的关系

> `https://blog.csdn.net/aitangyong/article/details/16858273`

1. `Main`是个非守护进程，不能设置成守护进行

    ```java
    public class MainTest
    {
        // 会抛出：Exception in thread "main"     java.lang.IllegalThreadStateException
        public static void main(String[] args)
        {
            System.out.println(" parent thread begin ");
            Thread.currentThread().setDaemon(true);
        }
    }
    ```

2. `Main`线程结束，其他线程一样可以正常运行
3. `Main`线程结束，其他线程也可以立刻结束，当且仅当这些子线程都是守护线程
