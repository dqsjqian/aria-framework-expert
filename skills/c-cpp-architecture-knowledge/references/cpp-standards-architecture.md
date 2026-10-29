# C++ 标准演进（C++11 → C++26）与架构设计方法

> 基准：2026-10-29。C++26 已于 2026-03-28 在伦敦 Croydon 会议完成技术工作（随后走 ISO 批准投票）。「本机实测」是在 macOS arm64 上用 Homebrew GCC 16.2.0、Homebrew Clang 23.1.0（libc++）、Apple Clang 21.0 编译并运行最小样例的结果。

## 一、每代一句话

| 标准 | 核心变化 |
|---|---|
| C++11 | 移动语义、lambda、`auto`、智能指针、`constexpr`、内存模型 + `<thread>/<atomic>`、可变参模板 |
| C++14 | 泛型 lambda、`make_unique`、放宽 `constexpr` |
| C++17 | 结构化绑定、`if constexpr`、`optional/variant/any/string_view`、折叠表达式、并行算法、`filesystem` |
| C++20 | concepts、ranges、协程（机制）、modules、`<=>`、`span`、`format`、`jthread/stop_token`、`atomic::wait` |
| C++23 | `std::expected`、`print/println`、deducing this、`mdspan`、`flat_map`、`generator`、`move_only_function`、`import std;`、`ranges::to` |
| C++26 | 静态反射、契约、`std::execution`、内存安全加固等，见第二节 |

## 二、C++26 特性清单与编译器现状

| 特性（提案） | 一句话 | GCC 16.2 实测 | Clang 23.1 实测 | Apple Clang 21 |
|---|---|---|---|---|
| 静态反射 P2996（`^^T`、`[: r :]`、`std::meta`） | 编译期自省与代码生成，替代宏/代码生成器 | ✅ 需 `-freflection` | ❌（上游无，仅 Bloomberg clang-p2996 fork） | ❌ |
| 展开语句 P1306（`template for`） | 编译期遍历元组/反射结果 | ✅（无需 `-freflection`） | ✅（遍历 `std::tuple` 实测通过） | ❌ |
| 契约 P2900（`pre`/`post`/`contract_assert`） | 语言级前后置条件 | ✅ 实验；macOS 上需 `-fcontracts` 才能链接，违约打印诊断后终止 | ❌ | ❌ |
| `std::execution` P2300 | sender/receiver 统一异步模型 | ❌（libstdc++ 16 无 `just`） | ❌ | ❌ |
| `std::inplace_vector` P0843 | 固定容量、无堆分配的 vector | ✅ | ❌ | ❌ |
| `std::simd` P1928 | 可移植 SIMD | GCC 16 发布说明列为改进中，但本机 `std::simd::vec` 不可用（需查证接口名） | ❌ | ❌ |
| `std::hive` P0447 | 稳定地址的桶容器 | ❌ | ❌ | ❌ |
| `std::linalg` P1673 | BLAS 风格线性代数 | ❌ | ❌ | ❌ |
| `#embed` P1967 | 编译期嵌入二进制 | ✅ | ✅ | ✅ |
| 包索引 P2662（`args...[0]`） | 直接取参数包第 N 个 | ✅ | ✅ | ✅ |
| 未初始化读取为“错误行为” P2795 | 不再是 UB，可诊断；`[[indeterminate]]` 显式退回 | GCC 16 发布说明列为已实现 | ❌（`[[indeterminate]]` 报未知属性） | ❌（同左） |
| 标准库加固 P3471 | 容器越界等前置条件检查 | libstdc++ `-D_GLIBCXX_ASSERTIONS` 越界即中止（实测） | libc++ `_LIBCPP_HARDENING_MODE_FAST` 越界即陷阱（实测） | — |
| 其他 | `constexpr` 异常、`std::optional<T&>`、`function_ref`、`copyable_function`、`indirect`/`polymorphic`、`<debugging>`、`= delete("原因")`、`_` 占位符、饱和算术 `add_sat` 等 | 多数已在 GCC 16 | 部分 | 少 |

不在 C++26 里的（防止误答）：模式匹配（P2688 未进）、Profiles（转向 C++29）、**平凡可重定位 P2786（2025-02 Hagenberg 进草案，2025-11 Kona 被移除，推迟到 C++29）**、网络库（Networking TS 未进）。

