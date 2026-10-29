# 绑定引擎与平台适配器速查（BindingEngine / IViewAdapter / ViewModel / Navigator / HTTP）

> 基准版本：Aria 3.2.6（tag v3.2.6，commit 85d3d38，2026-10-29 与 GitHub 远端 main 核对）
> Include：`#include "aria/binding/binding_engine.hpp"`。契约：lifecycle.md（L-N）、api-style.md（S-N）。指南：`docs/guide/binding.md`、`docs/guide/adapters/*.md`。
> 3.x 要点：绑定 API 与 2.0 一致；**HTTP 适配器 3.0 起改用 Mira 传输**（`native_server()` 删除、SSE 协程化、TLS/静态挂载由适配器自己实现）；3.2.1 起安装 `fake_adapter.hpp` 测试辅助。

## BindingEngine 构造与调度策略（L-4）

```cpp
aria::binding::BindingEngine engine{adapter};                    // Direct（默认）：单线程/测试
aria::binding::BindingEngine engine(adapter, dispatcher,
    aria::binding::BindingEngine::DispatchPolicy::SmartMarshal); // 生产推荐
```

| Policy | VM→View 路径 | 场景 |
|---|---|---|
| `Direct` | 同线程同步 | 单线程应用、测试 |
| `SmartMarshal` | `dispatcher.is_main_thread()` 时直发，否则 post | 生产应用 |
| `AlwaysPost` | 一律经 dispatcher post | 需要确定性顺序的测试 |

`adapter` 是 `std::shared_ptr<IViewAdapter>`——engine 触达 UI 工具包的唯一通道。

**View→VM 方向同样走策略**（2.0 起）：标量、转换文本、命令输入从 worker 线程（如 HTTP worker）进入时按 SmartMarshal/AlwaysPost 编组到图线程；借入文本 post 前先拷贝；排队回调在 view 绑定清除/替换/销毁、engine clear/析构时自动取消。同步 setter 回声仍被抑制。**绑定 setup/teardown 本身须在图拥有者线程**。

**3.0.1 起初始绑定加固**：先订阅源再执行初始投影/setter，并用 revision 计数丢弃过期的排队推送——投影或原生 setter 在初始绑定期间同步「规范化」源值时，后来的更新不会被旧值覆盖；初始投影抛异常会断开已建立的订阅。

## 绑定 API 总表

两条总规则：
- **单向（VM→View）接受任意只读响应式源**——`Property<T>` 或 `Computed<T>`。
- **双向只接受 `Property<T>`**。传 `Computed` 是编译错误。

### 标量绑定

| 方法 | 方向 | 源类型 | View 通道 |
|---|---|---|---|
| `bind_text` | 双向 | `Property<string>` | 文本输入（含 QComboBox 可编辑文本） |
| `bind_text_oneway` | 单向 | `Property/Computed<string>` | label 文本 |
| `bind_bool` / `bind_bool_oneway` | 双向/单向 | `bool` | 开关/复选 |
| `bind_int` / `bind_int_oneway` | 双向/单向 | `int` | spinbox/滑条/下拉索引 |
| `bind_int64` / `bind_uint64`（+oneway） | 双向/单向 | `int64_t` / `uint64_t` | 64 位数值 |
| `bind_float` / `bind_double`（+oneway） | 双向/单向 | `float` / `double` | 滑条/数值 |
| `bind_int_converted(prop, view, Converter<T,int>)` | 双向 | 任意 `T`（如 enum） | 整数控件；非连续 enum 值到下拉索引 |

### 文本投影与转换

| 方法 | 方向 | 说明 |
|---|---|---|
| `bind_text_projected(src, view, proj)` | 单向 | 任意 `T` → string；「带格式 label」首选 |
| `bind_optional_text(src, view, proj, empty_text)` | 单向 | `optional<T>`；适合 AsyncCommand 的 `last_result` |
| `bind_text_converted(prop, view, conv)` | 双向 | 完整 `Converter<T,string>` |
| `bind_text_converted_oneway` | 单向 | 只用 converter 的 to_view |

内置 Converter（`aria::binding::converters`）：`identity_string()` / `int_to_string()` / `double_to_string(precision=2)` / `bool_to_yes_no()`。View→Model 解析失败（`try_to_model` 返回 nullopt 或抛 `ConversionError`）时**跳过写回**，旧值保持权威（L-32.5）。有状态 converter 跨调用保留身份与状态。

