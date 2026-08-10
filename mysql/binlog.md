# MySQL binlog：三种格式，以及它能做的八件事

> 环境：MySQL 8.0 / InnoDB。本篇 SQL 与命令均为**示例**，未附实测输出。
>
> **一句话主线：binlog 把「数据库的状态」变成了「数据变更的流」。**
>
> 一旦变更可以被订阅，MySQL 就不只是一个存数据的地方，而成了整条数据链路的**源头**——
> 缓存、搜索、数仓、新分片库，全都可以挂在这条流上。

前四篇 MySQL 笔记讲的是「一张表怎么设计、怎么查得快、大了怎么拆」，都是**静态结构**。
这一篇换一个轴：**数据是怎么流出去的。**

---

## 一、先分清 binlog 和 redo log

不分清这两个，就理解不了"为什么 binlog 能被外部消费，redo log 不能"。

### 1.1 对照表

| | **redo log** | **binlog** |
|---|---|---|
| 在哪一层 | **InnoDB 引擎层** | **MySQL Server 层** |
| 记什么 | **物理**：某个表空间某页某偏移，改成了什么 | **逻辑**：哪一行数据，变成了什么 |
| 怎么写 | **循环写**，写满回头覆盖 | **追加写**，写满换下一个文件 |
| 能留多久 | 不能，会被覆盖 | 想留多久留多久（默认 30 天） |
| 干什么用 | **崩溃恢复**（crash-safe） | **复制、归档、订阅** |
| 谁能读 | 只有 InnoDB 自己 | **任何人**（从库、Canal、Flink CDC…） |
| 哪些引擎有 | 只有 InnoDB | 所有引擎都有 |

### 1.2 关键区别：「怎么改的」 vs 「改成了什么」

```
   UPDATE user SET age = 30 WHERE id = 7;

   redo log 记的是：
      "表空间 5、第 128 页、偏移 3021 处的 4 个字节，改成 0x0000001E"
      ↑ 只有 InnoDB 看得懂，而且换一台机器、换个版本可能就对不上

   binlog 记的是：
      "user 表，id=7 这一行，age 从 25 变成了 30"
      ↑ 任何人都看得懂，可以重放、可以转成 JSON 发给 Kafka
```

```
   ┌──────────────────────────────────────────────────────────┐
   │  redo log 是【怎么改的】—— 只对自己这份数据文件有意义      │
   │  binlog  是【改成了什么】—— 脱离原始文件依然成立           │
   │                                                           │
   │  所以只有 binlog 能被外部消费。这是整篇笔记的地基。         │
   └──────────────────────────────────────────────────────────┘
```

> 这个"日志即数据流"的思想不是 MySQL 独有的。《[LSM Tree 原理 · 第 5 步](../storage/LSM树原理.md)》里的 WAL 是同一个套路——
> 先把变更顺序写进日志，再慢慢作用到数据结构上。
> 区别在于 WAL 只服务于自己的崩溃恢复（对应 redo log），而 binlog 是**故意做成给别人看的**。

### 1.3 为什么需要两阶段提交

既然有两份日志，就要保证它们说的是同一件事。否则：

```
   只写了 redo，没写 binlog  →  本机数据变了，从库没变     →  主从不一致
   只写了 binlog，没写 redo  →  从库变了，本机重启后没变   →  主从不一致
```

MySQL 的解法是把提交拆成**三步**：

```
   ① InnoDB 写 redo log，标记为 prepare
                ↓
   ② Server 写 binlog                       ← 【提交的真正分界点】
                ↓
   ③ InnoDB 写 redo log，标记为 commit
```

崩溃恢复时的判定规则只有一条：

```
   redo 里有这个事务，但状态是 prepare：
       ├─ binlog 里【有】完整记录  →  提交它（因为从库已经可能收到了）
       └─ binlog 里【没有】       →  回滚它
```

```
   所以「事务是否提交」的唯一权威凭据是 binlog 里有没有它。
   这一条在第五节讲缓存一致性时会再次用到。
```

