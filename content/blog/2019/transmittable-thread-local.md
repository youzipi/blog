---
title: '【review-项目】TransmittableThreadLocal 导致的线程数据逸出'
date: 2019-02-18 18:40:13
updated: 2019-02-18 18:40:13
categories: 
- review
tags: 
- ThreadLocal
- InheritableThreadLocal
- TransmittableThreadLocal
description: 解决因使用 TransmittableThreadLocal 导致的线程数据逸出。ThreadLocal，ITL，TTL 的原理解析。
aliases:
    - /2019/02/18/transmittable-thread-local/
---

> 解决因使用 `TransmittableThreadLocal` 导致的线程数据逸出。ThreadLocal，ITL，TTL 的原理解析。

<!--more-->


alias：
TransmittableThreadLocal: TTL
InheritableThreadLocal: ITL

# 背景
## 现象
数据错乱：
- 商务人员认领线索后，线索的持有人是其他商务人员。
- 一些记录的操作人与事实不符。

## 原因
项目引入 TTL：
批量审批的操作很慢，于是改用了线程池去处理。
最后更新审批状态的 DB 的时候，需要填充操作人信息，这些信息是保存在 ThreadLocal 中的。
但是没法传递 threadLocal，使用 ITL 也不行。
于是，引入了TTL 用以解决：向`线程池中的线程`传递 当前请求的 threadLocal。

项目定义了一个 contextHandler 用于保存 当前登录用户的一些信息，token, currrentUserId 之类。
contextHandler 使用 `TransmittableThreadLocal` 保存内容。

项目启动后，有一些初始化的操作，其中需要通过 feign 去调用其他的服务。
而 feign 配置的拦截器中，有从 contextHandler 中获取 token 的操作。即调用了 `threadLocal.get()` 方法，导致主线程 threadLocal 的数据初始化了，并保存到了 thread 中。
以上都是发生在 `main 线程`中的。

之后，新的 http 请求进入，从主线程创建新的线程去处理。
因为 TTL 继承自 ITL，所以，创建线程的过程中，主线程的 threadLocal 中的数据被复制到了 处理 http 请求的子线程中。
因为 threadLocal 保存的是一个 map 对象。所以，事实上，所有的 处理 http 请求的线程 `读取`、`写入`、`清空` 操作的都是同一个 map。即：`数据从 主线程 逸出到了整个应用中`。
准确的说，所有同时处理 http 请求的线程共享同一个 map。
因为 在 拦截器的 `afterCompletion`是配置了清空 threadLocal 数据的。
所以，实际上是：
- 应用启动后，threadLocal.get() 触发了 map 初始化。
- 新进来 http 请求，需要创建新的线程，此时 main 线程的 threadLocal.map 复制给了 servlet 线程。（传递引用）
- servlet 线程 处理请求，结束后，执行 `afterCompletion`，清空 threadLocal，同时也断开了 和 main 线程中 threadLocal 的关系。但是此时，其他未结束的 servlet 线程中的 threadLocal 也被清空了。
- 长时间运行后，servlet 所在线程池中的所有线程都被清空了一次。都是独立的了，再之后的数据就不会有问题了。

> ThreadLocal 是在第一次调用 get() 方法时 “初始化”的：注册到 thread 中。


## 解决
重新翻阅了 TTL 的 README，issue，源码。
最后在源代码里，看到了官方提供了关于这个问题的解决：
```java
 * You can turn on "disable inheritable for thread pool" by {@link com.alibaba.ttl.threadpool.agent.TtlAgent}
 * so as to wrap thread factory for thread pooling components
 * ({@link java.util.concurrent.ThreadPoolExecutor}, {@link java.util.concurrent.ForkJoinPool}) automatically and transparently.
 * <p>
 * ❷ or by overriding method {@link #childValue(Object)}.
 * Whether the value should be inheritable or not can be controlled by the data owner,
 * disable it <b>carefully</b> when data owner have a clear idea.
TransmittableThreadLocal<String> transmittableThreadLocal = 
new TransmittableThreadLocal<String>() {
    protected String childValue(String parentValue) {
        return initialValue();
        }
}
```
TTL 继承自 ITL，
ITL 的 childValue 方法是 `return parentValue;`。
这里，我们 override `childValue` 方法，
改为调用 `TTL.initialValue()` 也就是 `ThreadLocal.initialValue`，内容是 `return null;`

