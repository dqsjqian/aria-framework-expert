# 契约体系 / 错误模型 / 诊断 / 高频 FAQ 排查表

> 基准版本：Aria 3.2.6（tag v3.2.6，commit 85d3d38，2026-10-29 与 GitHub 远端 main 核对）

## 带编号契约文档体系

非平庸行为都钉在带编号的契约文档里，回答生命周期/线程/错误/diff 行为时**必须挂 ID**：

| 文档 | 前缀 | 范围 |
|---|---|---|
| `docs/reference/api-style.md` | `S-N` | 命名、命名空间、错误与异步风格 |
| `docs/reference/lifecycle.md` | `L-N` | 线程亲和（L-1/L-2）、调度策略（L-4/L-5）、订阅释放（L-10~L-15）、flush（L-20~L-22）、view 销毁（L-32/L-33）、VM/协程/AsyncCommand（L-34~L-38.1）、Container 拆除（L-40） |
| `docs/reference/error-model.md` | `E-N` | `aria::Error` / `ErrorKind` 分类学，何时 throw 何时 set |
| `docs/reference/list-diff-contract.md` | 文档内 **`D-N`**（README/index 称 `LD-N`） | 六种事件、确定性顺序、reconcile（D-14）、重入（D-20/21）、投影回调恢复（D-29，3.0.1 新增）、派生链（D-40/41）、一致性套件（D-50） |
| `docs/reference/diagnostics.md` | `D-N` | TraceEvent + TraceSink 协议、各子系统挂钩点（D-30~D-36，竞态仲裁 D-31.1） |
| `docs/reference/performance.md` | `PERF-N` | 复杂度上界、配对实测基线、门禁协议 |

⚠️ list-diff 与 diagnostics **两份文档都用 `D-N` 编号**，引用时写全文档名（如「list-diff D-14」「diagnostics D-22」），README 的 `LD-N` 只是别称。

迁移文档：`docs/migration-3.0.md`（2.x→3.0）、`docs/migration-2.0.md`（1.2.x→2.0）。`docs/cpp23-evaluation.md` 已标注为 2.0 历史评估。

**ABI 层（3.x 仍为 ABI 2）**：`ARIA_ABI_VERSION` = 2（3.2.0/3.2.1 changelog 明确保留）；`aria_abi` 编译型基础库（默认 SHARED）承载跨库共享响应式图/诊断/调度器；无 abi-dump 工具，CI 跑 `abi-smoke`（`cross_dylib_abi_smoke` 过真实 dylib 边界压 `IProperty<T>`）+ `tests/acceptance/cross_dylib_reactive`（跨库依赖跟踪）+ core-only 链接与安装包消费测试（`tests/package_core`、`tests/package_http`）；运行时 `aria::abi::abi_matches_headers()` 自检。RaceSlot 两阶段发布（CLAIM → WRITE→PUBLISH → RESUME）；`when_all` 父句柄原子交接。

## 错误模型速记

- `aria::Error` + `ErrorKind`；`ConversionError`、`OperationCancelled`、`TimeoutError`、`CircularDependencyError`（硬错误）。
- 回调边界统一上报 `aria::report_callback_failure`（L-31.6）：destroy hook、trace sink、派生列表投影回调异常都不逃逸。
- 转换失败：引擎跳过写回，旧值保持权威（L-32.5）。
- HTTP：形状/类型/范围错 400，未知 view/命令 404，非 JSON Content-Type 415，通知队列满或 SSE 超额 503；错误体 `{ "error": "…" }`。

## 诊断协议

`TraceEvent` / `TraceSink` / `GraphInspector` / `ScopedTraceSink`：未挂 sink 时只有一次原子存在性检查（D-24 / PERF-3），昂贵载荷先判 `has_trace_sink()`。排查「为什么没触发/触发两次」先想到它（`docs/guide/diagnostics-guide.md`）。追踪与节点 ID 跨动态库共享；sink 抛异常被报告但**不中断传播**（D-22）。

## 高频 FAQ 排查表

### 1. Property 变了，界面没刷新

