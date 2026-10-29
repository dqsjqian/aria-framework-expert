# Aria 框架总览（定位 · 架构 · 线程 · ABI · 构建 · 依赖 · 迁移）

> 基准版本：Aria 3.2.6（tag v3.2.6，commit 85d3d38，2026-10-29 与 GitHub 远端 main 核对）
> 仓库：https://github.com/dqsjqian/Aria（MIT）。`ARIA_ABI_VERSION` 仍为 **2**（`modules/abi/include/aria/abi/version.hpp`；`CHANGELOG.md 3.2.0/3.2.1` 明确 "retain ABI version 2"）。

## 一句话定位

Aria 把**响应式引擎和绑定层从 UI 框架里拆出来**，做成不绑定任何 UI 工具包的 **C++23** MVVM 库。ViewModel 是普通 C++ 类——不继承框架基类（可选 `aria::binding::ViewModel` 获得生命周期管理）、不需要宏、不需要代码生成器；UI 通过 `IViewAdapter` 接入。

**适合**：已有 C++ 业务内核、要多端复用同一份逻辑、各端保留原生 UI 的项目。
**不适合**：「一份代码连界面一起跨端」——那是 Flutter / Qt Quick 的领域。

## 关键取舍（选型前必读）

| 取舍 | 说明 |
|---|---|
| **C++23 是最低标准**（3.0 起） | 所有模块与**消费者**都必须 C++23；`CMAKE_CXX_STANDARD` 默认 23，低于 23 配置期 `FATAL_ERROR`（`CMakeLists.txt`）；导出 target 自动向消费者传递 C++23 要求（`CHANGELOG.md 3.0.1`） |
| 不提供控件 | 控件、布局、动画归 UI 工具包；Aria 只负责状态↔界面数据流 |
| 模板层仅源码兼容 | `aria-abi` / `aria-runtime` / `aria-binding` 主版本内 ABI 稳定（ABI 2）；`Property<T>` 等模板跨版本需重编 |
| ABI 一致性要求 | 跨动态库共享响应式图要求一致的编译器、标准库、构建选项与主版本 |
| 适配器要自己补 | 仓内 Qt6 / AppKit / UIKit / JNI / HTTP 五个；新工具包实现 `IViewAdapter`（cookbook 08） |
| 构建依赖 Python | 3.2.1 起依赖由 ForgeDeps + ForgeRecipes 解析，需 Python 3.14+（core-only 且关测试/HTTP 时不加载） |

## 跨平台映射（`README.md`）

| 平台 | UI 宿主 | 适配器 | 状态 |
|---|---|---|---|
| Windows | Qt6 | `aria-qt6` | ✅ MSYS2 UCRT64 + MSVC（VS2026） |
| macOS | AppKit / Qt6 | `aria-qt6` / `aria-appkit` | ✅ |
| Linux | Qt6 | `aria-qt6` | ✅（安装版共享库 `$ORIGIN` rpath） |
| iOS | UIKit | `aria-uikit` | ✅ |
| Android | Compose / View | `aria-jni` | ✅（README 写 NDK r30） |
| Web（服务端驱动） | 浏览器 HTML/JS | `aria-http` | ✅ REST + SSE，协议 2，传输层 3.0 起为 Mira |
| Web（浏览器内 C++） | WASM | — | 未实现；`-DARIA_BUILD_WASM=ON` 直接报错 |

## 模块架构（10 个模块）

```
应用层
  └─ 适配器（Qt6 / AppKit / UIKit / JNI / HTTP，按需启用）
       └─ aria-binding (SHARED)   BindingEngine + IViewAdapter
            ├─ aria-runtime (SHARED)  EventBus / Container / Dispatcher / Logger
            └─ aria-async (仅头文件)  Task<T> / Scheduler / Executor / schedule_on
                 └─ aria-core (仅头文件)  Property / Computed / Command / ObservableList / Validator / Subscription
                      └─ aria_abi (默认 SHARED，可 STATIC)  信号/槽、共享响应式图、诊断存储、调度器基类、版本信息
```

| 模块 | CMake target | 类型 | ABI |
|---|---|---|---|
| abi | `aria::abi` | 默认 SHARED（跟随 `ARIA_BUILD_SHARED`） | 主版本内稳定（ABI 2） |
| core | `aria::core` | 仅头文件，自动链 `aria::abi` | 仅源码兼容 |
| async | `aria::async` | 仅头文件 | 仅源码兼容 |
| runtime | `aria::runtime` | SHARED，单例集中一个 dylib | 主版本内稳定 |
| binding | `aria::binding` | SHARED | 主版本内稳定 |
| 适配器 | `aria::qt6` / `aria::http` / `aria::adapters::{appkit,uikit,jni}` | SHARED/STATIC | 源码兼容 |

