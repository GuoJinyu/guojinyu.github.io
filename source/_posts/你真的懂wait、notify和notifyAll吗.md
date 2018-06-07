title: 你真的懂wait、notify和notifyAll吗
date: 2018-06-03 00:24:00
update: 2018-06-03 00:24:00
categories: 技术
tags: [Java]
---

生产者消费者模型是我们学习多线程知识的一个经典案例，一个典型的生产者消费者模型如下：
```java
    public void produce() {
        synchronized (this) {
            while (mBuf.isFull()) {
                try {
                    wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            mBuf.add();
            notifyAll();
        }
    }

    public void consume() {
        synchronized (this) {
            while (mBuf.isEmpty()) {
                try {
                    wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            mBuf.remove();
            notifyAll();
        }

    }
```
这段代码很容易引申出来两个问题：**一个是wait()方法外面为什么是while循环而不是if判断，另一个是结尾处的为什么要用notifyAll()方法，用notify()行吗。**

很多人在回答第二个问题的时候会想当然的说notify()是唤醒一个线程，notifyAll()是唤醒全部线程，但是唤醒然后呢，不管是notify()还是notifyAll()，最终拿到锁的只会有一个线程，那它们到底有什么区别呢？

其实这是一个对象内部锁的调度问题，要回答这两个问题，首先我们要明白java中对象锁的模型，JVM会为一个使用内部锁（synchronized）的对象维护两个集合，**Entry Set**和**Wait Set**，也有人翻译为锁池和等待池，意思基本一致。

对于Entry Set：如果线程A已经持有了对象锁，此时如果有其他线程也想获得该对象锁的话，它只能进入Entry Set，并且处于线程的BLOCKED状态。

对于Wait Set：如果线程A调用了wait()方法，那么线程A会释放该对象的锁，进入到Wait Set，并且处于线程的WAITING状态。

还有需要注意的是，某个线程B想要获得对象锁，一般情况下有两个先决条件，一是对象锁已经被释放了（如曾经持有锁的前任线程A执行完了synchronized代码块或者调用了wait()方法等等），二是线程B已处于RUNNABLE状态。

那么这两类集合中的线程都是在什么条件下可以转变为RUNNABLE呢？

对于Entry Set中的线程，当对象锁被释放的时候，JVM会唤醒处于Entry Set中的某一个线程，这个线程的状态就从BLOCKED转变为RUNNABLE。

对于Wait Set中的线程，当对象的notify()方法被调用时，JVM会唤醒处于Wait Set中的某一个线程，这个线程的状态就从WAITING转变为RUNNABLE；或者当notifyAll()方法被调用时，Wait Set中的全部线程会转变为RUNNABLE状态。所有Wait Set中被唤醒的线程会被转移到Entry Set中。

然后，每当对象的锁被释放后，那些所有处于RUNNABLE状态的线程会共同去竞争获取对象的锁，最终会有一个线程（具体哪一个取决于JVM实现，队列里的第一个？随机的一个？）真正获取到对象的锁，而其他竞争失败的线程继续在Entry Set中等待下一次机会。

有了这些知识点作为基础，上述的两个问题就能解释的清了。

首先来看第一个问题，我们在调用wait()方法的时候，心里想的肯定是因为当前方法不满足我们指定的条件，因此执行这个方法的线程需要等待直到其他线程改变了这个条件并且做出了通知。那么为什么要把wait()方法放在循环而不是if判断里呢，其实答案显而易见，因为wait()的线程永远不能确定其他线程会在什么状态下notify()，所以必须在被唤醒、抢占到锁并且从wait()方法退出的时候再次进行指定条件的判断，以决定是满足条件往下执行呢还是不满足条件再次wait()呢。

就像在本例中，如果只有一个生产者线程，一个消费者线程，那其实是可以用if代替while的，因为线程调度的行为是开发者可以预测的，生产者线程只有可能被消费者线程唤醒，反之亦然，因此被唤醒时条件始终满足，程序不会出错。但是这种情况只是多线程情况下极为简单的一种，更普遍的是多个线程生产，多个线程消费，那么就极有可能出现唤醒生产者的是另一个生产者或者唤醒消费者的是另一个消费者，这样的情况下用if就必然会现类似过度生产或者过度消费的情况了，典型如IndexOutOfBoundsException的异常。所以所有的java书籍都会建议开发者**永远都要把wait()放到循环语句里面**。

然后来看第二个问题，既然notify()和notifyAll()最终的结果都是只有一个线程能拿到锁，那唤醒一个和唤醒多个有什么区别呢？

