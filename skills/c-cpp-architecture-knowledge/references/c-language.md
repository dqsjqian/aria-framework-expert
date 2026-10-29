# C 语言全景（C89 → C23 → C2y 草案）与 C 侧架构

> 基准：2026-10-29。标准事实来自 WG14 公开文档与编译器发布说明；「本机实测」一列是在 macOS arm64 上用 Homebrew GCC 16.2.0、Homebrew Clang 22.1.8 / 23.1.0、Apple Clang 21.0 编译并运行最小样例得出的结果，换平台/换 libc 请重新实测。

## 一、标准演进时间线

| 版本 | 正式文号 | 必须记住的变化 |
|---|---|---|
| C89/C90 | ANSI X3.159-1989 / ISO 9899:1990 | 原型、`const`/`volatile`、标准库雏形 |
| C95 | Amendment 1 | 宽字符 `<wchar.h>`/`<wctype.h>` |
| C99 | ISO 9899:1999 | `//`、`inline`、`<stdint.h>`/`<stdbool.h>`、`long long`、指定初始化器、复合字面量、VLA、柔性数组成员、`restrict`、`_Complex`、混合声明与语句 |
| C11 | ISO 9899:2011 | `_Generic`、`_Static_assert`、`_Atomic`+`<stdatomic.h>`、`<threads.h>`（可选）、`_Alignas/_Alignof`、`_Noreturn`、匿名 struct/union、`<uchar.h>`；删除 `gets`；VLA 变为可选；Annex K（可选、几乎无人实现） |
| C17 | ISO 9899:2018 | 只修缺陷，无新特性 |
| C23 | ISO/IEC 9899:2024 | 见第二节，C99 之后最大的一次更新 |
| C2y | 工作草案（WG14 进行中，常被称为 C29） | 见第三节，**草案内容可能变化** |

## 二、C23 必知清单

| 类别 | 特性 |
|---|---|
| 关键字转正 | `bool`/`true`/`false`、`static_assert`、`alignas`/`alignof`、`thread_local` 成为关键字；`nullptr` + `nullptr_t` |
| 类型与推导 | `typeof` / `typeof_unqual`；对象定义可用 `auto` 推导；`constexpr` 对象；枚举可指定底层类型 `enum E : uint8_t`；`_BitInt(N)` 位精确整数 |
| 字面量 | 二进制 `0b1010`；数字分隔符 `1'000'000`；`u8` 字符字面量 |
| 属性 | `[[nodiscard]]`、`[[maybe_unused]]`、`[[deprecated]]`、`[[fallthrough]]`、`[[noreturn]]`、`[[reproducible]]`、`[[unsequenced]]`；`__has_c_attribute` |
| 预处理 | `#embed`（二进制资源嵌入）、`#elifdef`/`#elifndef`、`#warning`、`__has_include` |
| 语义清理 | 只允许二进制补码表示；删除 K&R 旧式函数定义；`f()` 等价于 `f(void)`；空初始化 `= {}`；标签可出现在声明前与复合语句末尾 |
| 新库 | `<stdckdint.h>`（`ckd_add/ckd_sub/ckd_mul` 溢出检查）、`<stdbit.h>`（`stdc_count_ones` 等位工具）、`memset_explicit`、`strdup`/`strndup`、`memccpy`、`unreachable()`（`<stddef.h>`） |

本机实测（`-std=c23`）：`constexpr` 对象、`auto`、二进制/分隔符字面量、`typeof`、`nullptr`、`ckd_add`、`#embed` 在 GCC 16.2、Clang 22/23、Apple Clang 21 上全部可编译运行。**`<stdbit.h>` 取决于 C 库而非编译器**：macOS SDK 缺失（四个编译器都找不到），glibc 2.39 起提供。