默认语言版本（本机实测 `__cplusplus`）：GCC 16 默认 `202002L`（gnu++20，GCC 16 起从 gnu++17 改为 gnu++20）；Clang 23.1 默认 `201703L`；Apple Clang 21 的 `c++` 默认 `201402L`。**永远显式指定标准**。

落地建议：
- 想今天写 C++26 → GCC 16.x；反射在 Clang 侧只能用 Bloomberg fork；MSVC 还在补 C++23，**需要 MSVC 的项目按 C++23 设计**。
- 特性检测宏在两大标准库里都有不可靠案例，**用 `check_cxx_source_compiles` 实测编译**比查宏可靠。
- 契约当前更适合做「debug/测试构建的可执行文档」，发布构建的求值语义（ignore/observe/enforce/quick_enforce）要与团队约定。
- `std::execution` 生产可用实现：NVIDIA stdexec（参考实现）；Aria 的 `Task/AsyncCommand` 与 Mira 的协程走的是自有执行器，不依赖 `std::execution`。

## 三、Aria 系列的 C++ 基线（推荐时必须说清）

| 项目 | 语言标准 | 工具链下限（以仓库当前文档为准） |
|---|---|---|
| Aria 3.x | C++23 最低（3.0.0 起；`CMAKE_CXX_STANDARD` 默认 23，配置期拒绝更低） | 当前 README「环境要求」（3.2.2 起跟随 CI）：GCC ≥ 16、Clang ≥ 23、Xcode 27+ 的 AppleClang、MSVC v145 / VS2026（需 CMake ≥ 4.2，其余 CMake ≥ 3.21）、NDK r30。`docs/migration-3.0.md` 仍写 3.0.0 时的历史下限 GCC 14 / Clang 19 / AppleClang 21 / MSVC v143——**以 README/CI 为准**，旧下限只代表 3.0.0 当时能编过 |
| Mira 1.2.x | C++23 | CI 跑最新稳定工具链（GCC 16、Clang 23、VS 2026、Xcode 27、NDK r30） |
| SQLiteCpp 1.2.x | C++23 | 见 `sqlitecpp-storage.md` |
| Tutti 2.x | C++23，header-only | CI：GCC 14.2/15.2、Clang 19.1/20.1 等（README） |

## 四、架构设计方法（顾问的主战场）

### 4.1 分层与依赖方向

```
View（Qt / UIKit / Compose / Web 薄客户端）      ← 只做绑定与展示
   │ 绑定（BindingEngine / IViewAdapter）
ViewModel（纯 C++：Property / Computed / Command） ← 全部表现逻辑
   │
Model / Service（业务规则、仓储接口）            ← 纯 C++，可单测
   │ 接口（端口）
Infra（Mira 网络、SQLiteCpp 存储、文件、日志）   ← 适配器，可替换
```
- 依赖只能向下、向内；Infra 通过接口注入（构造注入或 ServiceHub/DI 容器），**禁止 Service Locator 全局单例**。
- 线程边界写进架构图：图线程（UI 线程）、IO 线程（事件循环）、计算线程池；跨线程只走调度器编组。
- AriaTools 是这套分层的完整样板：17 个模块各自一个静态库、EventBus 跨模块通信、MountRegistry 拓展点、Qt/iOS/Android/Web 四个 View 壳共享一份 ViewModel。

### 4.2 UI 架构模式选型

| 模式 | 适合 | C++ 落地 |
|---|---|---|
| MVVM（数据绑定） | 多端共享表现逻辑、状态多、表单多 | **Aria**（纯 C++ VM + 平台原生 View）；Qt Quick/QML 的属性绑定 |
| MVP / MVC | 绑定能力弱的 UI 工具包、遗留代码 | Qt Widgets 手写 presenter |
| 单向数据流（Redux/MVI/Elm） | 状态可回放、调试要求高 | 不可变 state + reducer；可与 Aria `Property` 结合 |
| 立即模式 GUI | 工具、调试面板、游戏内编辑器 | Dear ImGui |
| ECS | 游戏/仿真，大量同构实体 | EnTT、flecs |

### 4.3 二进制边界与 ABI

