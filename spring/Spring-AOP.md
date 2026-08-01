# Spring AOP 学习笔记

> 配套 demo：`java_learner` 项目 `aop/` 包（`ReturnLog` 注解 + `ReturnLogAspect` 切面），
> 验证入口在 `JavaLearnerApplication#main` 第 6 步。笔记里的日志均为实测。
> 容器如何创建代理的部分，与《[Spring-IOC-与-Bean](Spring-IOC-与-Bean.md)》第五、六章的 `BeanPostProcessor` 直接衔接。

---

## 一、AOP 解决什么问题：横切关注点

业务代码里总有一类逻辑，跟业务无关，却在每个方法里重复出现——日志、事务、权限、耗时统计：

```
                 下单()            退款()            查账户()
                ┌──────────┐     ┌──────────┐     ┌──────────┐
   日志 ──────▶ │ 打日志    │     │ 打日志    │     │ 打日志    │  ◀── 每个方法都要写一遍
                ├──────────┤     ├──────────┤     ├──────────┤
   事务 ──────▶ │ 开事务    │     │ 开事务    │     │ 开事务    │  ◀── 和业务无关
                ├──────────┤     ├──────────┤     ├──────────┤
   业务 ──────▶ │ 真正的业务 │     │ 真正的业务 │     │ 真正的业务 │  ◀── 唯一真正想写的部分
                ├──────────┤     ├──────────┤     ├──────────┤
   事务 ──────▶ │ 提交/回滚 │     │ 提交/回滚 │     │ 提交/回滚 │
                └──────────┘     └──────────┘     └──────────┘
                     │                │                │
                     └────────────────┴────────────────┘
                              竖着看是一个个方法（OOP 的模块化方向）
                              横着看是一条条公共逻辑（AOP 的模块化方向）
```

- **OOP** 纵向拆分：按"业务归属"把代码组织成类和方法。
- **AOP**（Aspect Oriented Programming，面向切面编程）横向拆分：把散落在 N 个方法里的同一类公共逻辑抽出来，集中写在一个"切面"里，再声明"它该织入到哪些方法"。

一句话：**业务方法只写业务，公共逻辑写一次、到处生效**。

---

## 二、核心概念地图

术语多是 AOP 劝退的主因，其实一张图就能摆平——全部用本项目的 demo 对号入座：

```
┌──────────────────────────── Aspect 切面 ─────────────────────────────┐
│  ReturnLogAspect（@Aspect + @Component）                             │
│                                                                     │
│  ┌───────────── Pointcut 切点 ──────────────┐   回答"在哪儿切"        │
│  │ @Pointcut("@annotation(…ReturnLog)")     │   —— 一个筛选表达式，    │
│  │ 命中：所有标了 @ReturnLog 的方法            │   圈出要增强的方法集合   │
│  └──────────────────────────────────────────┘                       │
│  ┌───────────── Advice 通知 ────────────────┐   回答"切了干什么"       │
│  │ @Around public Object around(pjp) {…}    │   —— 增强逻辑本身，      │
│  │ 干的事：方法返回后打印返回值                 │   以及执行时机          │
│  └──────────────────────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────────┘
                     │
                     │ Weaving 织入：容器启动时把切面逻辑"编"进目标对象
                     │ （Spring 的实现方式 = 运行期生成代理对象，见第三章）
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Target 目标对象：StorageServiceImpl                                  │
│                                                                     │
│  Join Point 连接点：findAccountByUsername() 的这一次调用               │
│  （理论上每个方法调用都是潜在的连接点，被 Pointcut 选中的才真正被增强）     │
└─────────────────────────────────────────────────────────────────────┘
```

