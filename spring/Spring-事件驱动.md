# Spring 事件驱动学习笔记

> 配套 demo：`java_learner` 项目 `event/` 包（`PushEvent` + `PushEventPublisher` + 两个 Listener + `EventConfig`），
> 验证入口在 `JavaLearnerApplication#main` 第 8 步。笔记里的日志均为实测，源码结论在同级 clone 的 spring-framework / spring-boot 和 JDK `src.zip` 里验证过。
> 异步依赖线程池与 AOP 代理，与《[Spring-AOP](Spring-AOP.md)》《[Spring-事务](Spring-事务.md)》是并列的 Spring 核心能力。

---

## 一、事件驱动解决什么问题：把"通知"从业务里摘出去

下单成功之后要做的事，会随着业务发展越堆越多：

```
              直接调用（耦合）                        事件驱动（解耦）
   ┌────────────────────────────┐        ┌────────────────────────────┐
   │ orderService.create() {    │        │ orderService.create() {    │
   │     保存订单;               │        │     保存订单;               │
   │     smsService.send();     │        │     publish(下单成功事件);   │ ← 只管喊一嗓子
   │     pushService.push();    │        │ }                          │
   │     couponService.grant(); │        └─────────────┬──────────────┘
   │     // 又来一个需求…         │                      │ 广播
   │ }                          │        ┌─────────────┴──────────────┐
   └────────────────────────────┘        ▼             ▼              ▼
   每加一个下游，就要改一次下单代码          短信监听器      推送监听器      发券监听器
   下单方"认识"所有下游                    （加新需求 = 加个监听器，下单代码一行不动）
```

发布方只负责"宣布发生了什么"，不关心谁在听、有几个人听。这就是**观察者模式**在 Spring 里的内置实现。

---

## 二、四个角色

```
   ┌──────────────┐  publishEvent()   ┌────────────────────────────┐
   │ Publisher     │ ────────────────▶ │ ApplicationEventMulticaster │  ← 广播器：真正干活的中间人
   │ 事件发布者     │                   │ （事件驱动的"总机"）          │
   └──────────────┘                   └─────────────┬──────────────┘
        ↑ 发布                                       │ 遍历匹配的监听器并逐个 invoke
   ┌──────────────┐                    ┌────────────┴────────────┐
   │ Event 事件    │                    ▼                         ▼
   │ 承载数据       │            ┌──────────────┐         ┌──────────────┐
   └──────────────┘            │ Listener01    │         │ Listener02    │
                               └──────────────┘         └──────────────┘
```

| 角色 | demo 里是谁 | 说明 |
|---|---|---|
| **Event** 事件 | `PushEvent extends ApplicationEvent` | 数据载体，`getSource()` 取出 `PushEventMessage` |
| **Publisher** 发布者 | `PushEventPublisher` | 注入 `ApplicationEventPublisher` 调 `publishEvent()` |
| **Listener** 监听器 | `PushEventListener01/02` | 两种写法，见第六章 |
| **Multicaster** 广播器 | 容器内置的 `SimpleApplicationEventMulticaster` | 遍历并调用监听器；默认在发布者线程上串行调 |

---

## 三、默认是同步的

### 3.1 源码：广播器没有线程池就同步跑

`SimpleApplicationEventMulticaster#multicastEvent`（spring-context，第 142 行）：

```java
Executor executor = getTaskExecutor();                  // ← 默认为 null
for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
    if (executor != null && listener.supportsAsyncExecution()) {
        executor.execute(() -> invokeListener(listener, event));   // 异步：丢给线程池
    }
    else {
        invokeListener(listener, event);                           // 默认走这里：发布者线程上直接调
    }
}
```

而广播器本身从哪来，看 `AbstractApplicationContext#initApplicationEventMulticaster`（第 849 行）：
容器只认一个**名字恰好是 `applicationEventMulticaster`** 的 Bean，没有就自己 new 一个默认的（`taskExecutor` 为 null）。

一句话：**不做任何配置时，事件在调用 `publishEvent()` 的那个线程上，一个接一个串行跑完。**

### 3.2 实测（把两个监听器的 `@Async` 注释掉再跑）

```
00:45:23.471 [main] PushEventPublisher   : [main] 发布事件前
00:45:23.775 [main] PushEventListener01  : [main] PushEventListener01 接收到推送事件：…
00:45:24.081 [main] PushEventListener02  : [main] PushEventListener02 接收到推送事件：…
00:45:24.082 [main] PushEventPublisher   : [main] 发布事件后
```

