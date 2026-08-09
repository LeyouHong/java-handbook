# InnoDB 的内存：Buffer Pool、change buffer 与刷盘

> 环境：MySQL 8.0 / InnoDB。本篇配置与命令为**示例**，未附实测输出。
>
> **一句话主线：InnoDB 的所有性能优化都指向同一件事——少读一次磁盘、少写一次磁盘。**
>
> Buffer Pool 负责"少读"，change buffer 和 redo log 负责"少写"（或至少"顺序地写"）。

前面几篇多次提到 Buffer Pool 和 change buffer 却没有定义它们——《[乐观锁与悲观锁 · 5.6 坑三](../concurrency/乐观锁与悲观锁.md)》说"唯一索引用不了 change buffer"，《[B+ 树原理](B+树原理.md)》整篇算的都是"读几次磁盘"。这一篇把它们补上。

---

## 一、Buffer Pool：磁盘页在内存里的副本

> ⤵ **依据**：《[B+ 树原理 · 第一章](B+树原理.md)》——InnoDB 读写的最小单位是 **16KB 的页**，
> 一次磁盘随机 IO 大约 10ms，而内存访问是纳秒级。差了六个数量级，所以一切都围绕"别去磁盘"设计。

```
   ┌─ 内存 ─────────────────────────────────────────┐
   │  Buffer Pool                                    │
   │  ┌────┬────┬────┬────┬────┬────┬────┬────┐    │
   │  │page│page│page│page│page│page│page│page│ …  │  每个 16KB
   │  └────┴────┴────┴────┴────┴────┴────┴────┘    │
   └─────────────────┬───────────────────────────────┘
                     │ 缺页时才去读
   ┌─ 磁盘 ──────────▼───────────────────────────────┐
   │  ibd 文件（表空间）                              │
   └─────────────────────────────────────────────────┘
```

读写流程：

```
   读：先看 Buffer Pool 里有没有
         有 → 直接返回                      （命中）
         无 → 从磁盘读进来，再返回           （缺页）

   写：【只改 Buffer Pool 里的页】，标记为"脏页"
       → 立刻返回，不等磁盘
       → 由后台线程择机刷盘
```

```
   ┌──────────────────────────────────────────────────────────┐
   │  注意第二条：UPDATE 返回成功时，数据【很可能还没落盘】。   │
   │  那凭什么说它不会丢？—— 靠 redo log（第五节）。            │
   │                                                           │
   │  这就是 WAL（Write-Ahead Logging）：                       │
   │  改内存 + 顺序写日志 → 就算提交成功；                      │
   │  真正的数据页慢慢刷。                                      │
   └──────────────────────────────────────────────────────────┘
```

> 同一个思想在《[LSM Tree 原理 · 第 5 步](../storage/LSM树原理.md)》里也出现过：
> 先顺序写日志保证不丢，再慢慢整理数据结构。**顺序写比随机写快几个数量级**，是两类引擎共同的地基。

### 配置

```ini
innodb_buffer_pool_size = 8G      # 【最重要的一个参数】默认才 128M
innodb_buffer_pool_instances = 8  # 拆成多个实例，减少内部锁竞争
                                  # 每个实例建议 ≥ 1G，否则拆了没意义
```

```
   经验值：
     数据库专用机  →  物理内存的 60% ~ 80%
     混部机器      →  物理内存的 50% 左右

   判断够不够，不看比例，看【命中率】和【有没有频繁淘汰】（第七节）。
```

---

## 二、改良版 LRU：防止一次全表扫描毁掉整个缓存

Buffer Pool 满了要淘汰页。朴素 LRU 有个致命问题：

```
   一条 SELECT * FROM big_table 全表扫描
        ↓
   几百万个页依次被读进来，全部挤到 LRU 头部
        ↓
   【真正的热点页被挤到尾部淘汰掉】
        ↓
   扫描结束后，这些页再也不会被访问，但缓存已经被彻底污染
```

InnoDB 的解法是把 LRU 链表分成两段：

