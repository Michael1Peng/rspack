# Rspack 源码分析：Cargo.toml 分析与初步架构

本文档记录了对 Rspack 项目根目录及核心 Crate (`rspack`, `rspack_core`) 的 `Cargo.toml` 文件的分析结果，并基于此绘制了初步的 Rspack 整体架构图。

## 1. 根 `Cargo.toml` 分析

文件路径: `/Cargo.toml`

**关键信息:**

*   **Workspace Members:** 工作区包含 `crates/*` 下的所有 crate 以及 `xtask`。这表明 `crates` 目录是 Rspack Rust 代码的核心组织单元。
*   **Excluded Member:** `crates/node_binding` 被明确排除在工作区之外。这通常是为了避免 Cargo 的特性统一（feature unification）问题，特别是在涉及 NAPI 绑定时。`node_binding` 可能需要单独构建或有特殊的构建配置。
*   **Workspace Dependencies:** 定义了在整个工作区内共享的依赖版本，如 `napi`, `serde`, `swc_core`, `tracing` 等，有助于保持依赖一致性。

## 2. `crates/rspack/Cargo.toml` 分析

文件路径: `crates/rspack/Cargo.toml`

**关键信息:**

*   **Core Dependency:** 直接依赖 `rspack_core`，表明 `rspack_core` 提供核心编译能力。
*   **Plugin Dependencies:** 依赖大量 `rspack_plugin_*` crates (e.g., `_asset`, `_css`, `_javascript`, `_html`, `_devtool`, `_runtime`, `_externals`, `_remove_empty_chunks`)，表明 `rspack` crate 负责集成和协调这些内置插件。
*   **Other Rspack Dependencies:** 依赖 `rspack_ids`, `rspack_loader_sass`, `rspack_symbol`, `rspack_tracing`, `rspack_error` 等辅助 crate。
*   **Async Runtime:** 使用 `tokio` 作为异步运行时，依赖 `async-trait`。
*   **Testing:** 包含用于测试和基准测试的开发依赖。

**结论:** `rspack` crate 作为顶层协调器，集成 `rspack_core` 和各种内置插件，提供完整的打包能力。

## 3. `crates/rspack_core/Cargo.toml` 分析

文件路径: `crates/rspack_core/Cargo.toml`

**关键信息:**

*   **基础数据结构和类型:** 依赖 `bitflags`, `dashmap`, `derivative`, `dyn-clone`, `hashlink`, `indexmap`, `itertools`, `once_cell`, `petgraph`, `rustc-hash`, `ustr` 等，定义了编译过程中的核心数据结构（如图、哈希表、映射等）。
*   **AST 处理:** 大量依赖 `swc_core` 及其特性（ecma, css），以及 `swc_css`, `swc_emotion`，负责处理 JS 和 CSS 的 AST。
*   **模块解析:** 依赖 `nodejs-resolver` 和 `sugar_path`，处理模块路径解析。
*   **Loader Runner 集成:** 依赖 `rspack_loader_runner`，集成 Loader 执行机制。
*   **源码表示:** 依赖 `rspack_sources`，用于表示和操作生成代码。
*   **共享组件:** 依赖 `rspack_database`, `rspack_error`, `rspack_regex`, `rspack_symbol`, `rspack_tracing` 等基础功能 crate。
*   **并发和异步:** 依赖 `crossbeam`, `futures`, `rayon`, `tokio`，大量使用并发和异步编程。

**结论:** `rspack_core` 是 Rspack 的引擎核心，负责定义核心数据结构、AST 处理、模块解析、Loader 执行、源码表示、插件钩子定义以及核心编译逻辑。

## 4. 初步架构图 (Mermaid)

```mermaid
graph TD
    subgraph NodeJs World ["Node.js World (JavaScript/TypeScript)"]
        direction TB
        RspackCLI[""@rspack/cli\n(CLI Interface)""]
        RspackDevServer[""@rspack/dev-server\n(Development Server)""]
        UserConfig[""User Config\n(rspack.config.js)""]
        NodeBindingJS[""crates/node_binding/binding.js\n(JS side of Bridge)""]

        UserConfig --> RspackCLI
        UserConfig --> RspackDevServer
        RspackCLI --> NodeBindingJS
        RspackDevServer --> NodeBindingJS
    end

    subgraph Rust World ["Rust World (Core Logic)"]
        direction TB
        subgraph NodeBridge ["Node.js Bridge"]
            NodeBindingRust[""crates/node_binding\n(NAPI Rust Bridge)""]
        end

        subgraph Facade ["Facade & Coordination"]
             RspackLib[""crates/rspack\n(Integrator, Plugin Runner)""]
        end

        subgraph CoreEngine ["Core Engine"]
            RspackCore[""crates/rspack_core\n(Compiler, Compilation, AST,\nResolver, Module/Chunk Graph,\nPlugin Hooks, Core Structures)""]
        end

        subgraph Plugins ["Built-in Plugins"]
            direction LR
            PluginAsset["rspack_plugin_asset"]
            PluginCSS["rspack_plugin_css"]
            PluginJS["rspack_plugin_javascript"]
            PluginHTML["rspack_plugin_html"]
            PluginDevtool["rspack_plugin_devtool"]
            PluginRuntime["rspack_plugin_runtime"]
            PluginExternals["rspack_plugin_externals"]
            PluginSplitChunks["rspack_plugin_split_chunks"]
            OtherPlugins["..."]
        end

        subgraph ToolingUtils ["Tooling & Utilities"]
            direction LR
            LoaderRunner["crates/loader_runner"]
            Resolver[""nodejs-resolver\n(External Crate)""]
            SWC[""swc_core / swc_css\n(External Crates)""]
            Error["rspack_error"]
            Tracing["rspack_tracing"]
            Database["rspack_database"]
            Utils["rspack_util"]
            Ids["rspack_ids"]
            Symbol["rspack_symbol"]
            Sources["rspack_sources"]
        end

        NodeBindingJS -- Calls --> NodeBindingRust
        NodeBindingRust -- Calls --> RspackLib

        RspackLib -- Uses --> RspackCore
        RspackLib -- Manages/Runs --> Plugins

        RspackCore -- Defines Hooks For --> Plugins
        RspackCore -- Uses --> LoaderRunner
        RspackCore -- Uses --> Resolver
        RspackCore -- Uses --> SWC
        RspackCore -- Uses --> Error
        RspackCore -- Uses --> Tracing
        RspackCore -- Uses --> Database
        RspackCore -- Uses --> Utils
        RspackCore -- Uses --> Ids
        RspackCore -- Uses --> Symbol
        RspackCore -- Uses --> Sources

        Plugins -- Interact With --> RspackCore # (Via Hooks & Core APIs)
        LoaderRunner -- Uses --> RspackCore # (For Context/Utilities)

    end

    style NodeJs World fill:#D0E0F0,stroke:#333,stroke-width:1px
    style Rust World fill:#D0F0D0,stroke:#333,stroke-width:1px
    style NodeBridge fill:#E0D0F0,stroke:#333,stroke-width:1px
    style Facade fill:#F0E0D0,stroke:#333,stroke-width:1px
    style CoreEngine fill:#FFDAB9,stroke:#333,stroke-width:2px,stroke-dasharray: 5 5
    style Plugins fill:#E0F0E0,stroke:#333,stroke-width:1px
    style ToolingUtils fill:#F0F0D0,stroke:#333,stroke-width:1px

```

**下一步:** 深入分析 `crates/rspack_core` 中的核心组件，首先从 `Compiler` 开始。