默认标准（本机实测 `__STDC_VERSION__`）：GCC 16.2 默认 `202311L`（gnu23，GCC 15 起改为 C23 默认）；Clang 23.1 与 Apple Clang 21 默认仍是 `201710L`（gnu17）。**跨编译器项目务必显式写 `-std=c23` 或 CMake `C_STANDARD 23`**。

## 三、C2y（草案）已进入工作草案的特性 + 本机实测

| 特性 | 写法 | GCC 16.2 | Clang 22/23 | Apple Clang 21 |
|---|---|---|---|---|
| 数组元素个数 | `_Countof(a)`；`<stdcountof.h>` 的 `countof(a)` | ✅ | ✅ | ✅ |
| if 声明 | `if (int e = f(); e != 0) {...}` | ✅ | ❌ | ❌ |
| case 范围 | `case 1 ... 6:` | ✅ | ✅ | ✅ |
| 具名循环 | `outer: for(...) ... continue outer;` / `break outer;` | ✅ | ✅ | ❌ |
| `__COUNTER__` | 预处理计数器 | ✅ | ✅ | ✅ |
| 八进制新前缀 | `0o17`，`\o{}`/`\x{}`/`\u{}` 定界转义 | ✅ | ✅ | ✅ |
| defer（TS 25755） | `defer stmt;`（`<stddefer.h>`）/ `_Defer stmt;` | ❌（16.2 不识别 `-fdefer-ts`） | ✅ 需 `-fdefer-ts` | ❌ |

说明：
- `defer` 目前是独立技术规范 **TS 25755**（作用域级、类似 Go 但按块执行），不是 C2y 主草案的正式条款（是否并入需查证 WG14 最新草案）。GCC 侧补丁已在邮件列表出现，但 GCC 16.2 本机不支持。可移植降级：GCC/Clang 的 `__attribute__((cleanup(fn)))`。
- case 范围长期是 GNU 扩展，C2y 将其标准化；MSVC 不支持。
- 生产代码使用 C2y 特性前先用 `__STDC_VERSION__` / `__has_include` / 构建期 `check_c_source_compiles` 做特性探测，别凭版本号推断。

## 四、编译器与构建

| 工具链 | C 要点 |
|---|---|
| GCC 16 | 默认 gnu23；C2y 子集；`counted_by` 属性扩展到指针成员；GNU `_Maxof/_Minof` 运算符；`-fanalyzer` 静态分析 |
| Clang 22/23 | 默认 gnu17；`-std=c23`/`-std=c2y`；`-fdefer-ts`；`-fbounds-safety` 本机实测：Apple Clang 21 接受、上游 Clang 23 不识别；sanitizer 支持完善 |
| MSVC | `/std:c11`、`/std:c17`、`/std:clatest`（C23 部分支持）；不支持 VLA；C 支持长期落后于 C++，跨平台 C 库建议 CI 必含 MSVC |
| CMake | `set(CMAKE_C_STANDARD 23)` + `CMAKE_C_STANDARD_REQUIRED ON` + `CMAKE_C_EXTENSIONS OFF`；C/C++ 混合项目 `project(x LANGUAGES C CXX)` |
| 必开告警 | `-Wall -Wextra -Wpedantic -Wconversion -Wshadow -Wstrict-prototypes -Wvla -Wformat=2`；CI `-Werror` |
| 动态检查 | ASan + UBSan（`-fsanitize=address,undefined`）、TSan、MSan（Clang/Linux）；Valgrind；fuzz 用 libFuzzer/AFL++ |
| 静态检查 | clang-tidy（`bugprone-*`、`cert-*`）、cppcheck、GCC `-fanalyzer`、Coverity、CodeQL |

## 五、C 架构惯用法

