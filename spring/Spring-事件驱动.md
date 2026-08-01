# Spring 事件驱动学习笔记

> 配套 demo：`java_learner` 项目 `event/` 包（`PushEvent` + `PushEventPublisher` + 两个 Listener + `EventConfig`），
> 验证入口在 `JavaLearnerApplication#main` 第 8 步。笔记里的日志均为实测，源码结论在同级 clone 的 spring-framework / spring-boot 里验证过。
> 异步广播依赖线程池，与《[Spring-AOP](Spring-AOP.md)》《[Spring-事务](Spring-事务.md)》是并列的 Spring 核心能力。

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
   │ 事件发布者     │                   │ （事件驱动的"总机"）          │     决定同步还是异步
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
| **Listener** 监听器 | `PushEventListener01/02` | 两种写法，见第五章 |
| **Multicaster** 广播器 | 容器内置的 `SimpleApplicationEventMulticaster` | **同步还是异步，全看它有没有线程池** ← 本篇重点 |

---

## 三、默认是同步的：广播器没有线程池

### 3.1 源码：两处决定性逻辑

`AbstractApplicationContext#initApplicationEventMulticaster`（spring-context，第 849 行）：

```java
if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
    this.applicationEventMulticaster = beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, …);
}
else {
    // 没有自定义的 → new 一个默认的，taskExecutor 是 null
    this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
}
```

其中 `APPLICATION_EVENT_MULTICASTER_BEAN_NAME = "applicationEventMulticaster"`（第 161 行常量）。

`SimpleApplicationEventMulticaster#multicastEvent`（第 142 行）：

```java
Executor executor = getTaskExecutor();                  // ← 没配线程池就是 null
for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
    if (executor != null && listener.supportsAsyncExecution()) {
        executor.execute(() -> invokeListener(listener, event));   // 异步：丢给线程池
    }
    else {
        invokeListener(listener, event);                           // 同步：就在发布者线程上跑
    }
}
```

一句话：**`taskExecutor` 为 null，事件就在调用 `publishEvent()` 的那个线程上，一个接一个串行跑完。**

### 3.2 实测（`event.async.enabled=false`）

```
00:30:49.243 [main] PushEventPublisher   : [main] 发布事件前
00:30:49.549 [main] PushEventListener01  : [main] PushEventListener01 接收到推送事件：…
00:30:49.853 [main] PushEventListener02  : [main] PushEventListener02 接收到推送事件：…
00:30:49.854 [main] PushEventPublisher   : [main] 发布事件后
```

三个信号全都指向同步（两个监听器各 `sleep(300)`）：

1. 监听器线程名是 **`main`**——和发布者同一个线程；
2. 两个监听器**串行**：`.549` → `.853`，隔了整整 300ms；
3. "发布事件后"排在**最末尾** `.854`——`publishEvent()` 把发布者**阻塞了 611ms**。

这就是同步事件最大的风险：**下游监听器慢，会拖垮上游主流程**；而且监听器抛异常会直接冲回发布者（默认与发布者同一个事务、同一个调用栈）。

---

## 四、加上线程池 → 异步广播

### 4.1 `EventConfig` 补齐后的样子

```java
@Slf4j
@EnableAsync
@Configuration
public class EventConfig {

    @Bean
    public ThreadPoolTaskExecutor eventTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("push-event-");            // 线程名前缀，日志里一眼认出
        executor.setWaitForTasksToCompleteOnShutdown(true);     // 关容器时等任务跑完
        executor.setAwaitTerminationSeconds(10);
        return executor;
    }

    /** 方法名（Bean 名）必须正好是 applicationEventMulticaster —— 见 6.1 */
    @Bean
    public ApplicationEventMulticaster applicationEventMulticaster(ThreadPoolTaskExecutor eventTaskExecutor) {
        SimpleApplicationEventMulticaster multicaster = new SimpleApplicationEventMulticaster();
        multicaster.setTaskExecutor(eventTaskExecutor);   // ★ 就是这一步把同步变成异步
        return multicaster;
    }
}
```

### 4.2 实测（默认即异步）

