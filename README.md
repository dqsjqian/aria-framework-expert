# C/C++ Architecture Expert / C/C++ 架构专家

简体中文 | [English](README.en.md)

职业头衔：**C/C++架构顾问**（C/C++ Architecture Consultant）

一个 WorkBuddy 专家包。专家 Ari 覆盖 C（C89 → C23，跟踪 C2y 草案）与 C++（C++11 → C++26）的语言设计、编译器支持与架构方法，深度支持 Aria 系列（Aria 3.2 C++23 响应式 MVVM 框架、AriaTools、AriaAgent、AriaRead、AriaTutorial）、Mira 协程网络库与 SQLiteCpp 存储库，并把它们与业界主流框架放在一起做选型对照。

## 类型

Agent 型（单个 AI 专家），行业分类：技术工程。

## 擅长领域

| 标签 | 优先推荐 | 并列对照的业界方案 |
|---|---|---|
| C/C++26 | Aria、Mira、SQLiteCpp、Tutti 中的 C++23 实践；AriaRead 的 C 库整合 | GCC 16 / Clang 23 / MSVC、Boost、Abseil、Folly、stdexec、{fmt}、vcpkg / Conan、Sanitizers |
| MVVM跨端 | Aria + AriaTools（四端、17 模块的架构样板）、AriaAgent、AriaRead、AriaTutorial | Qt Widgets / Qt Quick、Slint、Flutter、React Native、Compose Multiplatform、.NET MAUI、wxWidgets、Dear ImGui |
| 网络与存储 | Mira（TCP / UDP / TLS / WebSocket / HTTP/1.1 / HTTP/2 / QUIC / HTTP/3 / SOCKS5 / DoH / MQTT）、SQLiteCpp（C++23 纯头文件 SQLite 封装） | Boost.Asio、libuv、libcurl、cpp-httplib、gRPC、nghttp2 / ngtcp2；SRombauts/SQLiteCpp、sqlite_orm、SOCI、RocksDB、DuckDB |

## 功能

- **C 语言**：C23 全特性与 C2y 草案（`_Countof`、if 声明、具名循环、defer TS 25755），附 GCC 16.2、Clang 22 / 23、Apple Clang 21 的实测支持表；不透明句柄、错误模型、ABI 稳定等 C 架构惯用法；高频 UB；C/C++ 互操作。
- **现代 C++**：C++11 → C++26 演进；反射、契约、`std::execution`、标准库加固等 C++26 特性的定稿状态与实测支持，以及没有进入 C++26 的提案。
- **架构设计**：分层与依赖方向、UI 模式选型、二进制边界、错误处理、并发模型、构建与依赖管理、测试与性能门禁、代码审查清单。
- **Aria 系列**：响应式核心、绑定引擎与 Qt6 / AppKit / UIKit / JNI / HTTP 五个适配器、协程异步、集合 / 校验 / 导航、ABI 2、编号契约、1.x → 2.0 → 3.x 迁移；AriaTools、AriaAgent、AriaRead 的真实架构；ForgeDeps / ForgeRecipes 依赖构建。
- **网络与存储**：Mira 的模块与协议覆盖、协程模型、取消与超时、构建接入与能力边界；SQLiteCpp 的 API、语句缓存、事务、线程规则与常见坑，以及它与同名 SRombauts/SQLiteCpp 的区别。
- **自我进化**：版本类问题以各项目 GitHub 远端为准核对，核实后更新知识库；标准与编译器问题优先用最小样例实测。

内置知识库 `skills/c-cpp-architecture-knowledge/`（11 份参考文档）：

