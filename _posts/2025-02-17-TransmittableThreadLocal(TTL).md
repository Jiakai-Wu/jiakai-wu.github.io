---
title: TransmittableThreadLocal(TTL)
date: 2025-02-17 19:14:15 +0800
categories:
  - java
  - thread
tags:
  - java
  - ttl
---
### TransmittableThreadLocal(TTL)概要

#### 功能  
  
`TransmittableThreadLocal`(`TTL`)：在使用线程池等会池化复用线程的执行组件情况下，提供`ThreadLocal`值的传递功能，解决异步执行时上下文传递的问题。一个`Java`标准库本应为框架/中间件设施开发提供的标配能力，本库功能聚焦 & 0依赖，支持`Java 6~21`。  
  
`JDK`的[`InheritableThreadLocal`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/InheritableThreadLocal.html)类可以完成父线程到子线程的值传递。但对于使用线程池等会池化复用线程的执行组件的情况，线程由线程池创建好，并且线程是池化起来反复使用的；这时父子线程关系的`ThreadLocal`值传递已经没有意义，应用需要的实际上是把 **任务提交给线程池时**的`ThreadLocal`值传递到 **任务执行时**。  
  
本库提供的[`TransmittableThreadLocal`](ttl-core/src/main/java/com/alibaba/ttl3/TransmittableThreadLocal.java)类继承并加强`InheritableThreadLocal`类，解决上述的问题。  
  
整个`TransmittableThreadLocal`库的核心功能（用户`API`、线程池`ExecutorService`/`ForkJoinPool`/`TimerTask`及其线程工厂的`Wrapper`；开发者`API`、框架/中间件的集成`API`），只有 **_~1000 `SLOC`代码行_**，非常精小。  
  
#### 使用场景  
`ThreadLocal`的使用场景即`TransmittableThreadLocal`的潜在需求场景，如果你的业务需要『在使用线程池等会池化复用线程的执行组件情况下传递`ThreadLocal`值』则是`TransmittableThreadLocal`目标使用场景。  
  
下面是几个典型场景例子。  
1. 分布式跟踪系统 或 全链路压测（即链路打标）  
2. 日志收集记录系统上下文  
3. `Session`级`Cache`  
4. 应用容器或上层框架跨应用代码给下层`SDK`传递信息  

#### 设计理念
使用类[`TransmittableThreadLocal`](ttl-core/src/main/java/com/alibaba/ttl3/TransmittableThreadLocal.java)来保存值，并跨线程池传递。  
  
`TransmittableThreadLocal`继承`InheritableThreadLocal`，使用方式也类似。相比`InheritableThreadLocal`，添加了`protected`的`transmitteeValue()`方法，用于定制 **任务提交给线程池时** 的`ThreadLocal`值传递到 **任务执行时** 的传递方式，缺省是简单的赋值传递。  
  
