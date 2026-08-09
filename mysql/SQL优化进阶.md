# SQL 优化进阶：JOIN、优化器、排序与临时表

> 环境：MySQL 8.0 / InnoDB。本篇 SQL 为**示例**，未附实测输出。
>
> **一句话主线：索引建对了还慢，问题多半在这四件事上——JOIN 的驱动顺序、优化器选错了索引、排序落到了磁盘、以及临时表。**

《[索引优化](索引优化.md)》解决的是"这条 SQL 用没用上索引"。本篇往下一层：**用上了索引，为什么还是慢。**

前置：《[B+ 树原理](B+树原理.md)》的回表与树高、《[索引优化](索引优化.md)》的 `EXPLAIN` 字段。

---

## 一、先建立工作流：别凭感觉优化

```
   ① 找出慢的      慢查询日志 → pt-query-digest 聚合
        ↓
   ② 看它怎么执行的  EXPLAIN → EXPLAIN ANALYZE（8.0.18+，看【真实】耗时）
        ↓
   ③ 找出慢在哪     optimizer trace / profiling
        ↓
   ④ 改             加索引 / 改写 SQL / 调参数
        ↓
   ⑤ 验证           再跑一遍 ②，用【真实数据量】验证
```

### 1.1 打开慢查询日志

```ini
slow_query_log      = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time     = 1                    # 秒，超过就记
log_queries_not_using_indexes = ON         # 没用索引的也记
log_slow_admin_statements = ON             # ALTER 之类也记
min_examined_row_limit = 100               # 扫描行数太少的不记（降噪）
```

```
   ⚠️ log_queries_not_using_indexes 在小表多的库上会刷屏
      —— 小表全表扫描本来就是对的（⤵《[索引优化 · 第七章](索引优化.md)》："这不是索引建错了，是数据量不够"）。
      配合 min_examined_row_limit 一起用。
```

### 1.2 聚合分析

```bash
# 官方自带，简单
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# Percona Toolkit，强得多：会把参数不同但结构相同的 SQL 归成一类
pt-query-digest /var/log/mysql/slow.log
```

```
   ┌──────────────────────────────────────────────────────────┐
   │  优先级不是看"最慢的那条"，而是看【总耗时占比】。          │
   │                                                           │
   │  一条 10 秒的报表 SQL 每天跑 1 次   = 10 秒                │
   │  一条 50ms 的查询每天跑 100 万次    = 13.9 小时  ← 先优化它 │
   │                                                           │
   │  pt-query-digest 默认就是按总耗时排序的，用它。            │
   └──────────────────────────────────────────────────────────┘
```

### 1.3 `EXPLAIN ANALYZE`：看真实执行

`EXPLAIN` 给的是**优化器的预估**，可能和现实差很远。8.0.18 起有：

```sql
EXPLAIN ANALYZE SELECT ... ;
```

```
   它会【真的执行】这条 SQL，然后给出每一步的：
       actual time=首行耗时..全部耗时   rows=实际行数   loops=执行次数

   对比 EXPLAIN 的预估 rows 和这里的实际 rows —— 差得越远，
   说明优化器的统计信息越不准（第三节）。
```

---

## 二、JOIN：驱动顺序决定一切

### 2.1 两种算法（8.0 现状）

```
   ┌─ Index Nested-Loop Join（INLJ）── 被驱动表【走索引】 ────┐
   │                                                           │
   │   for 驱动表的每一行:                                      │
   │       用关联字段去被驱动表的【索引】里查                    │
   │                                                           │
   │   复杂度 ≈ 驱动表行数 × log(被驱动表)                      │
   │   ★ 这是我们想要的                                         │
   └───────────────────────────────────────────────────────────┘

   ┌─ Hash Join ── 被驱动表【没有可用索引】───────────────────┐
   │                                                           │
   │   ① 把小表读进内存建哈希表                                 │
   │   ② 扫大表，逐行去哈希表里探测                             │
   │                                                           │
   │   复杂度 ≈ 两表行数之和 —— 【线性】，比想象中好很多         │
   └───────────────────────────────────────────────────────────┘
```

