# Mira 网络库（C++23 协程 · 传输为基 · 协议其上）

> 基准版本：Mira 1.2.1（tag v1.2.1，commit cd123e4，2026-10-29 与 GitHub 远端 main 核对；`git ls-remote` 显示 main 与 v1.2.1 均为 cd123e4）
> 仓库：https://github.com/dqsjqian/Mira（MIT，© 2026 Mira contributors）。版本唯一来源：根 `CMakeLists.txt` 中的 `project(Mira VERSION 1.2.1)`，由它生成 `mira/core/version.hpp`（`docs/RELEASES.md`）。

## 一句话定位与设计哲学

**独立的、协程原生的 C++23 网络库**：底层是传输核心，各协议跑在它上面，HTTP 只是其中一个协议族，Mira 本身不是 HTTP 框架（`docs/ARCHITECTURE.md` “What this library is”）。

| 抉择 | 内容（出处：README “为什么是 Mira”、ARCHITECTURE） |
|---|---|
| 完成式而非就绪式 | 公开 API 在所有平台都是完成式：`co_await read_some(buf)` 直接返回读取结果。kqueue/epoll 的就绪语义只是 POSIX 后端的实现细节；`wait_readable/wait_writable` 只在 `#if MIRA_HAS_READINESS_API` 下提供，Windows 上不能编译 |
| 组合而非绑定 | 协议只依赖 `AsyncStream` 概念。`http` 模块从不引用 socket 类型，TLS 也不硬编码 HTTP。分层由 `scripts/ci/check_layering.py` 强制检查 |
| 错误显式 | `Result<T>` = `std::expected<T, std::error_code>`。取消、截止时间、资源上限是设计标准 |
| 不做兼容层 | 不追求与 cpp-httplib 功能对齐，也不保证兼容旧 Aria API（ARCHITECTURE “Design mandate”、“Decisions on record”） |
| C++23 硬基线 | CMake 中 `CMAKE_CXX_STANDARD < 23` 直接 `FATAL_ERROR`，不保留 C++20 兼容 |

- **库形态**：编译型静态库，按模块提供公开头文件（`include/mira/<module>/*.hpp`），命名空间 `Mira::`。
- **依赖哲学**：core、transport、HTTP/1 的基础构建**不依赖任何第三方库**；TLS、WS、H2、H3 等依赖只在打开对应模块时引入（README “快速开始”）。
- **工具链下限**（README）：CMake 3.21+；GCC 16+（GCC 13 的协程优化器有 ICE）；Linux Clang 23+（clang≤18 的 `__cpp_concepts` 会导致 libstdc++ 隐藏 `std::expected`）；Xcode 27 AppleClang；MSVC v145（VS2026）；Android **NDK r30+**（NDK 27/28 的 libc++ 关闭了 `std::stop_token`）。

## 平台与后端矩阵（README “平台矩阵”）

| 平台 | 后端 | 验证范围 |
|---|---|---|
| macOS | kqueue | 基础/TLS/H2/H3 运行测试，ASan/UBSan，独立 curl HTTP/3 与 Retry 互操作 |
| Linux | epoll | GCC/Clang 运行测试，GCC ASan/UBSan，独立 curl H3 互操作 |
| Windows | IOCP | MSVC 与 MinGW 原生基础/TLS/H2/H3 测试；MSVC 独立 curl H3 互操作 |
| iOS | kqueue | 只在 arm64 模拟器上跑过，**没有真机运行证据** |
| Android | epoll | 只在 arm64 模拟器上跑过，**没有真机运行证据** |

io_uring **没有落地**，按 ROADMAP P2-2 暂缓：没有实测瓶颈就不改核心循环。

## 模块 / 协议覆盖总览

