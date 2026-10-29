# 异步协程 / 集合 / 校验速查

> 基准版本：Aria 3.2.6（tag v3.2.6，commit 85d3d38，2026-10-29 与 GitHub 远端 main 核对）
> Include：`"aria/async/task.hpp"`、`"aria/async/async_command.hpp"`、`"aria/async/scope.hpp"`、`"aria/observable_list.hpp"`、`"aria/derived/*.hpp"`、`"aria/validator.hpp"`、`"aria/binding/form.hpp"`。契约：lifecycle.md（L-N，协程竞态模型、L-35~L-38）、list-diff-contract.md（文档内编号 `D-N`，README 称 `LD-N`）。指南：`docs/guide/async.md`、`collections.md`、`validation.md`。
> 3.x 要点：async 层 3.0 起要求 C++23；`AsyncCommand` 构造签名去掉了旧示例里的空 `{}` 占位参数；`when_all` 支持 `Task<void>` 子任务（`std::monostate` 槽）；3.0.1/3.2.0 多项协程、Channel、执行器与派生列表加固。

## Task\<T\>（C++ 协程）

- **lazy、single-shot**：不 `co_await` 就什么都不做；异常存在协程帧内、在 `co_await` 点重抛。
- `Task<void>`：`co_return;`。3.0.1 修复对空 Task 的 await。
- fire-and-forget：`task.start_detached()`——**无生命周期保护**；有作用域需求用 `CoroutineScope` / `ViewModelScope`。分离任务帧经编译器管理销毁（2.0 修掉 MSVC ASan 抓到的 final-awaiter UAF）。
- `blocking_get()` 只适用于在一次同步 resume 内完成的任务，**不能**用来等线程池上的工作（`docs/getting-started.md`）。

### 线程跳转

```cpp
aria::async::Task<UserProfile> load_profile(aria::async::IExecutor& worker, aria::async::IExecutor& ui, int uid) {
    co_await aria::async::schedule_on(worker);   // 跳后台线程（阻塞 OK）
    auto data = fetch_from_db(uid);
    co_await aria::async::schedule_on(ui);       // 切回 UI/图线程
    co_return data;
}
```

后台线程写 Property 的正规姿势：`co_await schedule_on(ui)` 之后再赋值。执行器：`ThreadPoolExecutor`、`MainThreadExecutor`、`InlineExecutor`、`VirtualTimeExecutor`（测试）。3.2.0 起线程池空闲等待/唤醒改用 C++ atomic wait/notify（语义不变，减少锁往返）；3.0.1 修复执行器关停/线程归属与虚拟时钟溢出。

## CancellationToken / CancellationSource

```cpp
aria::async::CancellationSource src;
auto tok = src.token();
while (!tok.is_cancelled()) {
    tok.throw_if_cancelled();   // 抛 OperationCancelled
    do_work();
}
```

三轴取消模型见 L-38.1；3.0.1 修复取消回调路径。⚠️ `docs/guide/async.md` 示例使用 `co_await aria::async::sleep_for(100ms)`，但 v3.2.6 的 `modules/*/include` 中**搜不到 `sleep_for` 定义**——需查证；定时等待请以 `IDelayedScheduler` / `VirtualTimeExecutor` / `timeout.hpp` 实际接口为准。

## AsyncCommand\<R, Args...\>

三态异步动作，UI 直接绑的 Property：

| Property | 类型 | 含义 |
|---|---|---|
| `is_executing` | `Property<bool>` | 在途 |
| `last_error` / `last_error_message` | `Property<optional<Error>>` / 文本 | 最近错误 |
| `last_result` | `Property<optional<R>>` | 最近成功结果（R≠void） |

```cpp
aria::async::AsyncCommand<SearchResult, std::string> search{ui, worker,
    [&worker](std::string q) -> aria::async::Task<SearchResult> {
        co_await aria::async::schedule_on(worker);
        co_return perform_search(q);
    }, aria::async::AsyncCommandPolicy::LatestOnly};   // 第 4 参可省，默认 Parallel
search.execute("hello");
```

- **结果类型约束**：`last_result` 是 `Property<std::optional<R>>`，所以 `R` 必须满足 `PropertyValue`（可拷贝 + 可 `==` 比较），自定义结构体记得 `bool operator==(const R&) const = default;`（实测缺失时编译失败）。
- **构造签名**（`async_command.hpp`）：`AsyncCommand(ui, worker, action, policy = Parallel)`；有模板化重载（编译期检查 `ui` 是图线程执行器、`worker` 可跑后台、禁止 `InlineExecutor` 当 ui 配异线程 worker）和 `IExecutor&` 类型擦除重载（DI 场景）。2.x 文档里的 `(ui, worker, action, {}, policy)` 写法已不适用。
- **可取消**：action 首参接 `CancellationToken`。
- **并发策略**：`Parallel`（默认）/ `LatestOnly`（新 execute 取消在途——即输即搜）/ `DropIfRunning`（忙时忽略——防双击）。
- **action 目标所有权（3.2.0，L-37）**：命令**只保留一个 action 对象**，跨调用复用，可变捕获的状态在顺序调用间保留；启动调用不再拷贝用户 callable；调用帧共享持有 action 与参数元组直到完成（命令析构后也安全）。`Parallel` 或 `LatestOnly` 取消后仍在收尾的工作可能**在不同 worker 上并发访问同一 action**——可变捕获须自行同步或固定在单线程执行器上。
- 命令析构会取消**每一个**在途调用的 token。
- UI 惯用法：`last_error_message` 用 `bind_text_projected`，`last_result` 用 `bind_optional_text`。写回线程契约见 L-5。