安装包另提供聚合 target **`aria::aria`**（= core + async + runtime + binding，`cmake/ariaConfig.cmake.in`）。

**ABI 2 架构要点（2.0 引入，3.x 沿用）**：响应式图、诊断、调度器由 `aria_abi` 跨动态库共享（一进程一图）；core-only 消费者无需链 runtime；共享部署要带上 `aria_abi`；独立链接的多份静态副本**不构成共享图**。3.2.2 起静态构建统一由 `ARIA_STATIC_LINKAGE` 清空所有模块导出宏（修 MSVC C4275 /WX 失败，`CHANGELOG.md 3.2.2`）。

## 线程模型（分治）

**1. 响应式图（Property/Computed/Effect）——单线程**
- 跑在唯一「图线程」（通常 UI 线程）；`get()/set()/mutate()` 断言在图线程（L-1）。
- 热路径无锁；`reactive::batch` 在最外层边界合并为一次 flush。
- 后台线程更新 Property 必须编组：`dispatcher->post([...]{ prop = v; })` 或 `co_await schedule_on(ui)`（L-2）。

**2. 信号类（ObservableList / EventBus / Command 的 CanExecuteSignal）——线程安全**
- `abi::SignalErased` 内部短暂加锁，回调在**不持锁**状态下触发。
- `release()` 跳过尚未启动的回调（含当前 fanout 后续项），已在执行的可跑完。
- `ObservableList<T>` 内部 `shared_mutex`；Command 的 **can_execute 谓词跑在图线程**。

## 内存所有权

- 图内依赖是侵入式双向链表边，非引用计数。
- `Subscription` 是 **move-only RAII 所有权柄**；断开经 `weak_ptr`，发布者已死则静默 no-op。
- `main_dispatcher()` 返回拥有型 `shared_ptr<IDispatcher>` 快照；`SimpleDispatcher::pump()/run_one()` 只能在创建线程跑。
- `BindingEngine`：view 侧由引擎管（per-view 订阅桶）；source 侧由调用方管（VM 必须比绑定活得久）。

## ABI 稳定边界速查（ABI 2）

- `ARIA_ABI_VERSION` = 2；ABI 1 与 ABI 2 的库**不得载入同一对象图**。
- **2.x → 3.x 虽同为 ABI 2，仍需全量重编**：3.0 把语言基线提到 C++23、HTTP 后端换成 Mira，且 `ariaConfigVersion.cmake` 用 `SameMajorVersion` 兼容策略，`find_package(aria 2.x)` 匹配不到 3.x 包（`CMakeLists.txt`）。「跨 2.x/3.x 混载二进制」仓库未给出支持声明——**需查证**，按不支持处理。
- Mira 只承诺源码兼容、不承诺二进制 ABI：换 Mira 版本必须重编（`CHANGELOG.md 3.1.0`）。
- 运行时自检：`aria::abi::abi_matches_headers()`、`runtime_abi_version()`、`runtime_version_string()`。
- 注意：`version.hpp` 中 `ARIA_VERSION_STRING` 在 v3.2.6 仍是 `"3.2.0"`（项目版本 3.2.6），**勿用版本字符串判定补丁版本**。
- 类型擦除接口 `IProperty`（get_any/set_any/subscribe_any/type）供插件跨边界观察/修改。

## 环境要求与工具链

**当前写法（以 `README.md` 为准，来源 `CHANGELOG.md 3.2.2`「Documented toolchain floor follows CI」）**：
- CMake **≥ 3.21**（3.1.0 起；VS2026 生成器需 CMake ≥ 4.2）
- GCC ≥ 16（Windows 可 MSYS2 UCRT64）/ Clang ≥ 23 / Xcode 27+ 的 AppleClang / **MSVC v145（VS2026）**；Android NDK r30
- 可选 Qt6 ≥ 6.4；构建依赖需 Python ≥ 3.14
- CI 实际：GCC 16、LLVM 23、Xcode 27 镜像、windows-2025-vs2026、NDK 30.0.16248370（`.github/workflows/ci.yml`）

**历史写法**：3.0.0 写的是 GCC 14 / Clang 19 / AppleClang 21 / MSVC v143（`CHANGELOG.md 3.0.0`、`docs/migration-3.0.md` 至今仍是此表）——这是**语言能力下限**的说明；「文档承诺/CI 验证」下限已上调为上面那组。回答用户时优先给 README 写法，并说明旧表来源。

## 构建

```bash
git clone https://github.com/dqsjqian/Aria.git && cd Aria
python -m pip install -r requirements-build.txt          # ForgeDeps + ForgeRecipes（按 tag 固定）
python -m forgedeps init /path/to/shared-dependencies    # 初始化共享依赖工作区
python3 scripts/build.py --test                          # 统一入口：配置+编译+CTest
# 或直接 CMake：
cmake -B build/flavors/release -DCMAKE_BUILD_TYPE=Release
cmake --build build/flavors/release -j
ctest --test-dir build/flavors/release --no-tests=error --output-on-failure
```