| 模块（CMake target） | 职责 | 状态 / 开关 |
|---|---|---|
| `Mira::core` | `Task`、`TaskScope`、`Result`、`AsyncStream`、`EventLoop`、`Executor`、`BoundedExecutor`、`LoopGroup`、`ResourceBudget`、`observe` | 始终构建 |
| `Mira::http_common` | H1/H2/H3 共用的字段类型，无依赖 | 始终构建 |
| `Mira::transport` | TCP/UDP、本地流、有界系统解析器、`tcp::dial`、`tcp::serve` 受管服务、文件传输（sendfile/TransmitFile） | 始终构建 |
| `Mira::http` | HTTP/1.1 的解析、序列化、`serve_connection`、`ClientConnection`、流式 body、`Expect: 100-continue` | 始终构建 |
| `Mira::socks` / `Mira::dns` / `Mira::mqtt` | SOCKS5（RFC1928/1929，含 BIND、UDP ASSOCIATE）；DNS 报文编解码、DoH（RFC8484）、UDP/TCP 查询；MQTT 3.1.1/5.0 | 始终构建，不引入 OpenSSL |
| `Mira::client` | HTTP/1 连接池 `client::HttpClient`、SOCKS5 拨号、重定向、gzip/deflate 解压 | 始终构建 |
| `Mira::tls` / `Mira::client_tls` | OpenSSL TLS 流、mTLS、多 ALPN、SNI 多证书、原子 reload、本地 CRL、OCSP stapling；`HttpsClient` | `MIRA_ENABLE_TLS` |
| `Mira::crypto` / `Mira::dnssec` | OpenSSL Crypto 原语；本地 DNSSEC 校验 | WS 或 `MIRA_ENABLE_DNSSEC` |
| `Mira::ws` | WebSocket/WSS，支持子协议、RFC7692 permessage-deflate、Extended CONNECT、`ws::ByteStream` | `MIRA_ENABLE_WEBSOCKET`（需 zlib） |
| `Mira::http2` / `Mira::client_http2` | 基于 nghttp2，含 HPACK、多流、Extended CONNECT、`SessionDriver`；`Http2Client` 池 | `MIRA_ENABLE_HTTP2` |
| `Mira::quic` / `Mira::http3` / `Mira::client_http3` | 基于 ngtcp2 + nghttp3，含 Retry、迁移、0-RTT、DCID 分发、DATAGRAM（RFC9221）、qlog；`Http3Client` 池 | `MIRA_ENABLE_HTTP3`。**1.2.0 起不再标为 experimental**（ROADMAP P1-2） |
| SSE | 不是独立模块。`examples/sse_server.cpp` 用 HTTP/1 chunked 编码演示，含 Last-Event-ID | 示例 |

target 名称取自 `modules/*/CMakeLists.txt` 里的 `add_library(Mira::*)`。

## 核心 API 与最小示例

### 关键类型（`modules/core/include/mira/core/`）

- `error.hpp`：`enum class Errc { ok, would_block, eof, cancelled, timed_out, limit_exceeded, invalid_argument, not_supported, internal }`；`using Error = std::error_code`；`template<class T> using Result = std::expected<T, Error>`；`fail(Errc)` / `fail(Error)` 返回 `std::unexpected`。
- `event_loop.hpp`：`static Result<EventLoop> create()`、`Result<void> run()`、`Result<void> run_until_complete(Task<void>)`、`void stop()`（任意线程都可调用）、`void post(move_only_function<void()>)`（可靠的 continuation 通道）、`Result<void> try_post(...)`（有界准入，满时返回 `would_block`）。
- `operation.hpp`：`struct OperationOptions { std::stop_token stop; deadline; }`，`deadline` 是 `Clock = std::chrono::steady_clock` 上的**绝对时间点**。
- `task_scope.hpp`：`spawn(Task<void>)`、`Task<void> join()`、`get_stop_token()`、`request_stop()`。
- 传输层：`tcp::Listener::bind(loop, Endpoint)`、`listener.accept()`、`tcp::connect(EventLoop&, Endpoint, ConnectOptions = {}, OperationOptions = {})`（`transport/tcp.hpp:236`）。
- TLS：`tls::Stream<T>::create(EventLoop&, Underlying&, const Context&, std::string_view peer_name = {}, std::optional<ResourceBudget> = {})` 返回 `Result<Stream>`，然后 `co_await handshake(OperationOptions)`、`negotiated_protocol()`（`tls/stream.hpp:50-135`）。

### HTTP/1.1 服务端（节选自 `examples/hello_world_server.cpp`）