```
   版本说明：
   · 8.0.18 引入 Hash Join，用于等值 JOIN 且无可用索引的情况
   · 8.0.20 起 Hash Join 【全面取代】了老的 Block Nested-Loop Join（BNL）
   → 所以在 8.0.20+ 上，"没索引的 JOIN"已经不像以前那么灾难了，
     但【有索引仍然显著更好】，尤其是驱动表小、被驱动表大的时候
```

看 `EXPLAIN` 的 `Extra`：

```
   Using index                    ← 被驱动表走了索引，INLJ  ✓
   Using join buffer (hash join)  ← 走了 Hash Join，说明没索引可用
```

### 2.2 小表驱动大表

```sql
SELECT * FROM a JOIN b ON a.bid = b.id WHERE a.x = 1;
```

```
   a 过滤后 100 行，b 有 100 万行，b.id 上有索引：

   ✓ a 驱动 b：  100 次索引查找                    → 快
   ✗ b 驱动 a：  100 万次，每次去 a 里找           → 慢 1 万倍
```

```
   ┌──────────────────────────────────────────────────────────┐
   │  "小表"指的是【过滤之后】参与 JOIN 的行数，                │
   │  不是表本身的总行数。                                      │
   │                                                           │
   │  一张 1 亿行的表，WHERE 之后只剩 10 行，它就是小表。       │
   └──────────────────────────────────────────────────────────┘
```

优化器通常会自己选对。选错时可以干预：

```sql
-- 强制按你写的顺序 JOIN（左边的做驱动表）
SELECT * FROM a STRAIGHT_JOIN b ON a.bid = b.id;
```

```
   ⚠️ STRAIGHT_JOIN 是把优化器的权力收回来，
      数据分布一变它就可能反而更差。
      只在【确认优化器选错且短期改不了】时用，并留注释说明原因。
```

### 2.3 JOIN 优化清单

```
   [ ] 被驱动表的关联字段【必须有索引】—— 这一条顶其他所有
   [ ] 两边关联字段的类型和字符集要【完全一致】
       （不一致会隐式转换 → 索引失效，⤵《[索引优化 · 第五章](索引优化.md)》的 `key=NULL` 排查项）
   [ ] 尽量减少 JOIN 的表数量（3 张以内）
   [ ] 让 WHERE 尽早过滤，缩小驱动表
   [ ] 大表 JOIN 考虑冗余字段避免 JOIN（⤵《数据表设计》经验④）
   [ ] 分库分表后跨库 JOIN 做不了，只能应用层拼
```

---

## 三、优化器为什么选错索引

现象：明明有更好的索引，`EXPLAIN` 显示走了另一个，或者干脆全表扫描。

### 3.1 根因：优化器是按「成本」选的，而成本靠「统计信息」估

```
   优化器不知道真实数据，它靠【采样统计】出来的两个数字判断：
       · 索引的区分度（cardinality，不同值有多少个）
       · 预估要扫描的行数（rows）

   采样不准 → 成本算错 → 选错索引
```

看统计信息：

```sql
SHOW INDEX FROM t;     -- 关注 Cardinality 列
```

```
   Cardinality 明显偏离真实值（比如 status 只有 3 种值却显示 10 万）
   → 统计信息过期了
```

### 3.2 修法一：重新统计

```sql
ANALYZE TABLE t;       -- 重新采样，很轻量，可以放心执行
```

```
   多数"突然选错索引"的问题，ANALYZE 一下就好了。先试这个。
```

采样精度可调：

```ini
innodb_stats_persistent = ON              # 统计信息持久化（默认）
innodb_stats_persistent_sample_pages = 20 # 采样页数，默认 20
                                          # 大表且分布不均时可调到 64~128
```

### 3.3 修法二：改写 SQL

```sql
-- 索引 (a, b)，但 ORDER BY 让优化器放弃了它
SELECT * FROM t WHERE a = 1 ORDER BY c LIMIT 10;

-- 改成先用索引定位主键，再回表 —— 延迟关联
SELECT t.* FROM t
  JOIN (SELECT id FROM t WHERE a = 1 ORDER BY c LIMIT 10) x
    ON t.id = x.id;
```

### 3.4 修法三：强制索引（最后手段）