---

## 二、三种格式

`binlog_format` 有三个取值：**STATEMENT / ROW / MIXED**。

### 2.1 STATEMENT：记 SQL 原文

```sql
UPDATE user SET age = age + 1 WHERE city = '北京';
```

binlog 里存的就是这条 SQL 原文。从库拿到后自己再执行一遍。

```
   ✓ 日志【极小】—— 一条 SQL 影响 100 万行，也只记一行文本
   ✗ 有些 SQL 在主库和从库上跑出来【结果不一样】
```

第二条是致命伤。典型的不安全语句：

```sql
-- ① 不确定的函数
INSERT INTO t VALUES (UUID());        -- ✗ 主从生成的 UUID 必然不同
INSERT INTO t VALUES (SYSDATE());     -- ✗ SYSDATE() 取执行那一刻的时间

-- ② 没有确定排序的 LIMIT
DELETE FROM t WHERE status = 0 LIMIT 1;   -- ✗ 【最经典的例子】
```

最后那条值得展开：

```
   表里有 3 行 status=0，主库按某个顺序删掉了 id=5，
   从库重放时可能删掉 id=9 —— 因为「LIMIT 1 删哪一行」取决于扫描顺序，
   而扫描顺序会受索引选择、数据分布影响，两边不保证一致。

   → 主从数据从此分叉，而且【悄无声息】
```

> ⚠️ 注意 `NOW()` 是**安全**的——binlog 会把执行时刻写进去，从库重放时用 `SET TIMESTAMP` 还原。
> 不安全的是 `SYSDATE()`，它每次调用都取当前真实时间，不受 `SET TIMESTAMP` 控制。
> 这两个函数经常被搞反。

遇到这类语句，MySQL 会在错误日志里记一条 `Unsafe statement written to the binary log`。

### 2.2 ROW：记每一行的前后镜像

```
   Update_rows event:
     table: user
     row 1:  BEFORE (id=7, name='张三', age=25)
             AFTER  (id=7, name='张三', age=26)
     row 2:  BEFORE (id=9, ...)
             AFTER  (id=9, ...)
     ... 共 100 万行
```

```
   ✓ 【绝对准确】—— 记的是结果，不存在重放出不同结果的可能
   ✓ 能拿到【变更前后的完整数据】← 这是第五节所有玩法的前提
   ✗ 日志可能【非常大】
```

大到什么程度：

```sql
UPDATE order SET status = 1;    -- 全表 1000 万行
```

```
   STATEMENT：一行 SQL 文本，几十字节
   ROW：      1000 万条 row event，每条含前后镜像 → 【好几个 GB】
```

这也是"禁止大事务"的一个理由——binlog 会瞬间暴涨，主从延迟随之飙升。

### 2.3 MIXED：默认走 STATEMENT，遇到不安全的自动切 ROW

听起来两全其美，但实际很少用：

```
   ✗ 「什么时候切」是 MySQL 判断的，你无法预期某条语句会以哪种格式落盘
   ✗ 下游 CDC 工具无法处理 STATEMENT 事件（见 2.4）
```

### 2.4 为什么现在几乎都用 ROW

MySQL **5.7.7 起默认就是 ROW**，8.0 依然是。两个原因：

```
   ① 主从一致性是底线，不能拿"日志小"去换
   ② 【整个 CDC 生态只认 ROW】
      Canal / Debezium / Flink CDC / Maxwell 都需要"哪一行变成了什么"，
      而 STATEMENT 只给你一句 SQL —— 工具没法知道它到底影响了哪些行
```

第二条是决定性的。**你想让 binlog 干第五节那八件事里的任何一件，就必须用 ROW。**

### 2.5 `binlog_row_image`：ROW 还能再瘦一点