| 文档 | 内容 |
|---|---|
| `c-language.md` | C 标准演进、C23 / C2y 与编译器实测、惯用法、UB、C/C++ 互操作、C 生态 |
| `cpp-standards-architecture.md` | C++11 → C++26 与编译器实测、架构设计方法 |
| `aria-ecosystem.md` | Aria 系列全景：各仓库定位、版本、依赖关系、构建方式 |
| `industry-landscape.md` | 三个擅长领域的选型对照 |
| `aria-overview.md` | Aria 框架定位、模块、ABI、线程模型、构建、依赖、迁移 |
| `aria-reactive-core.md` | Property / Computed / Effect / Subscription / batch |
| `aria-binding-adapters.md` | BindingEngine、五个适配器、ViewModel、Navigator、HTTP 适配器 |
| `aria-async-collections-validation.md` | Task / AsyncCommand / Channel、ObservableList 与派生视图、Validator |
| `aria-contracts-faq.md` | 编号契约、错误模型、诊断、FAQ 与迁移 FAQ |
| `mira-networking.md` | Mira 网络库 |
| `sqlitecpp-storage.md` | SQLiteCpp 存储库 |

版本快照（2026-10-29 与各项目 GitHub 远端核对）：Aria 3.2.6、AriaTools 1.1.0、AriaAgent 0.2.0、AriaRead 0.3.1、AriaTutorial 0.2.0、Mira 1.2.1、SQLiteCpp 1.2.1、Tutti 2.1.1、ForgeDeps 0.4.10、ForgeRecipes 0.1.8。知识库中的独立代码示例已用对应版本的头文件编译验证。

## 试试这样问

- 用 Aria + Mira + SQLiteCpp 给我设计一个跨端应用的分层架构
- 帮我把一个 C 库封装成安全的 C++23 接口，同时保留稳定的 C ABI
- 网络层选 Mira、Boost.Asio 还是 libcurl？结合我的场景对比一下
- 我要从 Aria 2.x 升级到 3.2，帮我梳理 breaking changes
- C++26 的反射和契约现在哪些编译器能用？给我最小样例

## 相关项目

[Aria](https://github.com/dqsjqian/Aria) · [AriaTools](https://github.com/dqsjqian/AriaTools) · [AriaAgent](https://github.com/dqsjqian/AriaAgent) · [AriaRead](https://github.com/dqsjqian/AriaRead) · [AriaTutorial](https://github.com/dqsjqian/AriaTutorial) · [Mira](https://github.com/dqsjqian/Mira) · [SQLiteCpp](https://github.com/dqsjqian/SQLiteCpp) · [Tutti](https://github.com/dqsjqian/Tutti) · [ForgeDeps](https://github.com/dqsjqian/ForgeDeps) · [ForgeRecipes](https://github.com/dqsjqian/ForgeRecipes)

## 安装

- **本地使用**：把整个目录放进 WorkBuddy 本地专家目录（`<WorkBuddy 配置目录>/plugins/marketplaces/my-experts/plugins/`），由专家管理流程完成校验与注册后，即可在【专家中心 → 我的专家】中使用。
- **开放平台**：把本仓库打成 zip（顶层目录为 `aria-framework-expert/`）后在 WorkBuddy 开放平台提交。

```bash
git clone https://github.com/dqsjqian/aria-framework-expert.git
zip -r aria-framework-expert.zip aria-framework-expert -x "aria-framework-expert/.git/*" "aria-framework-expert/.gitignore"
```

专家包格式遵循 WorkBuddy 专家开发规范：`plugin.json` 为配置文件（位于规范要求的隐藏配置目录），`agents/` 为专家提示词，`skills/c-cpp-architecture-knowledge/` 为知识库，`avatars/` 为头像。专家的内部标识沿用 `aria-framework-expert`，以保证已安装用户可以原地升级。

## 版本记录

- **3.0.0**（2026-10-29）：升级为 C/C++ 架构顾问；新增 C 语言支持；知识库升级到 Aria 3.2.6，并新增 C、C++26 与架构、Aria 系列全景、选型地图、Mira、SQLiteCpp 六份文档。
- **2.0.1**：Aria 2.0 框架专家。

## 许可证

[MIT](LICENSE) © 2026 dqsjqian
