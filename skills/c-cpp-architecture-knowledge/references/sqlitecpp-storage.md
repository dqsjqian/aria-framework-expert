# SQLiteCpp（dqsjqian 版）：C++23 纯头文件 SQLite 封装

> 基准版本：SQLiteCpp 1.2.1（tag v1.2.1 为附注 tag，解引用后指向 commit 1cac401，2026-10-29 与 GitHub 远端 main 核对；`git ls-remote` 显示 main 为 1cac401）
> 仓库：https://github.com/dqsjqian/SQLiteCpp（MIT，© 2026 dqsjqian）。版本宏 `SQLITE_CPP_VERSION_MAJOR/MINOR/PATCH` 定义在 `include/SQLiteCpp/config.h`，与 `project(SQLiteCpp VERSION 1.2.1)` 一致。

## ⚠️ 先分清：此 SQLiteCpp ≠ SRombauts/SQLiteCpp

**结论**：dqsjqian/SQLiteCpp 和业界常说的 SRombauts/SQLiteCpp（“SQLiteC++”）**同名但无关**，两者 API **完全不兼容**。dqsjqian 版的 API 谱系来自 **sqlite_modern_cpp**（SqliteModernCpp/aminroosta，MIT，最后一个 release 是 2017 年的 v3.2），在此基础上用 C++23 重写和扩展（上游许可同为 MIT）。

证据（均为只读核实）：

| 证据 | 内容 |
|---|---|
| 首个提交 `8ae9922`（2026-10-26，“SQLiteCpp 0.7.0”） | 目录结构是 `hdr/`、`configure.ac`、`Makefile.in`、`.travis.yml`，测试文件名 `tests/prepared_statment.cc`（原拼写错误照搬）、`variant.cc`、`sqlcipher.cc` 等，与 sqlite_modern_cpp 仓库一一对应。`hdr/SQLiteCpp/errors.h` 和上游 `hdr/sqlite_modern_cpp/errors.h` 的 diff 基本只有宏前缀不同（`SQLITE_MODERN_CPP_ERROR_CODE` 改为 `SQLITE_CPP_ERROR_CODE`）。`tests/variant.cc` 的 diff 只有 include 行和一个宏名 |
| 注释 | `tests/tmp_file.h` 写着 “Replaces the deprecated std::tmpnam used by the **original examples**” |
| API 形状 | `sqlite::database`、`db << sql << args >> result`、`sqlite_config`/`OpenFlags`、`sqlite::errors::constraint_*`、`sqlcipher_database`、`error_log`、`define`，全部是 sqlite_modern_cpp 的风格 |
| 下游命名 | AriaRead 的 `dependencies.json` 中，依赖名仍叫 **`sqlite_modern_cpp`**，来源却指向 `dqsjqian/SQLiteCpp` v1.2.1（1cac401）。AriaRead `CMakeLists.txt:173` 注释写着“来源已切到维护中的 SQLiteCpp 项目” |
| 代码来源 | git 历史中没有上游提交，属于导入 sqlite_modern_cpp（MIT，Copyright (c) 2017 aminroosta）代码后重新起仓、用 C++23 重写扩展；首个提交的 `errors.h` 与上游相比只有宏前缀和少量新增成员不同（2026-10-29 对照上游 master 核实） |

和 SRombauts/SQLiteCpp 的对照（SRombauts 的信息取自其 GitHub 主页，2026-10 时 master 是 4.0.0 开发线）：

| | dqsjqian/SQLiteCpp 1.2.1 | SRombauts/SQLiteCpp 4.x |
|---|---|---|
| 命名空间 / 主类型 | `sqlite::database`、流式 `<<` / `>>` | `SQLite::Database`、`SQLite::Statement`、`Column`、`Transaction` |
| 形态 | **纯头文件**（INTERFACE target） | 编译库（`src/*.cpp`） |
| 标准 | **C++23**（低于 C++23 直接 `#error`） | C++17（3.x 维护线为 C++11） |
| CMake | `find_package(SQLiteCpp CONFIG)`，target **`SQLiteCpp::SQLiteCpp`** | `find_package(SQLiteCpp)`，target `SQLiteCpp`（无命名空间） |
| SQLite | 自带 3.54.1 amalgamation，下限 3.38 | 下限 3.38，可选内置 `src/sqlite3` |