| 取值 | 记录内容 | 影响 |
|---|---|---|
| `FULL`（默认） | 前后镜像都记**所有列** | 最大，但**闪回和 CDC 都要靠它** |
| `MINIMAL` | 前镜像只记主键，后镜像只记改动的列 | 省空间，但**拿不到完整行** |
| `NOBLOB` | 除 BLOB/TEXT 外都记 | 折中 |

```
   为了省空间设了 MINIMAL，结果 Canal 同步到 ES 的数据缺列、
   误删数据也没法闪回 —— 这是个常见的事后追悔。

   除非 binlog 体积真的顶不住，否则保持 FULL。
```

### 2.6 一个例外：DDL 永远是 STATEMENT

```sql
ALTER TABLE user ADD COLUMN nickname VARCHAR(32);
```

**即使 `binlog_format = ROW`，DDL 也是以语句形式记录的。**想想也合理——建表改表本来就没有"行"可记。

带来的实际影响：

```
   → CDC 工具必须单独处理 DDL 事件（Canal/Debezium 都有专门的 schema 变更处理）
   → 从库上执行 DDL 是【串行阻塞】的，大表 ALTER 会造成严重主从延迟
```

---

## 三、怎么看 binlog（实操）

```sql
-- ① 开了吗？（MySQL 8.0 默认开启，5.7 默认关闭）
SHOW VARIABLES LIKE 'log_bin';

-- ② 什么格式？
SHOW VARIABLES LIKE 'binlog_format';
SHOW VARIABLES LIKE 'binlog_row_image';

-- ③ 有哪些文件？
SHOW BINARY LOGS;

-- ④ 当前写到哪了？（做主从、做 CDC 都要记这个位点）
SHOW MASTER STATUS;

-- ⑤ 看某个文件里的事件
SHOW BINLOG EVENTS IN 'binlog.000003' FROM 4 LIMIT 20;

-- ⑥ 手动切一个新文件（备份前常用）
FLUSH BINARY LOGS;
```

命令行解析（ROW 格式是二进制的，必须解码才能看）：

```bash
# -vv 把 row event 还原成可读的伪 SQL
mysqlbinlog --base64-output=DECODE-ROWS -vv binlog.000003

# 按时间/位点截取
mysqlbinlog --start-datetime="2026-08-09 14:00:00" \
            --stop-datetime="2026-08-09 15:00:00"  binlog.000003

mysqlbinlog --start-position=1234 --stop-position=5678  binlog.000003
```

解出来大致长这样（示意）：

```
### UPDATE `shop`.`user`
### WHERE
###   @1=7          /* id */
###   @2='张三'      /* name */
###   @3=25         /* age */
### SET
###   @1=7
###   @2='张三'
###   @3=26
```

**`WHERE` 段是前镜像，`SET` 段是后镜像**——第五节的闪回就是把这两段对调。

---

## 四、关键参数

```ini
[mysqld]
server_id  = 1                          # 做复制必须唯一
log_bin    = /var/log/mysql/binlog      # 开启并指定前缀
binlog_format = ROW
binlog_row_image = FULL

binlog_expire_logs_seconds = 2592000    # 保留 30 天（8.0 用这个，
                                        # expire_logs_days 已废弃）
max_binlog_size = 1G
sync_binlog = 1
```

### `sync_binlog`：安全和性能的那个旋钮

```
   sync_binlog = 0    交给操作系统决定什么时候刷盘 —— 【断电会丢】
   sync_binlog = N    每 N 个提交组刷一次盘        —— 断电丢最近 N 组
   sync_binlog = 1    每个提交组都 fsync           —— 【最安全】，8.0 默认
```

`= 1` 意味着每次提交都要等一次磁盘往返，而 fsync 是**毫秒级**的硬件动作。按理说 TPS 会被砸到几百。

它没有被砸死，靠的是 **binlog 组提交（group commit）**：

```
   并发提交的事务不各自 fsync，而是攒成一批，
   推举一个 leader 代表整批写一次盘：

   ┌──────────────────────────────────────────┐
   │  事务 A   事务 B   事务 C   …  同时到达     │
   └────────────────┬─────────────────────────┘
                    │ 合并
                    ▼
              【一次 fsync】服务整批
```