## ViewModelScope / CoroutineScope（结构化并发）

```cpp
class PollingVm : public aria::binding::ViewModel {
    aria::binding::ViewModelScope scope_;
public:
    PollingVm() { scope_.attach(*this); }
    void start() { scope_.launch([this](aria::async::CancellationToken tok) -> aria::async::Task<void> { /*...*/ co_return; }); }
};
```

- VM 析构 → `cancel_and_join()`（默认 5s 上限）→ 卡死协程按泄漏上报（L-35）。
- 控制台/非 VM 场景用 `aria::async::CoroutineScope`：`launch_simple(task)` + 结束时 `cancel_and_join()`（返回 bool，L-36）。
- scope join 计入挂起中的子协程；timer/timeout 路径保留完成状态（2.0 修复）。

## 组合子与竞态模型

| 组合子 | 语义 |
|---|---|
| `when_all(t1, t2, ...)` | 等全部完成，`auto [a,b,c] = co_await when_all(...)`；**`Task<void>` 子任务占 `std::monostate` 槽**（3.0.1，可与有值任务混用）；非竞态式，父句柄原子交接 |
| `when_any(t1, t2)` | 首个完成者胜出 |
| `when_any_cancellable(...)` | 工厂接 token，赢家取消输家 |
| `with_timeout(task, 5s[, OnTimeout])` | 超时截止 |
| `retry` / `safe_run` | 见 `retry.hpp` / `safe_run.hpp` |

**with_timeout 两大坑**（RaceSlot 两阶段发布：CLAIM → WRITE→PUBLISH → RESUME）：
- `OnTimeout::Cancel` + **无 token 工厂 = 只能事后观察**：等内层自然结束后才把结果标为 `TimeoutError`。要 fail-fast：传接 token 的工厂或改 `OnTimeout::Fail`。
- `OnTimeout::Fail` 对不协作的内层：内层继续 detached 跑，父协程及时解锁、迟到结果不投递，但杀不掉不检查 token 的内层。
- timeout/retry 包装器可能在绑定一次调用时**单独拷贝内层工厂**（不在 AsyncCommand 单 action 保证范围内，L-37）。

## Channel

- 每个投递值由**被选中的接收者**拥有；未选中的消费者不会观察到假 EOF。
- capacity-zero 通道双向会合；`close()` 保留缓冲与已分配值，释放停泊的 sender/receiver；`await_resume()` 不重读通道。
- 3.0.1 修复 Channel 异常路径。

## AsyncResource

缓存 + 去重的异步资源（L-38）；3.0.1 修复观察者重入。

## ObservableList\<T\>

线程安全（内部 `shared_mutex`）可观察集合，元素是 `std::shared_ptr<T>`，**不是响应式图节点**。

```cpp
// 要接派生视图时源必须是 shared_ptr（工厂签名收 std::shared_ptr<Source>）；ObservableList 不可拷贝、不可移动
auto tasks = std::make_shared<aria::ObservableList<Task>>();
tasks->push_back(std::make_shared<Task>("Write docs"));
auto t = tasks->emplace_back();                       // 返回新元素 std::shared_ptr<Task>
tasks->insert(0, x); tasks->remove_at(1); tasks->replace_at(0, y); tasks->move(1, 0);
tasks->insert_range(pos, first, last); tasks->remove_range(0, 3);
tasks->remove_all([](const Task& t){ return t.done; }); // 谓词收 const T&，返回删除个数
auto p = tasks->at(0);                                // 越界抛 std::out_of_range
auto i = tasks->index_of(p.get());                    // 参数是 const T*；找不到返回 size()
auto snap = tasks->snapshot();                        // 拷贝当前 shared_ptr 序列
auto sub = tasks->observe([](const aria::ListChange<Task>& c){ /* switch(c.kind) */ });
tasks->clear();
```

> 以上片段与派生视图片段已于 2026-10-29 用 Aria v3.2.6 头文件编译、链接并运行验证；`at()` 越界抛异常、`index_of` 未命中返回 `size()` 为实测行为。

- `T` 提供返回 `Subscription` 的 `on_changed` 成员时自动产生 `ItemChanged`（D-13）。3.2.0 起非响应式元素类型跳过订阅安装簿记（更快）。

### ListChangeKind 六种事件（拥有载荷）

