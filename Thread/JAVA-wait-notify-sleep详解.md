# JAVA wait(), notify(),sleep详解

来源:[zyplus的专栏](http://blog.csdn.net/zyplus/article/details/6672775)

在JAVA中，是没有类似于PV操作、进程互斥等相关的方法的。JAVA的进程同步是通过synchronized()来实现的，需要说明的是，JAVA的synchronized()方法类似于操作系统概念中的互斥内存块，在JAVA中的Object类型中，都是带有一个内存锁的，在有线程获取该内存锁后，其它线程无法访问该内存，从而实现JAVA中简单的同步、互斥操作。明白这个原理，就能理解为什么synchronized(this)与synchronized(static XXX)的区别了，synchronized就是针对内存区块申请内存锁，this关键字代表类的一个对象，所以其内存锁是针对相同对象的互斥操作，而static成员属于类专有，其内存空间为该类所有成员共有，这就导致synchronized()对static成员加锁，相当于对类加锁，也就是在该类的所有成员间实现互斥，在同一时间只有一个线程可访问该类的实例。如果只是简单的想要实现在JAVA中的线程互斥，明白这些基本就已经够了。但如果需要在线程间相互唤醒的话就需要借助Object.wait(), Object.nofity()了

Obj.wait()，与Obj.notify()必须要与synchronized(Obj)一起使用，也就是wait,与notify是针对已经获取了Obj锁进行操作，从语法角度来说就是Obj.wait(),Obj.notify必须在synchronized(Obj){...}语句块内。从功能上来说wait就是说线程在获取对象锁后，主动释放对象锁，同时本线程休眠。直到有其它线程调用对象的notify()唤醒该线程，才能继续获取对象锁，并继续执行。相应的notify()就是对对象锁的唤醒操作。但有一点需要注意的是notify()调用后，并不是马上就释放对象锁的，而是在相应的synchronized(){}语句块执行结束，自动释放锁后，JVM会在wait()对象锁的线程中随机选取一线程，赋予其对象锁，唤醒线程，继续执行。这样就提供了在线程间同步、唤醒的操作。Thread.sleep()与Object.wait()二者都可以暂停当前线程，释放CPU控制权，主要的区别在于Object.wait()在释放CPU同时，释放了对象锁的控制。

单单在概念上理解清楚了还不够，需要在实际的例子中进行测试才能更好的理解。对Object.wait()，Object.notify()的应用最经典的例子，应该是三线程打印ABC的问题了吧，这是一道比较经典的面试题，题目要求如下：

建立三个线程，A线程打印10次A，B线程打印10次B,C线程打印10次C，要求线程同时运行，交替打印10次ABC。这个问题用Object的wait()，notify()就可以很方便的解决。代码如下：

```
public class MyThreadPrinter2 implements Runnable {   
  
    private String name;   
    private Object prev;   
    private Object self;   
  
    private MyThreadPrinter2(String name, Object prev, Object self) {   
        this.name = name;   
        this.prev = prev;   
        this.self = self;   
    }   
  
    @Override  
    public void run() {   
        int count = 10;   
        while (count > 0) {   
            synchronized (prev) {   
                synchronized (self) {   
                    System.out.print(name);   
                    count--;  
                    
                    self.notify();   
                }   
                try {   
                    prev.wait();   
                } catch (InterruptedException e) {   
                    e.printStackTrace();   
                }   
            }   
  
        }   
    }   
  
    public static void main(String[] args) throws Exception {   
        Object a = new Object();   
        Object b = new Object();   
        Object c = new Object();   
        MyThreadPrinter2 pa = new MyThreadPrinter2("A", c, a);   
        MyThreadPrinter2 pb = new MyThreadPrinter2("B", a, b);   
        MyThreadPrinter2 pc = new MyThreadPrinter2("C", b, c);   
           
           
        new Thread(pa).start();
        new Thread(pb).start();
        new Thread(pc).start();    }   
}  

```

先来解释一下其整体思路，从大的方向上来讲，该问题为三线程间的同步唤醒操作，主要的目的就是ThreadA->ThreadB->ThreadC->ThreadA循环执行三个线程。为了控制线程执行的顺序，那么就必须要确定唤醒、等待的顺序，所以每一个线程必须同时持有两个对象锁，才能继续执行。一个对象锁是prev，就是前一个线程所持有的对象锁。还有一个就是自身对象锁。主要的思想就是，为了控制执行的顺序，必须要先持有prev锁，也就前一个线程要释放自身对象锁，再去申请自身对象锁，两者兼备时打印，之后首先调用self.notify()释放自身对象锁，唤醒下一个等待线程，再调用prev.wait()释放prev对象锁，终止当前线程，等待循环结束后再次被唤醒。运行上述代码，可以发现三个线程循环打印ABC，共10次。程序运行的主要过程就是A线程最先运行，持有C,A对象锁，后释放A,C锁，唤醒B。线程B等待A锁，再申请B锁，后打印B，再释放B，A锁，唤醒C，线程C等待B锁，再申请C锁，后打印C，再释放C,B锁，唤醒A。看起来似乎没什么问题，但如果你仔细想一下，就会发现有问题，就是初始条件，三个线程按照A,B,C的顺序来启动，按照前面的思考，A唤醒B，B唤醒C，C再唤醒A。但是这种假设依赖于JVM中线程调度、执行的顺序。具体来说就是，在main主线程启动ThreadA后，需要在ThreadA执行完，在prev.wait()等待时，再切回线程启动ThreadB，ThreadB执行完，在prev.wait()等待时，再切回主线程，启动ThreadC，只有JVM按照这个线程运行顺序执行，才能保证输出的结果是正确的。而这依赖于JVM的具体实现。考虑一种情况，如下：如果主线程在启动A后，执行A，过程中又切回主线程，启动了ThreadB,ThreadC，之后，由于A线程尚未释放self.notify，也就是B需要在synchronized(prev)处等待，而这时C却调用synchronized(prev)获取了对b的对象锁。这样，在A调用完后，同时ThreadB获取了prev也就是a的对象锁，ThreadC的执行条件就已经满足了，会打印C，之后释放c,及b的对象锁，这时ThreadB具备了运行条件，会打印B，也就是循环变成了ACBACB了。这种情况，可以通过在run中主动释放CPU，来进行模拟。代码如下：

```
public void run() {   
    int count = 10;   
    while (count > 0) {   
        synchronized (prev) {   
            synchronized (self) {   
                System.out.print(name);   
                count--;  
                try{
                Thread.sleep(1);
                }
                catch (InterruptedException e){
                 e.printStackTrace();
                }
                
                self.notify();   
            }   
            try {   
                prev.wait();   
            } catch (InterruptedException e) {   
                e.printStackTrace();   
            }   
        }   

    }   
}

```

运行后的打印结果就变成了ACBACB了。为了避免这种与JVM调度有关的不确定性。需要让A,B,C三个线程以确定的顺序启动，最终代码如下：

```
  
public class MyThreadPrinter2 implements Runnable {   
  
    private String name;   
    private Object prev;   
    private Object self;   
  
    private MyThreadPrinter2(String name, Object prev, Object self) {   
        this.name = name;   
        this.prev = prev;   
        this.self = self;   
    }   
  
    @Override  
    public void run() {   
        int count = 10;   
        while (count > 0) {   
            synchronized (prev) {   
                synchronized (self) {   
                    System.out.print(name);   
                    count--;  
                    try{
                    Thread.sleep(1);
                    }
                    catch (InterruptedException e){
                     e.printStackTrace();
                    }
                    
                    self.notify();   
                }   
                try {   
                    prev.wait();   
                } catch (InterruptedException e) {   
                    e.printStackTrace();   
                }   
            }   
  
        }   
    }   
  
    public static void main(String[] args) throws Exception {   
        Object a = new Object();   
        Object b = new Object();   
        Object c = new Object();   
        MyThreadPrinter2 pa = new MyThreadPrinter2("A", c, a);   
        MyThreadPrinter2 pb = new MyThreadPrinter2("B", a, b);   
        MyThreadPrinter2 pc = new MyThreadPrinter2("C", b, c);   
           
           
        new Thread(pa).start();
        Thread.sleep(10);
        new Thread(pb).start();
        Thread.sleep(10);
        new Thread(pc).start();
        Thread.sleep(10);
    }   
}  

```

这样才可以完美的解决该问题。通过这个例子也是想说明一下，很多理论、概念如Obj.wait(),Obj.notify()等，理解起来，比较简单，但是在实际的应用当中，这里却是往往出现问题的地方。需要更加深入的理解。并在解决问题的过程中不断加深对概念的掌握。