```cpp
#include <mira/core/event_loop.hpp>
#include <mira/core/task.hpp>
#include <mira/core/task_scope.hpp>
#include <mira/http/connection.hpp>
#include <mira/transport/tcp.hpp>
namespace tcp = Mira::transport::tcp;
using Mira::Task; using Mira::Result;

Task<void> serve_one(tcp::Socket socket) {              // 按值持有，生命周期由协程帧负责
    auto served = co_await Mira::http::serve_connection(socket,
        [](const Mira::http::Request&, auto& writer, std::span<const std::byte>)
            -> Task<Result<void>> {
            Mira::http::Response rsp; rsp.status = 200;
            rsp.headers.append("Content-Type", "text/plain");
            std::string_view body = "hello";
            co_return co_await writer.send(rsp, {reinterpret_cast<const std::byte*>(body.data()), body.size()});
        });
    (void)served;                                        // eof 表示正常结束
}
Task<void> serve(tcp::Listener& listener) {
    Mira::TaskScope scope;
    for (;;) {
        auto accepted = co_await listener.accept();
        if (!accepted) { scope.request_stop(); break; }
        scope.spawn(serve_one(std::move(*accepted)));
    }
    co_await scope.join();                               // 必须 join
}
int main() {
    auto loop = Mira::EventLoop::create();               // Result<EventLoop>
    auto listener = tcp::Listener::bind(*loop, Mira::transport::Endpoint::loopback(8080));
    return loop->run_until_complete(serve(*listener)) ? 0 : 1;
}
```

TCP echo（`examples/echo_server.cpp`）的写法：`socket.read_some(buffer, {.stop = stop})` 加 `Mira::write_all(socket, span, {.stop = stop})`，stop token 来自 `scope.get_stop_token()`，这样 `request_stop()` 才能真正结束连接。

### HTTP/1.1 客户端（`examples/http1_client.cpp`）

```cpp
const Mira::OperationOptions io{.deadline = Mira::Clock::now() + std::chrono::seconds(3)};
auto socket = co_await tcp::connect(loop, endpoint, {}, io);
Mira::http::ClientConnection client{*socket};
Mira::http::Request req; req.target = "/"; req.headers.append("Host", endpoint.to_string());
auto started = co_await client.start(req, {}, io);
for (;;) { auto chunk = co_await client.read_body(); if (!chunk || chunk->empty()) break; /*...*/ }
// 复用或销毁 ClientConnection 之前，必须把 body 读到空块为止；之后可用 client.reusable() 判断能否复用
```

需要连接池、DNS 解析和交错拨号时，改用 `client::HttpClient`（`Mira::client`）或 `client::HttpsClient`（`Mira::client_tls`）。

更多示例见 `examples/`：`ws_server/ws_client`、`h2_client`、`h2_prior_knowledge_server`、`h3_server/h3_client`、`h3_multi_server`（单端口、按 CID 分发、可开 Retry）、`https_managed_server`（按 ALPN 分发 H1/H2）、`loop_group_server`、`sse_server`、`socks5_*`、`doh_*`、`mqtt_client`、`tiny_file_server`。

## 线程/协程模型、生命周期、错误、取消与超时

- **Task**：惰性启动。立即完成的循环等待也只占常量原生栈（GCC Debug 下同样成立）。`co_await` 一个空 `Task` 会抛 `std::logic_error`。
- **TaskScope**：结构化并发，只能单线程使用。`join()` 只能调用一次，调用后不再接受新任务。第一个子任务抛异常会触发 `request_stop()`，join 收齐后重抛。**scope 用过却没 join 就析构，会直接 `std::terminate()`**，这是有意的 fail-fast。
- **EventLoop**：一个 loop 对应一个线程。stop 可以从任意线程发起，但会在 loop 线程上投递（ARCHITECTURE “Threading”）。`LoopGroup` 让每个 worker 拥有一个独立的、绑定线程的 loop；socket 必须在所属 worker 上创建和使用，不能跨 worker 迁移。
- **错误模型**：可预期的运行期失败都通过 `Result<T>`/`std::error_code` 返回。`Task` 仍可能传播 body 里的异常，所以**并不保证不抛异常**。handler 抛出的异常在库边界会被映射为 `Errc::internal`。
- **取消/截止时间**（README 取消节）：

| 情形 | 结果 |
|---|---|
| 提交时 token 已经 stop | `Errc::cancelled`，不提交操作 |
| 提交时已过截止时间 | `Errc::timed_out`，不提交操作 |
| 两者同时命中 | 返回 `cancelled` |
| 同一批里既有真实完成又有取消/超时 | 真实完成优先 |

  取消不会回滚已经发生的 I/O。IOCP 上被取消的读可能丢掉内核已经搬运的字节，这条连接**不能再复用**。TLS 请求到期或被取消后，整个 TLS 会话失效，不会为了更新期限去重放密文操作。