### 状态 / 命令 / 生命周期

| 方法 | 说明 |
|---|---|
| `bind_visible(src, view)` / `bind_enabled(src, view)` | 单向 |
| `bind_command(cmd, view[, arg])` | 点击 → `execute(arg...)`；can_execute → enabled 自动驱动 |
| `bind_view_lifetime(view, fn)` | view 销毁（或引擎拆除）时回调一次；用于取消在途异步 |
| `adopt(view, sub)` | 把手写订阅托管进该 view 的订阅桶 |
| `clear()` | 释放全部绑定（析构自动调） |

没有合适绑定时：手写 `on_changed` + `adopt()`；复合 label 优先在 VM 上写 `Computed` 再 `bind_text_oneway`。

## 反馈环抑制与 view 生命周期

- **双向环**由引擎自动抑制（VM→View 推送时打 writing 标记，回写回调见标记即跳过）。
- **view 销毁安全**（L-32/L-33）：`IView::on_destroy` → 引擎清空该 view 订阅桶 → 之后变化静默跳过。
- Qt6 / AppKit / UIKit 的 view 拆除会让排队回调与列表通知失效；3.0.1 起原生适配器在 teardown 回调运行**之前**关闭注册，避免拆除期间新注册泄漏。

## ViewModel 基类（aria::binding::ViewModel）

```cpp
class GreetingVm : public aria::binding::ViewModel {
public:
    aria::Property<std::string> name{"World"};
    GreetingVm() { track(name.on_changed([this](const std::string&){ /* ... */ })); }
};
```

| API | 说明 |
|---|---|
| `activate()` / `deactivate()` | 幂等；`is_active` 在 batch 内翻转；deactivate 顺序：子先、hook、标志 |
| `on_activate()` / `on_deactivate()` | 覆写钩子 |
| `add_child(vm)` | 父子树；**3.0.1 起拒绝所有权成环与生命周期递归** |
| `track(sub)` | 存入内部 SubscriptionBag（逆序释放） |
| `add_destroy_hook(fn)` | 非订阅类清理；逆序执行；异常走 `aria::report_callback_failure`，不逃出析构 |
| `IViewModel` | 兼容别名，仍是具体基类 |

销毁顺序见 L-34；异步作用域 `ViewModelScope` 见 async 参考文档（L-35）。

## Navigator（aria::binding::Navigator，`navigation.hpp`）

| API | 说明 |
|---|---|
| `push(vm[, Presentation::Modal])` / `push<VM>(args...)` | 入栈；Modal 覆盖不打乱返回栈 |
| `pop()` / `dismiss_modal()` / `pop_to_root()` / `clear()` | pop_to_root 落到最深非 Modal 项 |
| `replace(vm)` / `replace<VM>(args...)` | 替换栈顶 |
| `current` / `depth` | 成员 `Property<shared_ptr<ViewModel>>` / `Property<size_t>`，可绑定 |
| `push_for_result<R>(vm[, kind])` | 返回 `shared_future<optional<R>>`；被调方 `dismiss_with(result)`，直接 pop 得 nullopt |
| `top_token()` | 栈顶 entry 的 CancellationToken，pop 时取消 |
| `register_route(pattern, factory)` / `route(path, RouteOptions{clear_stack, presentation})` | 深链路由（`{id}` 参数） |

（上表按 `navigation.hpp` 核对；2.0 版知识库写的 `replace_root` 在头文件中不存在。）

## 五平台适配器接线

三步永远一样：① 造平台适配器 ② 用它造 BindingEngine ③ 把控件和 Property 绑上。界面照旧用平台原生方式写。

### Qt6（Windows/macOS/Linux）

```cpp
auto adapter = std::make_shared<aria::adapters::qt6::QtAdapter>();
aria::binding::BindingEngine engine{adapter};
engine.bind_text(vm.name, adapter->view_for(name_edit));          // view_for：适配器持有包装，随控件/适配器失效
engine.bind_text_projected(vm.per_person, adapter->view_for(label),
    [](double v){ return std::format("¥{:.2f}", v); });
```

