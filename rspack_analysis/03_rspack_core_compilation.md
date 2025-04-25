# Rspack 源码分析：`rspack_core` 之 `Compilation`

本文档分析了 `crates/rspack_core/src/compiler/compilation.rs` 中定义的 `Compilation` 结构体。`Compilation` 代表了 Rspack 的一次具体构建过程，是管理构建状态和执行核心构建逻辑的引擎。

## 1. `Compilation` 结构体定义

`Compilation` 结构体包含了大量字段，用于存储构建过程中的各种状态和数据结构：

*   **核心图结构:**
    *   `module_graph: ModuleGraph`: 存储所有模块及其依赖关系的核心图。
    *   `chunk_graph: ChunkGraph`: 存储代码块（Chunk）及其包含的模块关系的核心图。
*   **输入与配置:**
    *   `options: Arc<CompilerOptions>`: 共享的编译选项。
    *   `entries: BundleEntries`: 入口点配置。
    *   `entry_dependencies: HashMap<String, Vec<DependencyId>>`: 入口名称到其直接依赖 ID 的映射。
*   **输出与资源:**
    *   `assets: CompilationAssets`: 存储最终生成的资源（文件名 -> `CompilationAsset`）。
    *   `emitted_assets: DashSet<String, ...>`: 记录已写入磁盘的资源文件名（线程安全）。
*   **Chunk 与 ChunkGroup:**
    *   `chunk_by_ukey: Database<Chunk>`: 通过唯一键存储 Chunk 数据。
    *   `chunk_group_by_ukey: HashMap<ChunkGroupUkey, ChunkGroup>`: 存储 Chunk 组。
    *   `entrypoints: HashMap<String, ChunkGroupUkey>`: 入口名称到对应 ChunkGroup Ukey 的映射。
    *   `named_chunks`, `named_chunk_groups`: 按名称存储 Chunk/ChunkGroup Ukey。
    *   `used_chunk_ids: HashSet<String>`: 已使用的 Chunk ID。
*   **运行时:**
    *   `runtime_modules: IdentifierMap<Box<dyn RuntimeModule>>`: 存储运行时模块实例。
    *   `runtime_module_hashes: IdentifierMap<u64>`: 运行时模块的哈希。
*   **诊断与状态:**
    *   `diagnostics: IndexSet<Diagnostic, ...>`: 存储本次构建的错误和警告。
    *   `last_module_diagnostics: IdentifierMap<Vec<Diagnostic>>`: 存储上次构建中各模块的诊断信息。
    *   `hash: String`: 本次编译的整体哈希。
*   **Tree Shaking:**
    *   `used_symbol`, `used_indirect_symbol`: 存储被标记为使用的直接和间接符号。
    *   `bailout_module_identifiers`: 存储因特定原因（如动态导入、CommonJS）而需要跳过 Tree Shaking 优化的模块。
    *   `tree_shaking_result` (debug only): 存储详细的 Tree Shaking 分析结果。
*   **代码生成:**
    *   `code_generation_results: CodeGenerationResults`: 存储模块代码生成的结果。
    *   `code_generated_modules: IdentifierSet`: 记录已生成代码的模块。
*   **依赖跟踪:**
    *   `file_dependencies`, `context_dependencies`, `missing_dependencies`, `build_dependencies`: 跟踪文件系统依赖，用于缓存和增量构建。
*   **共享组件:**
    *   `plugin_driver: SharedPluginDriver`: 共享插件驱动。
    *   `loader_runner_runner: Arc<LoaderRunnerRunner>`: 共享 Loader 执行器。
    *   `cache: Arc<Cache>`: 共享缓存。

## 2. 核心方法与构建阶段

`Compilation` 实现了驱动构建流程的核心方法：

### 2.1 `make()` - 模块构建阶段

*   **入口:** 由 `Compiler::compile()` 调用。
*   **核心职责:** 从入口点开始，并发地解析、加载、转换和分析所有模块，构建 `ModuleGraph`。
*   **并发模型:** 使用 `tokio` 和多个任务队列 (`FactorizeQueue`, `AddQueue`, `BuildQueue`, `ProcessDependenciesQueue`) 实现高度并发。
    *   **Factorize:** 解析依赖请求，创建模块对象（可能从缓存恢复）。
    *   **Add:** 将新创建或复用的模块添加到 `ModuleGraph`。
    *   **Build:** 对新模块执行 Loader 链，解析 AST，提取新的依赖。
    *   **Process Dependencies:** 处理 `Build` 阶段发现的新依赖，触发新的 `Factorize` 任务。
*   **增量构建:** 处理 `SetupMakeParam`，识别需要强制重新构建的模块和依赖。
*   **清理:** 在并发任务结束后，清理图中未被任何入口引用的孤立模块。

### 2.2 `optimize_dependency()` - Tree Shaking 阶段

*   **入口:** 由 `Compiler::compile()` 在 `make` 之后调用（如果启用 Tree Shaking）。
*   **核心职责:** 进行静态分析，识别并标记未使用的代码（导出和符号）。
*   **AST 分析:** 并行遍历 JS/TS 模块，使用 `ModuleRefAnalyze` 访问 AST，收集符号使用、导出、导入和副作用信息。
*   **星号导出处理:** 分析 `export * from '...'` 语句，构建继承关系图。
*   **符号标记:** 从入口点、有副作用的模块以及被标记为使用的符号出发，递归遍历依赖关系图和符号引用链，标记所有可达的、被使用的符号。
*   **模块剪枝:** 根据符号使用情况和副作用分析，标记 `ModuleGraphModule` 的 `used` 字段，供后续 `seal` 阶段使用。

### 2.3 `seal()` - 代码生成与优化阶段

*   **入口:** 由 `Compiler::compile()` 在 `optimize_dependency` (如果执行) 之后调用。
*   **核心职责:** 基于 `ModuleGraph` 和优化结果，生成最终的 Chunk、计算哈希、注入运行时代码，并准备好要输出的资源。
*   **主要步骤:**
    1.  **`build_chunk_graph()`:** 根据模块图、入口点和优化设置（如代码分割）构建 `ChunkGraph`。
    2.  **触发 `optimizeChunks` 钩子:** 允许插件进一步优化 Chunk 结构。
    3.  **触发 `moduleIds`, `chunkIds` 钩子:** 确定模块和 Chunk 的最终 ID。
    4.  **`code_generation()`:** 并行生成所有需要包含在输出中的模块的代码。
    5.  **`process_runtime_requirements()`:** 分析每个 Chunk 所需的运行时功能（如模块加载、HMR），并将对应的运行时模块添加到 Chunk 中。
    6.  **`create_hash()`:** 计算所有模块、运行时模块、Chunk 以及整体编译的哈希值。
    7.  **`create_chunk_assets()`:** 调用插件的 `renderManifest` 钩子，为每个 Chunk 生成最终的资源文件内容 (`CompilationAsset`)。
    8.  **触发 `processAssets` 钩子:** 允许插件在资源输出前进行最后修改。

## 3. 总结

*   `Compilation` 是 Rspack 构建过程的“工作台”，持有所有中间状态和数据结构。
*   它通过 `make` 和 `seal` 两个核心方法，驱动了从模块分析到代码生成的整个流程。
*   `ModuleGraph` 和 `ChunkGraph` 是其管理的核心数据结构。
*   其内部实现大量运用了并发、缓存和精细的状态管理，以实现高性能构建。
*   `Compilation` 实例是插件与之交互的主要对象，插件通过钩子读取和修改 `Compilation` 的状态。

**下一步:** 分析 `Plugin` trait 和 Rspack 的插件系统。