- **脚本收敛（3.2.1，24→7）**：`build.py`（唯一构建入口：`--platform qt|web|ios|android`、`--toolchain msvc|mingw`、`--config`、`--cmake-arg=`、`--dry-run`、`--offline`、`--build-dir`、`--arch`、`--ndk`）、`bench.py`（profile/check/scenarios/compare/validate/verify-release）、`docs.py`（filter/html prepare|check/api-check/doxygen 安装）、`dependencies.py`（resolve/update）、`tidy-gate.sh`、`pick-ios-simulator.py`、`run-android-tests.py`（`scripts/README.md`）。**`build.sh` / `build.ps1` / `build-msvc.ps1` / `update_dependencies.py` / `check-bench.sh` 均已删除**（`scripts/` 目录已核实）。
- 常用组合：`build.py --config Debug --cmake-arg=-DARIA_ENABLE_ASAN=ON --cmake-arg=-DARIA_ENABLE_UBSAN=ON --test`；TSan 同理；Windows `--toolchain msvc`（默认）/ `--toolchain mingw`。
- `build/` 是构建树**容器**，直接配置进 `build/` 或源码目录会被 guard 拒绝（逃生：`-DARIA_ALLOW_BUILD_ROOT=ON`，不推荐）；`build.py` 默认树在 `build/unified/<suffix>`。

**CMake 选项**：`ARIA_BUILD_TESTS`(ON) `ARIA_BUILD_BENCHMARK`(ON) `ARIA_BUILD_DOCS`(OFF) `ARIA_BUILD_SHARED`(ON) `ARIA_BUILD_QT6/APPKIT/UIKIT/JNI/HTTP`(OFF) `ARIA_HTTP_ENABLE_TLS`(**ON**) `ARIA_ENABLE_ASAN/UBSAN/TSAN`(OFF)。

## 依赖模型（3.0 起重构，3.2.1 再换引擎）

- **无 vendored 源码、无 git submodule、无 CPM**；`third_party/` 与 OpenSSL 子模块已删除（3.0.0）。
- 唯一锁文件 `dependencies.json`（schema 2）：每项记录 source、请求策略（缺省 = latest stable，首次解析后锁定）、完整 commit 与 **SHA256**，`request_hash` 绑定来源与策略。当前锁定：**Mira 1.2.0**（显式指定）、nlohmann/json 3.12.0、doctest 2.5.3、OpenSSL 4.0.3（TLS 构建）。
- **3.2.1 起**由 ForgeDeps 负责解析/下载/校验/共享缓存，ForgeRecipes 提供 JSON/doctest/OpenSSL 构建配方，构建一次后在共享工作区复用；`build.py` 全程持有依赖图**读取租约**。版本：CHANGELOG 3.2.1 写 ForgeDeps 0.4.4 / ForgeRecipes 0.1.4，当前 `requirements-build.txt` 已固定 **ForgeDeps v0.4.10 / ForgeRecipes v0.1.8**（以文件为准）。
- 更新锁：`python3 scripts/dependencies.py update --file dependencies.json [--only mira] [--version json=3.12.0]`；离线校验 `resolve --offline`。
- CMake 覆盖：`-DARIA_DEP_{JSON,DOCTEST,MIRA,OPENSSL}_VERSION=`（写入构建目录的生效 lock，不改源文件）；`-DARIA_PIN_<NAME>_SOURCE_DIR=` 本地源码；`-DARIA_DEPENDENCIES_OFFLINE=ON`；`FORGEDEPS_HOME` 选工作区；`-DFORGEDEPS_PYTHON_EXECUTABLE=` 指定解释器；Qt 精确版本 `-DARIA_DEP_QT_VERSION=6.10.0`。详见 `docs/dependencies.md`、`docs/dependency-updates.zh.md`。

## 接入自己项目

- **方式 A（推荐）**：`cmake --install` 后
  ```cmake
  find_package(aria 3.0 CONFIG REQUIRED COMPONENTS core)   # 按需加 async/runtime/binding/qt6/http/appkit/uikit/jni
  target_link_libraries(app PRIVATE aria::core)            # 或 aria::aria 聚合
  ```
  只请求 `core` 时不会去解析 Qt/Mira/OpenSSL（适配器导出集已拆分，3.0.1）。**静态 HTTP** SDK 需消费端能 `find_package(Mira <精确版本>)`；共享构建 Mira 已并入 `aria_http` dylib。捆绑 OpenSSL 的 TLS SDK 遇到宿主已导入的其他 OpenSSL target 会在配置期拒绝 http 组件；3.2.6 起包配置为 Windows 的 `OpenSSL::Crypto` 补链 `ws2_32 gdi32 advapi32 crypt32 user32`；3.2.3~3.2.5 修复宿主已 `find_package(Qt6)` 时的重复导入问题。