**风险**：两者的包名都叫 `SQLiteCpp`，同一个 `CMAKE_PREFIX_PATH` 里同时装了两个会互相遮蔽。dqsjqian 版的包版本是 1.x，SRombauts 版是 3.x/4.x，`SameMajorVersion` 下用 `find_package(SQLiteCpp 1.2 CONFIG)` 可以区分开。

## 定位与设计哲学

README 的定位是“最先进的现代 C++23 SQLite 封装库：流式参数绑定、强类型结果、结构化行映射、预编译语句缓存。纯头文件、自闭环、只认一个 SQLite 版本”。

- **C++23 硬要求**（`config.h`）：GCC 13 的 `__cplusplus == 202100L` 也会被接受；MSVC 要求 `/std:c++latest`。
- **SQLite 版本下限 3.38**（`fwd.h`）：因为库无条件调用了 `sqlite3_error_offset`。仓库在 `vendor/sqlite` 内置 **SQLite 3.54.1** amalgamation，与官方原版一致，SHA3-256 已校验，见 `vendor/sqlite/README.md`。
- **零构建依赖**：克隆后直接 cmake 即可，不需要包管理器、Python 或联网。1.0 之前曾用 ForgeDeps，`adba66a` 移除了 ForgeDeps 工具链。
- 所有公开类型都在 `namespace sqlite`。唯一入口头文件是 `#include <SQLiteCpp.h>`；`sqlcipher.h` 和 `log.h` **不在**入口里，需要单独 include。
- 许可证 MIT；内置的 SQLite 属于公有领域。

## API 总览（`include/SQLiteCpp/*.h`）

| 领域 | 入口（签名均摘自头文件） |
|---|---|
| 连接 | `database(const std::string&, const sqlite_config& = {})`、`database(const std::u16string&, ...)`、`database(std::shared_ptr<sqlite3>, std::size_t statement_cache_size = 32)`；`connection()` 返回 `std::shared_ptr<sqlite3>` |
| 配置 | `sqlite_config{ flags = READWRITE\|CREATE, zVfs, encoding, statement_cache_size = 32, busy_timeout_ms = 0 }`；`OpenFlags{READONLY, READWRITE, CREATE, NOMUTEX, FULLMUTEX, SHAREDCACHE, PRIVATECACH, URI}`（`PRIVATECACH` 是源码原拼写） |
| 控制 | `busy_timeout(ms)`、`configure(config_flag, bool)` / `configured(flag)`（`foreign_keys`、`triggers`、`views`、`defensive`、`trusted_schema`、`writable_schema`、`load_extension`、`scanstatus`）、`interrupt()`、`progress_handler(std::function<int()>, int instructions = 1000)`、`wal(bool = true)`、`load_extension(file, entry)` |
| 语句 | `db << sql` 返回 `database_binder`；`binder << v`、`>> 变量 / tuple / lambda`、`execute()`、`used(bool)`、范围 for 配合 `row.as<Ts...>()`、`column_count/column_name/column_declared_type`；`column_table_name/column_origin_name` 需要编译期定义 `SQLITE_ENABLE_COLUMN_METADATA` |
| 执行 | `sqlite3_int64 execute(sql)`、`execute_many(sql, range_of_tuples)`、`execution_result try_execute(sql, args...) noexcept`（返回 `{changes, code, error}`，通过 `explicit operator bool` 判断成败） |
| 结果 | `select<Record>(sql, &Record::a, &Record::b...)` 返回 `std::vector<Record>`；`pragma(sql)` 返回 `std::string`；`explain(sql)` 返回 `std::vector<std::string>` |
| 事务 | `transaction(db, transaction_behavior::{deferred,immediate,exclusive})`，提供 `commit()`、`rollback()`、`active()`；`savepoint(db, name)` 可嵌套 |
| 数据 | `blob(db, table, column, rowid, read_only = false, schema = "main")`，提供 `read/write(std::span, offset)`、`size()`；`no_copy(text或bytes)`（对应 `SQLITE_STATIC`）；`serialize(schema)` / `deserialize(span, schema)`；`backup(src, dst, ...)`，提供 `step(pages)`、`run()`、`remaining()` |
| 函数 | `define(name, f)`、`define(name, function_flags, f)`（`deterministic`、`innocuous`、`direct_only`、`subtype`）；聚合函数 `define(name, step, final)`；`define_window(name, step, inverse, value, final)` |
| 钩子 | `on_update(handler)` / `on_update(nullptr)`、`on_commit(std::function<int()>)`、`on_rollback(...)` |
| 诊断 | `last_insert_rowid()`、`rows_modified()`、`total_changes()`（均为 64 位）、`cache_used()`、`schema_used()`；`read_header(path)` 返回 `header_info`，`has_sqlite_header(path)` |
| 字面量 | `using namespace sqlite::literals; "SELECT 1"_sql` |
| 可绑定类型 | 整型、`float`、`double`、`std::string(_view)`、`std::u16string(_view)`、`std::vector<T>` 或 `std::span<T>`（作为 BLOB）、`std::nullptr_t`、`std::monostate`、`std::optional<T>`、`std::unique_ptr<T>`、`std::variant<...>`；约束概念为 `sqlite::bindable` |

