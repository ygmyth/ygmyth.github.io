---
title: "单例模式的安全性"
date: 2020-10-01T15:36:46+08:00
draft: false
---
单例模式，我想大家再熟悉不过了，不过本文不是介绍单例模式该怎么写的。本文来说说怎么破坏一个单例，让你写的单例变成一个假的单例。当然，本文也会给出怎么进行防守的方法。

## 一个简单的单例
来一个简单的单例模式例子：
```java
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();

    private String name;

    public String getName() {
        return this.name;
    }

    private Singleton() {
        this.name = "Neo";
    }

    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```
上面是一个比较简单的饿汉写法的单例模式，我们看看 client 调用：

```java
public class APP {
    // 由于构造方法上加了 private 修饰，所以我们已经不能通过 ‘new’ 来产生实例了
    // Singleton intance = new Singleton();

    Singleton instance = Singleton.getInstance();
    System.out.println(instance.getName());
}
```
## 通过反射破坏单例
原理很简单，通过反射获取其构造方法，然后重新生成一个实例。
```java
class APP {
    public static void main(String[] args) throws Exception {
        Singleton instance1 = Singleton.getInstance();

        // 下面我们通过反射得到其构造方法，并且修改其构造方法的访问权限，并用这个构造方法构造一个对象
        Constructor constructor = Singleton.class.getDeclaredConstructor();
        constructor.setAccessible(true);
        Singleton instance2 = (Singleton) constructor.newInstance();

      // 是不是产生了两个实例了？
        System.out.println(instance1 == instance2); // false
    }
}
```
显然，说好的单例已经不单一了，上面的程序运行结果肯定是：false

## 防止反射方式破坏
如果要避免单例被反射破坏，Java 提供了枚举，举个例子：
```java
public enum Singleton {
    INSTANCE;// 这里只有一项

    private String name;

    Singleton() {
        this.name = "Neo";
    }
    public static Singleton getInstance() {
        return INSTANCE;
    }

    public String getName() {
        return this.name;
    }
}
```
这个时候，如果我们再想通过反射获取类的构造方法：
```
Constructor constructor = Singleton.class.getDeclaredConstructor();
```
会抛出 `NoSuchMethodException` 异常：
```java
Exception in thread "main" java.lang.NoSuchMethodException: com.javadoop.Singleton.<init>()
    at java.lang.Class.getConstructor0(Class.java:3082)
    at java.lang.Class.getDeclaredConstructor(Class.java:2178)
    at com.javadoop.singleton.APP.main(APP.java:11)
```
对于枚举，JVM 会自动进行实例的创建，其构造方法由 JVM 在创建实例的时候进行调用。

我们在代码中是获取不到 enum 类的构造方法的。

通过序列化破坏
下面，我们再说说另一种破解方法：序列化、反序列化。

我们知道，序列化是将 java 对象转换为字节流，反序列化是从字节流转换为 java 对象。
```java
class APP {
    public static void main(String[] args) throws ... {
        Singleton instance1 = Singleton.getInstance();

        Constructor constructor = Singleton.class.getDeclaredConstructor();
        constructor.setAccessible(true);
        Singleton instance2 = (Singleton) constructor.newInstance();

        // instance3 将从 instance1 序列化后，反序列化而来
        Singleton instance3 = null;
        ByteArrayOutputStream bout = null;
        ObjectOutputStream out = null;
        try {
            bout = new ByteArrayOutputStream();
            out = new ObjectOutputStream(bout);
            out.writeObject(instance1);

            ByteArrayInputStream bin = new ByteArrayInputStream(bout.toByteArray());
            ObjectInputStream in = new ObjectInputStream(bin);
            instance3 = (Singleton) in.readObject();
        } catch (Exception e) {
        } finally {
            // close bout&out
        }

        // 显然，instance3 和 instance1 不是同一个对象了
        System.out.println(instance1 == instance3); // false
    }
}
```
毫无疑问，`instance1 == instance3` 也会返回 false。

## 防止序列化破坏
在序列化之前，我们要在类上面加上 implements Serializable。

我们需要做的是，在类中加上 readResolve() 这个方法，返回实例。
```java
public class Singleton implements Serializable {
    private static final Singleton INSTANCE = new Singleton();

    private String name;

    public String getName() {
        return this.name;
    }

    private Singleton() {
        this.name = "Neo";
    }

    public static Singleton getInstance() {
        return INSTANCE;
    }

    // 看这里
    public Object readResolve() throws ObjectStreamException {
        return INSTANCE;
    }
}
```
你再试一下，会发现变成 true 了。

因为在反序列化的时候，JVM 会自动调用 readResolve() 这个方法，我们可以在这个方法中替换掉从流中反序列化回来的对象。

更具体的信息，请参考官方文档：[对象序列化规范](https://docs.oracle.com/javase/7/docs/platform/serialization/spec/input.html#5903) ，这个方法完整的描述是这样的：
```
ANY-ACCESS-MODIFIER Object readResolve() throws ObjectStreamException;
```
总结
文中没有示例说反序列化在 enum 类中的表现,enum 类自带这种特殊光环，不用写 readResolve() 方法就可以自动防止反序列化方式对单例的破坏。

本文有点较真了，不过也算是曲线地介绍了点知识吧。