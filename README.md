# frontend-universal-standards-skill

> 前端全栈工程化、架构分层、网络请求、状态管理、路由鉴权、多端开发与代码质量通用 Agent Skill。

---

## 📖 简介 (Introduction)

`frontend-universal-standards-skill` 是一套遵循 **Agent Skills 开放标准 (SKILL.md)** 打造的企业级通用前端全栈开发与代码审查规范体系。

本 Skill 旨在为 AI Coding Agent 提供统一、标准、工业级的前端工程指导，涵盖现代 Web 应用、企业级中后台系统、移动端 H5 以及跨平台小程序等多种主流应用形态。指导 Agent 在进行前端脚手架搭建、模块分层、API 封装、路由鉴权、状态管理、组件编写及样式适配时严格遵循最佳实践，坚决避免代码生成中的过度设计与反模式。

---

## 🎯 核心规范矩阵 (Core Matrix)

1. **零过度设计原则 (Zero Over-Engineering)**：严格按当前项目实际依赖与技术栈（Vue 3 / React / Uni-app / Vue 2）精准按需匹配，严禁堆砌无关代码与框架。
2. **工业级网络请求 (Axios / Uni.request)**：双轨拦截、BaseURL 环境变量动态注入、Token 自动附加、双 Token 无感静默刷新（Silent Token Refresh）、统一解构核心载荷、业务错误码字典映射、API 纯函数隔离。
3. **RBAC 动态鉴权与防御性路由**：常量路由与异步权限路由解耦，通配 404 挂载时序控制，白名单机制，NProgress 进度条与文档标题同步。
4. **统一状态管理与跨端持久化**：Pinia Setup Store、Zustand 与 Redux Toolkit 选型标准，State 最小化原则，Uni-app Storage 跨端适配器。
5. **Uni-app 与小程序跨端特化**：严守 2MB 主包红线，主包与分包架构划分，`preloadRule` 空闲预加载，Easycom 自动扫描。
6. **五重代码质量卡点**：ESLint 9+ Flat Config + Stylelint (Recess 属性排序) + Prettier + Husky + Commitlint 打造坚不可摧的提交前防线。
7. **复杂场景与工程亮点**：大文件分片断点续传与秒传（SparkMD5）、图片视口交叉懒加载（`IntersectionObserver`）、WebSocket/STOMP 长连接容灾。

---

## 📂 Skill 目录结构 (Directory Layout)

```text
frontend-universal-standards-skill/
├── references/                                 # 垂直领域核心规范手册
│   ├── architecture_and_directory.md           # 目录分层、路由守卫与 RBAC 权限
│   ├── coding_and_component_standards.md       # TypeScript、Vue 3.5+ / React 编码范式
│   ├── network_and_api_standards.md            # Axios / Uni.request 工业级封装与无感刷新
│   ├── uniapp_and_miniprogram_standards.md     # 小程序 2MB 分包与跨端开发
│   ├── engineering_and_quality.md              # 五重卡点、ESLint 9 Flat 与工程卫生
│   └── ui_ux_and_style_standards.md            # Tailwind CSS、SCSS、多端适配
├── README.md                                   # Skill 说明文档
└── SKILL.md                                    # Agent Skill 标准规范入口
```

---

## 🚀 使用指南 (Usage Guidelines)

* **日常编码与脚手架搭建**：Agent 根据目标工程的技术栈类型，自动加载对应的 `references/` 细分文档，输出符合工业级标准的前端代码结构。
* **代码审查 (Code Review)**：对标本 Skill 提出的“API 纯函数原则”、“GET 请求安全规范”、“通配路由挂载时序”、“生命周期注销清理”等核心铁律，审查代码中的潜在缺陷与反模式。
* **渐进式重构 (Refactoring)**：指导老旧项目在不破坏现有业务的前提下，向 TypeScript 强类型、组合式 API 以及现代工程化卡点体系平滑演进。

---

## 🤖 跨平台兼容性 (Compatibility)

本 Skill 符合通用 Agent Skill 规范，开箱即用兼容以下所有 AI 编程助手与 Agent 工具：
* **OpenAI Codex (Desktop / CLI)**
* **Claude Code**
* **Cursor**
* **Windsurf**
* **Cline / Roo Code**
* **OpenCode / Kiro / Trae / Antigravity**