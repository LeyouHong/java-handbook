# Java 学习手册

个人 Java 学习笔记合集。每篇都尽量做到：**从现象出发 → 追到源码 → 落回可运行的 demo**，而不是罗列 API。

## 目录

### Spring

| 笔记 | 内容 |
|---|---|
| [Bean 与 Spring IOC](spring/Spring-IOC-与-Bean.md) | IOC/DI 概念、容器内部的两个 Map、`doGetBean` 源码流程、Bean 生命周期、动态注册 Bean、获取容器的几种方式 |

### MySQL

| 笔记 | 内容 |
|---|---|
| [数据表设计](mysql/数据表设计.md) | 六项设计原则（数据量/数据关系/数据规范 + 命名/字段类型/索引）、四条实战经验（字段≤20、全 NOT NULL、索引≤6、冗余换查询）、烂表改造实例、建表 Checklist |
| [索引优化](mysql/索引优化.md) | B+ 树主键索引 vs 辅助索引、回表、冗余索引识别、最左前缀与索引字段顺序、`key_len` 反推用了几列、索引下推 ICP、索引覆盖、EXPLAIN 字段速查 |
| [B+ 树原理](mysql/B+树原理.md) | 索引优化的底层篇：为什么不用哈希/红黑树/B 树、3 层为什么能装 2190 万行、页内 Page Directory 二分、页分裂与自增主键、聚簇索引 vs 二级索引、深分页为什么慢 |

<!-- 新增笔记时在上面的表格里加一行 -->

## 约定

- 按主题分目录：`spring/`、`jvm/`、`concurrency/` ……
- 一篇笔记一个 md，文件名用中文或英文都行，保持可读优先
- 涉及源码的结论尽量**实际跑一遍验证**，并把真实日志贴进笔记
- 代码片段标注出处（类名 + 方法名），方便回头断点跟踪

## 配套代码

笔记里引用的示例代码（`io.github.leyouhong.java_learner` 包下的 `ImoocBean`、`CustomBeanRegistry`、`MessageHandler` 等）在本地的 `java_learner` Spring Boot 项目中，暂未纳入本仓库。本仓库只收录 md 笔记。

## License

笔记内容仅供个人学习参考。