```
00:30:40.089 [main] PushEventPublisher   : [main] 发布事件前
00:30:40.089 [main] PushEventPublisher   : [main] 发布事件后（异步广播时这行不会等监听器执行完）
00:30:40.395 [push-event-2] PushEventListener02 : [push-event-2] …
00:30:40.395 [push-event-4] PushEventListener01 : [push-event-4] …
```

三个信号全反过来了：

1. 线程名变成 **`push-event-2` / `push-event-4`**——线程池里的线程；
2. 两个监听器**并行**：时间戳都是 `.395`，不再串行等待；
3. "发布事件后"紧跟"发布事件前"，**同一毫秒 `.089`**——发布者零阻塞，扔完就走。

### 4.3 两种模式对比

```
   同步（无线程池）                              异步（有线程池）
   main ├─ 发布前                               main ├─ 发布前
        ├─ Listener01 ──300ms──┐                     ├─ 发布后  ← 立刻返回
        ├─ Listener02 ──300ms──┤ 611ms 阻塞          └─ (继续干别的)
        └─ 发布后 ←────────────┘                push-event-2 ├─ Listener02 ┐ 并行
                                               push-event-4 ├─ Listener01 ┘
```

| | 同步（默认） | 异步（配线程池） |
|---|---|---|
| 执行线程 | 发布者线程（`main`） | 线程池线程（`push-event-N`） |
| 多个监听器 | 串行 | 并行 |
| 发布者 | 阻塞到全部跑完 | 不阻塞 |
| 监听器异常 | 冲回发布者 | **发布者感知不到**（见 6.3） |
| 事务 | 与发布者同一事务 | **脱离发布者事务**（见 6.4） |

---

## 五、两种监听器写法

```java
// 写法一：实现接口。类型由泛型 <PushEvent> 指定
@Component
public class PushEventListener01 implements ApplicationListener<PushEvent> {
    @Override
    public void onApplicationEvent(PushEvent event) { … }
}

// 写法二：@EventListener 注解。更灵活——一个类里可以监听多种事件
@Component   // ★ 别漏！见 6.2
public class PushEventListener02 {
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

## 六、踩坑清单（都在本项目验证过）

### 6.1 广播器的 Bean 名必须一字不差

```java
@Bean
public ApplicationEventMulticaster myEventMulticaster(...) { … }   // ✗ 名字错了，静默失效
```

`initApplicationEventMulticaster` 是按常量 `"applicationEventMulticaster"` **精确查名**的（3.1 源码）。
名字对不上，容器直接 new 一个默认的同步广播器——**不报错、不警告**，你的线程池就在旁边闲着，事件依然同步。
这类"静默降级"最难查，配完一定要跑一次看线程名。

### 6.2 监听器忘了 `@Component`

demo 初版的 `PushEventListener02` 就漏了 `@Component`：它压根不是 Bean，`@EventListener` 从来没被扫描注册过，
整个监听器是**死代码**，加什么线程池都不会执行。同样是不报错的静默失效。
（对照 AOP 笔记 7.3：切面只写 `@Aspect` 忘了 `@Component` 也是一模一样的坑——**注解生效的前提永远是"它先得是个 Bean"**。）

### 6.3 异步后，监听器的异常发布者收不到

同步时异常顺着调用栈冲回发布者；异步后异常发生在线程池线程里，发布者早就返回了，**默认只会打一条日志，业务无感**。
要兜住得给广播器设 `setErrorHandler(...)`，或者在监听器内部自己 try-catch 落库重试。

### 6.4 异步后，监听器脱离发布者的事务

发布者的事务还没提交，异步监听器就可能已经跑起来了——此时去查库很可能**查不到那条还没提交的数据**。
需要"事务提交后再执行"的语义，用 `@TransactionalEventListener(phase = AFTER_COMMIT)`。
注意它默认不支持异步执行（`ApplicationListener#supportsAsyncExecution` 就是给这类监听器留的口子，3.1 源码里那个 `&&` 判断），
要异步得在方法上再加 `@Async`。事务相关概念见《[Spring-事务](Spring-事务.md)》。

### 6.5 容器关闭把异步任务掐断

