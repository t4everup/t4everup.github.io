+++
title = 'java基础(一)'
date = '2023-12-13T12:41:51+08:00'
categories = ['Java']
tags = ['java基础']
+++

## 前言

java大一学的，学的很水。重学一下，打下基础。

### javaEE

javaEE:Java EE平台构建于Java SE平台之上，Java EE平台提供一组API和运行环境来开发和运行大规模的，多层的，可扩展的，可靠的和安全的网络应用程序。主要针对于web应用开发。

### java特点

- java语言是面向对象的(oop)

- java语言是健全的。强类型机制，异常处理，垃圾回收(GC)机制。

- java跨平台性。使用javac编译器生成的.class文件在不同操作系统都能够运行。

  ![image-20231209135444802](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202312091354853.png)

- java是`解释性`语言。解释性语言:php,java,JavaScript 。编译性语言:c/c++

  区别:`编译语言`直接转换为处理器可以执行的机器代码，`解释性语言`编译后的代码不能被处理器执行，需要解释器来执行。

### JVM

java核心机制:java虚拟机(JVM)

- jvm具有指令集并使用不同的存储区域。解释指令集(字节码文件)，管理数据，内存，寄存器。
- 不同的平台有不同的jvm(jvm映射规则不同)
- 一次编译，处处运行。

### JDK

JDK:java开发工具包。是用来编译、调试Java程序的开发工具包。包括Java工具（javac/java/jdb等）和Java基础的类库（java API ）。

### JRE

JRE:java运行环境。是Java平台，所有的程序都要在JRE下才能够运行。包括JVM和Java核心类库和支持文件。

#### 关系

 **JDK = JRE +** **开发工具集**（例如 Javac,java 编译工具等)

**JRE = JVM + Java SE** **标准类库**（java 核心类库）

## 面向对象

### 类与对象

如果说我养了两只猫，他们有不同的特征，比如一个白色的比较可爱，一个黑色比较捣蛋。

- 类：那么我们把这些属性提取出来，就成了一个猫(cat)的类。
- 对象：对象是类的个体，具体到哪一个个体，比如一个猫叫煤球。

看下图解
![image-20231209185340860](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202312091853891.png)

我们既然知道了一些属性，那么我们是不是就能将这个Cat类建立起来。

```java
public class Cat {
    public String color;
    public String name;

}
```

上面我们建立了一个Cat类，其中有两个属性，颜色和名字。
我们现在去实例化它，我们就拿到了一个对象。

```java
public class Cat {
    public String color;
    public String name;
    public static void main(String[] args) {

        Cat cat1 =new Cat();
    }

}
```

这个时候我们已经拿到了一个具体的Cat类的对象 ：cat1。但是我们并没有给他赋任何属性啊，我们现在给他添加具体的属性值。

```java
public class Cat {
    public String color;
    public String name;
    public static void main(String[] args) {
        
        Cat cat1 =new Cat();
        cat1.color="black";
        cat1.name="煤球";
        System.out.println(cat1.color);
        System.out.println(cat1.name);
    }
}
//black
//煤球
```

这个时候 cat1就是我们的煤球啦。得到一个比较完整的对象。

### 方法

如果说猫要叫，那么我们怎么操作呢? 当然是去类里面添加一些东西啦(方法)

```java
public class Cat {
    public String color;
    public String name;
    public void Mio(){//叫 这个方法，也就是函数
        System.out.println("喵喵");
    }
}
```

我们通过方法来实现了猫叫，也就是函数。那么如果我们要改变猫的颜色呢，而且这个颜色可以控制，那么就要用到`有参`函数了

```java
public class Cat {
    public String color;
    public String name;
    public void mio(){
        System.out.println("喵喵");
    }
    public void cgColor(String color){//color 通过参数传递
        this.color= color;
    }
}
```

最后看一下输出

```java
public class Cat {
    public String color;
    public String name;
    public void mio(){
        System.out.println("喵喵");
    }
    public void cgColor(String color){
        this.color= color;
    }

    public static void main(String[] args) {

        Cat csat1 =new Cat();
        cat1.color="black";
        cat1.name="煤球";
        cat1.mio();
        cat1.cgColor("bule");
        System.out.println(cat1.color);
        System.out.println(cat1.name);
    }
}
//喵喵
//bule
//煤球
```

#### 方法重载

可以在一个类中定义多个方法。具有相同的方法名，`不同`的参数。比如重载一下mio方法。

```java
public class Cat {
    public String color;
    public String name;
    public void mio(){
        System.out.println("喵喵");
    }
    public void cgColor(String color){
        this.color= color;
    }
    public void mio(String mio){//重载
        System.out.println(mio);
    }
}
```

#### 方法重写

当一个子类继承一父类，而子类中的方法与父类中的方法的名称、参数个数、类型都完全一致时，就称子类中的这个方法**重写**了父类中的方法。(相对于继承)

```java
//Cat1.java
public class  Cat1 extends Cat  {
    @Override
    public void mio() {
        super.mio();//super关键字 用来访问父类
    }

    public static void main(String[] args) {
        Cat1 cat1 = new Cat1();
        cat1.mio();
    }
}
//可以通过 alt + ins 快捷键进行重写
```

#### 构造方法

前面的代码都是在创建Cat对象后才对其属性进行赋值的，如果我们想要在`创建对象后`就带有属性呢？那就要用到`构造方法`了。

