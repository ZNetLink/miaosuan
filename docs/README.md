# 📚 智网妙算 | 文档中心

欢迎来到智网妙算官方文档中心。这里汇集了从平台入门、模型开发到 API 参考的全套指南，旨在帮助你快速上手，利用 Python 与云端算力构建高效的网络仿真实验。

**智网妙算文档中心正在不断完善中。**

API参考手册：
- [核心API](./engine-api.md)
- [MMS API](./mms-api.md)

---

以下均为规划内容，请勿点击：


## 🗺️ 文档导航 (Navigation)

为了方便查阅，我们将文档分为以下几个模块：

### 1\. 🏁 快速入门 (Getting Started)

*适合初次接触平台的新用户。*

  * **[平台概览](https://www.google.com/search?q=./getting-started/overview.md)**：了解智网妙算的界面布局与基础功能。
  * **[Hello World: 第一个仿真实验](https://www.google.com/search?q=./getting-started/hello-world.md)**：5分钟内搭建并运行一个简单的点对点网络。
  * **[账号与配额](https://www.google.com/search?q=./getting-started/account.md)**：了解免费额度、教育优惠与充值说明。

### 2\. 🧩 核心概念 (Core Concepts)

*理解仿真的底层逻辑与建模基础。*

  * **[仿真引擎机制](https://www.google.com/search?q=./concepts/engine.md)**：离散事件仿真 (DES) 在本平台是如何工作的。
  * **[节点与链路](https://www.google.com/search?q=./concepts/nodes-links.md)**：如何定义网络拓扑元素。
  * **[数据包与协议栈](https://www.google.com/search?q=./concepts/packet-stack.md)**：自定义数据包格式与协议层级交互。
  * **[统计与数据采集](https://www.google.com/search?q=./concepts/statistics.md)**：如何收集吞吐量、延迟等关键指标并绘图。

### 3\. 🐍 Python 建模指南 (Modeling Guide)

*利用 Python 原生能力进行深度定制。*

  * **[编写自定义模型](https://www.google.com/search?q=./modeling/custom-models.md)**：如何继承基类编写你自己的协议算法。
  * **[外部库支持](https://www.google.com/search?q=./modeling/libraries.md)**：NumPy, Pandas, SciPy 等科学计算库的使用规范。
  * **[调试与日志](https://www.google.com/search?q=./modeling/debugging.md)**：如何在云端环境中排查代码错误。

### 4\. 🧠 AI 与智能网络 (AI Integration)

*发挥平台“AI 友好”特性的进阶指南。*

  * **[AI 交互接口](https://www.google.com/search?q=./ai/interface.md)**：仿真环境与智能体 (Agent) 的状态/动作交互机制。
  * **[强化学习示例](https://www.google.com/search?q=./ai/rl-example.md)**：结合 PyTorch/Stable-Baselines3 训练路由优化算法。
  * **[数据离线导出](https://www.google.com/search?q=./ai/data-export.md)**：批量导出仿真数据用于机器学习训练。

### 5\. 📖 API 参考手册 (API Reference)

*详细的类、方法与参数说明。*

  * **[Core API](https://www.google.com/search?q=./api/core.md)**
  * **[Events API](https://www.google.com/search?q=./api/events.md)**
  * **[Topology API](https://www.google.com/search?q=./api/topology.md)**

-----

## 💡 常见问题 (FAQ)

在深入阅读文档前，您可以查看 [FAQ 页面](https://www.google.com/search?q=./faq.md)，这里收录了：

  * "为什么我的仿真运行速度很慢？"
  * "如何复现经典的 TCP 拥塞控制算法？"
  * "内测期间的数据会被保留吗？"

-----

## 🤝 如何贡献文档

本文档与平台代码库一同托管在 GitHub 上。如果您在阅读过程中发现错别字、死链，或者有更好的补充说明，我们非常欢迎您的贡献！

1.  **轻量修改**：直接点击文档页面右上角的 "Edit" 按钮提交 Pull Request。
2.  **重大补充**：请先在 [可疑链接已删除] 中提交建议，与我们讨论文档结构。

-----

## 🆘 遇到困难？

如果文档没能解决您的问题：

1.  请先搜索 **Issue**，查看是否有其他用户遇到过类似问题。
2.  如果是新问题，请提交一个新的 Issue，并打上 `question` 或 `documentation` 的标签。
