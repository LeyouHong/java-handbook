# MySQL binlog：是什么，怎么用

> 环境：MySQL 8.0 / InnoDB。本篇命令为**示例**，未附实测输出。
>
> **一句话：binlog 是数据库的「操作录像」——每一次数据改动都被记了下来。**
>
> 有了这份录像，你能做四件事：**找回误删的数据、恢复到任意时刻、搭从库、把变更同步给别人。**

---

## 一、它是什么

### 1.1 一个例子

你执行了一条 SQL：

```sql
UPDATE user SET age = 26 WHERE id = 7;
```

MySQL 除了改数据，还会往 binlog 里记一笔：

```
   user 表，id=7 这一行：
       改之前  age = 25
       改之后  age = 26
```

就这么简单。**每一次 INSERT / UPDATE / DELETE，都会留下这样一条记录。**

```
   ┌──────────────────────────────────────────────────────────┐
   │  binlog 里记的是【数据变成了什么】，                       │
   │  不是【数据现在是什么】。                                  │
   │                                                           │
   │  它是一本流水账，不是账户余额。                            │
   └──────────────────────────────────────────────────────────┘
```

### 1.2 所以它能干什么

正因为记的是"每一步的改动"，就有了这些用法：

```
   ① 倒着放 ────▶ 把改动反过来执行 = 【找回误删的数据】
   ② 正着放 ────▶ 从某个备份开始重放 = 【恢复到任意时刻】
   ③ 发给别人 ──▶ 另一台机器照着执行 = 【主从复制】
   ④ 让程序订阅 ▶ 数据一变就通知你 = 【同步到 ES / 删缓存】
```

后面四节就是这四件事的具体做法。

### 1.3 它和 redo log 不是一回事

MySQL 里有好几种日志，最容易混的是这两个：

| | **binlog** | **redo log** |
|---|---|---|
| 记什么 | user 表 id=7 的 age 从 25 变成 26 | 第 128 页偏移 3021 的 4 字节改成 0x1E |
| 谁看得懂 | **任何人** | 只有 InnoDB 自己 |
| 留多久 | 想留多久留多久（默认 30 天） | 写满就覆盖，留不住 |
| 干什么用 | **复制、备份、订阅** | 崩溃后恢复数据 |

```
   redo log 记的是「怎么改的」—— 只对自己那份数据文件有意义，
                                 换台机器就没用了

   binlog  记的是「改成了什么」—— 脱离原始文件依然成立，
                                 所以能发给别人、能重放

   → 本篇讲的所有用法，都只有 binlog 能做。
```

---

## 二、三种格式：直接用 ROW

`binlog_format` 有三个值，但结论很简单——**用 ROW，别的别选**。

```
   STATEMENT   记 SQL 原文       "UPDATE user SET age=age+1 WHERE city='北京'"
   ROW         记每行前后值      "id=7 的 age 从 25 变成 26"（每一行都记）
   MIXED       两者自动切换
```

### 为什么必须是 ROW

STATEMENT 有个致命问题：**同一条 SQL 在两台机器上可能跑出不同结果**。

```sql
DELETE FROM t WHERE status = 0 LIMIT 1;
```

```
   表里有 3 行 status=0
   主库删掉了 id=5，从库重放时可能删掉 id=9
        ↑ "LIMIT 1 删哪一行"取决于扫描顺序，两边不保证一致

   → 主从数据从此对不上，而且【悄无声息】
```

还有一条更现实的理由：

```
   ROW 记了【前后完整数据】
        ↓
   所以才能倒着放（找回误删）、才能被程序订阅（同步到 ES）

   STATEMENT 只给你一句 SQL —— 上面这些一个都做不了。
```

MySQL 5.7.7 起默认就是 ROW，一般不用改。

### 顺带两个要注意的

```
   binlog_row_image = FULL   保持默认。
        设成 MINIMAL 会只记主键和改动的列 → 省空间，
        但【闪回救不回数据、同步到 ES 会缺列】—— 得不偿失

   DDL 永远是 STATEMENT
        ALTER TABLE 这类语句本来就没有"行"可记，
        所以不管什么格式，它都以语句形式记录
```

---

## 三、怎么开、怎么看

### 3.1 检查有没有开

```sql
SHOW VARIABLES LIKE 'log_bin';          -- ON 就是开了（8.0 默认开，5.7 默认关）
SHOW VARIABLES LIKE 'binlog_format';    -- 应该是 ROW
```

### 3.2 配置