三个信号全都指向同步（两个监听器各 `sleep(300)`）：

1. 监听器线程名是 **`main`**——和发布者同一个线程；
2. 两个监听器**串行**：`.775` → `.081`，隔了整整 300ms；
3. "发布事件后"排在**最末尾** `.082`——`publishEvent()` 把发布者**阻塞了 611ms**。

这就是同步事件最大的风险：**下游监听器慢，会拖垮上游主流程**；监听器抛的异常也会顺着调用栈冲回发布者。

---

## 四、异步化的两条路

```
   路线 A：异步广播器（全局）                  路线 B：@Async 监听器（局部）★ 本 demo
   给 applicationEventMulticaster            在监听器方法上加 @Async
   设 taskExecutor                           （类路径上需要 @EnableAsync）
        │                                          │
        ▼                                          ▼
   容器里【所有】事件都变异步                    只有标了 @Async 的【那个方法】异步
   ——包括 ContextRefreshedEvent 等             ——其余监听器仍然同步
     框架生命周期事件，它们的框架监听器
     也会挤占你这个池子
```

| | 路线 A 异步广播器 | 路线 B `@Async` |
|---|---|---|
| 影响范围 | 全局，所有事件所有监听器 | 精确到方法 |
| 框架事件受影响 | ✓ 会（`ContextRefreshedEvent` 等也异步） | ✗ 不会 |
| 开关位置 | 配置类里一个 Bean | 监听器方法上一个注解 |
| 异常兜底 | `multicaster.setErrorHandler(...)` | `AsyncUncaughtExceptionHandler`（见 7.3） |
| 何时用 | 想让整个应用的事件都异步 | **大多数场景**：只想让某几个慢监听器异步 |

**两条路不要同时开**——原因见 7.5。

### 4.1 本 demo 的配置（路线 B）

```java
@Slf4j
@EnableAsync              // 打开 @Async 的开关，没有它 @Async 不生效
@Configuration
public class EventConfig {

    @Bean("eventAsyncTaskExecutor")
    public ThreadPoolTaskExecutor eventAsyncTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);                            // 参数含义见第五章
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("push-event-");            // 线程名前缀，日志里一眼认出
        executor.setWaitForTasksToCompleteOnShutdown(true);     // 关容器时等任务跑完（见 7.6）
        executor.setAwaitTerminationSeconds(10);
        return executor;
    }
}
```

监听器上按名字指定这个池子：

```java
@Async("eventAsyncTaskExecutor")     // 不写名字就用默认池，写了更明确、也方便多池隔离
@EventListener(PushEvent.class)
public void receive(PushEvent event) { … }
```

### 4.2 实测

```
00:45:05.662 [main] PushEventPublisher   : [main] 发布事件前
00:45:05.663 [main] PushEventPublisher   : [main] 发布事件后
00:45:05.970 [push-event-2] PushEventListener02 : [push-event-2] …
00:45:05.970 [push-event-1] PushEventListener01 : [push-event-1] …
```

三个信号全反过来了：

1. 线程名变成 **`push-event-1` / `push-event-2`**——自定义池子里的线程；
2. 两个监听器**并行**：时间戳都是 `.970`，不再串行等待；
3. "发布事件后" `.663` 紧跟 "发布事件前" `.662`——发布者**只花了 1ms**，扔完就走（对比同步的 611ms）。

### 4.3 同步 / 异步对比

```
   同步（无 @Async）                            异步（@Async + 线程池）
   main ├─ 发布前                               main ├─ 发布前
        ├─ Listener01 ──300ms──┐                     ├─ 发布后  ← 1ms 就返回
        ├─ Listener02 ──300ms──┤ 611ms 阻塞          └─ (继续干别的)
        └─ 发布后 ←────────────┘                push-event-1 ├─ Listener01 ┐ 并行
                                               push-event-2 ├─ Listener02 ┘
```

| | 同步（默认） | 异步（`@Async`） |
|---|---|---|
| 执行线程 | 发布者线程（`main`） | 线程池线程（`push-event-N`） |
| 多个监听器 | 串行 | 并行 |
| 发布者 | 阻塞到全部跑完（611ms） | 不阻塞（1ms） |
| 监听器异常 | 冲回发布者 | **发布者感知不到**（见 7.3） |
| 事务 | 与发布者同一事务 | **脱离发布者事务**（见 7.4） |
| `@Order` 排序 | 有效 | 基本失去意义（并行执行） |