```sql
SELECT * FROM t FORCE INDEX (idx_a_b) WHERE ...;
SELECT * FROM t IGNORE INDEX (idx_c) WHERE ...;
```

```
   ✗ 索引名写死在 SQL 里 —— 以后改索引名/删索引，这条 SQL 直接报错
   ✗ 数据分布变化后可能反而更慢

   → 当成止血手段，同时排查根因（多半是统计信息或索引设计问题）
```

### 3.5 排查工具：optimizer trace

想知道优化器**为什么**这么选：

```sql
SET optimizer_trace = "enabled=on";
SELECT ... ;                                    -- 跑一遍你的 SQL
SELECT * FROM information_schema.OPTIMIZER_TRACE\G
SET optimizer_trace = "enabled=off";
```

```
   输出是一大段 JSON，重点看：
       rows_estimation        →  每个候选索引预估扫多少行
       considered_execution_plans → 各方案的 cost 分别是多少
       chosen: true           →  最终选了谁

   把预估 rows 和真实 rows 一对比，就知道是不是统计信息的锅。
```

---

## 四、排序：`filesort` 到底在做什么

`EXPLAIN` 的 `Extra` 里出现 `Using filesort`，意思是**没法靠索引拿到有序结果，得自己排**。

```
   注意："filesort" 不一定用到磁盘 ——
   能在 sort_buffer 里排完就是内存排序，排不完才落磁盘（外部归并）。
   真正要避免的是【落磁盘】那种。
```

### 4.1 两种排序算法

```
   ┌─ 全字段排序 ────────────────────────────────────────────┐
   │  把 SELECT 需要的【所有列】都放进 sort_buffer 一起排      │
   │  ✓ 排完直接返回，不用回表                                 │
   │  ✗ 单行占空间大 → sort_buffer 装不下 → 落磁盘             │
   └──────────────────────────────────────────────────────────┘

   ┌─ rowid 排序 ────────────────────────────────────────────┐
   │  只放【排序字段 + 主键】进 sort_buffer                    │
   │  ✓ 单行小，能排更多行，不容易落磁盘                        │
   │  ✗ 排完还要拿主键【回表】取其他列 → 多一轮随机 IO          │
   └──────────────────────────────────────────────────────────┘
```

MySQL 自己按"单行长度"决定用哪种：

```ini
max_length_for_sort_data = 4096   # 单行超过这个长度就用 rowid 排序
                                  # ⚠️ 8.0.20 起已废弃，
                                  #    优化器改为自动判断，不用再调
sort_buffer_size = 256K           # 每个【连接】独占，别设太大（第七节）
```

### 4.2 根治：让索引本身有序

```sql
SELECT * FROM t WHERE a = 1 ORDER BY b;
```

```
   索引 (a)       →  找到 a=1 的行，再排 b        → Using filesort
   索引 (a, b)    →  a=1 的行在索引里【本来就按 b 有序】→ 直接顺序读，免排序 ✓
```

> ⤵ **依据**：《[B+ 树原理 · 第八章](B+树原理.md)》——联合索引的叶子层是按元组顺序排好的，
> 所以"等值列在前、排序列在后"能天然免掉 `filesort`。这是列序三步法的核心理由之一。

### 4.3 `ORDER BY` + `LIMIT` 的陷阱

```sql
SELECT * FROM t WHERE status = 1 ORDER BY id LIMIT 10;
```

```
   优化器可能这样想：
     "反正按 id 排序，那我沿着【主键索引】顺序扫，
      碰到 10 个 status=1 的就停" → 看起来很聪明

   但如果 status=1 的数据都在表的最后面：
     → 要扫完几乎整张表才凑够 10 条 → 【极慢】
     而走 status 索引再排序反而快得多
```

```
   识别信号：EXPLAIN 显示走了 PRIMARY 而不是你建的那个索引，
            且 rows 预估很小但实际执行很慢。
   解法：延迟关联（3.3）或 FORCE INDEX。
```

---

## 五、临时表

`Extra` 里的 `Using temporary` 意味着 MySQL 建了一张内部临时表来中转。

### 5.1 什么时候会产生