- **方式 B（源码）**：`add_subdirectory("${ARIA_SOURCE_DIR}" "${CMAKE_BINARY_DIR}/aria" EXCLUDE_FROM_ALL)`，Aria 独立 clone 到项目外（README 已不再推荐 `third_party/aria`）。
- **测试辅助**：3.2.1 起安装 `aria/binding/testing/fake_adapter.hpp`（`aria::binding::testing::FakeView` / `FakeAdapter`），下游测试无需引用 Aria 测试源码。

## 性能基线与门禁

- 公开对比表仍是 **1.2.1 → 2.0**（Apple M3 Pro / AppleClang 21 / **C++20** Release，`aa039f6` vs `2f22e17`，五轮交替配对取中位）：Property set 无观察者 20.5→13.2 ns、单观察者 98.9→35.0 ns、五级 Computed 链 570.8→229.0 ns、batch 内 10 次 set 250.6→102.5 ns、List 随机 move 307→31 μs、FilteredList 尾追加 11.18→0.23 μs、SortedList 随机键追加 8.38→17.28 μs（+106%，稳定排序契约代价）、AsyncCommand 往返 ≈6.25 μs 持平。**3.x/C++23 下没有发布同口径全表**——需查证（`docs/reference/performance.md`）。
- **门禁两套独立结果**（3.2.0 建立）：① 历史绝对预算 `benchmark/thresholds.json`，仅物理 M3 Pro 可校准，其他主机返回 3（unavailable）；② 同机回归：候选 vs 固定发布版 `v3.1.1`，`paired-median-scenario-v3.1` 设计，64 宏块、R64/P512、A-A 与 20% 真实延迟对照先通过，10%（1.10）限值；3.2.1 起（protocol v3.1）**12 个 batch-mean 统计量决策**，batch-P99 仅记为观察。退出码 0 过 / 1 回归 / 2 数据错误 / 3 不确定或不可用。入口 `scripts/bench.py check|validate`。

## 1.x / 2.x → 3.x 迁移要点

**2.x → 3.0（`docs/migration-3.0.md`）**：
1. 工程改 C++23（或删掉 `CMAKE_CXX_STANDARD` 覆盖），升级到满足下限的工具链；CMake ≥ 3.21。
2. `find_package(aria 3.0 ...)`；全量重编 app + 插件 + 适配器。
3. HTTP：`native_server()` 删除 → 自定义路由改 `register_command()`；SSE 不再占 worker，`max_sse_clients` 为唯一准入上限（0 = 64）；地址必须数字（`localhost`→IPv4 回环）。线协议仍是协议 2，浏览器/SDK 无需改。
4. 依赖：删除对 `third_party/`、子模块的假设；首次配置需联网（或预热共享工作区）；装 `requirements-build.txt` + `forgedeps init`。
5. 脚本：`build.sh/build.ps1/build-msvc.ps1` → `python3 scripts/build.py`。

**1.2.x → 2.0（`docs/migration-2.0.md`，精简）**：ABI 1/2 不混载并全量重编；`ListChange::item` 变 `shared_ptr<T>`、Reset 带非空 `snapshot`、只用事件载荷；`Subscription::detach()` → `release()` 且 move-only；`main_dispatcher()` 返回 `shared_ptr` 快照；HTTP 协议 2（BigInt）与 SDK 显式通道 `subscribe(id, cb, "enabled")`。

## 文档地图

| 想查什么 | 去哪 |
|---|---|
| 入门 | `docs/getting-started.md` |
| 九章指南 | `docs/guide/`（reactive-core/viewmodel/async/collections/validation/binding/navigation/diagnostics-guide/adapters） |
| 六份契约 | `docs/reference/`（api-style `S-N` / lifecycle `L-N` / error-model `E-N` / list-diff（文档内编号 `D-N`，README 称 `LD-N`）/ diagnostics `D-N` / performance `PERF-N`） |
| 迁移 | `docs/migration-3.0.md`（2.x→3.0）、`docs/migration-2.0.md`（1.2.x→2.0）、`docs/migration-guide.md`（Qt/Rx/WPF 概念映射） |
| 依赖 / 配方 | `docs/dependencies.md`、`docs/dependency-updates.zh.md` / `docs/cookbook/`（8 个） |
| 路线图 / 变更 | `docs/ROADMAP.md`（SwiftUI/WASM 需具体消费者触发）/ `CHANGELOG.md` |
| API 参考 | https://dqsjqian.github.io/Aria/（主分支自动发布）；本地 `-DARIA_BUILD_DOCS=ON` → `aria_docs` target |
| 教程 / 示例 | AriaTutorial（20 章）、AriaTools（旗舰四端示例） |