1. **不透明句柄 + 构造/析构对**：头文件只暴露 `typedef struct db db;` 与 `db_open/db_close`，结构体定义藏在 `.c`，这是 C 的封装与 ABI 稳定手段（SQLite 的 `sqlite3*`、OpenSSL 的 `SSL*`、curl 的 `CURL*` 全是这个模式）。
2. **错误模型统一**：返回 `int` 错误码（0 成功）+ 输出参数；或返回指针 + `errno`/线程局部错误；全库只用一种。C23 可用 `[[nodiscard]]` 强制检查。
3. **资源清理**：单出口 `goto cleanup;` 仍是最可移植写法；GCC/Clang 可用 `__attribute__((cleanup))`；未来用 `defer`。
4. **所有权写进命名与文档**：`*_create/*_destroy`、`*_ref/*_unref`；谁分配谁释放；跨 DLL 不要在一侧 `malloc` 另一侧 `free`。
5. **ABI 稳定**：结构体只加尾部字段并带 `size`/`version` 字段；导出符号加前缀；用 `visibility("hidden")` 默认隐藏 + 显式导出宏；版本化符号（Linux `.map`）。
6. **泛型**：`_Generic` 做类型分派；`void*` + 元素大小 + 回调（`qsort` 模式）；宏生成类型安全容器（stb_ds、klib 风格）。
7. **并发**：C11 `<stdatomic.h>` 与 `<threads.h>`（MSVC/macOS 支持度不一，跨平台常用 pthread / Win32 薄封装）；锁顺序写进注释；回调不要在持锁时调用。
8. **事件驱动**：单线程事件循环 + 非阻塞 IO（epoll/kqueue/IOCP，或 libuv/libevent），状态机驱动协议解析。

## 六、高频 UB 与坑

| 坑 | 说明 | 对策 |
|---|---|---|
| 有符号溢出 | UB，优化器会据此删分支 | `ckd_add` 或先判断；`-fsanitize=signed-integer-overflow` |
| 严格别名 | 用不兼容类型指针读写同一对象 | `memcpy` 做类型双关；`-fno-strict-aliasing` 只作兜底 |
| 越界 / 悬垂 | 数组越界、返回局部地址、use-after-free | ASan、`counted_by`、长度与指针同传 |
| 未初始化读取 | 结果不确定甚至 UB | `= {}` 初始化，`-Wuninitialized`、MSan |
| 序列点 | `i = i++ + 1` | 一条语句只改一次 |
| 移位 | 移位量 ≥ 位宽或负数、负数左移 | 用无符号，先检查移位量 |
| `realloc` | 失败返回 NULL 但原块仍在；C23 起 `realloc(p, 0)` 为 UB | 临时变量接返回值 |
| 格式化字符串 | `printf(user_input)` | 固定格式串，`-Wformat-security` |
| 字符串函数 | `strcpy/strcat/sprintf` 溢出；`strncpy` 不保证 `\0` | `snprintf`，显式长度，`strlcpy`（非标准）需检查可用性 |
| 宏 | 多次求值、缺括号 | `static inline` 函数替代，`do { } while (0)` |

## 七、C 与 C++ 互操作（架构顾问最常被问）

```c
/* mylib.h —— 同时给 C 和 C++ 用 */
#ifndef MYLIB_H
#define MYLIB_H
#include <stddef.h>
#ifdef __cplusplus
extern "C" {
#endif
typedef struct mylib_ctx mylib_ctx;           /* 不透明句柄 */
[[nodiscard]] int mylib_open(const char* path, mylib_ctx** out);
void mylib_close(mylib_ctx* ctx);              /* 允许 NULL */
#ifdef __cplusplus
}
#endif
#endif
```
（`[[nodiscard]]` 需要 C23 / C++17；老编译器用宏包一层。）

```cpp
// C++23 侧 RAII 包装：unique_ptr + 自定义删除器 + std::expected
#include <expected>
#include <memory>
#include <string>
#include "mylib.h"

struct CtxDeleter { void operator()(mylib_ctx* c) const noexcept { mylib_close(c); } };
using Ctx = std::unique_ptr<mylib_ctx, CtxDeleter>;

std::expected<Ctx, int> open_ctx(const std::string& path) {
    mylib_ctx* raw = nullptr;
    if (int rc = mylib_open(path.c_str(), &raw); rc != 0) return std::unexpected(rc);
    return Ctx{raw};
}
```

