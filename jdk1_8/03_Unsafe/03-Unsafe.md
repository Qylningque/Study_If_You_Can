```java
import sun.misc.Unsafe;
```

# 前言

Unsafe是位于sun.misc包下的一个类，主要提供一些用于执行低级别、不安全操作的方法，如直接访问系统内存资源、自主管理内存资源等，这些方法在提升Java运行效率、增强Java语言底层资源操作能力方面起到了很大的作用。但由于Unsafe类使Java拥有了类似C语言指针一样操作内存空间的能力，这无疑也增加了程序发生相关指针问题的风险。在程序中过度、不正确使用Unsafe类会使得程序出错的概率变大，使得Java不在那么“安全”，因此对Unsafe的使用一定要慎重。

Unsafe提供了底层访问的机制，这种机制仅供Java核心类库使用，而不应该被普通用户使用。

# 基本介绍

## 1 通过反射获取Unsafe实例

```java
public final class Unsafe {
  
  private static final Unsafe theUnsafe;

  private Unsafe() {
  }
  @CallerSensitive
  public static Unsafe getUnsafe() {
    Class var0 = Reflection.getCallerClass();
    
    if(!VM.isSystemDomainLoader(var0.getClassLoader())) {    
      throw new SecurityException("Unsafe");
    } else {
      return theUnsafe;
    }
  }
}
```

上示为Unsafe源码一部分，可以得知Unsafe类是一单例实现，提供静态方法`getUnsafe()`获取Unsafe的实例，当且仅当调用`getUnsafe()`方法的类为引导类加载器所加载时才合法，否则抛出`SecurityException`异常。

上示源码最后`return theUnsafe;`，那是否可以通过theUnsafe获取Unsafe的一个实例呢？继续看源码：

```java
private static final Unsafe theUnsafe;

static {
        registerNatives();
        Reflection.registerMethodsToFilter(Unsafe.class, new String[]{"getUnsafe"});
        theUnsafe = new Unsafe();
        ARRAY_BOOLEAN_BASE_OFFSET = theUnsafe.arrayBaseOffset(boolean[].class);
        ARRAY_BYTE_BASE_OFFSET = theUnsafe.arrayBaseOffset(byte[].class);
        ARRAY_SHORT_BASE_OFFSET = theUnsafe.arrayBaseOffset(short[].class);
        ARRAY_CHAR_BASE_OFFSET = theUnsafe.arrayBaseOffset(char[].class);
        ARRAY_INT_BASE_OFFSET = theUnsafe.arrayBaseOffset(int[].class);
        ARRAY_LONG_BASE_OFFSET = theUnsafe.arrayBaseOffset(long[].class);
        ARRAY_FLOAT_BASE_OFFSET = theUnsafe.arrayBaseOffset(float[].class);
        ARRAY_DOUBLE_BASE_OFFSET = theUnsafe.arrayBaseOffset(double[].class);
        ARRAY_OBJECT_BASE_OFFSET = theUnsafe.arrayBaseOffset(Object[].class);
        ARRAY_BOOLEAN_INDEX_SCALE = theUnsafe.arrayIndexScale(boolean[].class);
        ARRAY_BYTE_INDEX_SCALE = theUnsafe.arrayIndexScale(byte[].class);
        ARRAY_SHORT_INDEX_SCALE = theUnsafe.arrayIndexScale(short[].class);
        ARRAY_CHAR_INDEX_SCALE = theUnsafe.arrayIndexScale(char[].class);
        ARRAY_INT_INDEX_SCALE = theUnsafe.arrayIndexScale(int[].class);
        ARRAY_LONG_INDEX_SCALE = theUnsafe.arrayIndexScale(long[].class);
        ARRAY_FLOAT_INDEX_SCALE = theUnsafe.arrayIndexScale(float[].class);
        ARRAY_DOUBLE_INDEX_SCALE = theUnsafe.arrayIndexScale(double[].class);
        ARRAY_OBJECT_INDEX_SCALE = theUnsafe.arrayIndexScale(Object[].class);
        ADDRESS_SIZE = theUnsafe.addressSize();
    }
```