注意：如果传递的对象（引用类型）会被修改，且没有做深拷贝（如直接传递引用或是浅拷贝），那么  
- 因为跨线程传递而不再有线程封闭，传递对象在多个线程之间是有共享的。  
- 与`JDK`的[`InheritableThreadLocal.childValue()`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/InheritableThreadLocal.html#childValue(T))一样，需要使用者/业务逻辑注意保证传递对象的线程安全。  

#### 1. 简单使用  
父线程给子线程传递值。  
示例代码：  
  
```java  
TransmittableThreadLocal<String> context = new TransmittableThreadLocal<>();  
  
// =====================================================  
  
// 在父线程中设置  
context.set("value-set-in-parent");  
  
// =====================================================  
  
// 在子线程中可以读取，值是"value-set-in-parent"  
String value = context.get();  
```  
这其实是`InheritableThreadLocal`的功能，应该使用`InheritableThreadLocal`来完成。  
  
但对于使用线程池等会池化复用线程的执行组件的情况，线程由线程池创建好，并且线程是池化起来反复使用的；这时父子线程关系的`ThreadLocal`值传递已经没有意义，应用需要的实际上是把 **任务提交给线程池时**的`ThreadLocal`值传递到 **任务执行时**。  

#### 2. 保证线程池中传递值  
##### 2.1 修饰`Runnable`和`Callable`  
使用[`TtlRunnable`](ttl-core/src/main/java/com/alibaba/ttl3/TtlRunnable.java)和[`TtlCallable`](ttl-core/src/main/java/com/alibaba/ttl3/TtlCallable.java)来修饰传入线程池的`Runnable`和`Callable`。  
```java  
TransmittableThreadLocal<String> context = new TransmittableThreadLocal<>();  
  
// =====================================================  
  
// 在父线程中设置  
context.set("value-set-in-parent");  
  
Runnable task = new RunnableTask();  
// 额外的处理，生成修饰了的对象ttlRunnable  
Runnable ttlRunnable = TtlRunnable.get(task);  
executorService.submit(ttlRunnable);  
  
// =====================================================  
  
// Task中可以读取，值是"value-set-in-parent"  
String value = context.get();  
```  
  
**_注意_**：  
即使是同一个`Runnable`任务多次提交到线程池时，每次提交时都需要通过修饰操作（即`TtlRunnable.get(task)`）以抓取这次提交时的`TransmittableThreadLocal`上下文的值；即如果同一个任务下一次提交时不执行修饰而仍然使用上一次的`TtlRunnable`，则提交的任务运行时会是之前修饰操作所抓取的上下文。示例代码如下：  
  
```java  
// 第一次提交  
Runnable task = new RunnableTask();  
executorService.submit(TtlRunnable.get(task));  
  
// ...业务逻辑代码，  
// 并且修改了 TransmittableThreadLocal上下文 ...context.set("value-modified-in-parent");  
  
// 再次提交  
// 重新执行修饰，以传递修改了的 TransmittableThreadLocal上下文  
executorService.submit(TtlRunnable.get(task));  
```  
上面演示了`Runnable`，`Callable`的处理类似  
```java  
TransmittableThreadLocal<String> context = new TransmittableThreadLocal<>();  
  
// =====================================================  
  
// 在父线程中设置  
context.set("value-set-in-parent");  
  
Callable call = new CallableTask();  
// 额外的处理，生成修饰了的对象ttlCallable  
Callable ttlCallable = TtlCallable.get(call);  
executorService.submit(ttlCallable);  
  
// =====================================================  
  
// Call中可以读取，值是"value-set-in-parent"  
String value = context.get();  
```  
##### 整个过程的完整时序图  
  
[![时序图](/assets/img/2025-02-17-TransmittableThreadLocal(TTL)/transmittable（ttl）.png)]  
  
#### 2.2 修饰线程池  
省去每次`Runnable`和`Callable`传入线程池时的修饰，这个逻辑可以在线程池中完成。  
通过工具类[`TtlExecutors`](ttl-core/src/main/java/com/alibaba/ttl3/executor/TtlExecutors.java)完成，有下面的方法：  
- `getTtlExecutor`：修饰接口`Executor`  
- `getTtlExecutorService`：修饰接口`ExecutorService`  
- `getTtlScheduledExecutorService`：修饰接口`ScheduledExecutorService`  
  
示例代码：  
```java  
ExecutorService executorService = ...  
// 额外的处理，生成修饰了的对象executorService  
executorService = TtlExecutors.getTtlExecutorService(executorService);  
  
TransmittableThreadLocal<String> context = new TransmittableThreadLocal<>();  
  
// =====================================================  
  
// 在父线程中设置  
context.set("value-set-in-parent");  
  
Runnable task = new RunnableTask();  
Callable call = new CallableTask();  
executorService.submit(task);  
executorService.submit(call);  
  
// =====================================================  
  
// Task或是Call中可以读取，值是"value-set-in-parent"  
String value = context.get();  
```  
#### 2.3 使用`Java Agent`来修饰`JDK`线程池实现类  
  
这种方式，实现线程池的传递是透明的，业务代码中没有修饰`Runnable`或是线程池的代码。即可以做到应用代码 **无侵入**。  
\# 关于 **无侵入** 的更多说明参见文档[`Java Agent`方式对应用代码无侵入](docs/developer-guide.md#java-agent%E6%96%B9%E5%BC%8F%E5%AF%B9%E5%BA%94%E7%94%A8%E4%BB%A3%E7%A0%81%E6%97%A0%E4%BE%B5%E5%85%A5)。  
  
示例代码：  
```java  
// ## 1. 框架上层逻辑，后续流程框架调用业务 ##TransmittableThreadLocal<String> context = new TransmittableThreadLocal<>();  
context.set("value-set-in-parent");  
  
// ## 2. 应用逻辑，后续流程业务调用框架下层逻辑 ##ExecutorService executorService = Executors.newFixedThreadPool(3);  
  
Runnable task = new RunnableTask();  
Callable call = new CallableTask();  
executorService.submit(task);  
executorService.submit(call);  
  
// ## 3. 框架下层逻辑 ##// Task或是Call中可以读取，值是"value-set-in-parent"  
String value = context.get();  
```  
<blockquote>  
<details>  
  
<summary>关于<code>java.util.TimerTask</code>/<code>java.util.Timer</code> 的 展开说明</summary>  
<br>  
  
<p><code>Timer</code>是<code>JDK 1.3</code>的老类，不推荐使用<code>Timer</code>类。  
  
<p>推荐用<a href="https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ScheduledExecutorService.html" rel="nofollow"><code>ScheduledExecutorService</code></a>。<br>  
<code>ScheduledThreadPoolExecutor</code>实现更强壮，并且功能更丰富。  
如支持配置线程池的大小（<code>Timer</code>只有一个线程）；<code>Timer</code>在<code>Runnable</code>中抛出异常会中止定时执行。更多说明参见 <a href="https://alibaba.github.io/Alibaba-Java-Coding-Guidelines/#concurrency" rel="nofollow">10. <strong>Mandatory</strong> Run multiple TimeTask by using ScheduledExecutorService rather than Timer because Timer will kill all running threads in case of failing to catch exceptions. - Alibaba Java Coding Guidelines</a>。</p>  
  
</details>  
</blockquote>  
##### `Java Agent`的启动需要了解，待继续学习

####  使用`TTL`的好处与必要性  
  
**_好处：透明且自动完成所有异步执行上下文的可定制、规范化的捕捉与传递。_**  
这个好处也是`TransmittableThreadLocal`的目标。  
  
**_必要性：随着应用的分布式微服务化并使用各种中间件，越来越多的功能与组件会涉及不同的上下文，逻辑流程也越来越长；上下文问题实际上是个大的易错的架构问题，需要统一的对业务透明的解决方案。_**  
  
使用`ThreadLocal`作为业务上下文传递的经典技术手段在中间件、技术与业务框架中广泛大量使用。而对于生产应用，几乎一定会使用线程池等异步执行组件，以高效支撑线上大流量。但使用`ThreadLocal`及其`set/remove`的上下文传递模式，在使用线程池等异步执行组件时，存在多方面的问题：  
  
**_1. 从业务使用者角度来看_**  
1. **繁琐**  
- 业务逻辑要知道：有哪些上下文；各个上下文是如何获取的。  
- 并需要业务逻辑去一个一个地捕捉与传递。  
1. **依赖**  
- 需要直接依赖不同`ThreadLocal`上下文各自的获取的逻辑或类。  
- 像`RPC`的上下文（如`Dubbo`的`RpcContext`）、全链路跟踪的上下文（如`SkyWalking`的`ContextManager`）、不同业务模块中的业务流程上下文，等等。  
1. **静态（易漏）**  
- 因为要 **_事先_** 知道有哪些上下文，如果系统出现了一个新的上下文，业务逻辑就要修改添加上新上下文传递的几行代码。也就是说因 **_系统的_** 上下文新增，**_业务的_** 逻辑就跟进要修改。  
- 而对于业务来说，不关心系统的上下文，即往往就可能遗漏，会是线上故障了。  
- 随着应用的分布式微服务化并使用各种中间件，越来越多的功能与组件会涉及不同的上下文，逻辑流程也越来越长；上下文问题实际上是个大的易错的架构问题，需要统一的对业务透明的解决方案。  
1. **定制性**  
- 因为需要业务逻辑来完成捕捉与传递，业务要关注『上下文的传递方式』：直接传引用？还是拷贝传值？拷贝是深拷贝还是浅拷贝？在不同的上下文会需要不同的做法。  
- 『上下文的传递方式』往往是 **_上下文的提供者_**（或说是业务逻辑的框架部分）才能决策处理好的；而 **_上下文的使用者_**（或说是业务逻辑的应用部分）往往不（期望）知道上下文的传递方式。这也可以理解成是 **_依赖_**，即业务逻辑 依赖/关注/实现了 系统/架构的『上下文的传递方式』。  
  
**_2. 从整体流程实现角度来看_**  
  
关注的是 **上下文传递流程的规范化**。上下文传递到了子线程要做好 **_清理_**（或更准确地说是要 **_恢复_** 成之前的上下文），需要业务逻辑去处理好。如果业务逻辑对**清理**的处理不正确，比如：  
- 如果清理操作漏了：  
- 下一次执行可能是上次的，即『上下文的 **_污染_**/**_串号_**』，会导致业务逻辑错误。  
- 『上下文的 **_泄漏_**』，会导致内存泄漏问题。  
- 如果清理操作做多了，会出现上下文 **_丢失_**。  
  
上面的问题，在业务开发中引发的`Bug`真是**屡见不鲜** ！本质原因是：**_`ThreadLocal`的`set/remove`的上下文传递模式_** 在使用线程池等异步执行组件的情况下不再是有效的。常见的典型例子：  
  
- 当线程池满了且线程池的`RejectedExecutionHandler`使用的是`CallerRunsPolicy`时，提交到线程池的任务会在提交线程中直接执行，`ThreadLocal.remove`操作**清理**提交线程的上下文导致上下文**丢失**。  
- 类似的，使用`ForkJoinPool`（包含并行执行`Stream`与`CompletableFuture`，底层使用`ForkJoinPool`）的场景，展开的`ForkJoinTask`会在任务提交线程中直接执行。同样导致上下文**丢失**。  
  
#### 参考文档
1、[github-TransmittableThreadLocal(TTL)](https://github.com/alibaba/transmittable-thread-local/tree/2.x?tab=readme-ov-file#dummy)