| 术语 | 一句话 | 本 demo 里是谁 |
|---|---|---|
| **Join Point** 连接点 | 程序里"可以插逻辑"的位置，Spring AOP 里=方法调用 | `findAccountByUsername()` 的每次调用 |
| **Pointcut** 切点 | 筛选表达式，从所有连接点里圈出要增强的那批 | `@annotation(…ReturnLog)` |
| **Advice** 通知 | 插进去的逻辑 + 执行时机（前/后/环绕…） | `@Around` 的 `around()` 方法 |
| **Aspect** 切面 | 切点 + 通知的打包体，一个类 | `ReturnLogAspect` |
| **Target** 目标对象 | 被增强的原始对象 | `StorageServiceImpl` 原始实例 |
| **Proxy** 代理对象 | 织入切面后生成的"替身"，容器里存的是它 | `StorageServiceImpl$$SpringCGLIB$$0` |
| **Weaving** 织入 | 把切面和目标"编"到一起的过程 | Spring 在运行期用动态代理完成 |

记忆锚点：**Pointcut 管"在哪切"，Advice 管"切了干啥"，俩加起来打包成 Aspect**。

---

## 三、AOP 的本质：一个代理对象

### 3.1 调用链路

Spring AOP 没改你的字节码原文件，它玩的是"偷梁换柱"：容器里注册的不再是你写的那个对象，而是包了一层切面逻辑的**代理对象**。

```
              没有 AOP                              有 AOP
   ┌────────┐        ┌────────────┐    ┌────────┐        ┌───────────────────────┐
   │ 调用方  │ ─────▶ │ Target      │    │ 调用方  │ ─────▶ │ Proxy 代理对象          │
   │        │        │ 直接执行业务 │    │        │        │ ┌───────────────────┐ │
   └────────┘        └────────────┘    └────────┘        │ │ 前置逻辑            │ │
                                                         │ ├───────────────────┤ │
                                       容器里存的是代理！   │ │ target.业务方法()   │──┼──▶ Target
                                       调用方毫不知情      │ ├───────────────────┤ │
                                                         │ │ 后置逻辑（打印返回值）│ │
                                                         └─┴───────────────────┴─┘
```

实测验证（`main` 第 6 步，从容器里拿 `IStorageService` 打印真实类型）：

```
i.g.l.j.JavaLearnerApplication : storageService 的真实类型 = io.github.leyouhong.java_learner.service.StorageService.StorageServiceImpl$$SpringCGLIB$$0
```

类名后面挂着 `$$SpringCGLIB$$0`——容器里的确不是 `StorageServiceImpl` 本尊，而是 CGLIB 动态生成的子类代理。

### 3.2 两种代理方式：JDK 动态代理 vs CGLIB

```
   JDK 动态代理（基于接口）                 CGLIB（基于继承）
   ┌────────────────┐                   ┌──────────────────────┐
   │ IStorageService │◀───实现───┐       │ StorageServiceImpl    │
   └────────────────┘           │       └──────────┬───────────┘
        ▲                       │                  │ 继承
        │ 实现                   │       ┌──────────▼─────────────────┐
   ┌────┴───────────┐   ┌───────┴────┐  │ StorageServiceImpl          │
   │ $Proxy123      │   │ 原始 Target │  │ $$SpringCGLIB$$0（子类）     │
   │ (代理和 Target  │   └────────────┘  │ 重写方法，插入切面逻辑后        │
   │  是"兄弟"关系)   │                   │ 再调 super/target 的原方法    │
   └────────────────┘                   └────────────────────────────┘
```

| | JDK 动态代理 | CGLIB |
|---|---|---|
| 原理 | 运行期生成一个**实现相同接口**的代理类 | 运行期生成一个**继承目标类**的子类 |
| 前提 | 目标必须实现接口 | 类和方法不能是 `final` |
| Spring Boot 默认 | ✗ | ✓（`spring.aop.proxy-target-class` 默认 `true`） |