在Unsafe源码中有一个静态方法，该方法中创建了一个Unsafe的实例`theUnsafe = new Unsafe();`

具体通过反射获取单例对象`theUnsafe`:

```java
import sun.misc.Unsafe;
import java.lang.reflect.Field;
public class UnsafeStu {

    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
        Field f = Unsafe.class.getDeclaredField("theUnsafe");
        f.setAccessible(true);
        Unsafe unsafe = (Unsafe) f.get(f);
        System.out.println(unsafe.getClass());
    }

}
```

控制台打印：

```java
class sun.misc.Unsafe
```

## 2 使用Unsafe实例化一个类

```java
class User{
    int age;
    public User(){
        this.age = 10;
    }

    public int getAge(){
        return this.age;
    }
}
public class UnsafeStu {

    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, InstantiationException {
        Field f = Unsafe.class.getDeclaredField("theUnsafe");
        f.setAccessible(true);
        Unsafe unsafe = (Unsafe) f.get(null);
        
        User user = new User();
        System.out.println(user.getAge());
        
        User user1 = (User) unsafe.allocateInstance(User.class);
        System.out.println(user1.getAge());
    }

}
```

控制台打印：

```java
10
0
```

通过Unsafe实例化的User类的对象user1，为什么age属性为0呢？

```java
public native Object allocateInstance(Class<?> var1) throws InstantiationException;
```

从Unsafe的`allocateInstance()`得知，该方法只会给对象分配内存，并不会调用对象的构造方法，所以在User类的实例化user1对象并没有根据无参构造函数中定义的`this.age = 10;`为age属性赋值，而是使用的int类型的默认值0。

那Unsafe实例化的类，能不能对类中的属性进行修改呢？来试一试：

```java
public class UnsafeStu {

    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, InstantiationException {
        Field f = Unsafe.class.getDeclaredField("theUnsafe");
        f.setAccessible(true);
        Unsafe unsafe = (Unsafe) f.get(null);

        User user1 = (User) unsafe.allocateInstance(User.class);
        System.out.println(user1.getAge());
        //通过反射获得age属性
        Field age = user1.getClass().getDeclaredField("age");
        unsafe.putInt(user1,unsafe.objectFieldOffset(age),20);
        System.out.println(user1.getAge());
    }

}
class User{
    int age;
    public User(){
        this.age = 10;
    }

    public int getAge(){
        return this.age;
    }
}
```

控制台打印：

```java
0
20
```

由上可知，通过反射获取到user1的age属性后，就可以通过Unsafe将其值更改。

为什么能修改呢？因为Unsafe可以直接操作内存，比如上述的要修改user1对象的age属性，就要先获得age属性在内存中的偏移量，Unsafe就提供了得到某个属性在对象中的偏移量的方法：`unsafe.objectFieldOffset()`

```java
public native long objectFieldOffset(Field var1);
```

## 3 Unsafe怎么操作内存呢？

Unsafe可以直接操作堆外内存的分配、拷贝、释放和给定地址值操作等方法。

```java
public native long allocateMemory(long bytes);

public native long reallocateMemory(long address, long bytes);

public native void freeMemory(long address);

public native void setMemory(Object o, long offset, long bytes, byte value);

public native void copyMemory(Object srcBase, long srcOffset, Object destBase, long destOffset, long bytes);

public native Object getObject(Object o, long offset);

public native void putObject(Object o, long offset, Object x);

public native byte getByte(long address);

public native void putByte(long address, byte x);
```

通常，在Java中创建的对象都处于堆内内存（heap）中，堆内内存是由JVM所管控的Java进程内存，并且它们遵循JVM的内存管理机制，JVM会采用垃圾回收机制统一管理堆内存。