---

## 五、插播：线程池到底是什么

上面配的 `ThreadPoolTaskExecutor` 是异步的发动机，参数配错会直接把服务拖垮，值得单独讲清楚。

### 5.1 为什么不能"来一个任务开一个线程"

```java
new Thread(() -> doSomething()).start();   // 看着简单，生产上是灾难
```

- **建线程很贵**：每个 Java 线程要向操作系统申请栈空间（默认约 1MB），创建和销毁都要陷入内核态；
- **数量不可控**：流量一冲，几千个线程一起建出来，内存直接爆掉（`OutOfMemoryError: unable to create native thread`）；
- **切换有成本**：线程数远超 CPU 核数时，CPU 大部分时间在做上下文切换，真正干活的时间反而变少。

线程池的思路是**把线程当成可复用的资源**：预先建好一批，任务来了就找个空闲线程执行，执行完线程不销毁、回去等下一个任务。

```
   没有池                                   有池
   任务1 → 建线程 → 执行 → 销毁              ┌──────── 线程池 ────────┐
   任务2 → 建线程 → 执行 → 销毁              │ 线程1  线程2  线程3  线程4 │ ← 建好就常驻
   任务3 → 建线程 → 执行 → 销毁              └──────────┬─────────────┘
   （建销毁的开销可能比任务本身还大）           任务1 任务2 任务3 … 排队领取
```

### 5.2 核心：任务提交后的三步决策

这是线程池最容易记反的地方。JDK `ThreadPoolExecutor#execute` 的注释里写得很清楚（`src.zip` 第 1292 行起，
"Proceed in 3 steps"），配合 demo 的参数（core=4、queue=100、max=8）：

```
                          新任务到达
                               │
                               ▼
        ① 当前线程数 < corePoolSize(4) ? ──是──▶ 新建【核心线程】立刻执行
                               │否                （demo：第 1~4 个任务）
                               ▼
        ② 队列(100)还没满 ?        ──────是──▶ 【进队列排队】，不新建线程
                               │否                （demo：第 5~104 个任务）
                               ▼
        ③ 当前线程数 < maxPoolSize(8) ? ─是──▶ 新建【临时线程】救急
                               │否                （demo：第 105~108 个任务）
                               ▼
        ④ 执行【拒绝策略】                            （demo：第 109 个任务起）
```

用银行窗口打比方：

```
   corePoolSize   = 常驻窗口数（4 个，开门就在岗）
   queueCapacity  = 等候区座位（100 个，窗口忙就先坐着等）
   maxPoolSize    = 最多能开的窗口数（8 个，等候区坐满了才加开 4 个临时窗口）
   keepAliveTime  = 临时窗口空闲多久就撤掉（Spring 默认 60 秒）
   拒绝策略        = 等候区满了、窗口也开到头了，保安怎么劝退新来的人
```

**关键点：队列没满，绝不会开临时线程。** 很多人以为"任务多了线程数就会自动涨到 max"，其实中间隔着一整个队列。

### 5.3 实测：把 20 个任务丢进这个池子

把 `main` 第 8 步改成循环发 10 次事件（10 个事件 × 2 个监听器 = 20 个任务），统计线程名：

```
  10 push-event-1
  10 push-event-2
  10 push-event-3
  10 push-event-4
```

（每行日志出现两次线程名，所以 40 次匹配 = 20 个任务，每个线程各干 5 个。）

**只用了 4 个线程，`maxPoolSize=8` 一次都没碰到**——因为 20 个任务离"填满 100 个队列座位"差得远，
完全符合 5.2 的三步决策。这也说明：**队列开得越大，maxPoolSize 越像个摆设**。

### 5.4 参数速查（附 Spring 默认值）

| 参数 | 含义 | Spring `ThreadPoolTaskExecutor` 默认值 | demo 配置 |
|---|---|---|---|
| `corePoolSize` | 常驻线程数，不会被回收 | **1** | 4 |
| `maxPoolSize` | 线程数上限（队列满后才用得上） | **`Integer.MAX_VALUE`** | 8 |
| `queueCapacity` | 等待队列容量 | **`Integer.MAX_VALUE`** | 100 |
| `keepAliveSeconds` | 超出核心的线程空闲多久被回收 | 60 | 默认 |
| `allowCoreThreadTimeOut` | 核心线程也允许超时回收 | false | 默认 |
| `threadNamePrefix` | 线程名前缀 | `xxxExecutor-` | `push-event-` |