耐心看下面这个两个生产者两个消费者的场景，如果我们代码中使用了notify()而非notifyAll()，假设消费者线程1拿到了锁，判断buffer为空，那么wait()，释放锁；然后消费者2拿到了锁，同样buffer为空，wait()，也就是说此时Wait Set中有两个线程；然后生产者1拿到锁，生产，buffer满，notify()了，那么可能消费者1被唤醒了，但是此时还有另一个线程生产者2在Entry Set中盼望着锁，并且最终抢占到了锁，但因为此时buffer是满的，因此它要wait()；然后消费者1拿到了锁，消费，notify()；这时就有问题了，此时生产者2和消费者2都在Wait Set中，buffer为空，如果唤醒生产者2，没毛病；但如果唤醒了消费者2，因为buffer为空，它会再次wait()，这就尴尬了，万一生产者1已经退出不再生产了，没有其他线程在竞争锁了，只有生产者2和消费者2在Wait Set中互相等待，那传说中的死锁就发生了。

但如果你把上述例子中的notify()换成notifyAll()，这样的情况就不会再出现了，因为每次notifyAll()都会使其他等待的线程从Wait Set进入Entry Set，从而有机会获得锁。

其实说了这么多，一句话解释就是**之所以我们应该尽量使用notifyAll()的原因就是，notify()非常容易导致死锁**。当然notifyAll并不一定都是优点，毕竟一次性将Wait Set中的线程都唤醒是一笔不菲的开销，如果你能handle你的线程调度，那么使用notify()也是有好处的。

最后我把完整的测试代码放出来，供大家参考：

```java
import java.util.ArrayList;
import java.util.List;

public class Something {
    private Buffer mBuf = new Buffer();

    public void produce() {
        synchronized (this) {
            while (mBuf.isFull()) {
                try {
                    wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            mBuf.add();
            notifyAll();
        }
    }

    public void consume() {
        synchronized (this) {
            while (mBuf.isEmpty()) {
                try {
                    wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            mBuf.remove();
            notifyAll();
        }
    }

    private class Buffer {
        private static final int MAX_CAPACITY = 1;
        private List innerList = new ArrayList<>(MAX_CAPACITY);

        void add() {
            if (isFull()) {
                throw new IndexOutOfBoundsException();
            } else {
                innerList.add(new Object());
            }
            System.out.println(Thread.currentThread().toString() + " add");

        }

        void remove() {
            if (isEmpty()) {
                throw new IndexOutOfBoundsException();
            } else {
                innerList.remove(MAX_CAPACITY - 1);
            }
            System.out.println(Thread.currentThread().toString() + " remove");
        }

        boolean isEmpty() {
            return innerList.isEmpty();
        }

        boolean isFull() {
            return innerList.size() == MAX_CAPACITY;
        }
    }

    public static void main(String[] args) {
        Something sth = new Something();
        Runnable runProduce = new Runnable() {
            int count = 4;

            @Override
            public void run() {
                while (count-- > 0) {
                    sth.produce();
                }
            }
        };
        Runnable runConsume = new Runnable() {
            int count = 4;

            @Override
            public void run() {
                while (count-- > 0) {
                    sth.consume();
                }
            }
        };
        for (int i = 0; i < 2; i++) {
            new Thread(runConsume).start();
        }
        for (int i = 0; i < 2; i++) {
            new Thread(runProduce).start();
        }
    }
}
```

- 上面的栗子是正确的使用方式，输出的结果如下：

```
Thread[Thread-2,5,main] add
Thread[Thread-1,5,main] remove
Thread[Thread-3,5,main] add
Thread[Thread-0,5,main] remove
Thread[Thread-3,5,main] add
Thread[Thread-0,5,main] remove
Thread[Thread-2,5,main] add
Thread[Thread-1,5,main] remove

Process finished with exit code 0
```

- 如果把while改成if，结果如下，程序可能产生运行时异常：

``` 
Thread[Thread-2,5,main] add
Thread[Thread-1,5,main] remove
Thread[Thread-3,5,main] add
Thread[Thread-1,5,main] remove
Thread[Thread-3,5,main] add
Thread[Thread-1,5,main] remove
Exception in thread "Thread-0" Exception in thread "Thread-2" java.lang.IndexOutOfBoundsException
	at Something$Buffer.add(Something.java:42)
	at Something.produce(Something.java:16)
	at Something$1.run(Something.java:76)
	at java.lang.Thread.run(Thread.java:748)
java.lang.IndexOutOfBoundsException
	at Something$Buffer.remove(Something.java:52)
	at Something.consume(Something.java:30)
	at Something$2.run(Something.java:86)
	at java.lang.Thread.run(Thread.java:748)

Process finished with exit code 0
```

- 如果把notifyAll改为notify，结果如下，死锁，程序没有正常退出：

```
Thread[Thread-2,5,main] add
Thread[Thread-0,5,main] remove
Thread[Thread-3,5,main] add
```

另，转载请注明出处！文中若有什么错误希望大家探讨指正！