```
   ┌────────────── young 区（5/8）──────────────┬── old 区（3/8）──┐
   │  真正的热数据                                │  新读进来的页     │
   └──────────────────────────────────────────────┴───────────────────┘
        ↑ 头部                                            ↑ 中点插入   ↑ 尾部淘汰

   新页【不插头部，插在中点】（old 区的头部）
```

然后加一条时间规则：

```
   一个页进入 old 区后，
     · 【1 秒内】再被访问 → 判定为"扫描"，【留在 old 区】
     · 【1 秒后】才被访问 → 判定为"真热点"，晋升到 young 区头部
```

```
   为什么这条规则有效？

   全表扫描时，一个数据页里的 N 行会被【连续读完】，
   这些访问全部发生在读进来后的极短时间内 → 判定为扫描 → 不晋升
   → 扫完就在 old 区被淘汰，【碰不到 young 区的热点数据】✓
```

```ini
innodb_old_blocks_pct  = 37     # old 区占比，默认 37%（≈3/8）
innodb_old_blocks_time = 1000   # 毫秒，默认 1000
```

```
   有大量全表扫描（报表、备份）的库，可以把 old_blocks_time 调大，
   进一步降低污染。绝大多数情况保持默认即可。
```

---

## 三、脏页与刷盘

被修改过但还没写回磁盘的页叫**脏页**。

### 3.1 什么时候刷

```
   ① redo log 快写满了     → 【必须】立刻刷，否则没地方写新日志
   ② 脏页比例太高          → 主动刷
   ③ 系统空闲              → 顺手刷
   ④ 正常关闭              → 全刷完
```

第 ① 种最危险：

```
   ┌────────────────────────────────────────────────────────┐
   │  redo log 写满时，MySQL 会【停止所有更新】，              │
   │  强制推进检查点、把脏页刷下去，然后才继续。               │
   │                                                         │
   │  表现：写入 TPS 突然掉到 0，持续几秒到几十秒。            │
   │  这是最典型的"数据库莫名其妙卡一下"。                     │
   └────────────────────────────────────────────────────────┘
```

### 3.2 关键参数

```ini
innodb_io_capacity      = 2000   # 【必须按你的磁盘调】告诉 InnoDB 磁盘每秒能做多少 IOPS
innodb_io_capacity_max  = 4000   # 紧急情况下的上限

innodb_max_dirty_pages_pct     = 90   # 脏页比例上限
innodb_max_dirty_pages_pct_lwm = 10   # 超过这个就开始预刷（低水位）

innodb_flush_neighbors = 0       # 刷一个页时要不要连带刷相邻的脏页
                                 # 机械盘 = 1（合并随机 IO）
                                 # SSD    = 0（没必要，8.0 已默认 0）
```

```
   innodb_io_capacity 是最常被配错的参数：
     默认 200 —— 这是【机械盘】的水平。
     SSD 能做几万 IOPS，还配 200 的话，InnoDB 会以为磁盘很慢，
     刷脏页刷得畏畏缩缩 → 脏页堆积 → 最后触发第 ① 种强制刷 → 卡顿。

   NVMe SSD 配 2000~20000 都合理，用 fio 实测后再定。
```

### 3.3 doublewrite：为什么要写两遍

```
   问题：InnoDB 的页是 16KB，操作系统/磁盘的原子写单位通常是 4KB。
        刷一个页刷到一半断电 → 【页损坏】（partial page write）
        → redo log 也救不了，因为 redo 是"在原页基础上做修改"，
          原页本身都坏了，改无可改

   解法：刷脏页时先把页【顺序写】到一块专门的 doublewrite 区域，
        再写到真正的位置。
        崩溃后发现页损坏 → 从 doublewrite 区域拿完好的副本恢复
```

```ini
innodb_doublewrite = ON     # 保持开启。关掉能提升写性能，但会赌上数据完整性
```

```
   代价没有想象中大：doublewrite 区域是【顺序写】的，
   而顺序写比随机写便宜得多。
```

---

## 四、change buffer：终于定义它

前面几篇提了它 6 次，这里说清楚。

### 4.1 它解决什么问题