```
   没有组提交：100 个并发提交 = 100 次 fsync
   有组提交：  100 个并发提交 = 1 次 fsync

   fsync 的耗时和数据量几乎无关，所以并发越高，摊薄效果越好。
   ——「既要断电不丢，又要高吞吐」只有这一条路。
```

两个可调项（用延迟换更大的批）：

```ini
binlog_group_commit_sync_delay = 0            # 微秒；故意等一会儿凑更多事务
binlog_group_commit_sync_no_delay_count = 0   # 凑够这么多就不等了
```

写多的库把 `sync_delay` 设成几百微秒，往往能显著降低 fsync 次数——**用一点点延迟换吞吐**。

---

## 五、能用它做什么

八个用途，分成两个家族：

```
   ┌─ 家族一：MySQL 自己用 ────────────────────────┐
   │   ① 主从复制                                  │
   │   ② 时间点恢复（PITR）                        │
   │   ③ 误操作闪回                                │
   └───────────────────────────────────────────────┘

   ┌─ 家族二：外部订阅 = CDC ──────────────────────┐
   │   ④ 同步到异构存储（ES / 数仓 / HBase）        │
   │   ⑤ 缓存一致性              ★ 最实用           │
   │   ⑥ 分库分表迁移            ★ 接上一篇         │
   │   ⑦ 业务解耦（数据变更驱动事件）               │
   │   ⑧ 审计与数据追溯                            │
   └───────────────────────────────────────────────┘

   家族一是 binlog 的本职，家族二是它的红利 ——
   而红利的前提只有一个：binlog_format = ROW。
```

### 5.1 主从复制（本职工作）

```
   ┌─ 主库 ────────┐                    ┌─ 从库 ──────────────┐
   │  写 binlog     │ ──── dump ────▶   │  IO 线程 → relay log │
   │                │                    │  SQL 线程 → 重放      │
   └────────────────┘                    └──────────────────────┘
```

主从复制是读写分离、高可用、异地容灾的基础。**它是 binlog 存在的原始理由**，其余七件事都是顺带的。

### 5.2 时间点恢复（PITR）

单靠备份只能恢复到备份那一刻。**全量备份 + binlog 增量 = 恢复到任意时间点。**

```
   周日 02:00 全量备份
        │
        ├──────── binlog 持续记录 ────────┐
        │                                  │
   周三 14:23  某人执行了 DELETE FROM order;（没带 WHERE）
```

恢复步骤：

```bash
# ① 用全备恢复到周日 02:00 的状态
mysql -uroot -p < backup_sunday.sql

# ② 重放 binlog，但【停在事故语句之前】
mysqlbinlog --start-datetime="2026-08-05 02:00:00" \
            --stop-position=98765 \
            binlog.000012 | mysql -uroot -p
```

```
   实操要点：停止边界优先用 --stop-position 而不是 --stop-datetime。
   时间戳的粒度是秒，同一秒内可能有几十个事务；
   而 position 能精确停在那条 DELETE 的【前一个字节】。

   position 从哪来？先 mysqlbinlog 解出来人肉定位那条语句。
```

### 5.3 误操作闪回：ROW 格式的杀手级红利

ROW 格式记了前后镜像，那就可以**反着来**：

```
   binlog 里的                 反向生成的
   ─────────────────────────────────────────────
   DELETE  (id=7, age=25)  →  INSERT (id=7, age=25)
   INSERT  (id=9, ...)     →  DELETE WHERE id=9
   UPDATE  前(25) 后(26)   →  UPDATE 前(26) 后(25)   ← 前后镜像对调
```

把这些反向 SQL 按**倒序**执行，数据就回来了。工具：

```
   binlog2sql   (Python，最常用)
   MyFlash      (美团开源)
```