> ⚠️ **Spring 的默认值是个陷阱**：`new ThreadPoolTaskExecutor()` 什么都不配的话，
> 核心线程只有 **1** 个、队列却是**无界**的——等于一个单线程池，任务全在内存里无限堆积，
> `maxPoolSize` 永远用不上，压力大时先 OOM 的是队列。**自己 new 的池子，这三个参数必须显式配。**

### 5.5 四种拒绝策略

队列满 + 线程数到顶时的兜底动作（JDK 内置，Spring 用 `setRejectedExecutionHandler` 设置）：

| 策略 | 行为 | 适用 |
|---|---|---|
| `AbortPolicy` | 抛 `RejectedExecutionException`（**默认**） | 任务不能丢，要让调用方知道 |
| `CallerRunsPolicy` | 谁提交谁自己执行 | **常用**：天然给上游降速（发布者被迫等待），形成背压 |
| `DiscardPolicy` | 静默丢弃 | 可丢的任务，如埋点日志 |
| `DiscardOldestPolicy` | 丢掉队列里最老的，再塞进来 | 只关心最新数据，如行情推送 |

### 5.6 参数怎么估

```
   CPU 密集型（算法、压缩）：核心数 + 1
       线程多了只会增加上下文切换，干不完的活也快不了

   IO 密集型（查库、调接口、发消息）：核心数 × 2 起步，按实测压测调
       线程大部分时间在等 IO，可以多开几个填满 CPU 空档
```

事件监听器基本都是 IO 密集型（发短信、写库、调推送网关），所以 demo 用了 core=4。
真实项目应该压测后再定，并且**给不同业务配不同的池子**——把慢的、可能雪崩的业务隔离开，
别让发短信的任务把发券的队列占满（对照 7.7 的多池隔离）。

### 5.7 Spring 的封装与 JDK 原生的关系

```
   ThreadPoolTaskExecutor（Spring）
        │ 内部持有
        ▼
   ThreadPoolExecutor（JDK 原生，真正干活的）
```

Spring 这层封装多做了三件事：把参数变成 setter 方便配置类/配置文件注入；
把线程池的启动和销毁挂到 Bean 生命周期上（`initialize()` / `shutdown()` 自动调用）；
提供 `setWaitForTasksToCompleteOnShutdown`、`setAwaitTerminationSeconds` 这类优雅停机开关（见 7.6）。
需要原生对象时用 `executor.getThreadPoolExecutor()` 取出来。

---

## 六、两种监听器写法

```java
// 写法一：实现接口。类型由泛型 <PushEvent> 指定
@Component
public class PushEventListener01 implements ApplicationListener<PushEvent> {
    @Async("eventAsyncTaskExecutor")
    @Override
    public void onApplicationEvent(PushEvent event) { … }
}

// 写法二：@EventListener 注解。更灵活——一个类里可以监听多种事件
@Component   // ★ 别漏！见 7.2
public class PushEventListener02 {
    @Async("eventAsyncTaskExecutor")
    @EventListener(PushEvent.class)
    public void receive(PushEvent event) { … }
}
```

| | `implements ApplicationListener` | `@EventListener` |
|---|---|---|
| 一个类监听多种事件 | ✗ 一个类一种 | ✓ 多个方法各监听各的 |
| 条件过滤 | ✗ 只能方法里 if | ✓ `@EventListener(condition = "#event.xxx")` SpEL |
| 排序 | `@Order` / 实现 `Ordered` | `@Order` |
| 推荐度 | 老写法 | **首选** |

发布方也有两个入口，效果完全一致（`ApplicationContext` 本身就实现了 `ApplicationEventPublisher`，走的是同一个广播器）：

```java
publisher.publishEvent(new PushEvent(message));           // 首选：依赖更轻
applicationContext.publishEvent(new PushEvent(message));  // 等价
```

> demo 初版两行都写了，结果**每个监听器收到两次事件**——想演示两种写法的话，分成两个方法更清楚。

---

## 七、踩坑清单（都在本项目验证过）

### 7.1 `@EnableAsync` 不等于事件异步

