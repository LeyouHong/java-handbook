# Spring 事务学习笔记

> 配套 demo：`java_learner` 项目 `transaction/` 包（`EcommerceAccountServiceImpl` + 两个 Wrapper DAO），
> 验证入口在 `JavaLearnerApplication#main` 第 7 步；数据库用 H2 内存库（`schema.sql` 启动建表），笔记里的日志均为实测。
> 声明式事务的底层就是 AOP，与《[Spring-AOP](Spring-AOP.md)》直接衔接；源码结论在同级 clone 的 spring-framework 里验证过。

---

## 一、事务解决什么问题：同成同败

demo 的业务场景：注册一个电商账户要写**两张表**——账户表 + VIP 表。要是账户写成功了、VIP 写失败了，数据就"瘸"了：

```
             没有事务                                有事务
   ┌──────────────────────────┐          ┌──────────────────────────────┐
   │ INSERT 账户表   ✓ 成功     │          │ ┌─── 事务边界 ───────────────┐ │
   │ INSERT VIP 表  ✗ 抛异常   │          │ │ INSERT 账户表   ✓          │ │
   │                          │          │ │ INSERT VIP 表  ✗ 抛异常    │ │
   │ 结果：账户表多了一条        │          │ └──────────┬────────────────┘ │
   │ "半成品"数据，没有 VIP 记录 │          │            ▼                  │
   │ ——脏数据！                │          │   整体回滚：两张表都当没发生过   │
   └──────────────────────────┘          └──────────────────────────────┘
```

事务的承诺就是 ACID 里的 **A（原子性）**：边界内的所有写操作，要么全部提交，要么全部回滚。
剩下三个字母：**C** 一致性（结果）、**I** 隔离性（并发事务互不干扰，见隔离级别）、**D** 持久性（提交了就不丢）。

Spring 不自己实现事务——底层靠数据库的 `commit`/`rollback`，Spring 做的是**把"开启/提交/回滚"这套样板代码管起来**，给了两种用法。

---

## 二、两种形式总览

```
   形式一：编程式（Programmatic）              形式二：声明式（Declarative）
   事务边界 = 自己用代码圈                     事务边界 = AOP 代理帮你圈

   public void addUser() {                   @Transactional              ← 只贴个注解
       transactionTemplate.execute(...  ┐    public void addUser() {
           写账户表;                     │事      写账户表;                 ← 方法体全是业务
           写 VIP 表;                    │务      写 VIP 表;
       ...);                            ┘界   }
   }                                         （事务代码一行都看不见，
   （事务边界肉眼可见，但和业务代码缠在一起）        它们藏在代理对象里）
```

| | 编程式 `TransactionTemplate` | 声明式 `@Transactional` |
|---|---|---|
| 事务边界 | 代码显式圈定，可以只包方法中的**几行** | 整个方法，粒度最小到方法级 |
| 侵入性 | 高——业务代码里混着事务模板 | 低——一个注解，业务纯净 |
| 实现机制 | 直接调用事务管理器 | AOP 代理 + `TransactionInterceptor` |
| 失效风险 | 几乎没有（不依赖代理） | 自调用/private/吞异常等一堆坑（见第六章） |
| 适用场景 | 长方法里只想给一小段加事务；批处理分批提交 | 绝大多数业务方法（主流选择） |

---

## 三、形式一：编程式 `TransactionTemplate`

### 3.1 代码（demo：`addAccountUserTransactionTemplate`）

```java
private final TransactionTemplate transactionTemplate;   // Boot 有事务管理器时自动配好，直接注入

public void addAccountUserTransactionTemplate(String username, boolean mockFail) {
    // execute 带返回值；不需要返回值用 executeWithoutResult（老写法是 new TransactionCallbackWithoutResult() 匿名类）
    transactionTemplate.executeWithoutResult(status -> {
        ecommerceAccountWrapper.insert(username);        // 写账户表
        if (mockFail) {
            throw new IllegalStateException("模拟 VIP 开通失败");
        }
        ecommerceAccountVipWrapper.insert(username);     // 写 VIP 表
    });
}
```

### 3.2 模板到底替你干了什么（spring-tx 源码 `TransactionTemplate#execute`，已验证）