## 最小可编译示例（取自 README / `tests/`）

```cpp
#include <SQLiteCpp.h>
#include <tuple>
#include <vector>

struct item_record { int id = 0; std::string value; };

int main() {
    sqlite::database db(":memory:");
    db << "CREATE TABLE item (id INTEGER, value TEXT)";

    // 批量写入加 RAII 事务。没调用 commit() 时，析构会自动回滚
    const std::vector<std::tuple<int, std::string>> rows{{1, "one"}, {2, "two"}};
    sqlite::transaction tx(db, sqlite::transaction_behavior::immediate);
    db.execute_many("INSERT INTO item VALUES (?, ?)", rows);
    tx.commit();

    // 单值读取：要求结果恰好一行，0 行抛 errors::no_rows，多于 1 行抛 errors::more_rows
    int count = 0;
    db << "SELECT COUNT(*) FROM item" >> count;

    // 结构化读取
    auto items = db.select<item_record>("SELECT id, value FROM item ORDER BY id",
                                        &item_record::id, &item_record::value);
    for (auto&& row : db << "SELECT id, value FROM item") {
        auto [id, text] = row.as<int, std::string>();
    }

    // 持有语句并复用：execute() 后自动重置，下一轮 << 从 ?1 开始绑定
    auto insert = db << "INSERT INTO item VALUES (?, ?)";
    insert << 3 << "three"; insert.execute();

    // 不抛异常的路径
    if (auto r = db.try_execute("SELECT * FROM no_such_table"); !r) { /* r.code / r.error */ }
    return count == 2 && items.size() == 2 ? 0 : 1;
}
```

`tests/maintenance_regression.cpp` 覆盖了 1.x 新增的 API，包括 savepoint 嵌套、`define_window`、`try_execute` 和缓存旁路，可以作为权威用法参考。

## 语义要点：语句缓存、绑定、线程、错误