> **为什么 demo 里没写 `@EnableAspectJAutoProxy` 切面也生效了？**
> Spring Boot 的 `AopAutoConfiguration`（spring-boot-autoconfigure 里可查）在满足两个条件时自动等效开启它：
> ① classpath 上有 `org.aspectj.weaver.Advice`（我们 pom 里引了 `aspectjweaver`）；
> ② `spring.aop.auto` 没被设成 `false`（默认开）。
> 且其内部 `CglibAutoProxyConfiguration` 标着 `matchIfMissing = true`——不配置就走 CGLIB。
> 所以哪怕 `StorageServiceImpl` 实现了接口，拿到的仍是 CGLIB 子类代理，与实测吻合。

### 3.3 代理是什么时候生成的：接上 IOC 笔记

这正是《Spring-IOC-与-Bean》五、六章埋的伏笔——AOP 代理就是一个 `BeanPostProcessor` 干的活：

```
   实例化 ──▶ 属性注入 ──▶ BPP.before ──▶ 初始化方法 ──▶ BPP.after ──▶ 入 singletonObjects
                                                          │
                                          AnnotationAwareAspectJAutoProxyCreator
                                          （它就是个 BeanPostProcessor）
                                          在 postProcessAfterInitialization 里判断：
                                          这个 Bean 有没有被某个 Pointcut 命中？
                                            ├─ 没命中 → 原样返回
                                            └─ 命中   → 返回 CGLIB/JDK 代理 ←═ 偷梁换柱发生在这
```

所以最终进一级缓存、被注入到别人身上的，是代理对象。这也解释了循环依赖三级缓存里
`getEarlyBeanReference` 为什么存在（IOC 笔记 7.x）：有 AOP 时得提前把代理暴露出去。

---

## 四、五种 Advice 与执行时序

```
                        ┌────────────────────────────────────┐
        进入代理方法 ──▶  │  @Around 前半段（proceed() 之前）      │
                        │   ┌──────────────────────────────┐  │
                        │   │ @Before                      │  │
                        │   │  ┌────────────────────────┐  │  │
                        │   │  │   目标方法真正执行        │  │  │
                        │   │  └───────────┬────────────┘  │  │
                        │   │   正常返回 ───┴─── 抛异常      │  │
                        │   │      │              │        │  │
                        │   │ @AfterReturning  @AfterThrowing │
                        │   │      └──────┬───────┘        │  │
                        │   │       @After（finally，都执行） │  │
                        │   └──────────────────────────────┘  │
                        │  @Around 后半段（proceed() 之后）      │
                        └────────────────────────────────────┘
```

| 注解 | 时机 | 拿得到返回值？ | 能改变执行流程？ |
|---|---|---|---|
| `@Before` | 目标方法前 | ✗ | ✗ |
| `@AfterReturning` | 正常返回后 | ✓（只读） | ✗ |
| `@AfterThrowing` | 抛异常后 | ✗ | ✗ |
| `@After` | 无论成败都执行（类似 finally） | ✗ | ✗ |
| `@Around` | 包住整个调用 | ✓ 且可替换 | ✓ 可以不调 `proceed()`、改参数、改返回值、吞异常 |

`@Around` 是超集：它手握 `ProceedingJoinPoint`，`proceed()` 调不调、何时调、传什么参数都由它说了算。
demo 里要"方法返回后打印返回值"，用 `@AfterReturning` 其实也够；选 `@Around` 是因为它能拿到返回值引用、必要时还能加耗时统计等扩展。

---

## 五、Pointcut 表达式速览

| 指示符 | 含义 | 例子 |
|---|---|---|
| `execution` | 按方法签名匹配（最常用） | `execution(public * io.github..service.*.*(..))` |
| `@annotation` | 方法上标了某注解 | `@annotation(io.github.leyouhong.java_learner.aop.ReturnLog)` ← 本 demo |
| `within` | 某包/某类下的所有方法 | `within(io.github.leyouhong..*)` |
| `@within` | 类上标了某注解 | `@within(org.springframework.stereotype.Service)` |
| `bean` | 按 Bean 名字匹配 | `bean(storageServiceImpl)` |
| `args` / `@args` | 按参数类型/参数注解匹配 | `args(java.lang.String)` |

