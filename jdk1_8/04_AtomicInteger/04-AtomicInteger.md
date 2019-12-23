```java
import java.util.concurrent.atomic.AtomicInteger;
```

# 前言

AtomicInteger，见名知意，该类可以保障Integer操作的原子性。

AtomicInteger是一个支持原子操作的Integer类，它提供了原子自增方法，原子自减方法以及原子赋值方法等。其底层是基于`volatile`和`CAS`实现的，其中volatile保证其并发下的内存可见性，CAS算法保证其并发下的原子性。

首先举个简单的例子，看看AtomicInteger在并发自增情况下保障变量的原子性：

- 在不使用AtomicInteger的情况下，多线程情况下直接将Integer类型的变量进行自增；

```java
public class AtomicIntegerStu {
    
    private static volatile Integer a = 0;
    
    public static void main(String[] args) {
        Thread[] threads = new Thread[5];
        for (int i = 0;i<5;i++){
            threads[i] = new Thread(() ->{
               for (int j = 0 ; j<10; j++){
                   a++;
                   System.out.println(a);
                   try {
                       Thread.sleep(500);
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   }
               }
            });
            threads[i].start();
        }
    }
}
```

理想情况下5个线程对一个Integer类型变量a进行10次操作，那a的值最后应该是50，结果是不是呢？看看控制台打印的：

```java
26
30
29
28
27
31
31
33
31
32
34
36
35
34
34

Process finished with exit code 0
```

可以看到，非但没有达到预期结果50，在操作过程中还有重复值出现。

- 使用AtomicInteger的情况下，多线程情况下直接将Integer类型的变量进行自增；

```java
public class AtomicIntegerStu {

    static AtomicInteger atomicInteger = new AtomicInteger(0);

    public static void main(String[] args) {
        Thread[] threads = new Thread[5];
        for (int i = 0;i<5;i++){
            threads[i] = new Thread(() ->{
               for (int j = 0 ; j<10; j++){
                   System.out.println(atomicInteger.incrementAndGet());
                   try {
                       Thread.sleep(500);
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   }
               }
            });
            threads[i].start();
        }
    }
}
```

```java
45
44
41
42
46
48
47
49
50

Process finished with exit code 0
```

从控制台打印的结果可以看到，使用AtomicInteger后，多线程条件下能够保证atomicInteger的自增原子性操作。

那为什么a++就不能保证变量的原子性操作呢？对于a++的操作，其实可以分解为3步：

1. 从主存中读取a的值；
2. 对a进行+1操作；
3. 将a的新值刷新到主存；

在单线程条件下对a进行自增操作可以保障以上3步的顺序性，但是在多线程情况下进行操作就不能保证每一步没有其它线程操作，比如对a进行+1操作，但是还没来得及将其新值刷新到主存，其它线程就进来读取了a在主存中的旧值，然后进行相应的自增操作，这样肯定会出现问题。AtomicInteger正是用来**避免**这种情况发生。

# 源码阅读

前面例子中使用了AtomicInteger的有参构造函数和`incrementAndGet()`方法。

## 1. AtomicInteger的构造函数；

```java
	// 获取Unsafe的实例
    private static final Unsafe unsafe = Unsafe.getUnsafe();
	// 标识value字段的偏移量
    private static final long valueOffset;   
	// 静态代码块，通过unsafe获取value的偏移量
	static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
	private volatile int value;    

    /**
     * Creates a new AtomicInteger with the given initial value.
     * 根据指定值创建一个AtomicInteger对象
     * @param initialValue the initial value
     */
    public AtomicInteger(int initialValue) {
        value = initialValue;
    }

    /**
     * Creates a new AtomicInteger with initial value {@code 0}.
     * 创建一个默认值为0的AtomicInteger对象
     */
    public AtomicInteger() {
    }
```

- 可以看到`value`使用`volatile`关键字修饰，volatile主要作用就是保证变量在内存中的可见性，以及防止指令重排，在AtomicInteger中的作用也是基于前述两点。
- 在静态代码块中使用了`Unsafe.objectFieldOffset()`来获取value字段在类中的偏移量，用于之后的`CAS`操作使用。

## 2. AtomicInteger的`get`和`set`

### 2.1 基本的get和set方法。

```java
    /**
     * Gets the current value.
     *
     * @return the current value
     */
    public final int get() {
        return value;
    }

    /**
     * Sets to the given value.
     *
     * @param newValue the new value
     */
    public final void set(int newValue) {
        value = newValue;
    }
```

### 2.2 常用get和set方法

- 获取当前的值，并设置新的值；

```java
    /**
     * Atomically sets to the given value and returns the old value.
     *
     * @param newValue the new value
     * @return the previous value
     */
    public final int getAndSet(int newValue) {
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }
```

- 如果输入的数值等于预期值，则以原子方式将该值设置为输入值（update）；

```java
    /**
     * Atomically sets the value to the given updated value
     * if the current value {@code ==} the expected value.
     *
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful. False return indicates that
     * the actual value was not equal to the expected value.
     */
    public final boolean compareAndSet(int expect, int update) {
        //参数依次为：操作的对象、对象中字段的偏移量、原来的值（期望的值）、要修改的值
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }

	//Unsafe中该方法源码
	public native void putOrderedInt(Object var1, long var2, int var4);
```

`Unsafe.putOrderedInt()`方法是native方法，底层使用C/C++编写，主要是调用CPU的CAS指令实现，能够保证只有当对应偏移量处的字段值是期望值时才更新。通过CPU的CAS指令可以保证比较和更新步骤作为一个整体，避免对值进行操作时比较和赋值步骤中变量值不一致的问题。

- 获取当前的值，并自增；

```java
    /**
     * Atomically increments by one the current value.
     *
     * @return the previous value
     */
    public final int getAndIncrement() {
        //参数依次为：操作的对象、对象中字段的偏移量、要增加的值
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }

	//Unsafe中源码
    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```

`Unsafe.getAndInt()`方法:

1.首先获取当前的值；

2.然后再调用`compareAndSwapInt()`尝试更新对应偏移量处的值：

​	Ⅰ 如果更新成功，则跳出循环；

​	Ⅱ 如果更新失败，则一直尝试直到成功为止；

- 获取当前的值，并自增；

```java
    /**
     * Atomically decrements by one the current value.
     *
     * @return the previous value
     */
    public final int getAndDecrement() {
        return unsafe.getAndAddInt(this, valueOffset, -1);
    }
```

- 获取当前的值，并加上预期的值；

```java
    /**
     * Atomically adds the given value to the current value.
     *
     * @param delta the value to add
     * @return the previous value
     */
    public final int getAndAdd(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta);
    }
```

- 最终设置为newValue，使用lazySet设置之后可能导致其它线程在之后一小段时间内还是可以读到旧的值；

```java
    /**
     * Eventually sets to the given value.
     *
     * @param newValue the new value
     * @since 1.6
     */
    public final void lazySet(int newValue) {
        unsafe.putOrderedInt(this, valueOffset, newValue);
    }
```



- 本文对于源码阅读记录基于`JDK1.8`，个人浅见，如有不妥，敬请指正。*
- *文中 "`//TODO`"处代表个人暂时未懂或标示之后会学习的地方，如有高见，敬请指教。*
- 记录中涉及参考及引用部分在文末都会指明原链接，侵删。



# 参考及引用：

[**详解 java 并发原子类 AtomicInteger（基于 jdk1.8 源码分析）**](https://juejin.im/post/5da80de7f265da5b8529279c)

[**死磕 java 并发包之 AtomicInteger 源码分析**](https://juejin.im/post/5cd05e2ef265da0354032107)