- `QtDispatcher`（事件循环桥）、`ObservableListModel<T>`（任意 ListSource → `QAbstractListModel`）。
- **下拉选择**：固定选项用 `bind_int` / `bind_int_converted`（保持 VM 为 enum 类型）；选项会变用 `bind_combo_box_selection(Property<optional<Key>>&, QComboBox&, key_from_data, role)` 按**稳定 ID** 选择，允许重复标签与空选；同一控件不要同时绑索引和 ID（`docs/guide/adapters/qt6.md`）。
- CMake：`find_package(aria 3.0 CONFIG REQUIRED COMPONENTS qt6)` + `aria::qt6`。3.2.3~3.2.5 修复宿主已导入 Qt6 时 `aria_find_qt6` 重复导入（MSYS2 Qt 6.10 `EntryPointMinGW32` 报错）。

### iOS/UIKit 与 macOS/AppKit（.mm 接线）

```objc++
auto adapter = std::make_shared<aria::adapters::uikit::UIKitAdapter>();
aria::binding::BindingEngine engine(adapter, ui_dispatcher,
    aria::binding::BindingEngine::DispatchPolicy::SmartMarshal);
auto label = std::make_shared<aria::adapters::uikit::UIKitView>(self.totalLabel);
engine.bind_text_projected(vm.per_person, *label, [](double v){ return std::format("¥{:.2f}", v); });
```

`UIKitView`/`AppKitView` 用 ARC 强持有原生 view；桥接对象经 `objc_setAssociatedObject` 锚在原生 view 上；析构时在原生 view 存活期间通知引擎。CMake target `aria::adapters::uikit` / `aria::adapters::appkit`（需 `APPLE` + 对应选项）。

### Android/JNI

```cpp
auto adapter = std::make_shared<aria::adapters::jni::JniAdapter>(env);  // Android UI 线程上
aria::binding::BindingEngine engine{adapter};
aria::adapters::jni::JniView total_view(env, total_text_view);
```

- Kotlin 监听器转发 `adapter->notify_text_changed(...)` / `notify_click(...)`；Compose 无可寻址 view → side-channel 形态。
- typed JniAdapter 通过完整 conformance 电池；JNI 显式管理线程附着、局部/全局引用、UTF-8/16 转换与 pending 异常。
- 构建：`python3 scripts/build.py --platform android --ndk <path>`（旧 `build.sh android` 已删）。NDK 版本 README 写 r30，jni 指南仍写「NDK 29+ (Clang 20)」——以 README/CI（NDK 30）为准。

### Web/HTTP（C++ 在服务端，浏览器薄客户端）

```cpp
aria::adapters::http::HttpAdapterConfig config; config.port = 9090;
auto http = std::make_shared<aria::adapters::http::HttpAdapter>(config);
auto dispatcher = std::make_shared<aria::runtime::SimpleDispatcher>();
aria::binding::BindingEngine engine(http, dispatcher,
    aria::binding::BindingEngine::DispatchPolicy::SmartMarshal);
auto& search = http->register_view("search_query", "text");
engine.bind_text(query, search);
if (!http->start()) return 1;   // 等监听就绪；失败或已运行返回 false
// 图线程上持续 dispatcher->pump()；actual_port() 停止后归零
```

**3.0 传输层换为 Mira**（`CHANGELOG.md 3.0.0`、`docs/guide/adapters/http.md`）：
- 一个事件循环线程（kqueue/epoll/IOCP）以协程服务全部连接；同步路由逻辑与用户回调跑在 worker 池。阻塞的自定义命令只拖慢 worker 池，不卡网络循环。
- **`native_server()` 已删除**；自定义路由改 `register_command(view_id, name, handler)`（handler 收 JSON 参数串、返回 JSON 串，调用时 view 必须存在）。
- **SSE 不再占 worker**：`max_sse_clients`（默认 64，0 = 64）是唯一准入上限，与 worker 数无关；超额 503。3.2.1：挂起的 SSE 等待者在唤醒/关闭前计入 Mira 事件循环，stop 不会在 SSE 协程挂起时提前完成。
- **TLS**：`ARIA_HTTP_ENABLE_TLS` 默认 **ON**（开发可 `=OFF`），映射到 Mira TLS（OpenSSL 3+，验证 4.0）；`tls_cert_file`/`tls_key_file`，`tls_ca_file` 强制客户端证书校验（无不安全绕过），`tls_min_version` `1.2`/`1.3`；TLS 1.0/1.1 不可用。WebSocket / HTTP/2 / HTTP/3 在本集成中**关闭**（3.1.0）。
- **静态挂载**：`static_root` 由适配器自己服务：默认 `index.html`、按扩展名 Content-Type、拒绝路径穿越；不支持 Range 和目录列表；无内置 dashboard、不自动挂载 SDK（3.0.1 起允许 `/aria_client.js` 等静态资产与 API 前缀并存）。
- 绑定地址必须是数字（`127.0.0.1`/`0.0.0.0`），`localhost` 映射 IPv4 回环；独占绑定经 Mira `ListenOptions::exclusive`。
- 依赖：当前锁定 Mira **1.2.0**（3.2.1；3.2.0 为 1.1.1，3.1.0 为 1.0.0）。共享构建 Mira 并入 `aria_http` dylib；静态构建消费端需 `find_package(Mira <精确版本>)`。

