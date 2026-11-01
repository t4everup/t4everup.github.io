+++
title = 'java反射机制'
date = '2024-02-22T14:32:09+08:00'
categories = ['Java']
tags = ['java安全']
+++

## 反射概念

JAVA反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性；这种动态获取的信息以及动态调用对象的方法的功能称为java语言的反射机制。

## Class类

在 `java` 世界里，一切皆对象。从某种意义上来说，`java` 有两种对象：实例对象和 `Class` 对象。每个类的运行时的**类型信息**就是用 `Class` 对象表示的，它包含了与类有关的信息，实例对象就是通过 `Class` 对象来创建的。`Java` 使用 `Class` 对象执行其 `RTTI` (运行时类型识别，Run-Time Type Identification)，多态就是基于 `RTTI` 实现的。

1. Class类存在于JDK的java.lang包中
2. Class类也是继承object类
3. Class类对象是系统创建的（类 是Class类的一个对象）
4. 一个类的Class对象只在内存中存在一份，类只需要加载一次（在类加载的时候，jvm会创建唯一的class对象）

##  获取Class对象

1. 除了int等基本类型外，java的其他类型全都是class。  class的本质是数据类型。

在反射中，我们如果想获取一个类或者调用一个类的方法，需要先获取这个类的Class对象。

而`class`是由JVM在执行过程中动态加载的。JVM在第一次读取到一种`class`类型时，将其加载进内存。每加载一种`class`，JVM就为其创建一个`Class`类型的实例，并关联起来

类加载:(类加载机制 后边再深入学习)
![image-20240127155444174](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202401271554271.png)

获取方式：

1. 类名.class
2. 对象.getClass()
3. Class.forName(全限定类名)  //什么是全限定类名:包名.类名

![image-20240127161034590](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202401271610678.png)

#### forName有两个函数重载

- Class<?> forName(String name)
- Class<?> forName(String name, **boolean** initialize, ClassLoader loader)

```java
第一种是常见的获取class的方式，可以理解为第二种方式的一个封装
Class.forName(className)
// 等于
Class.forName(className, true, currentLoader)
```

第一个参数为类名； 第二个参数表示是否初始化；第三个参数为ClassLoader（类加载器）

第二个初始化 可以理解为类的初始化。

```java
public class DaiMa {
    {//构造块
        System.out.println("我是构造块");
    }
    static {
        System.out.println("我是类中的静态代码块");
    }
    public DaiMa(){//构造方法
        System.out.println("我是构造方法");
    }
    public static void main(String[] args) {
        String a ="第三个";
        System.out.println(a);
        new DaiMa();
    }
}
```

![image-20240219145801913](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202402191458957.png)

可以看到代码执行的顺序，首先执行static{}中的，这个就是类初始化时执行的。

也就是说，如果 forName 中的 initialize=true，就告诉java虚拟机进行初始化。

#### forName可控 static恶意攻击

```java
 public void ref(String shell) throws ClassNotFoundException {
      Class.forName(shell);
    }
```

这个函数，shell参数可控，我们可以构造一个恶意类，将恶意代码放在static{}中，从而在类初始化时执行恶意代码

```java
package fanshe.edu.xcu;

public class Shell {
    static {
        Runtime calc = Runtime.getRuntime();
        try {
            calc.exec("calc");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

```java
package fanshe.edu.xcu;

public class Chush {
    public void ref(String shell) throws ClassNotFoundException {
        Class.forName(shell);
    }

    public static void main(String[] args) throws ClassNotFoundException {
        Chush chush = new Chush();
        chush.ref("fanshe.edu.xcu.Shell");
    }
}
```

这样在类初始化时就执行了static{}中的恶意代码。
![image-20240219151925494](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202402191519583.png)

## 	通过反射获取类的信息

#### class 的常用方法

```java
Field[] getFields()             //返回一个包含Field对象的数组，存放该类或接口的所有可访问公共属性（含继承的公共属性）。
Field[] getDeclaredFields()     //返回一个包含Field对象的数组，存放该类或接口的所有属性（不含继承的属性）。
Field getField(String name)     //返回一个指定公共属性名的Field对象。
Method[] getMethods()           //返回一个包含Method对象的数组，存放该类或接口的所有可访问公共方法（含继承的公共方法）。
Method[] getDeclaredMethods()   //返回一个包含Method对象的数组，存放该类或接口的所有方法（不含继承的方法）。
Constructor[] getConstructors() //返回一个包含Constructor对象的数组，存放该类的所有公共构造方法。
Constructor getConstructor(Class[] args)//返回一个指定参数列表的Constructor对象。
Class[] getInterfaces()         //返回一个包含Class对象的数组，存放该类或接口实现的接口。
T newInstance()                 //使用无参构造方法创建该类的一个新实例。
String getName()                //以String的形式返回该类（类、接口、数组类、基本类型或void）的完整名。
```

```java
package fanshe.edu.xcu;

import java.lang.reflect.Field;
import java.lang.reflect.Method;

public class GetClass {
  