demo 初版 `EventConfig` 上只有 `@EnableAsync`，没有任何监听器标 `@Async`——**事件依然是纯同步的**。
`@EnableAsync` 只是"打开开关"，开关背后没接任何东西时它什么也不做。异步真正生效需要**开关 + 注解 + 线程池**三者齐备。

### 7.2 监听器忘了 `@Component`

demo 初版的 `PushEventListener02` 就漏了 `@Component`：它压根不是 Bean，`@EventListener` 从来没被扫描注册过，
整个监听器是**死代码**，加什么线程池都不会执行。而且**不报错、不警告**。

（对照 AOP 笔记 7.3：切面只写 `@Aspect` 忘了 `@Component` 也是一模一样的坑——**注解生效的前提永远是"它先得是个 Bean"**。
同理 `@Async` 也走 AOP 代理，AOP 笔记 7.2 的自调用失效对它同样成立：同类里 `this.asyncMethod()` 不会异步。）

### 7.3 异步后，监听器的异常发布者收不到

同步时异常顺着调用栈冲回发布者；异步后异常发生在线程池线程里，发布者早就返回了，**业务无感**。
两条路线的兜底入口不一样：

- 路线 B（`@Async`）：void 方法的异常交给 `AsyncUncaughtExceptionHandler`，默认实现只打日志；
  要自定义就实现 `AsyncConfigurer#getAsyncUncaughtExceptionHandler`。
- 路线 A（异步广播器）：`multicaster.setErrorHandler(...)`。

保险起见，监听器内部自己 try-catch 落库重试才是业务上最稳的做法。

### 7.4 异步后，监听器脱离发布者的事务

发布者的事务还没提交，异步监听器就可能已经跑起来了——此时去查库很可能**查不到那条还没提交的数据**。
需要"事务提交后再执行"的语义，用 `@TransactionalEventListener(phase = AFTER_COMMIT)`；
它默认不支持异步执行（3.1 源码里 `listener.supportsAsyncExecution()` 那个判断就是给这类监听器留的口子），
要异步得在方法上再加 `@Async`。事务相关概念见《[Spring-事务](Spring-事务.md)》。

### 7.5 两条路线同时开 = 每个监听器被提交两次

这是本项目真实踩到的：既给 `applicationEventMulticaster` 设了 `taskExecutor`，又在监听器上加了 `@Async`。
代码能跑、日志看着也正常，但每个监听器**在线程池里过了两趟**：

```
   main ──publishEvent()──▶ 广播器发现 taskExecutor != null
                             └─▶ 池子线程 A：调用 listener（其实是 @Async 代理）
                                   └─▶ @Async 拦截器又提交一次任务
                                         └─▶ 池子线程 B：真正执行方法体 ← 日志打在这
```

这是 `multicastEvent` 的 `executor.execute(...)` 和 `@Async` 代理各提交一次的必然结果。
实测线程编号也对得上（core=4，每来一个任务新建一个线程直到 4 个，见 5.2）：

```
两条都开：push-event-1、push-event-4    ← 2 个监听器却用掉 4 个线程
只留 @Async：push-event-1、push-event-2  ← 正好 2 个
```

后果是白白多占一倍线程、多一次上下文切换，排查问题时线程名也对不上号。**二选一。**

### 7.6 容器关闭把异步任务掐断

`main` 里发完事件立刻关容器，异步监听器的日志可能一条都看不到——线程池被 shutdown 了。
所以 demo 里配了 `setWaitForTasksToCompleteOnShutdown(true)` + `setAwaitTerminationSeconds(10)`，
`main` 第 8 步还 `Thread.sleep(1000)` 等了一下。

### 7.7 自定义 `Executor` Bean 会顶掉 Boot 的默认线程池

spring-boot 的 `TaskExecutorConfigurations.TaskExecutorConfiguration` 上标着 `@ConditionalOnMissingBean(Executor.class)`，
且它产出的 Bean 同时挂了 `applicationTaskExecutor` 和 `AsyncAnnotationBeanPostProcessor.DEFAULT_TASK_EXECUTOR_BEAN_NAME`（即 `taskExecutor`）两个名字。

```
   没有自定义 Executor                        定义了 eventAsyncTaskExecutor
   Boot 自动配 applicationTaskExecutor        Boot 整个退让，不再自动配
   （别名 taskExecutor，@Async 默认用它）       → 你这个池子成了 @Async 的默认池
```

