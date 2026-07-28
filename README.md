# Java 学习手册

个人 Java 学习笔记合集。每篇都尽量做到：**从现象出发 → 追到源码 → 落回可运行的 demo**，而不是罗列 API。

## 目录

### Spring

| 笔记 | 内容 |
|---|---|
| [Bean 与 Spring IOC](spring/Spring-IOC-与-Bean.md) | IOC/DI 概念、容器内部的两个 Map、`doGetBean` 源码流程、Bean 生命周期、动态注册 Bean、获取容器的几种方式 |

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