# TransmittableThreadLocal
## 为什么 TTL 不直接继承 ThreadLocal，而继承 ITL？
解决了这个问题后，又有了新的疑问：库的名字叫做 Transmittable ，那是不是 不应该提供 inheritable 的功能？
出现这个问题的原因是，我看到这个名字，下意识地把它当成了 传递的，忘记了它同时也是 继承的。
issue 里面也有相似的讨论：
[disable Inheritable when it's not necessary and buggy(eg. has potential memory leaking problem) ](https://github.com/alibaba/transmittable-thread-local/issues/100)
我在这个 issue 下面进行了提问：
>为什么 TTL 不直接继承 ThreadLocal，而继承 ITL？
>建议 把 inheritable 改为可选参数。
作者回复：
1. 向下兼容
2. 需要提供在 `new Thread`的情况下，也能进行传递。
3. `inheritable` 在业务上 有潜在的（**_potential_**）的 泄漏或污染。

## 关于开源项目 功能实现修改 的讨论
这里还有另个一个事例：
[[代码优化] 把 TTL 中的 holder 封装为 Set，便于阅读](https://github.com/alibaba/transmittable-thread-local/pull/95)
TTL 中用来实际存储的 holder 是 一个 Map 类型，但是是当做 Set 来使用的。
看源代码的时候，这里也是被困扰了：这个 map 的所有 put 方法传入的 value 都是 null。
实现 与 语义不一致。
作者坚持没有合入这个 feature。
理由：
- `TTL`是非常下层的库，期望极致减少可能的Overhead。
- 因为库内部实现 且 实现逻辑的代码非常少 => 通用设计的封装性 有些 反模式 是在代码维护性上还是可以接受。
- 如果我合进来了，因为对一个下层库可能有的性能影响，肯定会有不同的同学来反复问我这个问题，而我需要反复给予解释（维护者的痛苦深渊）。

总结一下：
- 性能问题。越是底层的库，调用次数越是频繁，任何一点性能差别都会被放到很大。
- 功能的实现是 `内部的`，而且是 `简单的`，可以牺牲可读性。比如 JDK 源码中有很多的位运算，其实是可读性很差的。（但是，这个项目中，这点理由其实不成立。）
- 作为一个开源项目，要考虑向前兼容，使用者可能会以你意想不到的方式使用这个库，所以要尽可能地`遵从历史`。
  - 对于新的实现，用户出现问题，可能会导致巨大的沟通成本。（但我觉得，这是因为工具的局限性，没有一个特别好的 代码变更的讨论，（投票）工具。）
  - 对于开源项目，一定要尽可能地少暴露接口。如无必要，不要暴露实现。只给用户提供必要的接口。
- 从 issue 提出方，当更换了底层的实现（比如：数据结构）时，考虑提供 beanchmark 报告。

## 实现
包装 Runnable
把 threadLocal 保存到 `private final AtomicReference<Object> capturedRef;` 中。
在 线程执行 之前，把数据填充回 threadLocal 中。
```java
    private TtlRunnable(@Nonnull Runnable runnable, boolean releaseTtlValueReferenceAfterRun) {
        this.capturedRef = new AtomicReference<Object>(capture());
        this.runnable = runnable;
        this.releaseTtlValueReferenceAfterRun = releaseTtlValueReferenceAfterRun;
    }

    /**
     * wrap method {@link Runnable#run()}.
     */
    @Override
    public void run() {
        Object captured = capturedRef.get();
        if (captured == null || releaseTtlValueReferenceAfterRun && !capturedRef.compareAndSet(captured, null)) {
            throw new IllegalStateException("TTL value reference is released after run!");
        }

        Object backup = replay(captured);
        try {
            runnable.run();
        } finally {
            restore(backup);
        }
    }
```
```java
// com.alibaba.ttl.TransmittableThreadLocal.Transmitter#capture
        public static Object capture() {
            Map<TransmittableThreadLocal<?>, Object> captured = new HashMap<TransmittableThreadLocal<?>, Object>();
            for (TransmittableThreadLocal<?> threadLocal : holder.get().keySet()) {
                captured.put(threadLocal, threadLocal.copyValue());
            }
            return captured;
        }
```
![](https://github.com/alibaba/transmittable-thread-local/raw/master/docs/TransmittableThreadLocal-sequence-diagram.png)

# InheritableThreadLocal
`@since 1.2`
类注释：
> This class extends <tt>ThreadLocal</tt> to provide inheritance of values from parent thread to child thread: when a child thread is created, the child receives initial values for all inheritable thread-local variables for which the parent has values.  Normally the child's values will be identical to the parent's; however, the child's value can be made an arbitrary function of the parent's by overriding the <tt>childValue</tt> method in this class.
>
> <p>Inheritable thread-local variables are used in preference to ordinary thread-local variables when the per-thread-attribute being maintained in the variable (e.g., User ID, Transaction ID) must be automatically transmitted to any child threads that are created.

## 场景
- 线程中的变量 需要在 创建子线程的时候，自动传递给子线程。
ex: user-id, transaction-id

### ITL 在 Spring RequestContextHolder 中的使用
NamedInheritableThreadLocal 是 ITL 的子类
NamedThreadLocal 是 ThreadLocal 的子类
两个子类都是只增加了一个 name 属性。
```java
// org.springframework.web.context.request.RequestContextHolder#inheritableRequestAttributesHolder
public abstract class RequestContextHolder  {
    // 不继承的
    private static final ThreadLocal<RequestAttributes> requestAttributesHolder = new NamedThreadLocal<>("Request attributes");
    // 可继承的
    private static final ThreadLocal<RequestAttributes> inheritableRequestAttributesHolder = new NamedInheritableThreadLocal<>("Request context");

	public static RequestAttributes getRequestAttributes() {
		RequestAttributes attributes = requestAttributesHolder.get();
		if (attributes == null) {
			attributes = inheritableRequestAttributesHolder.get();
		}
		return attributes;
	}
	public static void setRequestAttributes(@Nullable RequestAttributes attributes, boolean inheritable) {
		if (attributes == null) {
			resetRequestAttributes();
		}
		else {
			if (inheritable) {
                // 如果是 继承的话？
				inheritableRequestAttributesHolder.set(attributes);
				requestAttributesHolder.remove();
			}
			else {
				requestAttributesHolder.set(attributes);
				inheritableRequestAttributesHolder.remove();
			}
		}
	}
}
```

```java
// org.springframework.web.filter.RequestContextFilter
// org.springframework.web.servlet.FrameworkServlet
// 这两个类 使用了 threadContextInheritable
RequestContextHolder.setRequestAttributes(requestAttributes, this.threadContextInheritable);
```
`setThreadContextInheritable` 的注释：
> Set whether to expose the LocaleContext and RequestAttributes as inheritable for child threads (using an {@link java.lang.InheritableThreadLocal}). 
Default is "false", to avoid side effects on spawned background threads. 
Switch this to "true" to enable inheritance for custom child threads which are spawned during request processing and only used for this request (that is, ending after their initial task, without reuse of the thread). 
<b>WARNING:</b> Do not use inheritance for child threads if you are accessing a thread pool which is configured to potentially add new threads on demand (e.g. a JDK {@link java.util.concurrent.ThreadPoolExecutor}), since this will expose the inherited context to such a pooled thread 

如果使用线程池的话，不要 在子线程上使用继承，因为 继承的 context 会被暴露给 池子里的 线程（这个线程后面会被其他的“父线程”使用）

实际使用时，如果 数据 保存在 ITL 中，则 创建 子线程的时候，会被传递给 子线程的 ITL。
如果 数据 保存在 `ThreadLocal` 中的话，就不处理。
即 如果数据要 继承的话，需要 指明保存到 `InheritableThreadLocal` 中。

使用线程池的情况下，新的线程不是由父线程创建的，而是由线程池创建的。
父线程做的事情是把 任务 提交给线程池。

如何 在使用线程池的情况下，提供 ThreadLocal 值 的传递，解决异步执行任务时， context 传递 的问题？
应用需要的实际上是把 `任务提交给线程池时`的ThreadLocal值传递到 `任务执行时`。

https://blog.csdn.net/v123411739/article/details/79117430


[配合线程池定义可继承的线程变量InheritableThreadLocal_csdn](http://xiangshouxiyang.iteye.com/blog/2391477)


## 实现
在创建子线程的时候，把 父线程中的 ITL中的 map 复制给 子线程

- childValue

需要被重写。
因为 ThreadLocal  里面如果存放的是 对象的话，那么根据代码，传递的就是引用，也就是，任何子线程的修改都会影响到 所有的父线程 以及 父线程所有的子线程。`变量 实际上逸出到了整个线程树。`，这常常不是我们想要的。

**This method is called from within the parent thread before the child is started.**

### 数据传递过程

创建子线程的 调用链：
```java
new Thread()
|-Thread.init()
  --this.inheritableThreadLocals = ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    |-new ThreadLocalMap(parentMap)
        --key.childValue(e.value); // key: ThreadLocal, e: ThreadLocal.ThreadLocalMap.Entry
```
相关实现：
```java
// java.lang.Thread#init
private void init(ThreadGroup g, Runnable target, String name,
long stackSize, AccessControlContext acc) {
    Thread parent = currentThread();
    // some code
    if (parent.inheritableThreadLocals != null)
        // 复制 父线程（当前线程）的 inheritableThreadLocals 到子线程
        this.inheritableThreadLocals = ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    // some code
}
// java.lang.ThreadLocal#createInheritedMap
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
    return new ThreadLocalMap(parentMap);
}
// java.lang.ThreadLocal.ThreadLocalMap#ThreadLocalMap(java.lang.ThreadLocal.ThreadLocalMap)
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                /**
                 * 这里调用 ITL 的 `childValue(parentValue)`，进行 值复制，
                 * 在 childValue 里面决定复制方案：直接传递（引用|值），浅复制，深复制
                 */ 
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```
ThreadLocalMap 是 ThreadLocal 的 static 内部类。
createInheritedMap 是 ThreadLocal 的一个 static 方法。
`private ThreadLocalMap(ThreadLocalMap parentMap)` 只在 `ThreadLocal.createInheritedMap()` 中被调用，通过 其中的 childValue 进行父子线程的数据传递。
ThreadLocal 的childValue 方法是：`throw new UnsupportedOperationException();`
那么，
### 为什么 childValue 是 定义在 TL 中的，而不是定义在 ITL 中？
[why childValue() is defined in ThreadLocal instead of in InheritableThreadLocal？](https://stackoverflow.com/questions/54104706/why-childvalue-is-defined-in-threadlocal-instead-of-in-inheritablethreadlocal)
在 sof 上提了问，没人回答。。。

jdk 文档的解释：
> Method childValue is visibly defined in subclass InheritableThreadLocal, 
> but is internally defined here `for the sake of providing createInheritedMap factory method without needing to subclass the map class in InheritableThreadLocal`. 
> This technique is preferable to the alternative of embedding instanceof tests in methods. 

是为了方便提供 createInheritedMap 这个工厂方法。
在这里实现，比最后需要通过 `instanceof` 去区分要好。
但是，IMO，ThreadLocal 根本没有 继承相关的语义啊，为什么不把所有相关的都定义在 ITL 中？
**todo**
因为 createInheritedMap 被提升到了 TL 中，
ITL 对于 TL 是不可知的。
#### 把 childValue 定义在 ITL 中，要做的工作
如果要把 childValue 定义在 ITL 中，
那么需要
- 把 `createInheritedMap` 定义在 ITL 中
- 把 `private ThreadLocalMap(ThreadLocalMap parentMap)` 定义在 ITL 中
  - 在 ITL 中提供 `ThreadLocal.ThreadLocalMap`的子类，实现 `private ThreadLocalMap(ThreadLocalMap parentMap)` 方法。

## ITL vs ThreadLocal
区别很明显，很好理解。
- getMap
很简单，就是 override 了 ThreadLocal 的 getMap 方法。
TL 从  thread.threadLocals 中 取 map
ITL 从 thread.inheritableThreadLocals 取 map
也就是 一个线程是同时存在 TL 和 ITL 的，把 需要发布的 放到 `inheritableThreadLocals` 里，需要封闭的 放到 `threadLocals` 里面。

- createMap
TL.createMap: 创建 Thread 的 threadLocals，把 ThrealLocal:value 存到 Thread 的 threadLocals 里面
ITL.createMap: 创建 Thread 的 inheritableThreadLocals ThrealLocal:value 存到 Thread 的 inheritableThreadLocals 里面

ITL 相关实现：
```java
    /**
     * Computes the child's initial value for this inheritable thread-local
     * variable as a function of the parent's value at the time the child
     * thread is created.  This method is called from within the parent
     * thread before the child is started.
     * <p>
     * This method merely returns its input argument, and should be overridden
     * if a different behavior is desired.
     *
     * @param parentValue the parent thread's value
     * @return the child thread's initial value
     */
    protected T childValue(T parentValue) {
        return parentValue;
    }
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
```
# ThreadLocal
![](https://user-images.githubusercontent.com/5629307/53001523-ca301200-3465-11e9-9215-226534cb2620.png)
**todo**
```java
tl = new ThreadLocal()
tl.get():
    ThreadLocalMap map = t.threadLocals
    Entry e = map.getEntry(t1)
    return e.value
```

```java
// java.lang.ThreadLocal.ThreadLocalMap.Entry
      static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

# 总结
## ThreadLocal
ThreadLocal 线程封闭的一种方案。
线程封闭 是用来避免 线程安全问题的。

## ITL
之后，为了方便在父子线程间方便的传递数据，引入了 ITL。
但是，ITL 有两个问题：
1. ITL.childValue 方法，是直接返回 `parentValue`，没有进行深复制，如果存储的不是基本类型数据的话，相当于父线程及其所有子线程共享相同的引用，对其中数据的修改会影响到整个线程数。
所以，实际使用中，需要根据实际情况，override `ITL.childValue`。 
2. ITL 的数据复制时机是：`线程创建时`，在线程池场景下，达不到想要的结果。

## TTL
针对 ITL 在线程池之类情况下的缺陷 ，出现了 TTL。
通过包装线程的方式，把复制时机 移到了 `线程启动时`，
TTL 的问题：
TTL 同时具有 ITL 的功能，如果不想要 继承 功能的话，需要注意手动地修改 childValue 方法。
正常使用 TTL 的方式，（在不适用 proxy 的情况下）是需要对线程或者线程池进行手动的封装的。不用担心这个问题。
但是，当 TTL 被当做普通 ThreadLocal 使用时，TTL 实际是等价于 ITL 的。



# todo
@Scheduled 定时任务 执行所在的线程？

CommandLineRunner 里面的 run 方法 和 定时任务和 @Configuration 中的 定时任务。