所以 `@Async` 即使不写名字也会用到它，全应用共用一个池子，容量要一起考虑。
想隔离就多定义几个池子（理由见 5.6），并像 demo 这样在 `@Async("eventAsyncTaskExecutor")` 里**显式指名**——
既避免了默认池被别人占满，也让阅读代码的人一眼看出用的是哪个池。

### 7.8 路线 A 的额外坑：广播器 Bean 名必须一字不差

如果选路线 A，方法名（Bean 名）必须正好是 `applicationEventMulticaster`：

```java
@Bean
public ApplicationEventMulticaster myEventMulticaster(...) { … }   // ✗ 名字错了，静默失效
```

`initApplicationEventMulticaster` 是按常量 `"applicationEventMulticaster"` **精确查名**的（3.1）。
名字对不上，容器直接 new 一个默认的同步广播器——不报错、不警告，线程池在旁边闲着，事件依然同步。

---

## 八、速查表

| 概念 | 一句话 |
|---|---|
| `ApplicationEvent` | 事件基类，`getSource()` 拿数据载体 |
| `ApplicationEventPublisher` | 发布接口，`ApplicationContext` 也实现了它 |
| `ApplicationListener<E>` | 监听器接口写法，一个类一种事件 |
| `@EventListener` | 注解写法，支持 SpEL `condition` 过滤，首选 |
| `ApplicationEventMulticaster` | 广播器，遍历并调用监听器 |
| `applicationEventMulticaster` | 自定义广播器**必须**用这个 Bean 名，否则静默失效（7.8） |
| `SimpleApplicationEventMulticaster#setTaskExecutor` | 路线 A：让**所有**事件异步 |
| `@Async("poolName")` | 路线 B：让**单个**监听器异步，本 demo 用法 |
| `@EnableAsync` | `@Async` 的总开关，单独用它什么也不会发生（7.1） |
| `supportsAsyncExecution()` | 监听器可声明"我必须同步跑"，默认 `true` |
| `ThreadPoolTaskExecutor` | Spring 对 JDK `ThreadPoolExecutor` 的封装（5.7） |
| 线程池三步决策 | 核心线程 → **队列** → 临时线程 → 拒绝策略；队列没满不开新线程（5.2） |
| `corePoolSize` / `maxPoolSize` / `queueCapacity` | 常驻窗口 / 最大窗口 / 等候区座位（5.2） |
| Spring 池默认值 | core=1、queue 无界——**等于单线程池，必须显式配**（5.4） |
| `CallerRunsPolicy` | 拒绝策略里最常用的一种：谁提交谁执行，天然背压（5.5） |
| `setWaitForTasksToCompleteOnShutdown` | 关容器时等任务跑完，异步 demo 必配 |
| `AsyncUncaughtExceptionHandler` | `@Async` void 方法的异常兜底入口（7.3） |
| `@TransactionalEventListener` | 事务提交后再触发，解决 7.4 的读不到数据 |
| `@ConditionalOnMissingBean(Executor.class)` | 自定义 Executor 会让 Boot 的默认池退让（7.7） |

---

## 九、可以继续挖的方向

1. **`@TransactionalEventListener` 实测** — 在事务方法里发事件，对比 `AFTER_COMMIT` 和默认监听器能不能查到那条数据，把 7.4 跑实。
2. **`AsyncUncaughtExceptionHandler` 兜异常** — 实现一个 `AsyncConfigurer`，验证 7.3 里异步监听器的异常能不能统一收口。
3. **把线程池打满** — 把 core/max/queue 调到很小（比如 1/1/1）再循环发事件，观察 `RejectedExecutionException`；
   再换成 `CallerRunsPolicy`，看任务是不是回到了 `main` 线程执行（5.5 的实证）。
4. **`@Order` 在异步下还有意义吗** — 同步模式下有效，异步并行后呢？值得实测一次。
5. **SpEL 条件过滤** — `@EventListener(condition = "#event.source.userId > 1000")`，只处理关心的事件。
6. **容器内置事件** — `ContextRefreshedEvent` / `ApplicationReadyEvent` 等，看看 Boot 启动过程本身就是一串事件（对照 IOC 笔记的 `refresh()` 十二步）；这也是路线 A 影响面大的原因。
7. **虚拟线程** — Java 21 的 `spring.threads.virtual.enabled=true` 会让 Boot 换用虚拟线程执行器，IO 密集场景下 5.6 那套估算规则会被彻底改写。