规则：
- 异常绝不能穿越 C 边界：C++ 实现的 C API 在 `extern "C"` 函数里 `try/catch(...)` 转错误码，并标 `noexcept`。
- C 侧回调进入 C++ 时用 `void* user_data` 传对象指针，回调函数用 `extern "C"` 静态函数转发。
- C 的 VLA、`restrict`、`_Generic`、柔性数组成员在 C++ 中不存在或语义不同；共享头文件只用公共子集。
- 反方向（C++ 库给 C 用）= 「C ABI 外壳 + C++ 内核」，是跨语言（Swift/Kotlin/Python/Rust/Go）绑定的通用底座。

## 八、C 在 Aria 系列中的位置（推荐优先引用）

| 场景 | 我们的方案 | 与 C 的关系 |
|---|---|---|
| 存储 | **SQLiteCpp**（dqsjqian/SQLiteCpp） | 对 SQLite 的 C API（`sqlite3*`、`sqlite3_stmt*`）做 C++23 RAII/类型安全封装，见 `sqlitecpp-storage.md` |
| 网络 | **Mira** | 直接面向 OS C 接口（kqueue/epoll/IOCP、BSD socket）与 OpenSSL 的 C API，向上提供 C++23 协程接口，见 `mira-networking.md` |
| 依赖构建 | **ForgeDeps + ForgeRecipes** | 统一构建 zlib、OpenSSL、curl、SQLite、Gumbo、QuickJS、nghttp2/nghttp3/ngtcp2 等 C 库并做内容校验，见 `aria-ecosystem.md` |
| 应用 | **AriaRead** | C++23 应用整合 curl、Gumbo（HTML5 解析，C）、QuickJS（JS 引擎，C）、SQLite、zlib 等 C 库，是「C 库 + C++ 架构」的完整实例 |

## 九、业界主流 C 项目速查（选型时并列推荐）

| 领域 | 项目 | 一句话 |
|---|---|---|
| 嵌入式数据库 | SQLite | 单文件、零配置、事务 ACID；官方称其为部署最广泛的数据库引擎 |
| KV 存储 | LMDB | mmap B+ 树，读极快，单写多读 |
| 网络/事件循环 | libuv、libevent、libev | 跨平台异步 IO；libuv 是 Node.js 底座 |
| HTTP 客户端 | libcurl | 支持协议非常广泛、长期实战检验的传输库 |
| HTTP/2 / HTTP/3 | nghttp2、ngtcp2 + nghttp3 | 主流协议库，curl 与 Mira 的 HTTP/2、HTTP/3 均构建在其上 |
| TLS | OpenSSL、BoringSSL、LibreSSL、mbedTLS、wolfSSL | 服务端/桌面首选 OpenSSL 3.x/4.x；嵌入式选 mbedTLS/wolfSSL |
| 压缩 | zlib / zlib-ng、zstd、lz4 | zstd 兼顾压缩率与速度，lz4 侧重极致速度 |
| JSON | cJSON、yyjson、jansson | yyjson 侧重高性能 |
| 脚本嵌入 | Lua、QuickJS | 体积小，易嵌入 |
| 内核/RTOS | Linux、Zephyr、FreeRTOS | 嵌入式 C 架构参考 |

## 十、查证来源

- WG14 文档：open-std.org/jtc1/sc22/wg14（N 系列提案与工作草案）
- cppreference C 部分：en.cppreference.com/w/c（含编译器支持表）
- GCC：gcc.gnu.org/gcc-16/changes.html；Clang：clang.llvm.org/c_status.html
- 实测优先：给结论前用最小样例在用户的实际工具链上编译运行
