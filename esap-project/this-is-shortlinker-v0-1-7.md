> *一个干净到你不想再看别家短链平台的开源项目。*

---

当我们在项目中需要一个短链接服务时，我们真正关心的是什么？

* ⚡ 快速部署
* 🧩 简洁 API
* 📊 可视化后台
* 🔐 自主可控的数据存储
* 🔌 灵活的扩展性

市面上的解决方案很多，但不是太重，就是不够自由。于是我们打造了 **ShortLinker** ——一个极简主义的短链接服务，**由 Rust 构建，界面优雅，功能清晰，注重开发者体验**。

今天，我们迎来了 ShortLinker 的 v0.1.7-alpha.3 版本，这不仅是一个迭代更新，更像是「第一次成型」的发布。👇

---

## ✨ 核心亮点：不仅好用，还很好看

### 🌈 1. 全新的管理面板上线

这次版本带来了从零设计的 **Admin Panel**，采用 Vue 3 + TailwindCSS 构建，暗黑主题、渐变风格，极致轻量美感：

* 登录系统（支持本地验证）
* 系统健康状态：服务在线监控
* 实时数据总览：链接数、响应延迟
* 存储状态可视化（支持 SQLite，PostgreSQL/MariaDB 待集成）
* 多语言 + 系统主题切换

> 开源项目也能拥有媲美 SaaS 的 UI，谁说不能？

[caption id="attachment_2244" align="aligncenter" width="1024"]<img src="https://www.esaps.net/wp-content/uploads/2025/06/Screenshot_11-6-2025_0224_127.0.0.1-1024x779.webp" alt="Shortlinker v0.1.7 仪表盘" width="1024" height="779" class="size-large wp-image-2244" /> Shortlinker v0.1.7 仪表盘[/caption]

---

### 🧠 2. 架构升级：缓存 & 存储模块解耦

我们将原本耦合在一起的逻辑做了彻底拆分，引入了如下架构理念：

* ✅ Cache 抽象为插件式结构（支持 L1 + L2 分层缓存）
* ✅ Storage 接口模块化（目前支持 SQLite，未来扩展 PostgreSQL、MySQL 等）
* ✅ 支持点击计数、过滤器（Bloom）作为可选项

> 架构设计未来将支持热插拔 —— 想接什么组件，自己选。

---

### 🧪 技术细节

| 模块    | 技术栈                        |
| ----- | -------------------------- |
| 后端    | Rust + Actix-web           |
| 前端    | Vue 3 + TS + Tailwind      |
| 存储层   | SQLite（默认）                 |
| 可视化面板 | Vite + Pinia + ECharts（即将） |

---

## 🚀 快速上手

```bash
git clone https://github.com/AptS-1547/shortlinker.git
cd shortlinker

# 启动
ADMIN_TOKEN=123456 ENABLE_FRONTEND_ROUTES=true cargo run

# 管理面板地址
http://127.0.0.1:8080/panel
```

---

## 🛣️ 下一步计划

* [ ] 权限控制 & 多用户支持
* [ ] 短链搜索功能
* [ ] Webhook / Click 统计 API 导出

---

## ❤️ 致开发者

ShortLinker 项目的初衷，是为了那些在部署时说一句「我不想用 Firebase、也不想连 Mongo 的人」。我们选择了 Rust，是因为它安全、现代、表达力强；我们选 Vue + Tailwind，是因为它简洁而高效。

如果你想加入共建，欢迎提 PR、开讨论。如果你用它做了什么酷东西，也欢迎分享给我们！

---

**Star 一下，不会迷路：**
👉 [GitHub 地址](https://github.com/AptS-1547/shortlinker)