```
                    transactionTemplate.execute(callback)
                                   │
        ┌──────────────────────────▼───────────────────────────┐
        │ ① status = transactionManager.getTransaction(this)   │ ← 开事务
        │ ② try {                                              │
        │       result = action.doInTransaction(status)        │ ← 执行你的回调（业务代码）
        │    }                                                 │
        │    catch (RuntimeException | Error ex) {             │
        │       rollbackOnException(status, ex); throw ex;     │ ← 异常 → 回滚
        │    }                                                 │
        │    catch (Throwable ex) {                            │
        │       rollbackOnException(status, ex);               │ ← 连受检异常也兜住回滚！
        │       throw new UndeclaredThrowableException(ex, …); │
        │    }                                                 │
        │ ③ transactionManager.commit(status)                  │ ← 没异常 → 提交
        └──────────────────────────────────────────────────────┘
```

划重点：**编程式对"任何 `Throwable`"都回滚**——受检异常也不放过（第二个 catch）。
这跟声明式的默认行为不一样，正是第六章失效②的伏笔。

### 3.3 实测

```
i.g.l.j.JavaLearnerApplication : 编程式事务抛出异常: 模拟 VIP 开通失败
i.g.l.j.JavaLearnerApplication : [编程式] tpl-ok 落库? true (vip: true) | tpl-fail 落库? false ← 回滚则应为 false
```

`tpl-ok` 两张表都写进去了（提交）；`tpl-fail` 在两次写库之间抛了异常，账户表那笔 INSERT 被回滚——查不到数据。

---

## 四、形式二：声明式 `@Transactional`，本质是 AOP

### 4.1 代码（demo：`addAccountUserAnnotation`）

```java
@Transactional(rollbackFor = Exception.class)
public void addAccountUserAnnotation(String username, boolean mockFail) {
    ecommerceAccountWrapper.insert(username);
    if (mockFail) {
        throw new IllegalStateException("模拟 VIP 开通失败");
    }
    ecommerceAccountVipWrapper.insert(username);
}
```

方法体里一行事务代码都没有。事务逻辑去哪了？——**它是一个 `@Around` 式的环绕通知**，套在 AOP 代理里：

```
   调用方 ──▶ ┌─ EcommerceAccountServiceImpl$$SpringCGLIB$$0（代理） ─────┐
             │                                                         │
             │  TransactionInterceptor（事务拦截器，就是个环绕通知）：       │
             │  ┌─────────────────────────────────────────────┐        │
             │  │ ① getTransaction()          ← 开事务          │        │
             │  │ ② target.addAccountUserAnnotation()          │        │
             │  │      └── 你的业务代码（原始对象上执行）           │        │
             │  │ ③ 抛异常且命中回滚规则 → rollback()             │        │
             │  │    正常返回              → commit()           │        │
             │  └─────────────────────────────────────────────┘        │
             └─────────────────────────────────────────────────────────┘
```

对照 AOP 笔记就是一套东西：
- **Pointcut**：标了 `@Transactional` 的方法（`TransactionAttributeSourcePointcut`）
- **Advice**：`TransactionInterceptor`（开启→执行→提交/回滚）
- **代理生成**：还是 `BeanPostProcessor` 在初始化后偷梁换柱（AOP 笔记 3.3）

所以 AOP 的所有失效场景，`@Transactional` **一个不落全继承**（第六章）。

### 4.2 常用参数（demo：`demo()` 方法）

```java
@Transactional(isolation = Isolation.DEFAULT,      // 隔离级别：跟随数据库默认
        propagation = Propagation.REQUIRED,        // 传播行为：有事务就加入，没有就新建（默认）
        rollbackFor = Exception.class,             // 回滚范围扩大到受检异常
        timeout = 30)                              // 超时秒数（注意单位是秒，不是毫秒！）
public void demo() { ... }
```

| 参数 | 默认值 | 说明 |
|---|---|---|
| `propagation` | `REQUIRED` | 传播行为：当前有事务就加入，没有就新建。常用的还有 `REQUIRES_NEW`（挂起外层、独立新事务）、`NESTED`（嵌套保存点） |
| `isolation` | `DEFAULT` | 隔离级别，`DEFAULT` = 用数据库自己的（MySQL InnoDB 是可重复读） |
| `rollbackFor` | 空 | **默认只回滚 `RuntimeException` 和 `Error`**；要连受检异常一起回滚，必须显式写 `rollbackFor = Exception.class` |
| `timeout` | -1（不限） | 单位是**秒**。课程 slide 上写 30000 意思是 30 秒的话就写错了单位 |
| `readOnly` | false | 只读事务，给数据库一个优化提示 |