```
   GROUP BY 的字段没有索引，或和 ORDER BY 不一致
   DISTINCT + ORDER BY
   UNION（会去重，所以要建临时表；UNION ALL 不会）
   派生表 / 子查询物化
   ORDER BY 和 GROUP BY 用了不同的列
```

### 5.2 内存临时表 vs 磁盘临时表

```
   先建在内存里（8.0 默认用 TempTable 引擎）
        ↓ 超过阈值
   转成磁盘临时表（InnoDB）→ 【性能断崖】
```

```ini
tmp_table_size  = 16M     # 单个内存临时表上限
max_heap_table_size = 16M # 两者取较小值生效
temptable_max_ram = 1G    # TempTable 引擎的【全局】内存上限
```

怎么知道有没有落磁盘：

```sql
SHOW GLOBAL STATUS LIKE 'Created_tmp%';
--  Created_tmp_tables            创建了多少临时表
--  Created_tmp_disk_tables       其中多少落到了磁盘  ← 这个占比要低
```

### 5.3 消除临时表

```sql
-- ✗ GROUP BY 无索引 → 临时表 + filesort
SELECT city, COUNT(*) FROM t GROUP BY city;

-- ✓ 给 city 建索引 → 索引本身有序，边扫边聚合，免临时表
ALTER TABLE t ADD INDEX idx_city (city);

-- ✓ 确实不需要排序时，显式关掉（MySQL 8.0 起 GROUP BY 默认不再排序，
--   但老代码里可能还有依赖）
SELECT city, COUNT(*) FROM t GROUP BY city ORDER BY NULL;

-- ✓ 能用 UNION ALL 就别用 UNION
SELECT ... UNION ALL SELECT ...;
```

---

## 六、几个高频问题

### 6.1 `COUNT(*)` 为什么慢，怎么写最快

InnoDB **不维护表的总行数**——因为 MVCC 下"有多少行"取决于你的 ReadView，不同事务看到的行数可能不同。

> ⤵ **依据**：《[事务与 MVCC · 第五节](事务与MVCC.md)》——可见性是逐行判定的，没法预先存一个总数。

所以 `COUNT(*)` 只能真的去数。写法对比：

```
   COUNT(*)      ✓ 【最快】。优化器会挑一棵【最小的索引树】去扫，
                    通常是某个二级索引，比主键树小得多
   COUNT(1)      ≈ 和 COUNT(*) 基本一样，没有传说中的差别
   COUNT(主键)    ≈ 差不多
   COUNT(字段)    ✗ 【最慢】。要逐行取出该字段判断是否为 NULL，
                    而且【不统计 NULL 行】—— 语义都不一样
```

```
   结论：无脑用 COUNT(*)。
        网上"COUNT(1) 比 COUNT(*) 快"的说法在现代 MySQL 上是错的。
```

真的要快，只能不数：

```
   · 单独一张计数表，用事务或 binlog 维护（⤵《binlog · 5.7》）
   · Redis 计数器（要处理一致性）
   · 能接受估算就用 EXPLAIN 的 rows，或 information_schema.TABLES.TABLE_ROWS
     （注意：这是【估算值】，误差可能很大）
```

### 6.2 深分页

```sql
SELECT * FROM t ORDER BY id LIMIT 1000000, 10;   -- 要丢弃 100 万行
```

> ⤵ 完整解释见《[B+ 树原理 · 5.4](B+树原理.md)》。

三种解法：

```sql
-- ① 游标分页（最好，但不能跳页）
SELECT * FROM t WHERE id > 上一页最后一个id ORDER BY id LIMIT 10;

-- ② 延迟关联（能跳页，先在索引上分页再回表）
SELECT t.* FROM t
  JOIN (SELECT id FROM t ORDER BY id LIMIT 1000000, 10) x USING(id);

-- ③ 产品上限制页数（很多场景最实际的方案）
```

### 6.3 子查询

```sql
-- ✗ IN + 子查询在老版本会退化成逐行执行
SELECT * FROM a WHERE id IN (SELECT aid FROM b WHERE x = 1);

-- ✓ 改写成 JOIN，语义等价且稳定
SELECT DISTINCT a.* FROM a JOIN b ON a.id = b.aid WHERE b.x = 1;
```