| Kind | 字段 | 含义 |
|---|---|---|
| `Insert` | index=插入点 | 加一项 |
| `Remove` | index=移除前槽位 | 删一项 |
| `Replace` | index | 换一项 |
| `Move` | index=目的地，from_index=源 | 挪一项 |
| `Reset` | index=0，带非空 `snapshot` | 整表替换——观察者丢弃镜像 |
| `ItemChanged` | index | 元素自身 on_changed 触发 |

**拥有语义（2.0 起，消费者必须适配）**：
- `ListChange<T>::item` 是 **`shared_ptr<T>`**；每个 Reset 带非空拥有 `snapshot`（D-3）。
- **用事件载荷，不要重读源**：批事件索引描述接收者的增量镜像（D-11），`source.at(index)` 拿不回中间载荷。
- 重入编辑跟随当前 batch；新订阅者跳过订阅前已提交的事件（D-20）；handler 抛异常被吞并上报（D-21）。
- 重复出现的共享 item 句柄是独立记录，各自收到 ItemChanged。

### reconcile（服务端快照同步，D-14）

```cpp
struct ById { int operator()(const Task& t) const noexcept { return t.id; } };
tasks->reconcile(std::move(fresh), ById{});   // fresh: std::vector<std::shared_ptr<Task>>，返回编辑数
```

- 按 key 求最小编辑流（Insert/Remove/Replace/Move），**不发 Reset**——保住选择/滚动/展开态。`clear()+insert_range` 轮询是反模式。
- 服务端每次给新对象就必须传 `key_of`（默认按地址）。下游有 Computed 时包进 `reactive::batch`。

### 派生视图（都是 ListSource，可链，D-40/D-41）

`FilteredList` / `SortedList` / `MappedList<U>` / `DistinctList` / `PagedList` / `GroupedList`。链式工厂函数（`aria/derived/*.hpp`，需单独 include，`aria/aria.hpp` 不包含；参数为 `std::shared_ptr<Source>`，返回 `std::shared_ptr<视图>`）：

```cpp
auto evens  = aria::filtered(tasks,  [](const Task& t){ return !t.done; });
auto unique = aria::distinct<int>(evens, [](const Task& t){ return t.group_id; });
auto asc    = aria::sorted(unique,  [](const Task& a, const Task& b){ return a.due < b.due; });
auto page   = aria::paged(asc, 20);
auto rows   = aria::mapped<RowViewModel>(asc, [](const Task& t){ return std::make_shared<RowViewModel>(t); });
```

每级强引用源，保住 `page` 即保住整条链。

**投影回调与恢复（D-29，3.0.1）**：mapper/谓词/比较器在视图读写锁**之外**运行（可读视图）；嵌套源编辑跟随当前更新。回调在源更新中抛异常时：经 ABI 回调边界上报、保留之前的有效投影；之后一次成功事件（或替换谓词/比较器）从镜像重建并先发一个拥有型 **Reset**，再恢复增量。回调对应用对象的副作用不回滚。

**SortedList 契约**：等效键顺序变化发 **Move**；每次源编辑后保持稳定源顺序；比较前校验活动键（含已改未通知的键）。3.0.1 起排序视图额外保留 O(n) 工作布局以安全比较/提交；3.2.0 起具体比较器类型在整次扫描中可内联、有序尾插先预分配再无异常提交（擦除型 Comparator 重载仍支持）。代价：随机键追加约 +106%（2.0 对 1.2.1，`performance.md`）。

**Selection**：`Selection<T>` / `MultiSelection<T>`（`aria/selection.hpp`），`bind_to(list)` 跟随源变化，Reset 时自清。

## 校验（Validator / FormValidator）

响应式优先：Validator 绑 `Property<T>`，源变即全规则重跑，结果暴露为 `Property<ValidationState>`。

```cpp
aria::Property<std::string> email{""};
aria::Validator<std::string> v{email, "email"};          // 第二参 field_path 可省
v.must([](auto& s){ return !s.empty(); }, "Email is required")
 .must([](auto& s){ return s.find('@') != std::string::npos; }, "Email must contain @");
const auto& st = v.state().get();                        // valid / errors / touched / dirty / pending
```

- 规则链：`must` / `rule`（错误级）、`should` / `warning`（警告级）、`rules({...})` 批量；表单状态 `touch()` / `reset_touched()` / dirty 跟踪。
- **异步规则**：`aria/async/async_validator.hpp`（`AsyncValidator<T>`，如用户名占用检查，state 的 `pending` 反映在途）。
- `ValidationState` + `ValidationKey` 定位字段；`ValidationResult` 为遗留别名。
- 表单聚合（`aria/binding/form.hpp`）：`FormField<T>`、`FormGroup`、`FormValidator`，逐字段 `track(field)`，聚合 valid/dirty/pending/error。**3.0.1 起聚合器不拥有字段**：被跟踪字段先销毁时，读取经 `NodeHandle` 判活并返回默认值（valid=true、dirty=false、pending=false、无错误），存活字段仍可触发重算——不再访问已销毁字段（自定义响应式源仍由调用方管理寿命）。

Cookbook：01 表单同步+异步规则、02 跨字段规则、03 列表筛选排序选择、04 下拉刷新无限滚动、06 超时竞态、07 view 销毁取消。