```
   UPDATE t SET name = 'x' WHERE id = 7;      -- name 上有普通索引

   改主键索引：id=7 的页大概率在 Buffer Pool 里（刚查过）  → 快
   改 name 索引：'x' 应该插到 name 索引树的哪一页？
                 那一页【很可能不在内存里】 → 要先从磁盘读进来 → 慢
```

```
   ┌──────────────────────────────────────────────────────────┐
   │  痛点：为了改一个二级索引，要付出一次【随机磁盘读】。       │
   │        而这个读，唯一的作用只是"把页拿进来好让我改它"。     │
   └──────────────────────────────────────────────────────────┘
```

### 4.2 它怎么做

```
   目标页不在内存 → 【先把这个改动记在 change buffer 里】，直接返回
                    ↓
   等到某次查询真的要读这个页时（或后台线程主动 merge）
                    ↓
   把页读进来，然后把攒着的改动一次性应用上去
```

```
   收益 = 省掉的那次随机读
   而且如果一个页上攒了 10 个改动，就省了 10 次读 → 攒得越多越赚
```

```
   注意 change buffer 也会持久化到 redo log，
   所以"记在 change buffer 里就返回"不会丢数据。
```

### 4.3 为什么唯一索引用不了

```
   插入前必须检查有没有重复 → 检查就要看那一页的内容
                            → 【必须把页读进内存】
                            → 那次随机读【躲不掉】
                            → change buffer 的唯一收益消失了
```

```
   ┌──────────────────────────────────────────────────────────┐
   │  唯一索引  = 必须读页才能判重 → change buffer 无用武之地    │
   │  普通索引  = 不用判重，可以先记账 → change buffer 生效      │
   │                                                           │
   │  主键索引也用不了（它本身就唯一，而且插入位置必须确定）      │
   └──────────────────────────────────────────────────────────┘
```

> 这就是《[乐观锁与悲观锁 · 5.6 坑三](../concurrency/乐观锁与悲观锁.md)》说的"别为了顺便快一点乱加 UNIQUE"——
> 加一个不必要的唯一约束，等于主动放弃了这一层写优化。

### 4.4 什么时候收益大

```
   ✓ 写多读少               改动能在 change buffer 里攒很久才 merge  → 赚
   ✗ 写完立刻读             刚记进去就要 merge → 白折腾一趟 → 【反而更慢】
   ✓ 数据量远大于内存        页大概率不在内存，省的读多
   ✗ 内存装得下整个库        页本来就在内存，没有读可省
```

```ini
innodb_change_buffering      = all   # none/inserts/deletes/changes/purges/all
innodb_change_buffer_max_size = 25   # 最多占 Buffer Pool 的 25%
```

```
   写密集型（日志表、行为流水）可以调到 50；
   读密集型或"写完马上读"的业务，考虑设成 none。
```

---

## 五、redo log buffer 与刷盘策略

```
   事务修改数据
        ↓
   ① 改 Buffer Pool 里的页（变脏页）
        ↓
   ② 写 redo log buffer（内存）
        ↓
   ③ 按策略写入 OS cache / fsync 到磁盘
```

```ini
innodb_log_buffer_size = 16M          # redo log 的内存缓冲
innodb_redo_log_capacity = 4G         # 8.0.30+ 用这一个参数
                                      # （旧版是 innodb_log_file_size × innodb_log_files_in_group）
```

### `innodb_flush_log_at_trx_commit`：又一个安全 vs 性能的旋钮

```
   = 1   每次提交都 write + fsync 到磁盘   【默认，不丢数据】
   = 2   每次提交只 write 到 OS cache，每秒 fsync
         → MySQL 进程崩溃不丢；【操作系统崩溃/断电会丢最近 1 秒】
   = 0   每秒才 write + fsync
         → MySQL 进程崩溃就【丢最近 1 秒】
```

```
   ┌──────────────────────────────────────────────────────────┐
   │  = 1 才是真正的"提交即持久"。                              │
   │  设 2 或 0 换来的性能，代价是明确的数据丢失窗口。          │
   │                                                           │
   │  它和 sync_binlog=1 是一对，合称"双 1"配置 ——              │
   │  金融类业务的底线。                                        │
   └──────────────────────────────────────────────────────────┘
```