1. **Subscription 丢弃了？** `label.bind(...);` 不接返回值 → 订阅当场断（`[[nodiscard]]`）。存成成员或 `adopt()`/`track()`。
2. **set 值和旧值相等？** 相等性门控静默 no-op（L-21）。确要强推用 `mutate()`。
3. **写在后台线程？** 图单线程（L-1）；SmartMarshal 管两个方向的派发，但写 Property 本身仍须在图线程（`schedule_on(ui)` 后再写）。
4. **view 已销毁？** on_destroy 后引擎静默跳过该 view（L-32）。
5. **双向绑了 Computed？** 双向只收 `Property`。
6. 都不是 → `TraceSink` 看事件流。

### 2. 去掉中间闪烁 / 多值只推一次

`aria::batch([&]{ ... });` 或 `BatchScope`。

### 3. 后台线程更新状态

```cpp
co_await aria::async::schedule_on(worker);   // 干重活
auto r = compute();
co_await aria::async::schedule_on(ui);       // 回图线程
my_property = r;
```
或 `dispatcher->post([...]{ prop = v; })`（L-2）。

### 4. CircularDependencyError

图不允许环。环上一处读取改 `untracked()` 或重构为单向数据流。

### 5. with_timeout 超时了但没「快速失败」

`OnTimeout::Cancel` + 无 token 工厂 = 事后观察。工厂接 `CancellationToken`，或用 `OnTimeout::Fail`（也杀不掉不检查 token 的内层）。

### 6. 列表刷新丢选择/滚动位置

`clear()+insert_range` 每次发 Reset。改 `reconcile(fresh, key_of)`（list-diff D-14）；Qt 下拉选项会变用 `bind_combo_box_selection`（稳定 ID）。

### 7. Effect stop 后想重启

不行。新建 Effect；或 `into_subscription()` 放 SubscriptionBag。

### 8. 订阅回调访问已析构对象崩溃

- 图订阅：断开经 weak 校验——查裸存的悬垂捕获。响应式 Subscription 必须在图线程析构（L-11）。
- view 侧交给引擎（bind_* / adopt）；VM 侧 `track()`；异步用 `ViewModelScope`（cancel_and_join，5s 上限）。

### 9. can_execute 没随依赖变化更新按钮

谓词读的必须是 Property/Computed，写入在图线程；谓词跑图线程。

### 10. 派生列表回调抛异常后视图「卡住」/ 突然收到 Reset

list-diff D-29（3.0.1）：投影回调异常被上报、保留旧投影；下一次成功事件会**先发 Reset 快照**再恢复增量——镜像端必须正确处理 Reset。

### 11. AsyncCommand 的可变捕获出现数据竞争

3.2.0 起命令只保留**一个** action 对象跨调用复用（L-37）；`Parallel` 或 `LatestOnly` 收尾期间可能多 worker 同时进入。自行加锁或把访问固定在单线程执行器。

## 2.x → 3.x 迁移 FAQ

### M1. 配置报 "Aria requires C++23 or later"

3.0 起 C++23 是所有模块与**消费者**的最低标准（`CMakeLists.txt`）。删除 `CMAKE_CXX_STANDARD 20` 覆盖或改 23；导入 target 也会把 C++23 传给你的 target。

### M2. 编译器版本够不够

当前文档下限：GCC 16+ / Clang 23+ / Xcode 27+ AppleClang / MSVC v145（VS2026）/ NDK r30（`README.md`，`CHANGELOG.md 3.2.2`「follows CI」）。`docs/migration-3.0.md` 仍列 GCC 14 / Clang 19 / AppleClang 21 / VS2022 v143（3.0.0 语言能力下限：GCC 13 协程 ICE、clang-18 下 libstdc++ `std::expected` 不可见）。低于 README 下限属**未验证**组合，出问题先升级工具链。VS2026 生成器需 CMake ≥ 4.2。

### M3. `find_package(aria 2.0 ...)` 找不到包 / CMake 版本报错

包版本兼容策略 `SameMajorVersion`：请求 `aria 2.x` 不匹配 3.x，改 `find_package(aria 3.0 CONFIG REQUIRED COMPONENTS ...)`（注意 `docs/getting-started.md` 的 Qt 小节仍写 `aria 2.0`，是文档遗留）。CMake 最低 **3.21**（3.1.0 起）。