```ini
[mysqld]
server_id  = 1                          # 做主从必须唯一，随便给个数
log_bin    = /var/log/mysql/binlog      # 开启，并指定文件名前缀
binlog_format = ROW
binlog_row_image = FULL

binlog_expire_logs_seconds = 2592000    # 保留 30 天
max_binlog_size = 1G                    # 单个文件多大就切下一个
sync_binlog = 1                         # 每次提交都刷盘，断电不丢
```

`sync_binlog` 这一个值得说一句：

```
   sync_binlog = 1   每次提交都写进磁盘  → 【断电不丢】，8.0 默认
   sync_binlog = 0   交给操作系统决定     → 断电可能丢

   设成 1 会不会很慢？不会。
   MySQL 会把同时提交的一批事务【攒起来一次性写盘】（叫组提交），
   所以并发越高，摊到每个事务上的开销越小。

   保持 1。它和 InnoDB 的 innodb_flush_log_at_trx_commit = 1
   合称"双 1"配置，是数据安全的底线。
```

### 3.3 常用命令

```sql
SHOW BINARY LOGS;          -- 有哪些 binlog 文件
SHOW MASTER STATUS;        -- 现在写到哪个文件的哪个位置了
FLUSH BINARY LOGS;         -- 手动切一个新文件（备份前常用）

-- 看某个文件里发生过什么
SHOW BINLOG EVENTS IN 'binlog.000003' LIMIT 20;
```

### 3.4 把 binlog 翻译成人能读的

binlog 是二进制的，直接 `cat` 是乱码。用官方工具解码：

```bash
mysqlbinlog --base64-output=DECODE-ROWS -vv binlog.000003
```

解出来大概长这样（示意）：

```
### UPDATE `shop`.`user`
### WHERE                      ← 改之前
###   @1=7          /* id */
###   @2='张三'
###   @3=25         /* age */
### SET                        ← 改之后
###   @1=7
###   @2='张三'
###   @3=26
```

```
   WHERE 段 = 改之前的值
   SET   段 = 改之后的值

   记住这两段的含义，后面"找回误删数据"就是把它们【对调】。
```

只看某一段时间的：

```bash
mysqlbinlog --start-datetime="2026-08-17 14:00:00" \
            --stop-datetime="2026-08-17 15:00:00" binlog.000003

# 或者按位置（更精确）
mysqlbinlog --start-position=1234 --stop-position=5678 binlog.000003
```

---

## 四、例子一：误删了数据，怎么找回来

**最常用、也最救命的一个用法。**

场景：有人执行了 `DELETE FROM orders WHERE ...`，把不该删的删了。

### 原理：把改动反过来

因为 ROW 格式记了前后完整数据，所以可以反着生成 SQL：

```
   binlog 里记的                      反过来就是
   ─────────────────────────────────────────────────────
   DELETE 了 (id=7, amount=100)  →   INSERT (id=7, amount=100)
   INSERT 了 (id=9, ...)         →   DELETE WHERE id=9
   UPDATE  前=25 后=26           →   UPDATE 前=26 后=25   ← 前后对调
```

把这些反向 SQL **倒着执行一遍**，数据就回来了。

### 操作步骤

```bash
# ① 先找到误操作发生的位置
mysqlbinlog --base64-output=DECODE-ROWS -vv \
            --start-datetime="2026-08-17 14:00:00" binlog.000012 | less
#    翻到那条 DELETE，记下它前后的 position（比如 98765 ~ 99999）

# ② 用工具生成回滚 SQL（-B 就是反向的意思）
python binlog2sql.py -h127.0.0.1 -uroot -p'xxx' -dshop -torders \
       --start-file='binlog.000012' \
       --start-position=98765 --stop-position=99999 \
       -B > rollback.sql

# ③ 检查一遍再执行
less rollback.sql
mysql -uroot -p shop < rollback.sql
```

常用工具：**binlog2sql**（Python，最常用）、**MyFlash**（美团开源）。

```
   ┌──────────────────────────────────────────────────────────┐
   │  两个前提，缺一不可：                                      │
   │                                                           │
   │  ① binlog_format = ROW                                    │
   │  ② binlog_row_image = FULL                                │
   │                                                           │
   │  设成 MINIMAL 的话，UPDATE 的"改之前"只记了主键，          │
   │  其他列的旧值根本没记 —— 救不回来。                        │
   └──────────────────────────────────────────────────────────┘
```

```
   ⚠️ 发现误删后，第一件事是【立刻停止对这张表的写入】。
      后续的新写入会让回滚变得复杂甚至无法进行。
```

---

## 五、例子二：恢复到某个时间点

场景：周三下午有人 `DELETE FROM orders;`（忘了带 WHERE），而你只有周日的全量备份。