> 那为什么"每次提交都 fsync"还能跑出高 TPS？靠**组提交**——
> 详见《[binlog · 第四节](binlog.md)》，那里讲了 fsync 是毫秒级硬件动作、
> 怎么用一次 fsync 服务一整批事务。

### redo log 太小的后果

```
   redo log 容量小 → 很快写满 → 频繁触发"强制刷脏页"（3.1 的第 ① 种）
                              → 写入 TPS 周期性掉零

   判断方法：SHOW ENGINE INNODB STATUS 里看 LOG 段，
            "Log sequence number" 和 "Last checkpoint at" 的差值
            如果长期接近 redo 总容量 → 该调大了
```

---

## 六、自适应哈希索引（AHI）

```
   InnoDB 观察到某个索引前缀被【反复等值查询】
        ↓
   自动在内存里为它建一个哈希表
        ↓
   下次同样的查询：哈希 O(1) 直达，【跳过 B+ 树的 3 层查找】
```

```ini
innodb_adaptive_hash_index = ON     # 默认开
```

```
   ⚠️ 它不总是好事：
      · 只对【等值查询】有效，范围查询用不上
      · 维护它需要加锁 —— 高并发下 AHI 的锁竞争可能成为瓶颈
      · 有过因为它导致性能反降的案例

   判断：SHOW ENGINE INNODB STATUS 的 INSERT BUFFER AND ADAPTIVE HASH INDEX 段，
        看命中率。命中率低而 CPU 高 → 试着关掉对比。
```

这是少数几个"默认开着但可能要关掉"的参数。

---

## 七、全局内存 vs 连接级内存

**这是内存配置最容易出事的地方。**

```
   ┌─ 全局共享（配一次，所有连接共用）────────────────┐
   │  innodb_buffer_pool_size      ← 大头，往大了配     │
   │  innodb_log_buffer_size                           │
   │  key_buffer_size（MyISAM 用，InnoDB 库可以调小）   │
   └───────────────────────────────────────────────────┘

   ┌─ 连接级（每个连接【各自】分配一份）──────────────┐
   │  sort_buffer_size                                 │
   │  join_buffer_size                                 │
   │  read_buffer_size / read_rnd_buffer_size          │
   │  tmp_table_size                                   │
   │       ↑ 这些要【保持小】                           │
   └───────────────────────────────────────────────────┘
```

```
   ┌──────────────────────────────────────────────────────────┐
   │  致命算术：                                                │
   │     sort_buffer_size = 16M  ×  500 个连接  =  8 GB         │
   │                                                            │
   │  "我把 sort_buffer 调大点提升排序性能" ——                  │
   │  在高连接数下，这句话等于 OOM。                            │
   └──────────────────────────────────────────────────────────┘
```

```
   正确做法：连接级参数保持默认（几百 KB），
            个别需要大 buffer 的慢 SQL 在【会话内】单独调：
               SET SESSION sort_buffer_size = 8*1024*1024;
```

粗略的内存预算公式：

```
   总内存 ≈ innodb_buffer_pool_size
          + innodb_log_buffer_size
          + max_connections × (sort_buffer + join_buffer + read_buffer + …)
          + MySQL 自身开销

   留够 20% 给操作系统 —— 被 OOM Killer 干掉的 MySQL 恢复起来很痛苦。
```

---

## 八、监控与排查

### 8.1 命中率

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
```

```
   Innodb_buffer_pool_read_requests   逻辑读（从 Buffer Pool 读）总次数
   Innodb_buffer_pool_reads           物理读（不得不去磁盘）次数

   命中率 = 1 - reads / read_requests

   OLTP 系统应该 > 99%。
   低于 95% → Buffer Pool 太小，或者有大量全表扫描在污染缓存