**配置项**（`http_config.hpp`）：`host`(127.0.0.1) `port`(9090，0=自动) `api_prefix`(/aria) `static_root` `worker_threads`(0=CPU 数且≥2；显式值须≥2) `heartbeat_sec`(25) `max_sse_clients`(64) `max_pending_sse_bytes`(4 MiB/客户端，溢出关流可重连拿快照) `max_pending_notifications`(1024，超出 503) `enable_cors`(false) 及 4 个 TLS 字段。

**端点**（前缀 `/aria`）：`GET health` / `GET views` / `GET state?view=X` / `POST state {view,field,value}` / `POST click {view}` / `POST command {view,command,args}` / `GET stream`（SSE）。view kind：`text bool int int64 uint64 float double click`。

**协议 2（2.0 引入，3.x 线协议未变）**：
- 64 位整数：JS 安全范围内走 number；超范围线上十进制字符串、SDK 里 `BigInt`；setter 接受 bigint/十进制字符串，**拒绝不安全的 number**。
- SSE 初始顺序：`hello` → 各 view 当前值（click 除外）→ visibility → enabled，然后才是后续更新。
- 校验：形状/类型/范围错误 400，未知 view/命令 404，错误体 `{ "error": "…" }`；**变更端点要求单一 `Content-Type: application/json`，否则 415**（3.0.1，可挡跨域简单表单 POST，但不替代鉴权）。
- 线程：state/click 订阅跑在 worker 线程且按进入注册表顺序串行；成功的 POST 只代表校验+派发完成，模型工作可能仍在排队。直接订阅适配器/自定义 handler 要自己编组到图线程。在 worker 回调里调用 `start()/stop()` 抛 `std::logic_error`。

**浏览器 SDK（`modules/adapters/http/web-sdk/aria_client.js`，零构建 ESM）**：`new AriaClient(location.origin)`、`onOpen/onError`、`subscribe(id, cb[, channel])`、`getState(id, "enabled")`、`setText/setInt64/...`、`connect()/close()`。通道显式（`"visibility"`/`"enabled"`），view ID 中的点号不拆分；unsubscribe 幂等。RFC 0001 仅为历史设计。

## 适配器共性事实

- 五个适配器同一 `IView`+`IViewAdapter` 契约，同一 `adapter_conformance` 电池（`aria/binding/testing/adapter_conformance.hpp`，12 用例/25 断言：text/bool/int/double 双向、click、command+can_execute→enabled、view 销毁安全、converter 反馈环抑制）。
- **测试辅助（3.2.1 起随 SDK 安装）**：`aria/binding/testing/fake_adapter.hpp` 提供 `aria::binding::testing::FakeView`（text/flag/integer/i64/u64/f32/number/visible/enabled 字段 + 每类型一个信号）与 `FakeAdapter`——下游 VM/绑定单测可直接用，无需引用 Aria 测试源码。
- `IView::~IView` 在原生句柄仍存活时 `fire_destroy_()`。
- int64→int 饱和截断（clamp），每 (label, 方向) 每进程最多一条 `Logger::warn`。
- 自写适配器：实现 `IViewAdapter`（可基于 `ViewAdapterBase`）+ 跑 conformance 电池，参考 cookbook 08；`platform_name` 与不支持控件的诊断见 L-39。
