# 响应式核心速查（Property / Computed / Effect / Subscription / batch / untracked）

> 基准版本：Aria 3.2.6（tag v3.2.6，commit 85d3d38，2026-10-29 与 GitHub 远端 main 核对）
> Include：`#include "aria/aria.hpp"`（伞头）或 `"aria/reactive/reactive.hpp"`。契约：`docs/reference/lifecycle.md`（L-N）。指南：`docs/guide/reactive-core.md`。
> 3.x 说明：本层 API 自 2.0 起无破坏性变更；3.0 起要求 C++23；3.0.1 做了若干异常安全/重入加固（见文末「3.x 行为修订」）。

五个日常原语 + 两个工具：`Property<T>`（源节点）、`Computed<T>`（派生节点）、`Effect`（自动副作用）、`Subscription`（RAII 订阅柄）、`batch()`（事务）、`untracked()`（逃生舱）。全部共享同一个进程级 DAG 依赖图引擎（ABI 2 起由 `aria_abi` 跨动态库共享同一张图，见总览）。这些名字在 `aria::reactive` 中定义，并由 `reactive.hpp` 以 `using` 导入 `aria::`（`aria::batch` 与 `aria::reactive::batch` 等价）。

## Property\<T\>

可观察的真相之源。构造 `aria::Property<T> name{"Alice"};`

**读**：
```cpp
std::string n  = name.get();             // 自动追踪：Computed/Effect 内读到即登记依赖
const std::string& nr = name.get_ref();  // 引用版，避免拷贝（仍登记依赖）
std::string snap = name.peek();          // 非追踪快照，永不登记依赖（另有 peek_ref()）
std::string s = name;                    // 隐式转换，等价 get()
```

**写**：
```cpp
name.set("Bob");     // 相等性门控：值相同则静默 no-op（L-21 / PERF-4）
name = "Charlie";    // operator= 即 set()
items.mutate([](std::vector<int>& v){ v.push_back(42); });  // 原地改：永远触发（无法检测 no-op）
```

**观察**（三个都返回 `Subscription`，均 `[[nodiscard]]`）：
```cpp
auto s1 = name.on_changed([](const std::string& v){...});   // 只在变化时触发（初值不触发，L-19）
auto s2 = age.bind([](int v){...});                          // 立即用当前值触发一次 + 之后每次变化
auto s3 = age.observe([](const int& old_v, const int& new_v){...});  // (旧值, 新值)
```

**约束**：
- `T` 必须 copyable + equality-comparable（`operator==`）。
- Property **不可拷贝不可移动**——图身份绑定在其地址上。
- **单线程**：读写必须在图线程（L-1），debug 构建违规即 assert；3.0.1 起 `mutate()` 也显式断言图线程。

## Computed\<T\>

只读派生值。依赖**自动发现**——compute lambda 里每次 `get()` 都成为上游：

```cpp
aria::Property<double> subtotal{100.0}, tax_rate{0.08};
aria::Computed<double> total{[&]{ return subtotal.get() * (1.0 + tax_rate.get()); }};
subtotal = 200.0;          // total.get() == 216.0，自动重算
```

- **构造时立即求值一次**并登记依赖（不需要 T 可默认构造，`performance.md`）。
- **惰性 + 记忆化**：clean 时 `get()` 返回缓存；上游真变才在 flush 时按拓扑序重算（L-22）。
- **glitch-free**：重算结果与旧值相等则传播停止。
- **条件依赖**：分支没走到的属性不登记（L-17：动态依赖会被释放）。
- 观察 API 与 Property 相同（`on_changed` / `bind` / `observe`）。
- 失败的重算不留悬挂边、保留原依赖集（2.0 修复）。
- 成本：单次求值 O(D²) 最坏依赖去重 + O(D) 边调和；flush 每轮 O(P log P)（`performance.md`）。

## Effect

自动副作用（≈ MobX autorun / SolidJS createEffect）。**构造时立即运行一次**（L-18），之后任何被追踪读取变化即重跑：

```cpp
aria::Effect logger{[&]{ std::cout << "count = " << count.get() << "\n"; }};
```

- `eff.stop()` 断开图连接；**不可重启**——要重来就新建 Effect。
- 转入订阅袋：`bag += std::move(eff).into_subscription();`

## Subscription / SubscriptionBag

统一 RAII 柄，覆盖响应式图、信号、EventBus 所有观察点。**move-only 所有权**（2.0 起）。

```cpp
aria::Subscription sub = name.on_changed(...);
sub.active();                 // true
sub.release();                // 立即断开（析构也会自动断开）
auto sub2 = std::move(sub);   // 转移所有权
```

- `detach()` 已于 2.0 删除——用 `release()`。
- `release()`：跳过所有尚未启动的回调（含当前 fanout 中排在后面的）；已在执行的回调允许跑完（L-10、L-13）。
- 响应式图背书的 Subscription **必须在图线程析构**（L-11）。
- `SubscriptionBag` 按注册**逆序**断开（L-12），允许回调内重入 `clear()`。
- 断开回调支持 move-only 捕获。`inplace_function` 仍要求目标 copy-constructible；3.0.1 起其内部存储为 `mutable`，**带可变捕获的目标跨调用保留状态**。
- 3.0.1 起 `function_ref` / `inplace_function` **拒绝会返回悬垂引用的可调用体**（返回引用类型但实际绑定到临时量时编译期不匹配）——升级后若遇到重载决议失败，检查返回类型是否经由临时量转换（`modules/core/include/aria/function_ref.hpp`）。