    public static void main(String[] args) throws ClassNotFoundException, InstantiationException, IllegalAccessException, NoSuchMethodException {
        Class <?> catClass=null;
        System.out.println("-----获取Cat类的Class对象-----");
        catClass=Class.forName("fanshe.edu.xcu.Cat");//获取class类
        System.out.println(catClass);
        Field[] fields1=catClass.getDeclaredFields();//获取全部成员变量
        Field[] fields2 =catClass.getFields();//获取全部public成员变量
        System.out.println("-----获取成员变量-----");
        for (Field field1: fields1){
            System.out.println(field1);
        }
        Method[] methods =catClass.getMethods();//获取成员方法
        System.out.println("-----获取成员方法-----");
        for(Method method1 : methods){
            System.out.println(method1);
        }

    }
}

```

![image-20240218211316841](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202402182113985.png)

## 反射创建类对象 

#### 反射中重要的方法:

- 获取类的⽅法： forName
- 实例化类对象的⽅法： newInstance
- 获取函数的⽅法： getMethod
- 执⾏函数的⽅法： invoke

```java
package fanshe.edu.xcu;

import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class GetClass {

    public static void main(String[] args) throws ClassNotFoundException, InstantiationException, IllegalAccessException, NoSuchMethodException, InvocationTargetException {
        Class <?> catClass=null;
        System.out.println("-----获取Cat类的Class对象-----");
        catClass=Class.forName("fanshe.edu.xcu.Cat");//获取class类
        System.out.println(catClass);
      
        System.out.println("-----通过newInstance创建对象-----");
        Object Cat2=catClass.newInstance();//创建类实例对象

        Cat cat= (Cat) catClass.newInstance();//创建一个Cat对象
        System.out.println(cat.age);
        System.out.printf(cat.name);
      
        Method method=catClass.getMethod("Mio");
        method.invoke(Cat2);//使用invoke  执行某个的对象的目标方法。


    }
}
#Cat 类
  //package fanshe.edu.xcu;
//
//public class Cat {
//    protected String name;
//    protected int age;
//    public int high=10;
//    public Cat(){
//        System.out.println("我被调用了");
//        this.age=18;
//        this.name="meiqiu";
//    }
//    public void Mio(){
//        System.out.println("miao");
//    }
//}
```

![image-20240219142848833](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202402191428878.png)

这里可以知道:

1. 使用newInstance() 实际内部调用了无参构造方法
2. invoke() 用于执行某个的对象的目标方法。一般会和getMethod方法配合进行调用。
3. 传统调用方法是：对象.方法()；在反射中，是 方法.invoke(对象)

如果使用newInstance() 不成功，可能是:

- 使用的类没有构造函数
- 使用的类的构造函数是私有的

## 利用反射构造Runtime类执行

在java.lang.Runtime类中，Runtime.exec方法有6个重载：

![image-20240220130032541](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202402201300612.png)

我们去构造一个执行代码

```java
package fanshe.edu.xcu;

import java.lang.reflect.InvocationTargetException;

public class Rtime {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, InstantiationException, IllegalAccessException, InvocationTargetException {
        Class RC=Class.forName("java.lang.Runtime");
        RC.getMethod("exec", String.class).invoke(RC.newInstance(),"calc.exe");

    }
}
	
```

执行是报错的。
![image-20240220130211763](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202402201302816.png)

因为Runtime类的构造方法是私有的，上边总结过了，如果使用newInstance() 不成功的情况。

这就是一种常见的模式:单例模式 
`这种模式涉及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。`

```java
public class Runtime {
    private static final Runtime currentRuntime = new Runtime();

    private static Version version;

    /**
     * Returns the runtime object associated with the current Java application.
     * Most of the methods of class {@code Runtime} are instance
     * methods and must be invoked with respect to the current runtime object.
     *
     * @return  the {@code Runtime} object associated with the current
     *          Java application.
     */
    public static Runtime getRuntime() {
        return currentRuntime;
    }

    /** Don't let anyone else instantiate this class */
    private Runtime() {}

```

上边代码 可以看出，Runtime类给我们创建了一个Runtime对象：currentRuntime，然后编写了一个静态方法getRuntime来获取currentRuntime对象。

所以说，Runtime类是单列模式，我们只能通过Runtime.getRuntime() 来获取到 Runtime 对象。

```java
package fanshe.edu.xcu;

import java.lang.reflect.InvocationTargetException;

public class Rtime {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, InstantiationException, IllegalAccessException, InvocationTargetException {
        Class RC=Class.forName("java.lang.Runtime");
//        RC.getMethod("exec", String.class).invoke(RC.newInstance(),"calc.exe");

//        Runtime test=Runtime.getRuntime();
//        test.exec
        RC.getMethod("exec", String.class).invoke(RC.getMethod("getRuntime").invoke(RC),"calc.exe");
    }
}
```

```java
//分解 RC.getMethod("exec", String.class).invoke(RC.getMethod("getRuntime").invoke(RC),"calc.exe");

        Class RC=Class.forName("java.lang.Runtime");
        Method exec= RC.getMethod("exec", String.class);
        Method getRuntime= RC.getMethod("getRuntime");
        Object runtime=getRuntime.invoke(RC);
        exec.invoke(runtime,"calc.exe");
```