### 4.3 实测

```
i.g.l.j.JavaLearnerApplication : 声明式事务抛出异常: 模拟 VIP 开通失败
i.g.l.j.JavaLearnerApplication : [声明式] anno-ok 落库? true (vip: true) | anno-fail 落库? false ← 回滚则应为 false
```

行为和编程式完全一致：`anno-ok` 提交、`anno-fail` 回滚。区别只在"事务边界谁来圈"。

---

## 五、Boot 里为什么零配置就能用

和 AOP 笔记 3.2 的 `AopAutoConfiguration` 同一个套路：

```
   classpath 上有 spring-boot-starter-jdbc + H2
        │
        ├─▶ DataSourceAutoConfiguration          自动配出 DataSource（内存 H2）
        ├─▶ DataSourceTransactionManagerAutoConfiguration
        │        自动配出 PlatformTransactionManager（JDBC 版）
        └─▶ TransactionAutoConfiguration
                 ├─ 等效开启 @EnableTransactionManagement（声明式可用）
                 └─ 自动配出 TransactionTemplate Bean（编程式直接注入）
```

所以 demo 里 `TransactionTemplate` 是直接 `@RequiredArgsConstructor` 注入的，一行配置都没写。

---

## 六、失效案例四连（全部实测）

声明式事务 = AOP + 回滚规则，所以失效也分两类：**AOP 层失效**（切面根本没生效）和**规则层失效**（切面生效了但判定不回滚）。

### 6.1 失效①：异常被 try-catch 吞掉（规则层）

```java
@Transactional(rollbackFor = Exception.class)
public void invalid1(String username) {
    try {
        ecommerceAccountWrapper.insert(username);
        throw new IllegalStateException("业务出错了，但异常被我自己吞了");
    } catch (Exception ex) {
        log.warn("invalid1 把异常吞掉了: {}", ex.getMessage());   // ← 异常到此为止
    }
}
```

```
   TransactionInterceptor（方法外层）
   ┌──────────────────────────────────┐
   │  invalid1() {                    │
   │     try { ...抛异常... }          │
   │     catch { 内部消化 }    ←───────┼── 异常没冲出方法，
   │  }                               │   拦截器只看到"正常返回"
   │  拦截器判断：没异常 → commit ✓      │
   └──────────────────────────────────┘
```

实测——异常吞了，数据照样落库：

```
i.g.l.j.t.EcommerceAccountServiceImpl    : invalid1 把异常吞掉了: 业务出错了，但异常被我自己吞了
i.g.l.j.JavaLearnerApplication           : [失效①吞异常] invalid1-user 落库? true ← 失效则为 true
```

**回滚的信号就是"异常冲出方法边界"**。要自己 catch 做善后可以，但要么重新 throw，要么手动 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`。

### 6.2 失效②：受检异常 + 默认回滚规则（规则层）

```java
@Transactional                                    // ← 没写 rollbackFor
public void invalid2(String username) throws Exception {
    ecommerceAccountWrapper.insert(username);
    throw new SQLException("受检异常：默认不在回滚范围内");
}
```

判定回滚的默认规则在 spring-tx 源码 `DefaultTransactionAttribute#rollbackOn`（已验证）：

```java
public boolean rollbackOn(Throwable ex) {
    return (ex instanceof RuntimeException || ex instanceof Error);   // 受检异常不在内！
}
```

实测——`SQLException` 冲出去了，事务照样提交：

```
i.g.l.j.JavaLearnerApplication : 失效②抛出受检异常: 受检异常：默认不在回滚范围内
i.g.l.j.JavaLearnerApplication : [失效②受检异常] invalid2-user 落库? true ← 失效则为 true
```

对比第三章：**编程式模板连 `Throwable` 都回滚，声明式默认只回滚运行时异常**——这是两种形式一个真实的行为差异。
习惯上声明式统一写 `@Transactional(rollbackFor = Exception.class)` 抹平这个坑。

### 6.3 失效③：private 方法（AOP 层）

```java
@Transactional(rollbackFor = Exception.class)
private void invalid3(String username) { ... }    // IDEA 直接标黄警告
```

