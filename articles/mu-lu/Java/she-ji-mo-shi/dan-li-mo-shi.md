# 单例模式



## 1. 饿汉式

```java
//构造方法私有化
private Singleton() {
}
private static Singleton singleton = new Singleton();

public static Singleton getInstance() {
    return singleton;
}
```

线程安全，占用空间

## 2. 懒汉式

```java
private static Singleton singleton;
public static Singleton getInstance() {
    if(singleton == null) {
        singleton = new Singleton();
    }
    return singleton;
}
```

线程不安全。

## 3. 方法加锁

```java
private static Singleton singleton;
public static synchronized Singleton getInstance() {
    if(singleton == null) {
        singleton = new Singleton();
    }
    return singleton;
}
```

线程安全，但是在方法上加锁开销大

## 4. 双重校验

```java
private volatile static Singleton singleton;

public static Singleton getInstance() {
    if (singleton == null) {
        synchronized (Singleton.class) {
            if (singleton == null) {
                singleton = new Singleton();
            }
        }
    }
    return singleton;
}
```

第一层判断null减少不必要的加锁，第二次判断，如果被另外一个线程new了不判空的话会再次new，造成不是单例。

## 5. 枚举

枚举是天然单例的。其他的单例可以被反射破坏(反射可以调用私有的构造方法)，而枚举是无法被反射调用构造方法的，原因在与Constructor的newInstance()时针对枚举的处理。

{% code lineNumbers="true" %}
```java
@CallerSensitive
    public T newInstance(Object ... initargs)
        throws InstantiationException, IllegalAccessException,
               IllegalArgumentException, InvocationTargetException
    {
        if (!override) {
            if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
                Class<?> caller = Reflection.getCallerClass();
                checkAccess(caller, clazz, null, modifiers);
            }
        }
        if ((clazz.getModifiers() & Modifier.ENUM) != 0)
            throw new IllegalArgumentException("Cannot reflectively create enum objects");
        ConstructorAccessor ca = constructorAccessor;   // read volatile
        if (ca == null) {
            ca = acquireConstructorAccessor();
        }
        @SuppressWarnings("unchecked")
        T inst = (T) ca.newInstance(initargs);
        return inst;
    }
```
{% endcode %}

12行，如果判断被反射的类是被enum修饰的，会抛出Cannot reflectively create enum objects异常。
