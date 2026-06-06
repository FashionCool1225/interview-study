# Interview Study - Java 后端面试备战系统

面向 3 年左右经验的 Java 后端开发者的面试知识体系与复习工具，涵盖 **155+ 高频面试题**，配套交互式单页学习应用，支持知识卡片浏览、自测、模拟面试和选择题速练。

## 项目结构

```
interview-study/
├── index.html                 # 交互式学习应用（单文件，浏览器直接打开）
├── PROJECT_GOALS.md           # 项目目标与设计文档
├── 00-复习计划.md              # 4 周分阶段复习计划（每日安排）
├── 01-Java基础.md              # Java 集合、并发编程、JVM（25 题）
├── 02-Spring-Boot-Cloud.md     # Spring 核心、Boot 自动配置、Cloud 微服务（20 题）
├── 03-MySQL.md                 # 索引、事务、MVCC、SQL 调优、锁机制（18 题）
├── 04-Redis.md                 # 数据结构、持久化、集群、缓存穿透/击穿/雪崩（24 题）
├── 05-消息队列.md              # MQ 选型、消息可靠性、顺序性、幂等（15 题）
├── 06-Docker-K8s.md            # Docker 容器化、K8s 核心概念与部署（18 题）
├── 07-项目经验梳理指南.md       # STAR 法则、亮点提炼、面试话术脚本（9 题）
├── resume.html                # 简历模板（标准版）
├── resume-A-minimalist.html   # 简历模板（极简版）
├── resume-B-dark.html         # 简历模板（暗色版）
├── resume-C-creative.html     # 简历模板（创意版）
├── resume-v2.html             # 简历模板（v2 改版）
├── resume-concise.pdf         # 简历 PDF（精简版）
└── resume-detailed.pdf        # 简历 PDF（详细版）
```

## 快速开始

直接用浏览器打开 `index.html` 即可使用，无需安装任何依赖或启动后端服务。所有学习进度通过浏览器 localStorage 本地保存。

## 知识覆盖

| 模块 | 题数 | 核心内容 |
|------|------|---------|
| Java 基础 | 25 | HashMap 原理、线程池、锁机制、JVM 内存模型与 GC |
| Spring 全家桶 | 20 | IOC/AOP、Bean 生命周期、事务传播、Boot 自动配置、Cloud 微服务 |
| MySQL | 18 | B+ 树索引、MVCC、事务隔离、SQL 优化、分库分表 |
| Redis | 24 | 五大数据结构、RDB/AOF、哨兵与集群、缓存三大问题、分布式锁 |
| 消息队列 | 15 | 解耦/异步/削峰、消息丢失与重复、RocketMQ vs Kafka |
| Docker & K8s | 18 | 镜像分层、Dockerfile、Pod/Service/Deployment、滚动更新 |
| 项目经验 | 9 | STAR 法则、CRUD 项目包装、技术亮点提炼、面试话术 |

## 复习计划

`00-复习计划.md` 提供了 **4 周每日学习计划**：

- **第 1 周** — Java 基础 + Spring 核心（每天 3-4 小时）
- **第 2 周** — Spring Boot/Cloud + MySQL 深入
- **第 3 周** — Redis + 消息队列 + Docker/K8s
- **第 4 周** — 项目经验打磨 + 模拟面试冲刺

## 交互式学习应用

`index.html` 是一个零依赖的单文件 Web 应用，功能包括：

- 知识卡片浏览与折叠展开
- 全文搜索
- 自测模式（隐藏答案，点击显示）
- 模拟面试（随机抽题，计时作答）
- 选择题速练
- 学习进度追踪与可视化
- 响应式布局，适配手机和桌面端

## 简历模板

项目包含多套简历 HTML 模板，可直接在浏览器中打印为 PDF 使用：

- **标准版** — 适合大多数技术岗位投递
- **极简版** — 信息密度低，排版清爽
- **暗色版** — 深色主题，适合设计感强的场景
- **创意版** — 色彩丰富，突出个性

## 技术栈

纯前端实现，HTML + CSS + JavaScript 单文件架构，无框架依赖。CSS 使用 `clamp()` 实现响应式布局，JavaScript 使用 localStorage 做状态持久化。

## License

MIT