```bash
# 生成回滚 SQL
python binlog2sql.py -h127.0.0.1 -uroot -p'xxx' -dshop -torder \
       --start-file='binlog.000012' \
       --start-position=98765 --stop-position=99999 \
       -B > rollback.sql
```

```
   ┌────────────────────────────────────────────────────────┐
   │  这是 ROW 相比 STATEMENT 最实在的一个好处：             │
   │  误删了能救回来，而且是【精确到行】的救。                │
   │                                                         │
   │  前提：binlog_row_image = FULL                          │
   │        MINIMAL 只记了主键，UPDATE 的前镜像不全，救不回。 │
   └────────────────────────────────────────────────────────┘
```

### 5.4 同步到异构存储（CDC 的典型用法）

```
   MySQL ──binlog──▶ Canal ──▶ Kafka ──┬──▶ Elasticsearch  （全文搜索）
                                        ├──▶ 数仓 / Hive     （离线分析）
                                        ├──▶ Redis           （缓存）
                                        └──▶ HBase           （宽表）
```

关键在于**业务代码完全不用改**：

```
   ✗ 传统做法：业务代码里写完 MySQL，再写一遍 ES
                → 两处写，容易漏；ES 挂了要不要回滚 MySQL？

   ✓ binlog 做法：业务只管写 MySQL
                → 同步逻辑收敛到一条链路，业务和同步彻底解耦
```

### 5.5 缓存一致性 ★

这是最实用的一个用途。先看老问题：

```java
@Transactional
public void updateUser(User u) {
    userMapper.update(u);
    redis.del("user:" + u.getId());   // 删缓存
}
```

```
   ✗ 删缓存失败了怎么办？→ 缓存里是脏数据，而且【永远不会自愈】
   ✗ 事务还没提交就删了缓存 → 别人读到旧值又写回缓存
   ✗ 事务最后回滚了 → 缓存白删（这个不严重）但说明时机不对
```

于是有了"延迟双删"这类补丁——**靠猜一个 sleep 时长**，本质上不可靠。

binlog 方案：

```
   业务代码只管写数据库（一件事都不用多做）
        ↓
   binlog → Canal → MQ → 消费者删缓存
```

它的可靠性来自第 1.3 节那个结论：

```
   ┌──────────────────────────────────────────────────────────┐
   │  binlog 里有这条记录 ⟺ 事务【真的提交了】                 │
   │                                                           │
   │  所以绝不会出现"缓存删了但数据库回滚了"。                  │
   │  而业务代码里的 redis.del() 给不了这个保证 ——              │
   │  它执行的时候，事务还没提交。                              │
   └──────────────────────────────────────────────────────────┘
```

再加上 MQ 的重试，**删缓存这件事变成了最终一定会成功**：

```
   删失败 → MQ 重投 → 再删 → …… 直到成功
   （删除操作天然幂等，重复删没有副作用）
```

### 5.6 分库分表迁移 ★

《[分区表与分库分表 · 第 11 步](分区表与分库分表.md)》给的迁移流程是：

```
   增量同步 → 迁存量 → 等追平 → 对账 → 灰度切读 → 切写
```

第一步"增量同步"，指的就是 binlog。**它替掉的是最朴素的那个做法——应用双写**：

```
   ✗ 应用双写
       业务代码要同时写老库和新分片库
       → 侵入业务；两边失败了怎么办？要不要分布式事务？

   ✓ binlog 增量同步
       业务只写老库，Canal 订阅 binlog 实时灌进新库
       → 业务代码【一行不改】
```

完整流程：

```
   ① 开启 binlog 订阅，把增量灌到新分片库（先跑起来，允许落后）
   ② 用全量工具迁移存量数据
   ③ 等增量追平（监控延迟趋近 0）
   ④ 对账：全量比对 + 抽样比对
   ⑤ 灰度切读：1% → 10% → 100%
   ⑥ 切写，老库保留一段时间以便回滚
```

```
   这里也回答了上一篇没展开的一个问题：
   「存量迁移期间新写入的数据怎么办？」—— 交给 binlog 增量流。
```

