## Java线程中断机制

线程中断即线程运行过程中被其他线程给打断了，线程中断相关的方法有：

- 1、java.lang.Thread#interrupt

中断目标线程，给目标线程发一个中断信号，线程被打上中断标记。

- 2.java.lang.Thread#isInterrupted()

判断目标线程是否被中断，不会清除中断标记。

- 3.java.lang.Thread#interrupted
Thread中的一个静态方法，判断目标线程是否被中断，如果是被标记了中断状态，会清除中断标记。

interrupt方法中断线程仅仅是设置了一个中断标记，如果在线程run方法中不去相应中断，则中断标记不会对线程产生任何影响。

如下代码，thread线程并不会被中断

```java
    public static void testInterrupt() {
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    System.out.println("thread " + Thread.currentThread().getName() + " is running");
                }
            }
        });
        thread.start();
        thread.interrupt();
    }
```
如果要中断线程，则需要判断interrupt标记位，然后结束线程

```java
    public static void testInterrupt() {
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    if (Thread.currentThread().isInterrupted()) {
                        System.out.println("thread " + Thread.currentThread().getName() + " is interrupted");
                        return;
                    }
                    System.out.println("thread " + Thread.currentThread().getName() + " is running");
                }
            }
        });
        thread.start();
        thread.interrupt();
    }
```
当让线程进入休眠状态后，中断标记失效，代码如下：

```java
 public static void testInterrupt2() throws InterruptedException {
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    if (Thread.currentThread().isInterrupted()) {
                        System.out.println("thread " + Thread.currentThread().getName() + " is interrupted");
                        return;
                    }
                    System.out.println("thread " + Thread.currentThread().getName() + " is running");
                    try {
                        Thread.sleep(3000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
        thread.start();
        Thread.sleep(2000);
        thread.interrupt();
    }
```

输出结果如下：

```java
thread Thread-0 is running
java.lang.InterruptedException: sleep interrupted
	at java.base/java.lang.Thread.sleep(Native Method)
	at thread.InterruptDemo$2.run(InterruptDemo.java:33)
	at java.base/java.lang.Thread.run(Thread.java:834)
thread Thread-0 is running
thread Thread-0 is running
thread Thread-0 is running
thread Thread-0 is running
thread Thread-0 is running
thread Thread-0 is running
thread Thread-0 is running
thread Thread-0 is running
...

```

可以看到线程抛出了一个中断异常，之后线程仍然在执行。

sleep状态的线程在调用interrupt后会抛出InterruptedException，并清除interrupt标记，因此线程会继续运行。如果需要打断sleep状态的线程，需要捕获InterruptedException异常后再次调用interrupt方法。如下:
```java
  public static void testInterrupt2() throws InterruptedException {
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    if (Thread.currentThread().isInterrupted()) {
                        System.out.println("thread " + Thread.currentThread().getName() + " is interrupted");
                        return;
                    }
                    System.out.println("thread " + Thread.currentThread().getName() + " is running");
                    try {
                        Thread.sleep(3000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                        Thread.currentThread().interrupt();
                    }
                }
            }
        });
        thread.start();
        Thread.sleep(2000);
        thread.interrupt();
    }
```

输出结果如下：
```java
thread Thread-0 is running
java.lang.InterruptedException: sleep interrupted
	at java.base/java.lang.Thread.sleep(Native Method)
	at thread.InterruptDemo$2.run(InterruptDemo.java:33)
	at java.base/java.lang.Thread.run(Thread.java:834)
thread Thread-0 is interrupted
```