`main` 里发完事件立刻关容器，异步监听器的日志可能一条都看不到——线程池被 shutdown 了。
所以 demo 里配了 `setWaitForTasksToCompleteOnShutdown(true)` + `setAwaitTerminationSeconds(10)`，
`main` 第 8 步还 `Thread.sleep(1000)` 等了一下。

### 6.6 自定义 `Executor` Bean 会顶掉 Boot 的默认线程池

spring-boot 的 `TaskExecutorConfigurations.TaskExecutorConfiguration` 上标着 `@ConditionalOnMissingBean(Executor.class)`，
且它产出的 Bean 同时挂了 `applicationTaskExecutor` 和 `AsyncAnnotationBeanPostProcessor.DEFAULT_TASK_EXECUTOR_BEAN_NAME`（即 `taskExecutor`）两个名字。

```
   没有自定义 Executor                        定义了 eventTaskExecutor
   Boot 自动配 applicationTaskExecutor        Boot 整个退让，不再自动配
   （别名 taskExecutor，@Async 默认用它）       → 你这个池子成了 @Async 的默认池
```

也就是说：**加了这个线程池之后，`@Async` 用的也是它**，两边共用一个池子，容量要一起考虑。
想隔离就多定义一个池子，并在 `@Async("xxx")` 里指定名字。

### 6.7 `@EnableAsync` 不等于事件异步

demo 初版 `EventConfig` 上只有 `@EnableAsync`——**事件依然是同步的**。这俩是两条独立的路：

```
   路线 A：异步广播（本篇）                路线 B：@Async 监听器
   给 applicationEventMulticaster        在监听器方法上加 @Async
   配 taskExecutor                       （@EnableAsync 才有意义）
   → 所有事件、所有监听器都异步            → 只有标注的那个监听器异步，粒度更细
```

`@EnableAsync` 只是打开 `@Async` 的开关；**没有任何方法标 `@Async` 时，它什么也不做**。

---

## 七、速查表

| 概念 | 一句话 |
|---|---|
| `ApplicationEvent` | 事件基类，`getSource()` 拿数据载体 |
| `ApplicationEventPublisher` | 发布接口，`ApplicationContext` 也实现了它 |
| `ApplicationListener<E>` | 监听器接口写法，一个类一种事件 |
| `@EventListener` | 注解写法，支持 SpEL `condition` 过滤，首选 |
| `ApplicationEventMulticaster` | 广播器，同步/异步的总开关 |
| `applicationEventMulticaster` | 自定义广播器**必须**用这个 Bean 名，否则静默失效 |
| `SimpleApplicationEventMulticaster#setTaskExecutor` | 设了线程池 = 异步，不设 = 同步（默认） |
| `supportsAsyncExecution()` | 监听器可声明"我必须同步跑"，默认 `true` |
| `ThreadPoolTaskExecutor` | Spring 的线程池封装，核心参数 core/max/queue |
| `setWaitForTasksToCompleteOnShutdown` | 关容器时等任务跑完，异步 demo 必配 |
| `@TransactionalEventListener` | 事务提交后再触发，解决 6.4 的读不到数据 |
| `@ConditionalOnMissingBean(Executor.class)` | 自定义 Executor 会让 Boot 的默认池退让（6.6） |
| `@EnableAsync` | 只是 `@Async` 的开关，**与事件是否异步无关**（6.7） |

---

## 八、可以继续挖的方向

1. **`@TransactionalEventListener` 实测** — 在事务方法里发事件，对比 `AFTER_COMMIT` 和默认监听器能不能查到那条数据，把 6.4 跑实。
2. **`setErrorHandler` 兜异常** — 给广播器配一个，验证 6.3 里异步监听器的异常能不能被统一收口。
3. **`@Order` 控制监听顺序** — 同步模式下有效，异步模式下还有意义吗？值得实测（提示：并行执行）。
4. **SpEL 条件过滤** — `@EventListener(condition = "#event.source.userId > 1000")`，只处理关心的事件。
5. **容器内置事件** — `ContextRefreshedEvent` / `ApplicationReadyEvent` 等，看看 Boot 启动过程本身就是一串事件（对照 IOC 笔记的 `refresh()` 十二步）。
6. **走路线 B 再实现一遍** — 监听器加 `@Async`，去掉自定义广播器，对比两种做法的线程名和粒度差异（6.7）。