### M4. 升级后加载崩溃 / 行为诡异

虽仍是 ABI 2，C++ 标准、Mira 传输与依赖都变了——**app、插件、适配器全部用 3.x 重编**；不要混用 2.x 与 3.x 二进制（仓库未声明支持，按不支持处理）。Mira 不承诺二进制 ABI（3.1.0）。ABI 1 与 ABI 2 绝不混载。启动自检 `abi_matches_headers()`；`runtime_version_string()` 在 v3.2.6 仍报 `"3.2.0"`（`version.hpp` 未随补丁版本更新），不能用来区分 3.2.x 补丁。

### M5. `HttpAdapter::native_server()` 不存在

3.0 删除（它暴露 cpp-httplib）。自定义路由改 `register_command(view_id, name, [](std::string_view args_json){ return std::string(...); })`。线协议仍是协议 2，浏览器与 SDK 不用改。

### M6. SSE 连接数 / worker 配置语义变了

3.0 起 SSE 挂在 Mira 事件循环上，**不占 worker**：`max_sse_clients` 是唯一准入上限（默认 64，0 = 64），旧的「workers − 1」上限已取消。`worker_threads` 仍是 0=自动（≥2）或显式 ≥2。⚠️ `docs/migration-guide.md` 末节仍写「SSE admission capped at workers minus one」——那是 2.0 语义，以 `docs/guide/adapters/http.md` 与 `CHANGELOG.md 3.0.0` 为准。

### M7. HTTP 启动失败：`localhost` / 主机名

绑定地址必须是数字 IP；`"localhost"` 映射到 IPv4 回环，其他主机名不支持。

### M8. POST 返回 415

3.0.1 起 state/click/command 必须带单一 `Content-Type: application/json`（允许大小写与 `charset` 参数）。自写客户端补头；官方 SDK 已处理。

### M9. TLS 相关

`ARIA_HTTP_ENABLE_TLS` 默认 ON（OpenSSL 3+，锁定 4.0.3）；纯 HTTP 开发 `-DARIA_HTTP_ENABLE_TLS=OFF`。`tls_ca_file` 强制客户端证书校验；无 TLS 1.0/1.1；WebSocket/HTTP2/HTTP3 关闭。消费捆绑 OpenSSL 的 SDK 时若宿主已导入别的 `OpenSSL::SSL`，http 组件配置期被拒（用 `OPTIONAL_COMPONENTS http` 保住 core，或用宿主 OpenSSL 重编 Aria）。

### M10. `third_party/` 和子模块没了 / 首次配置要联网

3.0 起无 vendored、无 submodule、无 CPM；依赖按 `dependencies.json`（SHA256 锁定）下载。3.2.1 起由 **ForgeDeps + ForgeRecipes** 管理：
```bash
python -m pip install -r requirements-build.txt      # Python ≥ 3.14
python -m forgedeps init /path/to/shared-dependencies
```
报 "Aria dependencies require Python 3.14+ and forgedeps" 即缺这一步（`cmake/ariaDependencies.cmake`）；可用 `-DFORGEDEPS_PYTHON_EXECUTABLE=` 指定解释器。离线：`-DARIA_DEPENDENCIES_OFFLINE=ON`（共享工作区须已有对应产物）、`-DARIA_PIN_<NAME>_SOURCE_DIR=`。旧 Aria-only 缓存可能需联网重新填充一次。core-only 且关闭测试与 HTTP 时不加载 ForgeDeps。

### M11. `scripts/build.sh` / `build.ps1` / `build-msvc.ps1` / `update_dependencies.py` 不见了

3.2.1 脚本收敛：统一 `python3 scripts/build.py [--test] [--config Debug] [--toolchain msvc|mingw] [--platform qt|web|ios|android --ndk ...] [--dry-run]`；依赖更新 `scripts/dependencies.py update`；基准 `scripts/bench.py`；文档 `scripts/docs.py`。

### M12. 静态 HTTP SDK 消费端找不到 Mira