```

### 8.2 详细状态

```sql
SELECT * FROM information_schema.INNODB_BUFFER_POOL_STATS\G
--  POOL_SIZE / FREE_BUFFERS / DATABASE_PAGES / MODIFIED_DATABASE_PAGES(脏页数)
--  PAGES_MADE_YOUNG / PAGES_NOT_MADE_YOUNG   ← 晋升情况，看 LRU 是否被污染

SHOW ENGINE INNODB STATUS\G
--  BUFFER POOL AND MEMORY 段：
--    Free buffers        剩多少空闲页，长期为 0 说明一直在淘汰
--    Modified db pages   脏页数量
--    Buffer pool hit rate  命中率（形如 1000 / 1000）
--  LOG 段：
--    Log sequence number / Last checkpoint at   ← 差值判断 redo 压力
```

### 8.3 重启预热

```ini
innodb_buffer_pool_dump_at_shutdown = ON   # 关机时把热点页的页号存下来
innodb_buffer_pool_load_at_startup  = ON   # 启动时预先加载回来
innodb_buffer_pool_dump_pct = 25           # 存最热的 25%
```

```
   5.7+ 默认就是开的。没有它，重启后 Buffer Pool 是空的，
   数据库要"冷跑"很久才恢复正常性能 —— 期间响应可能慢十几倍。
   做主从切换、滚动重启前要意识到这一点。
```

---

## 九、速查

### 内存分布

```
   ┌────────────────────────────────────────────────────────────┐
   │  Buffer Pool（大头，60~80% 物理内存）                       │
   │    ├─ 数据页 / 索引页    改良 LRU：young 5/8 + old 3/8      │
   │    ├─ change buffer      ≤ 25%，只服务【非唯一二级索引】     │
   │    └─ 自适应哈希索引     等值查询提速，高并发下可能要关       │
   │                                                             │
   │  redo log buffer（16M）                                     │
   │  ─────────────────────────────────────────────────────────  │
   │  连接级 buffer × 连接数   ← 保持小！会被乘以连接数            │
   └────────────────────────────────────────────────────────────┘
```

### 必调参数

```ini
innodb_buffer_pool_size          = 物理内存 60~80%    # 最重要
innodb_io_capacity               = 按磁盘实测（SSD 别留 200）
innodb_redo_log_capacity         = 4G                 # 8.0.30+
innodb_flush_log_at_trx_commit   = 1                  # 双 1
sync_binlog                      = 1                  # 双 1
innodb_flush_neighbors           = 0                  # SSD
```

### 三个"和直觉相反"

```
   ✗ "调大 sort_buffer_size 能加速排序"
     → 它是连接级的，×连接数 = OOM

   ✗ "自适应哈希索引默认开着，说明总是好的"
     → 高并发等值场景下它的锁可能是瓶颈，有关掉更快的案例

   ✗ "innodb_io_capacity 用默认就行"
     → 默认 200 是机械盘水平，SSD 上会导致刷脏页太慢→周期性卡顿
```

### 排查三步

```sql
-- ① 命中率够不够
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
-- ② 脏页多不多、空闲页还有没有
SHOW ENGINE INNODB STATUS\G     -- BUFFER POOL AND MEMORY 段
-- ③ redo 压力大不大
SHOW ENGINE INNODB STATUS\G     -- LOG 段，LSN 与 checkpoint 的差
```

---

## 相关笔记

- 《[B+ 树原理 · 第一章](B+树原理.md)》——16KB 页、磁盘 IO 才是瓶颈；本篇是"怎么让这些 IO 不发生"
- 《[乐观锁与悲观锁 · 5.6 坑三](../concurrency/乐观锁与悲观锁.md)》——唯一索引用不了 change buffer，本篇第四节给出原因
- 《[binlog · 第四节](binlog.md)》——`sync_binlog` 与组提交，和 `innodb_flush_log_at_trx_commit` 合称"双 1"
- 《[SQL优化进阶 · 第七节](SQL优化进阶.md)》——连接级 buffer 的具体用途（排序、JOIN、临时表）
- 《[LSM Tree 原理 · 第 5 步](../storage/LSM树原理.md)》——"先顺序写日志、再慢慢整理数据"是两类引擎共同的地基