demo 选 `@annotation` 的好处：**要不要被增强，由业务方法自己贴标签决定**，切面不用维护一份包名清单，加一个方法就贴一个注解，侵入最小。

---

## 六、实战逐行解读：`@ReturnLog` + `ReturnLogAspect`

### 6.1 自定义注解（开关）

```java
@Target(ElementType.METHOD)                 // 只能贴在方法上
@Retention(RetentionPolicy.RUNTIME)         // 保留到运行期——反射要读它，缺了这行切面拿不到
public @interface ReturnLog {
    String prefix() default StringUtils.EMPTY;   // 日志前缀，给使用方一个自定义口子
}
```

### 6.2 切面

```java
@Slf4j
@Aspect            // 声明这是切面
@Component         // 别忘了！@Aspect 不注册 Bean，两个注解缺一不可
public class ReturnLogAspect {

    @Pointcut("@annotation(io.github.leyouhong.java_learner.aop.ReturnLog)")
    private void pointcut() {}               // 空方法只是给切点起个名，方便复用

    @Around("pointcut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        String clazzName = joinPoint.getTarget().getClass().getName();  // 目标类名
        String methodName = joinPoint.getSignature().getName();          // 方法名

        Object result = joinPoint.proceed();  // ★ 放行：真正执行目标方法，拿到返回值

        // 从切点签名反射拿到 Method，再读方法上的 @ReturnLog 注解
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        ReturnLog returnLog = method.getAnnotation(ReturnLog.class);

        log.info("{}{}.{}() return {}", returnLog.prefix(), clazzName, methodName, result);
        return result;                        // ★ 必须把返回值传回去，否则调用方拿到 null
    }
}
```

### 6.3 使用方：贴注解即可

```java
@Override
@ReturnLog(prefix = "[账户查询]")
public Object findAccountByUsername(String username) {
    return "ImoocEngineeringGuide";
}
```

### 6.4 实测日志

`main` 第 6 步运行输出：

```
i.g.l.java_learner.aop.ReturnLogAspect   : [账户查询]io.github.leyouhong.java_learner.service.StorageService.StorageServiceImpl.findAccountByUsername() return ImoocEngineeringGuide
i.g.l.j.JavaLearnerApplication           : storageService 的真实类型 = io.github.leyouhong.java_learner.service.StorageService.StorageServiceImpl$$SpringCGLIB$$0
```

一个值得注意的现象：这条切面日志实际打了**两次**。因为 `main` 第 3 步
`accountService.queryAccountInfo()` 内部调用的 `storageService` 也是注入的代理对象——
**只要调用走的是代理，无论谁发起，切面都生效**；这也为 7.2 的"自调用失效"埋了个对照。

---

## 七、踩坑清单（都在本项目验证过）

### 7.1 SLF4J 占位符顺序 bug（实测抓到的）

切面初版写的是：

```java
log.info("{}{}.{}() return {}", clazzName, methodName, returnLog.prefix(), result);
```

占位符是"前缀 类名.方法名()"的意图，参数却按"类名, 方法名, 前缀"传，实测输出成了：

```
...StorageServiceImplfindAccountByUsername.[账户查询]() return ImoocEngineeringGuide
        └── 类名和方法名粘在一起了        └── 前缀跑到了方法名的位置
```

SLF4J 的 `{}` 只按**位置**依次填参，不看语义。修正为 `returnLog.prefix(), clazzName, methodName, result` 后输出才正确（见 6.4）。教训：**带多个 `{}` 的日志，写完对着占位符数一遍参数顺序**。

### 7.2 自调用失效：AOP 最经典的坑

```
   外部调用（走代理，切面生效✓）              自调用（不走代理，切面失效✗）
   ┌────────┐    ┌─────────────┐          ┌─────────────────────────┐
   │ 调用方  │ ─▶ │ Proxy        │          │ Target                  │
   └────────┘    │  切面逻辑     │          │  methodA() {            │
                 │  ┌─────────┐ │          │     this.methodB();  ←──┼── this 是原始对象，
                 │  │ Target  │ │          │  }                      │   不是代理！
                 │  └─────────┘ │          │  @ReturnLog methodB()   │   切面被绕过
                 └──────────────┘          └─────────────────────────┘
```