- **预编译语句缓存**（`detail/statement_cache.h`）：LRU 结构，默认容量 32，`statement_cache_size = 0` 表示关闭缓存（1.1.1 之前 0 实际被当作 1）。缓存的语句用 `sqlite3_prepare_v3` 加 `SQLITE_PREPARE_PERSISTENT` 编译；**1.2.1 起，绕过缓存的一次性语句不再带 PERSISTENT 标志**，这一项让单次编译加销毁的耗时下降了 58%。正在执行的语句不会被借出第二次，所以在行回调里嵌套执行同一条 SQL 也是安全的。
- **绑定语义**：每轮执行都从干净状态开始。本轮没有绑定的参数一律视为 `NULL`，不会残留上一次使用同一缓存语句时的值（这是 1.1.1 修复的 bug）。
- **析构即执行**（继承自 sqlite_modern_cpp 的语义）：`db << "..."` 产生的 binder 如果从未执行，**析构时会自动执行**；若当时正在展开异常，则不会执行。因此析构函数声明为 `noexcept(false)`，可能抛异常。`used(true)` 可以抑制自动执行。
- **多线程**：SQLite 以默认的 serialized 模式运行时（内置 amalgamation 就是这个模式），一个 `database` 可以被多个线程共享，缓存使用连接自身的 mutex，已在 TSan 下验证（`tests/concurrency.cc`）。约束有三条：单个语句对象同一时刻只能属于一个线程；钩子和进度回调必须在共享之前安装；以 `OpenFlags::NOMUTEX` 打开的连接只能单线程使用。
- **错误模型**：默认抛异常。`sqlite_exception`（继承 `std::runtime_error`）提供 `get_code()`、`get_extended_code()`、`get_sql()`、`get_error_offset()`（SQL 中的字节偏移）、`errstr()`。`sqlite::errors::*` 为每个 SQLite 主码和扩展码各定义了一个子类，可以按子类精确 catch，例如 `errors::constraint_primarykey`。不想用异常时用 `try_execute`。全局日志用 `#include <SQLiteCpp/log.h>` 里的 `sqlite::error_log(handler)`。
- **回调里的异常不会穿过 SQLite 的 C 栈帧**：进度回调抛异常会中断当前语句；commit 钩子抛异常会把提交改为回滚。
- **`no_copy`** 使用 `SQLITE_STATIC`：调用方必须保证缓冲区在 step 或重新绑定之前一直有效，而且 reset 并不会清除这类绑定。
- **SQLCipher**：`#include <SQLiteCpp/sqlcipher.h>` 后可用 `sqlcipher_database`、`set_key`、`rekey`，但需要自行链接 SQLCipher。`tests/sqlcipher.cc` **没有列入** CMake 测试集，`boost_optional.cc`、`flags.cc`、`prepared_statment.cc` 也没有。头文件中已经没有 boost::optional 支持（只剩 `std::optional`，以及 `SQLITE_CPP_EXPERIMENTAL_OPTIONAL_SUPPORT`）。这些功能**未经 CI 验证**。

## 性能（README，Release，Apple Silicon，`:memory:`，10 万次操作，取 9 轮中位数）

| 路径 | 相对裸 C API |
|---|---|
| 逐行 `db << sql << ...`（缓存开启） | +16%（约 33 ns/行） |
| 持有语句并重新绑定 | +4% |
| `execute_many` | +1%（基本持平） |
| 关闭缓存 | +242%（只作反例，不是建议；想省内存应把容量设为 1~4） |
| 单行 `>>` | +22%，其中 10 个百分点来自“确认恰好一行”所需的额外一次 step |
| 全表扫描（回调、range-for、`select<Record>`） | +1%~+2% |

README 在 1.2.0 中更正了 1.1.x 宣称的“`execute_many` 比 C 快 8%”：那个结论来自不对等的基准测试。

## 构建与集成

```sh
cmake -S . -B build -DBUILD_TESTING=ON && cmake --build build --config Release
ctest --test-dir build -C Release --output-on-failure
cmake --install build --prefix /path/to/sdk
```

```cmake
project(app LANGUAGES C CXX)                 # 编译 sqlite3.c 需要启用 C 语言
find_package(SQLiteCpp 1.2 CONFIG REQUIRED)  # 加上版本号，避免误找到 SRombauts 版
find_package(Threads REQUIRED)
add_library(sqlite3_core STATIC ${SQLiteCpp_SQLITE_DIR}/sqlite3.c)
target_include_directories(sqlite3_core PUBLIC ${SQLiteCpp_SQLITE_DIR})
target_link_libraries(sqlite3_core PUBLIC Threads::Threads ${CMAKE_DL_LIBS})
# 需要 column_table_name/origin_name 时，再给 sqlite3_core 加上 SQLITE_ENABLE_COLUMN_METADATA=1
target_link_libraries(app PRIVATE SQLiteCpp::SQLiteCpp sqlite3_core)
```