静态 `aria_http` 需要消费端 `find_package(Mira <精确版本> COMPONENTS transport http [tls])`（`ariaConfig.cmake.in`，当前 Mira 1.2.0）；共享构建已把 Mira 并入 `aria_http`，无需额外处理。

### M13. Windows 链接错 / 配置错（3.2.x 补丁已修）

- 链接 OpenSSL 静态库报 `__imp_select`/`send` 等未定义 → 升级到 **3.2.6**（包配置为 `OpenSSL::Crypto` 补 `ws2_32 gdi32 advapi32 crypt32 user32`）。
- MSYS2 + Qt 6.10 报 "cannot create imported target EntryPointMinGW32" → 升级到 **≥3.2.5**（3.2.3 引入复用、3.2.4 修宏展开、3.2.5 报告组件已找到）。
- `ARIA_BUILD_SHARED=OFF` 时 MSVC C4275 + /WX 失败 → 升级到 **≥3.2.2**（`ARIA_STATIC_LINKAGE` 统一清空导出宏）。

### M14. `AsyncCommand(ui, worker, action, {}, policy)` 编译失败

当前签名 `AsyncCommand(ui, worker, action, policy = Parallel)`，删掉 `{}` 占位参数（`docs/guide/async.md` 已改）。

### M15. `when_all` 里混 `Task<void>`

3.0.1 起支持，void 子任务在结果 tuple 中占 `std::monostate` 槽。

### M16. 升级后 `function_ref` / 回调包装编译不过

3.0.1 起拒绝「返回引用但实际绑定到临时量」的 callable（防悬垂引用）。修正返回类型或返回值类型。

### M17. 下游单测想要假适配器

3.2.1 起 SDK 安装 `aria/binding/testing/fake_adapter.hpp`（`aria::binding::testing::FakeView` / `FakeAdapter`），配合 `adapter_conformance.hpp` 使用。

### M18. 性能门禁返回 3 / "unavailable"

`scripts/bench.py check`：非物理 M3 Pro 或主机繁忙 → 绝对预算结果为 3（不可用，不等于失败）。`bench.py validate` 同机配对：A-A 或 20% 延迟对照不合格 → 3（measurement-unavailable，候选未测）；区间跨越 1.10 → 3（inconclusive）；上界 > 1.10 → 1（回归）。3.2.1（protocol v3.1）起只有 12 个 batch-mean 统计量参与决策，batch-P99 为观察值（`docs/reference/performance.md`）。

## 1.x → 2.0 迁移 FAQ（精简保留）

- **ABI 1/2 不混载**，全量重编；共享部署带上 `aria_abi`。
- `Subscription::detach()` → `release()`；Subscription move-only，聚合用 `SubscriptionBag`（`bag += std::move(sub)`）。
- `ListChange<T>::item` 是 `shared_ptr<T>`，Reset 带 `snapshot`；只用事件载荷，别 `source.at(c.index)`；SortedList 等效键顺序变化会收到 Move。
- `main_dispatcher()` 返回 `shared_ptr<IDispatcher>` 快照；`SimpleDispatcher::pump()/run_one()` 限创建线程。
- HTTP 协议 2：超安全范围整数线上为十进制字符串、SDK 为 `BigInt`（`JSON.stringify` 需手动转）；SDK 通道 `subscribe("view", cb, "enabled")` 取代 `"view.enabled"`。
- Container 重注册同时替换 value 与 lifetime；工厂保留可变状态、在锁外执行。

## 其他

- **「找不到这个 API」**：先对照知识库五份参考；再查 `docs/guide/*`、在线 API 参考 https://dqsjqian.github.io/Aria/ 或本地 `-DARIA_BUILD_DOCS=ON`；完整应用接线看 AriaTools，系统学习看 AriaTutorial。
- **接仓里没有的 UI 工具包**：实现 `IViewAdapter`（cookbook 08），用 `adapter_conformance` 电池钉行为。
- **贡献约定**：`.clang-format` / `.clang-tidy`（`scripts/tidy-gate.sh` 对比 `clang-tidy-baseline.txt`，只拦新增债务）；所有变更过 `ctest --output-on-failure`；新功能在对应 `modules/*/tests/` 补测试；架构改动先开 Issue。