同一个类里 `methodA()` 内部调 `this.methodB()`，哪怕 `methodB` 标了 `@ReturnLog` 也不会打日志——因为 `this` 指向原始对象，压根没经过代理这层壳。`@Transactional`、`@Async` 失效的头号原因也是它。解法：把 `methodB` 挪到另一个 Bean；或注入自身代理（`@Lazy` 注入自己）再调。

### 7.3 其他限制与暗坑

| 坑 | 原因 |
|---|---|
| `private` / `static` / `final` 方法增强不了 | CGLIB 靠**子类重写**插逻辑，这三种方法没法被重写 |
| 切面类不生效 | 只写了 `@Aspect` 忘了 `@Component`——切面自己首先得是个 Bean |
| JDK 代理下 `method.getAnnotation()` 拿到 `null` | 若切到接口方法（JDK 代理时 `getMethod()` 返回的是**接口**上的方法），注解贴在实现类上就读不到 → NPE。稳妥写法是 IOC 笔记速查表里的 `AnnotationUtils.findAnnotation`（能穿透代理和类层级）。Boot 默认 CGLIB 所以本 demo 没炸，但别依赖这个默认 |
| `@Around` 忘了 `return result` | 调用方拿到 `null`，且编译器不报错，线上才炸 |

---

## 八、速查表

| 概念 | 一句话 |
|---|---|
| Join Point | 可插入增强的位置，Spring AOP 里就是方法调用 |
| Pointcut | 筛选连接点的表达式，"在哪切" |
| Advice | 增强逻辑 + 时机，"切了干啥" |
| Aspect | Pointcut + Advice 的打包类，`@Aspect` 标注 |
| Weaving | 织入；Spring 用运行期动态代理实现 |
| `ProceedingJoinPoint` | 只有 `@Around` 能用，`proceed()` 放行目标方法 |
| `MethodSignature` | 从 JoinPoint 拿 `Method` 反射对象的桥梁 |
| JDK 动态代理 | 基于接口，代理与目标是兄弟关系 |
| CGLIB | 基于继承，生成 `$$SpringCGLIB$$` 子类；Boot 默认 |
| `spring.aop.proxy-target-class` | Boot 里默认 `true`（CGLIB），设 `false` 换 JDK 代理 |
| `AopAutoConfiguration` | Boot 自动配置，classpath 有 aspectjweaver 即等效 `@EnableAspectJAutoProxy` |
| `AnnotationAwareAspectJAutoProxyCreator` | 真正干活的 `BeanPostProcessor`，在初始化后把 Bean 换成代理 |
| 自调用失效 | `this.xxx()` 不走代理，切面/事务/异步全失效 |

---

## 九、可以继续挖的方向

1. **`@Transactional` 就是 AOP** — 事务切面 `TransactionInterceptor` 的 invoke 值得读，自调用失效在事务上复现一遍最有体感。已展开成独立笔记：《[Spring-事务](Spring-事务.md)》。
2. **多切面排序** — 两个切面命中同一方法时，用 `@Order` 控制洋葱的内外层，打日志验证执行顺序。
3. **循环依赖 + AOP** — IOC 笔记十一.3 的实验：给 `ImoocCycleA` 加 `@ReturnLog`，在 `getEarlyBeanReference` 打断点，看二级缓存里存的是不是代理。
4. **在 `postProcessAfterInitialization` 打断点** — 亲眼看 `AnnotationAwareAspectJAutoProxyCreator` 返回代理对象的瞬间（IOC 笔记十一.4）。
5. **换 JDK 代理再跑一遍** — `application.yml` 里设 `spring.aop.proxy-target-class=false`，观察真实类型变成 `$Proxy` 开头、以及 7.3 那个 `getAnnotation` 为 null 的 NPE 能否复现。