- **安装包只提供头文件和 SQLite 源码，不提供编译好的 sqlite3 库**。`SQLiteCppConfig.cmake` 会导出 `SQLiteCpp_SQLITE_DIR`，指向 `share/SQLiteCpp/sqlite`，由消费方自己编译。消费方也可以改用自己的 SQLite，但版本必须 ≥ 3.38。另外会安装 `SQLiteCpp.pc`（pkg-config）。
- 包版本兼容策略：`SameMajorVersion` 加 `ARCH_INDEPENDENT`。
- CMake 选项：`BUILD_TESTING`（CTest，默认 ON）、`SQLITECPP_WARNINGS_AS_ERRORS`、`SQLITECPP_COVERAGE`、`SQLITECPP_BUILD_FUZZERS`（仅 LLVM Clang）。仓库内的 `sqlite3_core` target 只在 `BUILD_TESTING` 打开时创建。
- 也可以 `add_subdirectory` 后直接链接 `SQLiteCpp::SQLiteCpp`，此时 SQLite 需要自己提供，可以编译 `vendor/sqlite/sqlite3.c`。
- 平台与 CI（README）：Linux x64（GCC 16，Clang 23 搭配 libstdc++ 和 libc++）、Linux ARM64、macOS、iOS 模拟器（只构建）、Android NDK r30（只构建）、Windows MSVC、clang-cl、ARM64、MinGW UCRT64，全部开启警告即错误。质量门禁包括 ASan+UBSan、TSan、clang-tidy、lcov、60 秒 libFuzzer，以及每个头文件单独编译检查。

## 与 Aria 系列的关系（只读核实 `dependencies.json`）

| 仓库 | 是否依赖 SQLiteCpp | 细节 |
|---|---|---|
| **AriaRead** | ✅ | 依赖名为 `sqlite_modern_cpp`，来源 `dqsjqian/SQLiteCpp` v1.2.1（1cac401）。SQLite 单独以 `sqlite3` 3.54.1 amalgamation 供应（`find_package(SQLite3Amalgamation)`），由 ForgeDeps 统一构建。代码中 `include/ariaread/database.h` 包含 `<SQLiteCpp.h>`，`src/infra/database.cpp` 使用 `std::unique_ptr<sqlite::database>` 并 catch `sqlite::sqlite_exception` |
| Aria / AriaTools / AriaAgent | ❌ | 它们的 `dependencies.json` 里都没有 sqlite 相关条目 |

AriaRead 的 `docs/dependencies.md` 提到 sqlite_modern_cpp 带有 ForgeRecipes 补丁，并且“仅适用于已审核版本”。升级 SQLiteCpp 时要同步检查配方。

## 版本演进（git log；仓库**没有 CHANGELOG**，只能看提交说明）

| 版本 | 要点 |
|---|---|
| 0.7.0（`8ae9922`） | 导入 sqlite_modern_cpp 代码，改名为 SQLiteCpp，升级到 C++23。当时需要外部 SQLite 和 ForgeDeps |
| 0.x 到 1.0.0（2026-10-26） | 重写执行路径，加入语句缓存；新增 `transaction`、`execute`、`execute_many`、`try_execute`（最初返回 `std::expected`）；`hdr/` 拆成模块化的 `include/`；内置 SQLite 3.54.1；移除 ForgeDeps |
| 1.0.0 之后（`0d8673e`） | **`try_execute` 的返回类型从 `std::expected` 改为 `sqlite::execution_result`**，原因是 Clang 搭配 libstdc++ 13 不提供 `std::expected`，MSVC 的 `_MSVC_LANG` 也偏低。只认 vendor 中的这一份 SQLite（不提供替换开关），低于 3.38 时编译期报错 |
| 1.1.0 | `busy_timeout`、`savepoint`、`configure`、`interrupt`、`progress_handler`、`blob`、`no_copy`、列元数据、`read_header`、`explain`、钩子、`define_window`、`load_extension` |
| 1.1.1 | 绑定残留修复（未绑定参数改为 NULL）；缓存改为线程安全；缓存容量 0 真正表示关闭；修复 `execute()` 失败后析构时重跑的问题；长度统一走 64 位接口；**`on_update` 回调参数改为 `std::string_view`**（旧的 `std::string` 写法仍可编译）；README 默认语言改为中文 |
| 1.2.0 | 读列时每个值只调用一次 SQLite；基准测试改为统一口径 |
| 1.2.1（2026-10-27） | 绕过缓存的语句不再带 `SQLITE_PREPARE_PERSISTENT`，无行为变化 |

