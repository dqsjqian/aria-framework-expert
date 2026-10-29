# C/C++ Architecture Expert

[简体中文](README.md) | English

Profession: **C/C++ Architecture Consultant**

A WorkBuddy expert package. The expert, Ari, covers language design, compiler support and architecture for C (C89 → C23, tracking the C2y draft) and C++ (C++11 → C++26). It has deep support for the Aria series (the Aria 3.2 C++23 reactive MVVM framework, AriaTools, AriaAgent, AriaRead, AriaTutorial), the Mira coroutine networking library and the SQLiteCpp storage library, and compares them with mainstream industry frameworks when you choose a stack.

## Type

Agent (a single AI expert), category: Engineering.

## Areas of expertise

| Tag | Recommended first | Industry options compared side by side |
|---|---|---|
| C/C++26 | C++23 practice in Aria, Mira, SQLiteCpp and Tutti; C library integration in AriaRead | GCC 16 / Clang 23 / MSVC, Boost, Abseil, Folly, stdexec, {fmt}, vcpkg / Conan, sanitizers |
| Cross-platform MVVM | Aria + AriaTools (a four-platform, 17-module reference architecture), AriaAgent, AriaRead, AriaTutorial | Qt Widgets / Qt Quick, Slint, Flutter, React Native, Compose Multiplatform, .NET MAUI, wxWidgets, Dear ImGui |
| Networking & storage | Mira (TCP / UDP / TLS / WebSocket / HTTP/1.1 / HTTP/2 / QUIC / HTTP/3 / SOCKS5 / DoH / MQTT), SQLiteCpp (a header-only C++23 SQLite wrapper) | Boost.Asio, libuv, libcurl, cpp-httplib, gRPC, nghttp2 / ngtcp2; SRombauts/SQLiteCpp, sqlite_orm, SOCI, RocksDB, DuckDB |

## Capabilities

- **C**: all of C23 and the C2y draft (`_Countof`, if-declarations, named loops, defer TS 25755), with a support table measured on GCC 16.2, Clang 22 / 23 and Apple Clang 21; C architecture idioms (opaque handles, error models, ABI stability); common undefined behavior; C/C++ interoperability.
- **Modern C++**: C++11 → C++26; the status and measured compiler support of C++26 features such as reflection, contracts, `std::execution` and library hardening, plus proposals that did not make C++26.
- **Architecture**: layering and dependency direction, UI pattern selection, binary boundaries, error handling, concurrency models, build and dependency management, testing and performance gates, code review checklists.
- **Aria series**: the reactive core, the binding engine and its five adapters (Qt6 / AppKit / UIKit / JNI / HTTP), coroutine async, collections / validation / navigation, ABI 2, numbered contracts, 1.x → 2.0 → 3.x migration; the real architectures of AriaTools, AriaAgent and AriaRead; ForgeDeps / ForgeRecipes dependency builds.
- **Networking and storage**: Mira's modules and protocol coverage, coroutine model, cancellation and timeouts, integration and limits; SQLiteCpp's API, statement cache, transactions, threading rules and pitfalls, and how it differs from the identically named SRombauts/SQLiteCpp.
- **Self-updating knowledge**: version questions are checked against each project's GitHub remote and the knowledge base is updated once verified; standards and compiler questions are answered with minimal compiled examples where possible.

Built-in knowledge base `skills/c-cpp-architecture-knowledge/` (11 reference documents):

| Document | Contents |
|---|---|
| `c-language.md` | C standards, C23 / C2y with measured compiler support, idioms, UB, C/C++ interop, C ecosystem |
| `cpp-standards-architecture.md` | C++11 → C++26 with measured compiler support, architecture methods |
| `aria-ecosystem.md` | The Aria series: each repository's role, version, dependencies and build |
| `industry-landscape.md` | Stack selection across the three areas of expertise |
| `aria-overview.md` | Aria's scope, modules, ABI, threading model, build, dependencies, migration |
| `aria-reactive-core.md` | Property / Computed / Effect / Subscription / batch |
| `aria-binding-adapters.md` | BindingEngine, the five adapters, ViewModel, Navigator, the HTTP adapter |
| `aria-async-collections-validation.md` | Task / AsyncCommand / Channel, ObservableList and derived views, Validator |
| `aria-contracts-faq.md` | Numbered contracts, error model, diagnostics, FAQ and migration FAQ |
| `mira-networking.md` | The Mira networking library |
| `sqlitecpp-storage.md` | The SQLiteCpp storage library |

Version snapshot (checked against each project's GitHub remote on 2026-10-29): Aria 3.2.6, AriaTools 1.1.0, AriaAgent 0.2.0, AriaRead 0.3.1, AriaTutorial 0.2.0, Mira 1.2.1, SQLiteCpp 1.2.1, Tutti 2.1.1, ForgeDeps 0.4.10, ForgeRecipes 0.1.8. Standalone code examples in the knowledge base were compiled against the headers of those versions.

## Try asking

- Design a layered cross-platform app architecture with Aria + Mira + SQLiteCpp
- Wrap my C library in a safe C++23 interface while keeping a stable C ABI
- Mira, Boost.Asio or libcurl for my networking layer? Compare them for my scenario
- I'm upgrading from Aria 2.x to 3.2 — walk me through the breaking changes
- Which compilers support C++26 reflection and contracts today? Show minimal examples

## Related projects

[Aria](https://github.com/dqsjqian/Aria) · [AriaTools](https://github.com/dqsjqian/AriaTools) · [AriaAgent](https://github.com/dqsjqian/AriaAgent) · [AriaRead](https://github.com/dqsjqian/AriaRead) · [AriaTutorial](https://github.com/dqsjqian/AriaTutorial) · [Mira](https://github.com/dqsjqian/Mira) · [SQLiteCpp](https://github.com/dqsjqian/SQLiteCpp) · [Tutti](https://github.com/dqsjqian/Tutti) · [ForgeDeps](https://github.com/dqsjqian/ForgeDeps) · [ForgeRecipes](https://github.com/dqsjqian/ForgeRecipes)

## Installation

- **Local use**: put the whole directory into WorkBuddy's local expert directory (`<WorkBuddy config dir>/plugins/marketplaces/my-experts/plugins/`); after the expert manager validates and registers it, it appears under Expert Center → My Experts.
- **Open platform**: zip this repository (top-level directory `aria-framework-expert/`) and submit it on the WorkBuddy open platform.

```bash
git clone https://github.com/dqsjqian/aria-framework-expert.git
zip -r aria-framework-expert.zip aria-framework-expert -x "aria-framework-expert/.git/*" "aria-framework-expert/.gitignore"
```

The package follows the WorkBuddy expert specification: `plugin.json` is the configuration file (in the hidden configuration directory the specification requires), `agents/` holds the expert prompt, `skills/c-cpp-architecture-knowledge/` the knowledge base and `avatars/` the avatar. The expert keeps its internal identifier `aria-framework-expert` so that existing installations upgrade in place.

## Changelog

- **3.0.0** (2026-10-29): became the C/C++ Architecture Consultant; added C support; knowledge base upgraded to Aria 3.2.6 with six new documents (C, C++26 and architecture, the Aria series, stack selection, Mira, SQLiteCpp).
- **2.0.1**: Aria 2.0 framework expert.

## License

[MIT](LICENSE) © 2026 dqsjqian