- 构造方法名称必须和类的名称一样
- 构造方法不能有任何的返回值类型声明
- 构造方法中不能return一个值

```java
simport java.util.Scanner;

public class Cat {
    public String color;
    public String name;
    public Cat(){//构造方法
        this.color="black";
        this.name="煤球";
    }

    public static void main(String[] args) {
      
        Cat cat1 =new Cat();
        System.out.println(cat1.color);
        System.out.println(cat1.name);
    }

}
```

在我们实例化cat1对象的时候，color和name属性就已经给赋值了。

### 匿名对象

匿名对象就是没有明确的给出名字的对象，是对象的一种简写形式。一般匿名对象只使用一次，而且匿名对象只在堆内存中开辟空间，而不存在栈内存的引用。

```java
import java.util.Scanner;

 class Cat {
    public String color;
    public String name;
    public  Cat(){
        this.color="black";
        this.name="煤球";
    }

    public void mio(String mio){
        System.out.println(mio);

    }

    public static void main(String[] args) {
        new Cat().mio("miaomiao");//匿名对象
    }

}
```

可以看到和我们之前new一个对象不同，没有任何栈内存引用它，该对象在使用后就被GC机制回收。

### 代码块

##### 普通代码块

直接在方法或语句中定义的代码

```java
public class DaiMa {

    public static void main(String[] args) {
        {
            int a=666;
            System.out.println(a);
        }
        int a=100;
        System.out.println(a);
    }
}
```

##### 构造块

写在类里面的代码块，其执行顺序优先于构造方法，在每实例化一个对象时都会执行一次。

```java
public class DaiMa {
    {//构造块
        String b="第一个";
        System.out.println(b);
    }
    public DaiMa(){//构造方法
        System.out.println("我是构造方法");
    }

    public static void main(String[] args) {
        new DaiMa();
        new DaiMa();

    }
}//第一个
 //我是构造方法
 //第一个
 //我是构造方法
```

##### 静态代码块

- static声明的代码块。其执行优先于`主方法`。
- 类中定义的`静态代码块`执行优先于`构造块`。
- 静态代码块只执行一次。

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

![image-20231210140047900](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202312101400969.png)

### 三大特征

##### 封装

封装就是把抽象出的数据（**属性**）和对数据的操作（**方法**）封装在一起，数据被保护在内部，程序的其他部分只有通过被授权的操作（**方法**）才能对数据进行操作。

封装实现的步骤

- 将属性进行私有化private （不能直接修改属性）
- 提供公共的setter方法，用于对属性判断并赋值
- 提供公共的getter方法，用于获取属性的值

```java
//People类
public class People {
    private String name;//属性设置为私有
    private int age;
    public void setAge(int age) {
        if (age > 18) {
            this.age = age;
        }
    }
    public void setName(String name) {//提供public方法去获取属性
        this.name = name;
    }
    public String getName() {
        return name;
    }
    public int getAge() {
        return age;
    }
}
```

```java
//GetPeople类
public class GetPeople {
    public static void main(String[] args) {
        People people1 = new People();
        people1.setAge(20);
        people1.setName("168");
        System.out.println(people1.getAge());
        System.out.println(people1.getName());
    }
}
```

##### 继承

如果说多个类中存在相同的属性和方法时，我们在这些类中抽象出`父类`，在父类中定义相同的属性和方法，其子类就不用定义这些属性和方法，只需要用extents来继承父类就能使用该属性和方法了。

```java
public class People {
    protected String name;
    protected int age;
    public People(String name,int age){
        this.name=name;
        this.age=age;
    }
    public People(){
        this.age=20;
        this.name="168";

    }
```

```java
public class People2 extends People{
    public People2(){
        super("168",22);
//        super();//注意 构造方法中只能出现一次super关键字
    }

    public static void main(String[] args) {
        People2 people2 = new People2();
        System.out.println(people2.age);
        System.out.println(people2.name);
    }
}
```

这里可以看到我们可以在People2类中并没有写name和age属性，但是能够访问，这就是继承的效果。

##### 多态

举个例子，比如说:叫，猫的话是喵喵。狗的话是汪汪。也就是`同一个行为的不同表现`。

###### 多态的必要条件:

- 继承
- 重写
- 父类引用指向子类对象

```java
public abstract class Sound {//abstract 修饰抽象类
    public String song;
    public void makeSound(String song){
        System.out.println(song);
    }	

    public abstract void makeSound();//抽象方法不能有主题
}

class Cat2 extends Sound{
    @Override
    public void makeSound() {
        super.makeSound("喵喵");
    }
}
class Dog extends Sound{
    @Override
    public void makeSound() {
        super.makeSound("汪汪");
    }
}
```

```java
public class MakeSound {

    public static void main(String[] args) {
        Cat2 cat2 = new Cat2();
        cat2.makeSound();
        Dog dog = new Dog();
        dog.makeSound();
    }
}
```

### 抽象类

在多态时，我们的Sound是用abstract修饰的抽象类，那么勇abstract修饰方法时这个方法就是抽象方法。

- 抽象类不能被实例化

- 抽象类不一定包含abstract方法，但是类中有abstract方法，类就必须要用abstract修饰

- 抽象方法不能有主体

  

### 访问修饰符

- 公开级别:用 public 修饰,对外公开
- 受保护级别:用 protected 修饰,对子类和同一个包中的类公开
- 默认级别:没有修饰符号,向同一个包的类公开. 
- 私有级别:用 private 修饰,只有类本身可以访问,不对外公开.