**⚠️ 最高频坑**：`label.bind(...);` 丢弃返回值 → 临时 Subscription 立即析构 → 订阅当场断掉。**必须接住返回值**（UI 里通常存成 View 成员 / `track()` / `adopt()`）。发布者先死也安全：断开走 weak 校验，静默 no-op。

## Dispatcher

- `aria::main_dispatcher()` 返回**拥有型 `shared_ptr<IDispatcher>` 快照**——接住持有再用。
- `SimpleDispatcher::pump()` / `run_one()` **只能在创建它的线程上跑**；worker 线程只能 `post()`。
- 极端延迟会饱和（不溢出）；回调里可以安全销毁自己的 dispatcher。
- 3.2.0 性能优化：单次主线程派发不再每次分配 vector，去掉 owner 自唤醒；批顺序、异常路由、跨线程唤醒与阻塞等待语义保持（`CHANGELOG.md 3.2.0`）。
- 启动顺序：executor 与 timer 先于 ViewModel 构造（L-5b）。

## Container（IoC）

- 重注册**同时替换 value 与 lifetime 模式**。
- 工厂在多次 resolve 之间**保留可变状态**；工厂在注册表锁**之外**执行（可再 resolve）。
- 拆除按**注册逆序**（L-40）。
- 依赖「每次调用拿全新副本」的代码要把重置写显式。

## batch() / BatchScope

把多次写合并为一次 flush，观察者只收到一次通知、看不到中间态：

```cpp
aria::batch([&]{ x = 10; y = 20; });     // sum 只重算一次
{ aria::BatchScope b; x = 10; y = 20; }  // RAII 版，作用域结束 flush（flush 抛错时析构仍收尾）
```

不包 batch：x、y 各触发一轮重算，Computed 链会闪过中间值。若最终结果与当前值相等，一次通知都不发（README 示例）。

## untracked() / UntrackedScope

在 Computed/Effect 内「读但不建立依赖边」：

```cpp
aria::Computed<int> c{[&]{
    int tracked = a.get();
    int snap = aria::untracked([&]{ return b.get(); });  // b 变化不触发重算
    return tracked + snap;
}};
```

## dep()

显式依赖提示——「没读它，但把它当上游」：`aria::dep(click_count);`（`reactive/computed.hpp`）。99% 场景 `get()` 自动追踪已够用。

## CircularDependencyError

图检测到环（A→B→A）抛 `aria::CircularDependencyError`（`std::runtime_error` 派生），是硬错误——解环。

## 运算子（`property_ops.hpp`，Rx 风格）

`combine_latest(a, b, fn)`、`scan(source, seed, reducer)`、`distinct_until_changed(p)`、`debounce(p, 300ms, scheduler)`、`throttle(p, cooldown, scheduler)`（后两者需 `IDelayedScheduler`）。映射关系见 `docs/migration-guide.md`。

## 速查表

| 原语 | 类型 | 产值 | 自动追踪 | 首次立即跑 |
|---|---|---|---|---|
| `Property<T>` | 源 | ✓ | — | — |
| `Computed<T>` | 派生 | ✓ | ✓ | ✓ |
| `Effect` | 反应 | ✗ | ✓ | ✓ |
| `batch()` | 事务 | — | — | — |
| `untracked()` | 逃生舱 | — | — | — |
| `dep()` | 提示 | — | — | — |

## Command（放核心层，与绑定联动）

```cpp
aria::Command<> increment{[&]{ count = count.get() + 1; }};
aria::Command<std::string> open{[&](const std::string& path){ /*...*/ }};
aria::Command<> logout{[]{ /*...*/ }, [&]{ return logged_in.get(); }};  // 第二参 = can_execute 谓词
increment();          // 或 increment.execute()；can_execute 为 false 时 no-op
```

- can_execute 谓词自身接入响应式图（内部 Effect），`bind_command` 后 enabled 自动联动；谓词跑在图线程。
- Command 析构会取消**每一个**在途调用的 token。异步版本见 `AsyncCommand<R, Args...>`（async 参考文档）。

## 3.x 行为修订（`CHANGELOG.md 3.0.1`）

| 修订 | 影响 |
|---|---|
| `Property::mutate()` 回调抛异常时**仍通知下游失效**，再重抛原异常 | 原地编辑无法回滚，读者不会看到「值已改但缓存未失效」；通知自身的异常走 `report_callback_failure` |
| `mutate()` 显式 `assert_on_graph_thread()` | 以前漏过的跨线程 mutate 在 debug 下会直接断言 |
| 拒绝返回悬垂引用的 callable wrapper | 部分以前能编译的 `function_ref<const T&()>` 用法变成编译错——修返回类型 |
| 初始绑定期间产生的更新被保留 | 投影/原生 setter 同步规范化源值时不再丢更新（binding 层，见绑定文档） |
| 部分失败的变更后，响应式依赖会被失效 | 异常路径一致性提升 |