**从 sqlite_modern_cpp 迁移的提示**：头文件 `<sqlite_modern_cpp.h>` 改为 `<SQLiteCpp.h>`；宏前缀 `MODERN_SQLITE_*` / `SQLITE_MODERN_CPP_*` 改为 `SQLITE_CPP_*`；要求升到 C++23 和 SQLite 3.38；boost::optional 支持已移除；事务建议改用 RAII 的 `sqlite::transaction`，不再手写 `begin;` / `commit;`。

## 常见坑 / FAQ

| 现象 | 原因 / 处理 |
|---|---|
| `#error "SQLiteCpp requires C++23"` | 编译器没有开启 `-std=c++23`；MSVC 需要 `/std:c++latest` |
| `#error ... requires SQLite 3.38` | 系统 sqlite3 太旧。改为编译 `SQLiteCpp_SQLITE_DIR` 里的 amalgamation |
| `find_package` 找到了 SRombauts 版，target 不存在 | 两者包名相同。加上版本号 `1.2`，并检查 `SQLiteCpp_DIR` |
| `column_table_name` 未定义 | 编译 SQLite 时需要定义 `SQLITE_ENABLE_COLUMN_METADATA`，而且这个宏在包含头文件时也必须可见 |
| 链接时报 `sqlite3_*` 未定义 | 安装包不包含编译好的 SQLite，需要自己 `add_library(sqlite3_core ...)` |
| 析构时抛出异常 | binder 没被执行，析构时自动执行并失败了。显式调用 `execute()`，或者用 `try_execute` |
| 单值 `>>` 抛 `more_rows` 或 `no_rows` | 单值提取要求结果恰好一行。多行结果请用 range-for 或 lambda |
| 多线程下数据竞争 | 同一个 binder 被跨线程共享，或者连接以 `NOMUTEX` 打开 |
| 内存紧张 | 不要把缓存容量设为 0，设为 1~4 |

## 与业界方案的定位对比（只列可核实的差异）

| 方案 | 差异 |
|---|---|
| SRombauts/SQLiteCpp | API 名称贴近 SQLite C API，采用面向对象的 `Statement`/`Column` 风格，C++17，编译库，支持 vcpkg、Meson，生态成熟。dqsjqian 版采用流式 `<<`/`>>`，纯头文件，C++23，自带 SQLite，并内置语句缓存 |
| sqlite_modern_cpp | 两者是同一 API 谱系，dqsjqian 版即其 C++23 演进版。上游是 C++14/17、靠 hunter 管理依赖，最后一个 release 是 2017 年。dqsjqian 版在此基础上补齐了语句缓存、RAII 事务、窗口函数、blob、backup 和钩子 |
| sqlite_orm | ORM 方案，用 C++ 类型描述表结构，不写原始 SQL 字符串，header-only、C++14 起。**许可证自 v1.7（2021-10）起为 AGPL-3.0，闭源使用需另购 MIT 授权**。dqsjqian 版是原生 SQL 加结构化映射，不是 ORM，MIT 许可 |
| SOCI | 多后端数据库访问层，支持 SQLite、PostgreSQL、MySQL 等。dqsjqian 版只支持 SQLite |
| 原生 sqlite3 C API | 性能上限所在。dqsjqian 版用 `execute_many` 或持有语句时与它持平，逐行表达式写法每行多约 33 ns（README 基准） |

## 诚实边界

- **不支持或未覆盖**：ORM 与 schema 迁移、连接池、异步（协程）接口、多数据库后端；SQLCipher 和 `flags`、`prepared_statment` 等遗留测试没有纳入 CI；没有 Meson、vcpkg、Conan 包；iOS 和 Android 只做了构建，**没有运行测试**。
- 仓库没有 CHANGELOG 和 docs 正文（只有 Doxyfile），版本说明只能从提交信息中获取。
