# 1.Java程序初始化的顺序是怎样的？

在Java语言中，当实例化对象时，对象所在类的所有成员变量需要先进行初始化，只有当类的所有成员都初始化后，才会调用对象所在类的构造函数创建对象。

**类的初始化一般遵循3个原则：**

- 静态对象（变量）优先于非静态对象（变量）初始化，静态对象（变量）只初始化一次，而非静态对象（变量）可能会初始化多次；
- 父类优先于子类进行初始化；
- 按照成员变量定义顺序进行初始化。即使成员变量定义散步于方法之中，他们依然在任何方法（包括构造函数）被调用之前先初始化；

**初始化加载顺序：**

- 父类（静态变量、静态语句块）
- 子类（静态变量、静态语句块）
- 父类（实例变量、普通语句块）
- 父类（构造函数）
- 子类（实例变量、普通语句块）
- 子类（构造函数）

实例：

```java
class Base{
    // 1.父类静态代码块
    static {
        System.out.println("Base static block");
    }
    // 2.父类非静态代码块
    {
        System.out.println("Base block");
    }
    // 3. 父类构造函数
    public Base(){
        System.out.println("Base constructor");
    }

}
public class Derived extends Base{
    //4. 子类静态代码块
    static{
        System.out.println("Derived static block");
    }
    //5. 子类非静态代码块
    {
        System.out.println("Derived block");
    }
    //6. 子类构造函数
    public Derived(){
        System.out.println("Derived constructor");
    }
    public static void main(String[] args) {
        new Derived();
    }
}
```

控制台打印：

```java
Base static block
Derived static block
Base block
Base constructor
Derived block
Derived constructor
```



# 2.Java和C++的区别

- Java是纯粹的面向对象语言，所有对象都继承自`java.lang.Object`，C++为了兼容C既支持面向对象也支持面向过程；
- Java通过虚拟机从而实现跨平台性，而C++依赖于特定平台；
- Java没有指针，它的引用可以理解为安全指针，而C++具有和C一样的指针；
- Java支持自动垃圾回收，而C++需要手动回收（C++11中引入智能指针，使用引用计数法垃圾回收）；
- Java不支持多重继承，可以通过实现多个接口来达到相同的目的，而C++支持多继承；
- Java不支持操作符重载，虽然可以对String对象支持加法运算，但是这是语言内置支持的操作，不属于操作符重载，而C++可以；
- Java内置了线程的支持，而C++需要依靠第三方库；
- Java的goto是保留字，不可用，C++可以使用goto；
- Java不支持条件编译，C++通过#ifdef #ifndef等预处理命令从而实现条件编译；

# 3.反射

![JVM内存模型](https://pic4.zhimg.com/v2-4face8109e0d52ef5894c41c69e4ec6b_r.jpg)