- **HTTP 超时**用两个时长而不是一个截止时间：`ServerOptions::idle_timeout` 和 `request_timeout`，`ClientOptions::request_timeout` 也一样，**1.2.0 起默认都是 60 秒**（之前为 0，即不限，见 ROADMAP P1-3；`http/connection.hpp:96/118`）。请求之间空闲超时视为**成功**结束；请求进行中超时返回 `timed_out` 并关闭连接；**不会发送 408**。显式设为 0 可关闭对应超时。`max_requests_per_connection` 默认 100。
- `EventLoop::stop()` 只负责让 `run()` 返回。要取消单个操作，用 `OperationOptions`。

## 构建与集成

### CMake 选项（根 `CMakeLists.txt`）

| 选项 | 默认 | 说明 |
|---|---|---|
| `MIRA_BUILD_TESTS` | 顶层工程为 ON | |
| `MIRA_BUILD_EXAMPLES` | 顶层且非交叉编译时为 ON | |
| `MIRA_WARNINGS_AS_ERRORS` | ON | 通过 `Mira::warnings` 接口 target 生效，不会污染消费方 |
| `MIRA_ENABLE_TLS` / `_DNSSEC` / `_WEBSOCKET` / `_HTTP2` / `_HTTP3` | 全部 OFF | 打开后才引入 OpenSSL、zlib、nghttp2、ngtcp2/nghttp3 |
| `MIRA_INSTALL` | 顶层为 ON | 以 `add_subdirectory` 方式嵌入时默认 OFF |
| `MIRA_BUILD_BENCH` / `MIRA_BUILD_FUZZERS` | OFF | fuzzer 只支持 clang；`MIRA_BENCH_LIBUV` 用于构建可选的 libuv 对照基准 |

### 消费方式（`docs/RELEASES.md`、README）

```cmake
# A. 安装后的 SDK
find_package(Mira 1.2.1 EXACT REQUIRED COMPONENTS core transport http)   # TLS 时加 tls 组件
target_link_libraries(app PRIVATE Mira::transport Mira::http)
# B. FetchContent，必须带 URL_HASH，钉 Release 源码包和 SHA256SUMS
set(MIRA_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(MIRA_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
FetchContent_Declare(Mira URL "${MIRA_SOURCE_URL}" URL_HASH "SHA256=${MIRA_SOURCE_SHA256}")
FetchContent_MakeAvailable(Mira)
# C. 先校验源码包再解压，然后 add_subdirectory(vendor/Mira)
```

- 包配置 `cmake/mira-config.cmake.in`：**只列出需要的组件时，只加载这些组件的依赖闭包**；不列组件时，加载安装时启用的全部模块。`client_http2` 会隐式带上 `http2 tls`，`client_http3` 会带上 `http3`。包版本兼容策略为 `SameMajorVersion`。
- **可选协议依赖的获取方式（1.2.1）**：`cmake/SharedDependencies.cmake` 只为已启用的模块选择依赖（openssl、zlib、nghttp2、nghttp3、ngtcp2）。它调用 **ForgeDeps**（`python -m forgedeps`，需 **Python 3.14+**），按 `scripts/protocol-dependencies.json` 的锁定信息（版本、revision、SHA256）准备依赖，构建配方来自外部仓库 **ForgeRecipes**。`requirements-build.txt` 钉住 `forgedeps@v0.4.10` 与 `forgerecipes@v0.1.8`。
  - 当前锁定版本：OpenSSL **4.0.3**、zlib 1.3.2、ngtcp2 1.22.1（官方）、nghttp2 **1.70.0-mira.1**、nghttp3 **1.18.0-mira.1**。后两个是 dqsjqian 维护的 fork，用来让 CONNECT 204 走隧道语义（`docs/PROTOCOL-DEPENDENCIES.md`）。
  - 离线构建：设置环境变量 `FORGEDEPS_OFFLINE=1`，或 `scripts/build.py --offline`（要求源码输入已经校验过）。用 `FORGEDEPS_HOME` 或 `--deps-home` 指定工作区。
  - **OpenSSL 属于锁定的依赖图**：随便设置一个 `OPENSSL_ROOT_DIR` 并不能替换掉它（PROTOCOL-DEPENDENCIES）。系统 openssl 可执行文件只能用来生成证书或做互操作测试。
  - 一键构建：`python -m forgedeps init <workspace>` 后执行 `python scripts/build.py --tls --websocket --http2 --http3`。