```
   MySQL 8.0 的优化器对半连接（semi-join）优化好了很多，
   IN 子查询大多能自动转成 JOIN。
   但【EXISTS / NOT EXISTS / NOT IN】仍然要小心，
   尤其 NOT IN 遇到 NULL 会返回空集 —— 这是语义坑不是性能坑。
```

### 6.4 隐式类型转换

```sql
-- phone 是 VARCHAR
SELECT * FROM t WHERE phone = 13800138000;    -- ✗ 传了数字
```

```
   字符串列 = 数字  →  MySQL 把【列】转成数字再比较
                    →  相当于对列做了函数运算
                    →  【索引失效，全表扫描】

   反过来，数字列 = 字符串 → 转的是【常量】，索引仍可用
```

```
   ┌────────────────────────────────────────────────────┐
   │  规律：转换发生在【列】上就失效，发生在【常量】上没事。 │
   │  实践：参数类型永远和列类型对齐，别指望隐式转换。      │
   └────────────────────────────────────────────────────┘
```

---

## 七、连接级内存：一个容易踩的放大器

这几个参数是**每个连接独占**的，不是全局共享：

```ini
sort_buffer_size   = 256K     # 排序
join_buffer_size   = 256K     # JOIN
read_buffer_size   = 128K     # 顺序扫描
read_rnd_buffer_size = 256K   # 随机读
tmp_table_size     = 16M      # 临时表
```

```
   ⚠️ 危险在于它们会【乘以连接数】：

      sort_buffer_size = 16M  ×  500 个连接  =  8 GB
                                                  ↑ 瞬间 OOM

   所以这些参数要【保持小】。
   个别慢 SQL 需要大 buffer，就在那个会话里单独调：
      SET SESSION sort_buffer_size = 8*1024*1024;
```

```
   全局内存（Buffer Pool 等）见《BufferPool与内存.md》——
   那些是共享的，才是该往大了配的。
```

---

## 八、速查

### 工作流

```
   慢查询日志 → pt-query-digest（按【总耗时】排，不是按最慢排）
              → EXPLAIN / EXPLAIN ANALYZE
              → optimizer trace（查为什么选这个索引）
              → 改 → 用真实数据量验证
```

### EXPLAIN Extra 对照

```
   Using index                     覆盖索引，不回表          ✓ 最好
   Using where                     引擎返回后还要过滤        · 正常
   Using index condition           索引下推 ICP              ✓ 好
   Using filesort                  额外排序                  ✗ 想办法消掉
   Using temporary                 用了临时表                ✗ 想办法消掉
   Using join buffer (hash join)   被驱动表没索引            ✗ 加索引
```

### 四类问题的第一反应

```
   JOIN 慢      →  被驱动表的关联字段有索引吗？两边类型一致吗？
   选错索引     →  先 ANALYZE TABLE；还不行看 optimizer trace
   filesort     →  能不能靠联合索引让结果天然有序（等值在前、排序在后）
   临时表       →  GROUP BY 的字段有索引吗？UNION 能换 UNION ALL 吗？
```

### 别信的几条老说法

```
   ✗ "COUNT(1) 比 COUNT(*) 快"        —— 一样，无脑用 COUNT(*)
   ✗ "没索引的 JOIN 是灾难"           —— 8.0.20+ 有 Hash Join，线性复杂度
   ✗ "filesort 一定是磁盘排序"        —— 装得下就在内存里排
   ✗ "sort_buffer_size 调大能提速"    —— 它是连接级的，调大会 OOM
```

---

## 相关笔记

- 《[索引优化](索引优化.md)》——本篇的前置：先确认索引用上了，再看这里
- 《[B+ 树原理 · 第八章 / 5.4](B+树原理.md)》——联合索引的有序性是消灭 `filesort` 的根据；深分页为什么慢
- 《[事务与 MVCC · 第五节](事务与MVCC.md)》——`COUNT(*)` 慢的根因：可见性是逐行判定的
- 《[BufferPool与内存](BufferPool与内存.md)》——全局内存与连接级内存的分界
- 《[数据表设计](数据表设计.md)》——用冗余字段避免大表 JOIN
