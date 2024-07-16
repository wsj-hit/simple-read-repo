> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/82025801?utm_campaign=shareopn&utm_medium=social&utm_psn=1796480364639813635&utm_source=wechat_session)

> 本文基于 OpenJDK 11

之前升级了 JDK 到 OpenJDK11，把遇到的问题以及解决方案列一下。 每篇文章会以提出问题，思路说明，解决问题的思路去行文。 这篇文章是关于堆栈信息获取的。

遇到的问题 - 调用堆栈获取
--------------

之前有做调用堆栈监控上报，某些仅采集调用类，某些需要采集调用方法，总体来说：在 Java8 中，我们可以这样去获取调用堆栈：

1.  通过 Reflection 类：

```
private static void getCallStackClassNames() {
    StringBuffer sbStack = new StringBuffer();
    int i = 0;
    Class<?> caller = Reflection.getCallerClass(i++);
    do {
        sbStack.append(i + ".").append(caller.getName())
            .append("\n");
        caller = Reflection.getCallerClass(i++);
    } while (caller != null);
    LOGGER.info("{}", sbStack);
}

```

这种方式可以灵活地获取调用类，不用一下子读取整个堆栈。但是缺点是：无法查看调用方法，信息不够详细

1.  通过`Thread.currentThread().getStackTrace()`:

```
StackTraceElement[] stackTraceElements = Thread.currentThread().getStackTrace();

```

这种方法获取的信息很详细，但是一下子返回整个堆栈的调用，不够方便。

升级到 OpenJDK11 之后，`sun.reflect.Reflection`类没有了。

思路说明
----

通过在 Java 9 之后 JDK 自带的工具 jdeps 来寻找可替代的类：

```
jdeps  --jdk-internals ./target/AppName.jar

```

显示：

```
...
JDK Internal API                         Suggested Replacement
----------------                         ---------------------
sun.reflect.Reflection                   Use java.lang.StackWalker @since 9

```

看到建议使用`java.lang.StackWalker` 我们考虑用这个类替换`sun.reflect.Reflection`。

解决问题
----

`StackWalker`可以灵活地查看每一帧调用。 初始化可以指定选项：

```
//一般这样就足够用了，可以把每个调用栈输出
StackWalker walker = StackWalker.getInstance();

```

也可以指定初始化参数：

```
//这样对于调用：StackWalker#getCallerClass()和StackFrame#getDeclaringClass()不会报异常，默认初始化是不支持这两个方法的
StackWalker walker = StackWalker.getInstance(StackWalker.Option.RETAIN_CLASS_REFERENCE);

```

如果你想看反射的调用栈，例如 Spring 动态代理反射，可以这么初始化：

```
StackWalker walker = StackWalker.getInstance(StackWalker.Option.SHOW_REFLECT_FRAMES);

```

如果你想看完整的调用栈没有隐藏任何的调用栈，可以这么初始化：

```
StackWalker walker = StackWalker.getInstance(StackWalker.Option.SHOW_HIDDEN_FRAMES);

```

之后应用，例如取第一个调用栈：

```
walker.walk(s -> s.limit(1).collect(Collectors.toList()));

```

将每一个输出：

```
walker.forEach(System.out::println);

```