```
   周日 02:00  全量备份 ●
                        │
                        ├─────── binlog 一直在记 ───────┐
                        │                               │
   周三 14:23                                    ✗ 事故发生
```

**全量备份 + binlog = 恢复到任意时刻。**

```bash
# ① 用周日的全备恢复
mysql -uroot -p < backup_sunday.sql
#    此时数据是周日 02:00 的状态

# ② 重放 binlog，但要【停在事故那条语句之前】
mysqlbinlog --start-datetime="2026-08-16 02:00:00" \
            --stop-position=98765 \
            binlog.000012 | mysql -uroot -p
```

```
   ┌──────────────────────────────────────────────────────────┐
   │  停止位置优先用 --stop-position，不要用 --stop-datetime。 │
   │                                                           │
   │  时间戳只精确到秒，同一秒内可能有几十个事务 ——             │
   │  用时间停，很可能把那条 DELETE 也重放进去了。              │
   │  position 能精确停在它【前一个字节】。                     │
   │                                                           │
   │  position 怎么找？先用 mysqlbinlog 解出来，肉眼定位。      │
   └──────────────────────────────────────────────────────────┘
```

---

## 六、例子三：搭一个从库

从库的原理就一句话：**主库把 binlog 发过去，从库照着执行一遍。**

```
   ┌─ 主库 ──────┐   binlog   ┌─ 从库 ──────┐
   │  写数据      │ ─────────▶ │  照着执行     │
   └──────────────┘            └──────────────┘
```

```sql
-- ── 主库上 ─────────────────────────────
-- my.cnf 里确保：server_id=1, log_bin=binlog, binlog_format=ROW
CREATE USER 'repl'@'%' IDENTIFIED BY 'xxx';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
```

```bash
# ── 导出主库数据 ───────────────────────
mysqldump --all-databases --single-transaction \
          --source-data=2 > dump.sql
#   --single-transaction 让导出过程不锁表
```

```sql
-- ── 从库上 ─────────────────────────────
-- my.cnf 里：server_id=2（要和主库不同）, read_only=ON

-- 先导入数据，再指向主库
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST     = '10.0.0.1',
  SOURCE_USER     = 'repl',
  SOURCE_PASSWORD = 'xxx',
  SOURCE_AUTO_POSITION = 1;

START REPLICA;

-- 检查有没有跑起来
SHOW REPLICA STATUS\G
--   要确认这三项：
--     Replica_IO_Running:  Yes    ← 有没有在接收
--     Replica_SQL_Running: Yes    ← 有没有在执行
--     Last_Error:          （空）
```

> 主从的延迟怎么监控、切换怎么做、读写分离有什么坑，是另一个话题，
> 见《[主从复制与高可用](主从复制与高可用.md)》。

---

## 七、例子四：数据一变就通知我（CDC）

场景：MySQL 里的商品数据改了，要同步到 Elasticsearch；或者商品改了，要删掉 Redis 缓存。

### 传统做法的问题

```java
@Transactional
public void updateProduct(Product p) {
    productMapper.update(p);
    esClient.index(p);          // 同步到 ES
    redis.del("product:" + p.getId());   // 删缓存
}
```

```
   ✗ ES 挂了怎么办？要不要回滚 MySQL？
   ✗ 删缓存失败了怎么办？缓存里就一直是脏数据，【永远不会自愈】
   ✗ 每加一个下游，就要改一次这段代码
```

### binlog 做法

```
   业务代码【只管写 MySQL】
        ↓
   binlog
        ↓
   Canal（伪装成一个从库，把 binlog 拉过来）
        ↓
   Kafka / RocketMQ
        ↓
   ┌──────────┬──────────┬──────────┐
   ▼          ▼          ▼          ▼
  同步 ES   删缓存    更新报表   通知下游
```

**业务代码一行都不用改**，同步逻辑完全独立出去。

### 怎么搭

Canal 的原理很朴素——**它假装自己是一个从库**，走的是和例子三一模一样的协议。所以准备工作也一样：

```sql
-- 主库上建个账号给 Canal 用
CREATE USER 'canal'@'%' IDENTIFIED BY 'canal';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
```

```
   REPLICATION SLAVE   拉取 binlog 的权限
   REPLICATION CLIENT  查看主库状态的权限
   SELECT              读表结构（要知道列名才能解析出"哪一列变了"）
```

然后启动 Canal，配置好要监听哪些库表、输出到哪个 MQ 就行。

同类工具：**Canal**（阿里，Java 生态最常用）、**Debezium**（Kafka 生态）、**Flink CDC**、**Maxwell**。

### ⚠️ 消费端必须做幂等

