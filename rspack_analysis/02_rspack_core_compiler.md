# Rspack 源码分析：`rspack_core` 之 `Compiler`

本文档分析了 `crates/rspack_core/src/compiler/mod.rs` 中定义的 `Compiler` 结构体及其相关实现。`Compiler` 是 Rspack 构建流程的顶层协调器。

## 1. `Compiler` 结构体定义

```rust
#[derive(Debug)]
pub struct Compiler {
  pub options: Arc<CompilerOptions>, // 编译选项 (共享)
  pub compilation: Compilation,       // 当前构建过程实例
  pub plugin_driver: SharedPluginDriver, // 插件驱动 (共享, 读写锁)
  pub loader_runner_runner: Arc<LoaderRunnerRunner>, // Loader 执行器 (共享)
  pub cache: Arc<Cache>,             // 缓存实例 (共享)
}
```

*   **职责:** `Compiler` 实例代表了一个完整的 Rspack 构建环境。它持有所有必要的配置、状态和工具（插件驱动、Loader 执行器、缓存）。
*   **共享与并发:** 大量使用 `Arc` 来共享不可变数据（如 `options`）或需要跨线程访问的组件（`plugin_driver`, `loader_runner_runner`, `cache`）。`plugin_driver` 使用 `RwLock` 进行内部可变性管理，允许多个读取者或一个写入者，确保插件状态的线程安全访问。
*   **`Compilation` 关联:** `Compiler` 持有 `Compilation` 实例。`Compilation` 代表一次具体的构建活动（从入口分析到资源输出），每次调用 `build` 时通常会创建一个新的 `Compilation`。

## 2. 核心方法与构建流程

### 2.1 `new()` - 构造函数

*   接收 `CompilerOptions` 和插件列表 (`Vec<Box<dyn Plugin>>`)。
*   初始化并组装所有核心组件：`ResolverFactory`, `PluginDriver`, `LoaderRunnerRunner`, `Cache`, `Compilation`。
*   将配置和共享组件包装在 `Arc` 或 `Arc<RwLock<>>` 中。

### 2.2 `run()` - 简单入口

*   调用 `build()` 方法启动构建。

### 2.3 `build()` - 构建入口

*   **生命周期管理:** 管理缓存状态（`end_idle`, `begin_idle`）。
*   **创建新 `Compilation`:** 为本次构建创建一个新的 `Compilation` 实例，传递必要的共享组件。
*   **触发 `this_compilation` 和 `compilation` 钩子:** 通过 `plugin_driver` 通知插件新的编译过程已开始。
*   **设置入口:** 调用 `compilation.setup_entry_dependencies()` 确定构建的起点。
*   **调用 `compile()`:** 获取入口依赖，启动核心编译流程。
*   **错误报告 (Debug):** 在 Debug 模式下，可选地使用 `Stats` 输出诊断信息。

### 2.4 `compile()` - 核心编译流程

*   **调用 `compilation.make()`:** 触发模块构建阶段。这是 Rspack 最核心和复杂的部分之一，包括：
    *   从入口点开始递归解析模块依赖。
    *   对每个模块运行相应的 Loader (`loader_runner_runner`)。
    *   解析模块代码（使用 SWC）生成 AST。
    *   分析 AST，提取依赖关系。
    *   构建模块图 (`ModuleGraph`)。
*   **Tree Shaking (可选):** 如果启用，调用 `compilation.optimize_dependency()` 进行依赖分析，标记未使用的代码/导出。
*   **调用 `compilation.seal()`:** 触发代码生成和优化阶段：
    *   基于模块图创建块图 (`ChunkGraph`)。
    *   优化模块和块（合并、拆分等）。
    *   生成运行时代码。
    *   为每个块生成最终代码。
    *   计算内容哈希。
    *   生成资源列表 (`Assets`)。
*   **处理诊断信息:** 合并来自 `compilation` 和 `plugin_driver` 的错误和警告。
*   **调用 `emit_assets()` (可选):** 如果需要输出文件，则调用资源输出方法。
*   **调用 `compilation.done()`:** 触发 `done` 钩子，通知插件构建完成。

### 2.5 `emit_assets()` - 资源输出

*   **触发 `emit` 钩子:** 通知插件即将输出资源。
*   **创建输出目录:** 确保 `output.path` 存在。
*   **并行写入:** 使用 `rayon` 并行遍历 `compilation.assets()`。
*   **调用 `emit_asset()`:** 对每个资源调用写入逻辑。
*   **触发 `after_emit` 钩子:** 通知插件资源输出完成。

### 2.6 `emit_asset()` - 单个资源写入

*   获取资源内容 (`Source` trait object)。
*   创建必要的父目录。
*   将资源内容写入文件。
*   记录已输出的文件名。

## 3. 总结

*   `Compiler` 是 Rspack 的最高层控制器，管理整个构建的生命周期和核心组件。
*   它通过创建和管理 `Compilation` 实例来执行具体的构建任务。
*   构建流程被清晰地划分为 `make`, `seal`, `emit` 等阶段，并通过 `PluginDriver` 在关键节点提供钩子给插件。
*   利用 Rust 的所有权、借用、`Arc`, `RwLock` 以及 `rayon` 等机制来实现高性能和线程安全的并发构建。

**下一步:** 分析 `Compilation` 结构体，理解其在单次构建中的具体职责和管理的数据。