### 5.7 业务解耦：数据变更驱动事件

```
   订单表 status 从 1（已支付）变成 2（已发货）
        ↓ binlog
   ┌────┬────┬────┐
   ▼    ▼    ▼    ▼
  发短信 加积分 更新报表 通知物流
```

```
   ✗ 都写在订单服务里 → 订单服务越来越胖，加一个下游就要改一次代码
   ✓ 挂在 binlog 流上 → 订单服务不知道下游是谁，加下游不用动它
```

> 这和《[Spring 事件驱动](../spring/Spring-事件驱动.md)》是同一个思想在不同层次的表达：
> Spring 事件解耦的是**同一个 JVM 内的模块**，binlog 解耦的是**不同的服务**。

### 5.8 审计与数据追溯

```
   "这条记录的金额昨天是多少？谁在什么时候改的？"
   → 解析 binlog 就能还原每一次变更的时间点和前后值
```

局限要说清楚：

```
   binlog 里【没有业务身份】——不知道是哪个用户操作的。
   它有的是 thread_id 和数据库连接账号。

   真要做业务审计，还是得业务表自己记 operator_id，
   binlog 只能作为「技术层面的兜底证据」。
```

---

## 六、Canal 的原理：伪装成一个从库

CDC 工具为什么能拿到 binlog？答案很朴素——**它假装自己是从库**。

```
   ┌─ MySQL 主库 ─┐                    ┌─ Canal ──────────────┐
   │              │ ◀── 我是 slave ─── │  发 COM_REGISTER_SLAVE │
   │              │ ◀── dump 请求 ──── │  发 COM_BINLOG_DUMP    │
   │              │ ─── binlog 流 ───▶ │  解析成结构化数据       │
   └──────────────┘                    └────────────────────────┘
                                              │
                                              ▼
                                        Kafka / RocketMQ
```

MySQL 完全不知道对面是不是真的从库——它只认协议。所以准备工作和加一个从库一样：

```sql
-- ① 确认配置
--    log_bin = ON, binlog_format = ROW, server_id 唯一

-- ② 建账号并授权
CREATE USER 'canal'@'%' IDENTIFIED BY 'canal';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
FLUSH PRIVILEGES;
```

```
   REPLICATION SLAVE   —— 拉取 binlog 流的权限
   REPLICATION CLIENT  —— 执行 SHOW MASTER STATUS 的权限
   SELECT              —— 读表结构（解析 row event 需要知道列名）
```

同类工具：

| 工具 | 特点 |
|---|---|
| **Canal** | 阿里开源，Java 生态最常用，社区中文资料多 |
| **Debezium** | Kafka Connect 生态，支持多种数据库 |
| **Flink CDC** | 基于 Debezium，直接接入 Flink 做流处理 |
| **Maxwell** | 轻量，直接输出 JSON |

---

## 七、坑

```
   ✗ binlog 没开 ──────────────── MySQL 5.7 默认【关闭】，
                                   8.0 才默认开。上线前先确认
   ✗ 格式是 STATEMENT/MIXED ───── CDC 工具直接歇菜（2.4）
   ✗ binlog_row_image = MINIMAL ─ 同步缺列、闪回救不回来（2.5）
   ✗ 大事务 ──────────────────── ROW 下 binlog 暴涨到 GB 级，
                                   主从延迟飙升（2.2）
   ✗ 保留时间设太短 ──────────── 从库断开久了无法追上，
                                   CDC 消费慢就【直接丢数据】
   ✗ 消费端不做幂等 ──────────── binlog 可能【重复投递】，
                                   消费者必须能重复处理同一条变更
   ✗ 用 binlog 做业务审计 ─────── 里面没有"谁操作的"（5.8）
   ✗ sync_binlog = 0 ─────────── 断电丢 binlog，
                                   主从从此不一致（第四节）
   ✗ 忘了 DDL 是 STATEMENT ───── 下游 schema 变更要单独处理（2.6）
   ✗ 位点丢了 ────────────────── CDC 必须持久化消费位点；
                                   丢了要么重头跑（数据洪水）要么丢数据
```