```
   Canal 重启后，会从上次记录的位置重新拉 —— 那一段变更【会重放一遍】

   删缓存：重复删没问题        ✓ 天然幂等
   加积分：重复加就出事了       ✗ 必须自己做幂等
```

> 幂等的标准做法：消费记录表 + `UNIQUE (msg_id)`，让数据库直接拒绝重复。
> 见《[乐观锁与悲观锁 · 5.7](../concurrency/乐观锁与悲观锁.md)》。

---

## 八、例子五：查"这条数据是谁改的"

```
   "这个订单的金额昨天是多少？什么时候被改的？"
```

解析 binlog 就能还原每一次变更：

```bash
mysqlbinlog --base64-output=DECODE-ROWS -vv \
            --start-datetime="2026-08-16 00:00:00" \
            binlog.000012 | grep -A 20 "order_no='X20260816001'"
```

**但有个重要局限：**

```
   ┌──────────────────────────────────────────────────────────┐
   │  binlog 里【没有业务身份】——                              │
   │  它不知道是"哪个用户"改的，只有数据库连接账号和线程 id。   │
   │                                                           │
   │  真要做业务审计，还是得业务表自己记 operator_id。          │
   │  binlog 只能当【技术层面的兜底证据】。                     │
   └──────────────────────────────────────────────────────────┘
```

---

## 九、几个坑

```
   ✗ binlog 没开
        MySQL 5.7 默认【关闭】，8.0 才默认开。上线前先确认

   ✗ 格式不是 ROW
        闪回、CDC 全都做不了

   ✗ binlog_row_image 设成了 MINIMAL
        为了省空间，代价是误删救不回、同步到 ES 缺列

   ✗ 保留时间设太短
        从库断开久一点就追不上了；
        CDC 消费慢一点就【直接丢数据】。建议至少 7 天

   ✗ 大事务
        一次 UPDATE 改 100 万行 → binlog 瞬间涨几个 GB
        → 主从延迟飙升。批量操作要拆成小批

   ✗ 消费端没做幂等
        binlog 可能重复投递（见第七节）

   ✗ 位点丢了
        CDC 工具必须持久化消费位置，丢了要么重头跑、要么丢数据

   ✗ 拿 binlog 当业务审计用
        里面没有"谁操作的"（见第八节）
```

---

## 十、速查

### 是什么

```
   binlog = 数据库的操作录像，记录每一次数据改动的【前后值】
   → 倒着放 = 找回误删
   → 正着放 = 恢复到任意时刻
   → 发出去 = 主从复制
   → 被订阅 = 同步 ES / 删缓存
```

### 配置（照抄即可）

```ini
server_id  = 1
log_bin    = /var/log/mysql/binlog
binlog_format = ROW
binlog_row_image = FULL
binlog_expire_logs_seconds = 2592000    # 30 天
sync_binlog = 1
```

### 命令

```sql
SHOW VARIABLES LIKE 'log_bin';       -- 开了没
SHOW BINARY LOGS;                    -- 有哪些文件
SHOW MASTER STATUS;                  -- 当前写到哪
FLUSH BINARY LOGS;                   -- 切新文件
```

```bash
# 看内容
mysqlbinlog --base64-output=DECODE-ROWS -vv binlog.000003

# 按位置截取
mysqlbinlog --start-position=1234 --stop-position=5678 binlog.000003

# 重放到数据库
mysqlbinlog --stop-position=98765 binlog.000012 | mysql -uroot -p
```

### 五个用法

| 想做什么 | 怎么做 | 在哪一节 |
|---|---|---|
| 找回误删的数据 | `binlog2sql -B` 生成反向 SQL | 第四节 |
| 恢复到某个时刻 | 全备 + `mysqlbinlog --stop-position` | 第五节 |
| 搭从库 | `CHANGE REPLICATION SOURCE TO` | 第六节 |
| 同步 ES / 删缓存 | Canal 订阅 → MQ → 消费 | 第七节 |
| 查数据改动历史 | `mysqlbinlog` 解析 | 第八节 |

---

## 相关笔记

- 《[主从复制与高可用](主从复制与高可用.md)》——例子三的展开：延迟怎么监控、主从怎么切换、读写分离的坑
- 《[乐观锁与悲观锁 · 5.7](../concurrency/乐观锁与悲观锁.md)》——消费 binlog 必须幂等，唯一索引是标准解法
- 《[分区表与分库分表 · 第 11 步](分区表与分库分表.md)》——分库分表迁移时，用第七节的 CDC 做增量同步，业务代码零改动
- 《[事务与 MVCC](事务与MVCC.md)》——undo log 是另一份日志，管的是回滚和读快照，和 binlog 完全两回事