| 手段 | 何时用 |
|---|---|
| C ABI 外壳（`extern "C"` + 不透明句柄） | 跨语言、跨编译器、插件系统的最稳边界 |
| Pimpl | 稳定类布局、缩短编译依赖 |
| 类型擦除接口（纯虚接口 + 工厂） | 动态库间共享对象；Aria 的 `IProperty`/`IViewAdapter` 即此类 |
| 版本化 inline namespace | 同进程多版本共存、ABI 断代 |
| 单独的 ABI 基础库 | Aria 2.0 起的 `aria_abi`：跨动态库共享响应式图/诊断/调度器；ABI 主版本不同不得混载 |
- 模板代码只能保证源码兼容，不能跨 ABI 边界暴露 STL 容器给不同编译器/不同 CRT 的模块。
- Windows：一侧分配另一侧释放、不同 CRT（/MD vs /MT、MSVC vs MinGW）混用是头号事故源。

### 4.4 错误处理策略

| 方式 | 适用 |
|---|---|
| 异常 | 构造失败、真正的异常路径；库边界要写清异常保证 |
| `std::expected<T,E>`（C++23） | 可预期失败（解析、IO、业务校验），热路径 |
| 错误码 + `std::error_code` | 系统调用/网络层，与 C API 对接 |
| 契约（C++26） | 程序员错误（前置条件违背），不是业务错误 |
| 终止 | 不变量破坏、内存损坏 |
规则：一层只选一种主模型；异常不跨 C 边界、不跨线程（用 `exception_ptr` 搬运）；协程里 `co_await` 的异常要在 `Task` 中传播或转成 `expected`。

### 4.5 并发与异步

| 层次 | 推荐 |
|---|---|
| UI 状态 | 单线程响应式图（Aria），后台结果经 `Dispatcher`/`schedule_on` 回图线程 |
| 业务异步 | 协程 + 结构化并发（Aria `Task`/`AsyncCommand`/`ViewModelScope`；业界 cppcoro 思想、`std::execution`/stdexec、Folly coro） |
| 网络 IO | 单事件循环协程（**Mira**；业界 Boost.Asio、libuv） |
| CPU 计算 | 线程池（**Tutti**：`std::jthread`+`move_only_function`+`expected`；业界 TBB、Taskflow、BS::thread_pool） |
铁律：取消要贯穿（stop_token / CancellationToken）；每个协程有明确的所有者作用域；不要在持锁时 `co_await`；回调和协程不要跨越对象生命周期。

### 4.6 构建与依赖

| 方案 | 特点 |
|---|---|
| CMake + Presets | 业界最常用的构建系统生成器；目标化（`target_link_libraries` + 导出 `Config.cmake`） |
| **ForgeDeps + ForgeRecipes**（Aria 系列在用） | Python 3.14+；锁文件 `dependencies.json`（版本+commit+SHA256）；共享工作区、按工具链/选项/源码内容复用产物；无 vendored 源码、无子模块 |
| vcpkg / Conan | 业界主流包管理，生态丰富 |
| FetchContent / CPM | 轻量，但每个项目重复构建；Aria 3.0 起已弃用 CPM 与子模块 |
| C++20 modules | GCC 16 仍需 `-fmodules`（实验）；CMake 3.28+ 支持；跨编译器生产使用仍要谨慎 |

### 4.7 质量闭环

- 测试：doctest（Aria 系列在用）、Catch2、GoogleTest；属性测试 RapidCheck；fuzz libFuzzer。
- 动态检查：ASan/UBSan/TSan 全进 CI；Aria 自身在 macOS 跑 TSan/fuzz。
- 性能：基准用固定样本、成对比较、A-A 对照，不用 best-of-N（Aria 3.2 的性能门禁协议即如此）；profiling 用 perf / Instruments / VTune / Tracy。
- 代码审查清单：所有权是否单一明确；生命周期是否被作用域表达；线程归属是否写明；错误模型是否一致；公共头是否最小化；是否有越界/溢出/未初始化；是否有测试覆盖失败路径。

## 五、查证来源

- 标准：eel.is/c++draft、open-std.org/jtc1/sc22/wg21、cppreference 编译器支持表
- 编译器：gcc.gnu.org/gcc-16/changes.html、clang.llvm.org/cxx_status.html、MSVC 文档
- 结论有分歧时：在用户的实际工具链上写最小样例实测