关于**幂等**再多说一句，它是消费 binlog 时最容易翻车的地方：

```
   Canal 崩溃重启 → 从上次持久化的位点重新拉 → 那一段变更【会重放一遍】

   删缓存：重复删没问题        ✓ 天然幂等
   加积分：重复加就出事了       ✗ 必须自己做幂等
```

> 幂等的标准做法见《[乐观锁与悲观锁 · 5.7](../concurrency/乐观锁与悲观锁.md)》——
> 用一张消费记录表 + `UNIQUE (msg_id)`，让数据库直接拒绝重复。

---

## 八、速查

### 三种格式

```
   ┌──────────────────────────────────────────────────────────┐
   │  STATEMENT  记 SQL 原文                                   │
   │             小；但 SYSDATE()/UUID()/LIMIT 会主从不一致     │
   │                                                           │
   │  ROW        记每行前后镜像            ← 默认，用这个       │
   │             准确、能被 CDC 消费、能闪回；但可能很大        │
   │                                                           │
   │  MIXED      自动切换                                      │
   │             看着两全，实际下游没法处理，很少用             │
   │                                                           │
   │  例外：DDL 在任何格式下都是 STATEMENT                      │
   └──────────────────────────────────────────────────────────┘
```

### 常用命令

```sql
SHOW VARIABLES LIKE 'log_bin';          -- 开了没
SHOW VARIABLES LIKE 'binlog_format';    -- 什么格式
SHOW BINARY LOGS;                       -- 有哪些文件
SHOW MASTER STATUS;                     -- 当前位点
FLUSH BINARY LOGS;                      -- 切新文件
```

```bash
mysqlbinlog --base64-output=DECODE-ROWS -vv binlog.000003
mysqlbinlog --start-position=1234 --stop-position=5678 binlog.000003 | mysql -uroot -p
```

### 八个用途一句话

```
   MySQL 自己用 ─┬─ 主从复制      本职工作，读写分离/高可用的基础
                 ├─ PITR          全备 + binlog = 恢复到任意时间点
                 └─ 闪回          前后镜像对调，误删能救回来

   外部订阅      ─┬─ 异构同步      ES / 数仓 / HBase，业务代码不用改
   （= CDC）      ├─ 缓存一致性    binlog 是"事务已提交"的凭据 ★
                 ├─ 分库分表迁移  替掉双写，业务零侵入 ★
                 ├─ 业务解耦      数据变更驱动下游，加下游不改上游
                 └─ 审计追溯      能还原变更，但没有业务身份
```

### 上线前 Checklist

```
   [ ] log_bin = ON
   [ ] binlog_format = ROW
   [ ] binlog_row_image = FULL
   [ ] server_id 全局唯一
   [ ] sync_binlog = 1
   [ ] binlog 保留时长 > 最长可容忍的消费延迟（建议 ≥ 7 天）
   [ ] 磁盘空间够吗？（ROW 格式很能吃）
   [ ] CDC 消费位点持久化了吗？
   [ ] 消费端幂等做了吗？
   [ ] DDL 事件下游怎么处理？
```

---

## 相关笔记

- 《[分区表与分库分表](分区表与分库分表.md)》——迁移流程里的"双写"可以用 binlog 增量同步替掉（5.6）
- 《[乐观锁与悲观锁 · 5.7](../concurrency/乐观锁与悲观锁.md)》——消费 binlog 必须幂等，唯一索引是标准解法
- 《[LSM Tree 原理 · 第 5 步](../storage/LSM树原理.md)》——WAL 和 binlog 是同一个"日志即数据流"的思想，一个对内一个对外
- 《[Spring 事件驱动](../spring/Spring-事件驱动.md)》——同样是事件解耦，Spring 在 JVM 内，binlog 在服务间