- OpenSSL 版本下限：`tls` 模块要求 `find_package(OpenSSL 3.0)`，`quic` 模块要求 `OpenSSL 3.5`（`modules/*/CMakeLists.txt`）。1.2.1 新增了对 OpenSSL 4 证书 API 的支持。生产环境应使用包含安全修复的版本，README 写的是截至 2026-09-30 的 3.5.9 / 3.6.5（CVE-2026-35189）。
- **ABI 策略**：只承诺同一主版本内**源码兼容**，任何版本之间（包括 patch 版本）**都不承诺二进制兼容**，换 SDK 必须全量重编（RELEASES “ABI policy”）。

## 与 Aria 系列的关系（只读核实各仓库 `dependencies.json` 与 CMake）

| 仓库（本地 HEAD） | 锁定的 Mira | 使用的组件 | 出处 |
|---|---|---|---|
| Aria v3.2.6 | **v1.2.0**（3f89424） | HTTP/REST/SSE 适配器：`transport http`，打开 `ARIA_HTTP_ENABLE_TLS` 时加 `tls`；**不启用 H2/H3/WS** | `modules/adapters/http/CMakeLists.txt`；CHANGELOG 3.0 写明 “cpp-httplib is replaced by Mira” |
| AriaAgent | v1.2.0 | LLM 客户端：`transport http tls`，负责解析、连接、TLS 握手（始终验证证书）和流式响应 | `core/CMakeLists.txt:18` |
| AriaRead | v1.2.0 | Web Server：`Mira::http` + `Mira::transport` | `src/apps/web_server/CMakeLists.txt` |
| AriaTools | v1.2.0 | Workbench Web 平台：`Mira::transport`、`Mira::http`（probe 自检直接用其客户端） | `Workbench/platform/web/CMakeLists.txt` |

- 结论：**Aria 系列目前都锁定在 Mira 1.2.0，还没有升到 1.2.1**。1.2.1 只涉及依赖工具链、OpenSSL 4 支持和 CI，没有公开 API 变化（CHANGELOG 1.2.1）。
- Aria 侧的覆盖开关：`-DARIA_PIN_MIRA_SOURCE_DIR=/path/to/Mira`（离线或本地源码），`-DARIA_DEP_MIRA_VERSION=`（参见 `aria-overview.md`）。
- 因为 Mira 不承诺 ABI，Aria 3.x 换 Mira 版本时必须重编（Aria CHANGELOG 3.1.0）。

## 版本演进要点（CHANGELOG）

| 版本 | 要点 / 迁移提示 |
|---|---|
| 0.3.0 | 包配置统一为 `MiraConfig.cmake`，CMake 选项统一为 `MIRA_*` 前缀 |
| 0.4.0 | `post()` 改为接收 `mira::move_only_function<void()>`；新增 `Executor` 概念探测；新增 `writev_all` 分散写 |
| 0.5.0 | 共享字段类型移到 `Mira::http_common`，**HTTP/3 不再依赖 nghttp2**；H2/H3 字段别名的规范命名空间变了，需要重编 |
| 1.0.0（2026-09-30） | 首个稳定版。**`tls::Stream::create` 第一个参数改为 `EventLoop&`**；引入 `tcp::dial`、H1 流式上传与 `exchange` 双工、H3 0-RTT、socks/dns/mqtt/client 模块、`LoopGroup`、`ResourceBudget`、QUIC Retry、按 DCID 分发 |
| 1.1.0 | H2/H3 `SessionDriver` 与多路客户端池（`client_http2/3`）；TLS SNI 多证书、原子 reload、CRL、OCSP；DNS 缓存与 UDP/TCP 查询；QUIC PMTUD、拥塞控制选择、DATAGRAM、qlog、ReplayStore；可观测性 |
| 1.1.1 | HTTP/1 请求 trailer 中禁止出现分帧、路由类字段 |
| 1.2.0（2026-10-26） | 文件传输策略（sendfile/TransmitFile）、TCP TLS 1.3 early data（需显式 opt-in）、客户端 Cookie/缓存状态、MQTT 出站主题别名、DNSSEC、CONNECT 204 隧道。**行为变化：HTTP 超时默认值从 0 改为 60 秒**（ROADMAP P1-3，CHANGELOG 未单列）。大文件或慢传输场景需要显式调大 `request_timeout` |
| 1.2.1（2026-10-27） | 依赖构建改走 ForgeDeps + ForgeRecipes（1.2.0 用的是 AriaDeps 0.3.0），**此后启用可选协议都需要 Python 3.14 和 forgedeps**；支持 OpenSSL 4；工具链全部升到最新（GCC 16 / Clang 23 / VS2026 / Xcode 27 / NDK r30） |

## 常见坑 / FAQ