CGLIB 靠**子类重写**插入事务逻辑，private 方法没法被重写；而且 private 方法从类外根本调不到，
只能自调用——撞上失效④，双重失效。这也是四个案例里唯一没法单独跑给你看的：它连被外部触发的入口都没有。

### 6.4 失效④：自调用（AOP 层，最经典）

```java
public void invalid4_1(String username) {
    invalid4_2(username);                 // ← 实际是 this.invalid4_2()，this 是原始对象！
}

@Transactional(rollbackFor = Exception.class)
public void invalid4_2(String username) {
    ecommerceAccountWrapper.insert(username);
    throw new IllegalStateException("自调用场景：这个异常不会触发回滚");
}
```

```
   调用方 ──▶ Proxy.invalid4_1()
              │  invalid4_1 没标注解 → 代理不开事务，直接进原始对象
              ▼
            Target.invalid4_1() {
                this.invalid4_2();   ← this = 原始对象，不再经过代理
            }                          invalid4_2 的 @Transactional 形同虚设
```

实测——异常抛了，数据没回滚：

```
i.g.l.j.JavaLearnerApplication : 失效④抛出异常: 自调用场景：这个异常不会触发回滚
i.g.l.j.JavaLearnerApplication : [失效④自调用] invalid4-user 落库? true ← 失效则为 true
```

和 AOP 笔记 7.2 是同一个坑的事务版。解法一样：拆到另一个 Bean，或注入自身代理再调。

### 6.5 失效场景分类总结

```
                        @Transactional 失效
                       ┌────────┴────────┐
              AOP 层：切面没套上            规则层：切面套上了但判定不回滚
              ├─ ③ private/final/static   ├─ ① 异常被 try-catch 吞掉
              ├─ ④ 自调用 this.xxx()       └─ ② 受检异常 + 默认 rollbackFor
              ├─ 类没被 Spring 管理
              │   （new 出来的对象没有代理）
              └─ 多线程：子线程抛异常，
                  主线程的拦截器看不到
```

---

## 七、速查表

| 概念 | 一句话 |
|---|---|
| `PlatformTransactionManager` | 事务管理器接口：`getTransaction` / `commit` / `rollback` 三件套 |
| `TransactionTemplate` | 编程式事务模板，把"开启→执行→提交/回滚"样板固化，Boot 自动配好 |
| `executeWithoutResult` | 无返回值回调（lambda 版）；老写法是 `TransactionCallbackWithoutResult` 匿名类 |
| `TransactionStatus` | 当前事务的状态句柄，可 `setRollbackOnly()` 手动标记回滚 |
| `@Transactional` | 声明式事务注解，本质是 AOP 环绕通知 |
| `TransactionInterceptor` | 声明式事务真正干活的 Advice |
| `@EnableTransactionManagement` | 声明式事务开关，Boot 的 `TransactionAutoConfiguration` 自动等效开启 |
| `rollbackFor` | 默认只回滚 `RuntimeException`/`Error`，建议显式写 `Exception.class` |
| `propagation = REQUIRED` | 默认传播行为：有事务加入，没有新建 |
| `REQUIRES_NEW` | 挂起外层，另开独立新事务（内层回滚不连累外层） |
| `timeout` | 单位是**秒** |
| 回滚的触发信号 | 异常**冲出方法边界**且命中回滚规则，缺一不可 |
| 自调用失效 | `this.xxx()` 不走代理——AOP 系注解的通病 |

---

## 八、可以继续挖的方向

1. **七种传播行为跑一遍** — 重点对比 `REQUIRED` vs `REQUIRES_NEW` vs `NESTED`：外层回滚内层怎样、内层回滚外层怎样，用两张表实测。
2. **四种隔离级别** — H2/MySQL 里复现脏读、不可重复读、幻读，体感比背表格深刻。
3. **读 `TransactionInterceptor#invokeWithinTransaction`** — 声明式事务的总入口，和 AOP 笔记的 `@Around` 对照着读。
4. **`setRollbackOnly()` 实验** — 在 catch 块里手动标记回滚，验证 6.1 的解法。
5. **`UnexpectedRollbackException`** — 内层 `REQUIRED` 方法把事务标了 rollback-only，外层却想提交时的经典报错，值得主动复现一次。
6. **事务 + 事件** — `@TransactionalEventListener(phase = AFTER_COMMIT)` 解决"事务还没提交，监听器就去查库查不到"的问题，见《[Spring-事件驱动](Spring-事件驱动.md)》6.4。