与堆内内存相对的是堆外内存，存在于JVM管控之外的内存区域，Java中对堆外内存的操作，依赖于Unsafe提供的操作堆外内存的native方法。

### 为什么Unsafe使用堆外内存呢？

- **对垃圾回收停顿的改善**。由于堆外内存是直接受操作系统管理而不是JVM，所以在使用堆外内存时，可以保持较小的堆内内存规模，从而在GC时减少回收停顿对于应用的影响；
- **提升程序I/O操作的性能**。通常在I/O通信中，会存在堆内内存到堆外内存的数据拷贝操作，对于需要频繁进行内存间数据拷贝且生命周期较短的暂存数据，都建议存储到堆外内存；

使用Unsafe的`allocateMemory()`可以直接在堆外分配内存，这个内存不收JVM管控，因此需要调用`freeMemory()`方法手动释放。

**典型应用**：`DirectByteBuffer`是Java用于实现堆外内存的一个重要类，通常用在通信过程中做缓冲池。`DirectByteBuffer`对于堆外内存的创建、使用、销毁等逻辑均有Unsafe提供的堆外内存API来实现。

![img](https://p0.meituan.net/travelcube/5eb082d2e4baf2d993ce75747fc35de6486751.png)

## 4 Unsafe的CAS相关操作

```java
public final native boolean compareAndSwapObject(Object o, long offset,  Object expected, Object update);

public final native boolean compareAndSwapInt(Object o, long offset, int expected,int update);
  
public final native boolean compareAndSwapLong(Object o, long offset, long expected, long update);
```

上述部分为Unsafe的`CAS`（`CompareAndSwap`）相关操作。

`CAS`，即为比较并替换，是实现并发算法时常用到的一种技术。CAS操作包含三个操作数：

- 内存位置
- 预期原值
- 新值

执行`CAS`操作的时候，将内存位置的值与预期原值比较，如果相匹配，那么处理器会自动将该值更新为新值，否则，处理器不做任何操作。

`CAS`是一条CPU的原子指令（cmpxchg指令），不会造成所谓的数据不一致问题，Unsafe提供的`CAS`方法底层实现即为CPU指令cmpxchg。

**典型应用**：`java.util.concurrent.atomic` 相关类、`Java AQS`、`CurrentHashMap`等。

## 5 Unsafe的线程调度相关

这部分，包括线程挂起、恢复、锁机制等方法。

```java
public native void unpark(Object thread);

public native void park(boolean isAbsolute, long time);

@Deprecated
public native void monitorEnter(Object o);

@Deprecated
public native void monitorExit(Object o);

@Deprecated
public native boolean tryMonitorEnter(Object o);
```

当一个线程正在等待某个操作时，JVM调用Unsafe的`park()`方法来`阻塞（挂起）`此线程，调用该方法后，线程将一直阻塞直到超时或者中断等条件出现；

当阻塞的线程需要再次运行时，JVM调用Unsafe的`unpark()`来`唤醒（恢复）`此线程；

**典型应用**：Java 锁和同步器框架的核心类`AbstractQueuedSynchronizer`，就是通过调用`LockSupport.park()`和`LockSupport.unpark()`实现线程的阻塞和唤醒的，而`LockSupport`的`park`和`unpark`方法实际是调用Unsafe的`park`和`unpark`方法来实现。

## 6 Unsafe的Class相关操作

主要提供Class和它的静态字段的操作相关方法，包含静态字段内存定位、定义类、定义匿名类、校验&确保初始化等。

```java
public native long staticFieldOffset(Field f);

public native Object staticFieldBase(Field f);

public native boolean shouldBeInitialized(Class<?> c);

public native void ensureClassInitialized(Class<?> c);

public native Class<?> defineClass(String name, byte[] b, int off, int len, ClassLoader loader, ProtectionDomain protectionDomain);

public native Class<?> defineAnonymousClass(Class<?> hostClass, byte[] data, Object[] cpPatches);

```

**典型应用**：Java 8 开始，JDK 使用 `invokedynamic` 及 `VM Anonymous Class` 结合来实现 Java 语言层面上的 Lambda 表达式。

## 7 Unsafe的对象操作

主要包含对象成员属性相关操作及非常规的对象实例化方式等相关方法。

```java
public native long objectFieldOffset(Field f);

public native Object getObject(Object o, long offset);

public native void putObject(Object o, long offset, Object x);

public native Object getObjectVolatile(Object o, long offset);

public native void putObjectVolatile(Object o, long offset, Object x);

public native void putOrderedObject(Object o, long offset, Object x);

public native Object allocateInstance(Class<?> cls) throws InstantiationException;
```

**典型应用**：

- **常规对象实例化方式**：我们通常所用到的创建对象的方式，从本质上来讲，都是通过 new 机制来实现对象的创建。但是，new 机制有个特点就是当类只提供有参的构造函数且无显示声明无参构造函数时，则必须使用有参构造函数进行对象构造，而使用有参构造函数时，必须传递相应个数的参数才能完成对象实例化。
- **非常规的实例化方式**：而 Unsafe 中提供 `allocateInstance` 方法，仅通过 Class 对象就可以创建此类的实例对象，而且不需要调用其构造函数、初始化代码、JVM 安全检查等。它抑制修饰符检测，也就是即使构造器是 private 修饰的也能通过此方法实例化，只需提类对象即可创建相应的对象。由于这种特性，`allocateInstance` 在 `java.lang.invoke`、`Objenesis`（提供绕过类构造器的对象生成方式）、`Gson`（反序列化时用到）中都有相应的应用。

如下图所示，在 `Gson` 反序列化时，如果类有默认构造函数，则通过反射调用默认构造函数创建实例，否则通过 `UnsafeAllocator` 来实现对象实例的构造，`UnsafeAllocator` 通过调用 Unsafe 的 `allocateInstance` 实现对象的实例化，保证在目标类无默认构造函数时，反序列化不够影响。

![img](https://p1.meituan.net/travelcube/b9fe6ab772d03f30cd48009920d56948514676.png)

## 8 Unsafe数组相关的操作

`arrayBaseOffset`与`arrayIndexScale`方法结合起来，即可定位数组中每个元素在内存中的位置。

```java
public native int arrayBaseOffset(Class<?> arrayClass);

public native int arrayIndexScale(Class<?> arrayClass);
```

**典型应用**： `java.util.concurrent.atomic` 包下的 `AtomicIntegerArray`（可以实现对 Integer 数组中每个元素的原子性操作）

## 9 Unsafe的内存屏障相关

```java
public native void loadFence();

public native void storeFence();

public native void fullFence();
```

在Java8中引入，用于定义内存屏障（也称为内存栅栏、内存栅障、屏障指令等，是一类同步屏障指令，是CPU或编译器在对内存随机访问的操作中的一个同步点，使得此点之前的所有读写操作都执行后才可以开始执行此点之后的操作），可以避免代码重排序。

**典型应用**：Java 8 中引入了一种锁的新机制——`StampedLock`，它可以看成是读写锁的一个改进版本。

## 10 Unsafe的系统相关

获取系统相关信息。

```java
public native int addressSize();  

public native int pageSize();
```

**典型应用**： java.nio 下的工具类 Bits。





- 本文对于源码阅读记录基于`JDK1.8`，个人浅见，如有不妥，敬请指正。*
- *文中 "`//TODO`"处代表个人暂时未懂或标示之后会学习的地方，如有高见，敬请指教。*
- 记录中涉及参考及引用部分在文末都会指明原链接，侵删。

## 参考及引用：

[**Java 魔法类：Unsafe 应用解析 - 美团技术团队**](https://tech.meituan.com/2019/02/14/talk-about-java-magic-class-unsafe.html)

[**死磕 java 魔法类之 Unsafe 解析**](https://juejin.im/post/5ccf160b51882541ac5c512a#heading-0)