| 现象 | 原因 / 处理 |
|---|---|
| 程序退出时 `std::terminate` | `TaskScope` 用过却没 `co_await scope.join()` 就析构 |
| `request_stop()` 后连接不结束 | 读写调用没有传 `{.stop = scope.get_stop_token()}` |
| 升到 1.2.x 后大下载、长轮询被断开 | HTTP 默认超时变成 60 秒。调大 `request_timeout`，或显式设为 0 |
| Linux clang 报找不到 `std::expected` | clang≤18 搭配 libstdc++ 的已知问题，换 Clang 23+ |
| Android 编译 `stop_token` 失败 | NDK < r30 |
| 启用 TLS/H3 时配置期报 “Install Mira requirements-build.txt…” | 缺 Python 3.14 或 forgedeps。执行 `pip install -r requirements-build.txt`，并用 `FORGEDEPS_PYTHON_EXECUTABLE` 指定解释器 |
| 设置了 `OPENSSL_ROOT_DIR` 却不生效 | OpenSSL 由锁定依赖图提供，属于设计如此 |
| 需要 H1/H2 自动切换 | 库**不会自动切换**。读 `negotiated_protocol()` 后自己分发，参照 `examples/https_managed_server.cpp` |
| CONNECT 返回 204 却得到 `not_supported` | 链接的是上游 nghttp2/3，缺少 `*_HAS_CONNECT_2XX_TUNNEL`。改用 `-mira.1` fork，且头文件与库必须来自同一次安装 |
| 跨 loop 使用 socket 出问题 | socket 与创建它的 loop 线程绑定，不能迁移 |
| Windows 上读取 UDP 辅助数据或 Unix socket 返回 `not_supported` | 平台不支持，库明确拒绝，不会静默降级 |

## 与业界方案的定位对比（只列可核实的差异）

| 方案 | 与 Mira 的差异 |
|---|---|
| Boost.Asio / standalone Asio | Asio 也是 proactor 风格，支持回调、future、协程等多种完成令牌，C++ 标准要求更低，生态成熟。Mira 只提供协程这一种接口，以 C++23 的 `std::expected` 加 `stop_token` 作为统一的取消与错误模型，HTTP/TLS/QUIC 等协议都在同一仓库内 |
| libuv | C 库，回调式，生态广（Node.js）。Mira 在 `bench/libuv_echo_server.cpp` 中把它作为可选的单 loop 对照基准，README 明确说明 loopback 数据**不构成跨库性能排名** |
| cpp-httplib | 头文件库，同步阻塞、线程池模型，上手简单。Mira 在 ARCHITECTURE 里声明不以与它功能对齐为目标；Aria 3.0 把 HTTP 适配器从 cpp-httplib 换成了 Mira |
| libcurl | C 客户端，协议覆盖最广，经过长期实战检验（AriaRead 仍用 curl 8.22 做书源抓取）。Mira 的 client 是协程化的 H1/H2/H3 连接池，服务端和客户端共用同一套协议栈 |
| nghttp2 / ngtcp2 / nghttp3 | 这三个是协议引擎，不是对手。Mira 的 H2/H3 直接构建在它们之上，在外层提供协程驱动、资源预算和生命周期管理 |

## 诚实边界（README “仍需验证的边界”、ROADMAP）

- **未实现或不支持**：io_uring、splice、MSG_ZEROCOPY；客户端记忆服务端 SETTINGS；使用原生系统信任（Keychain、Windows 证书库）；SSM 组播；Windows UDP ancillary 数据与 Unix-domain socket；SOCKS5 UDP 分片；普通（非 Extended）CONNECT 代理；H3 DATAGRAM / WebTransport（只有 QUIC 层 RFC9221 DATAGRAM）；MQTT broker、自动重连策略、磁盘存储后端；分布式或持久化的 0-RTT 防重放存储。gRPC、Redis、WebRTC 属于生态层，不在本库范围内。
- **证据边界**：移动端只有模拟器证据；多机/WAN 和进程 RSS 硬上限没有验证；aioquic 互操作只覆盖 aioquic 客户端到 Mira 服务端这一方向；Retry 不保证 token 只用一次，也不构成完整的抗 DDoS 能力；`ResourceBudget` 是计量配额，不等于进程 RSS 上限。
- **需查证**：Mira 是否提供 HTTP 路由器。`modules/http/include` 中没有找到 router 或 route 相关 API，handler 需要自己分发，Aria 适配器自己实现了 REST 路由。按“不提供”处理。
