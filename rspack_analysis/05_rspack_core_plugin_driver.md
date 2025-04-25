# Rspack 源码分析：`rspack_core` 之 `PluginDriver`

本文档分析了 `crates/rspack_core/src/plugin/plugin_driver.rs` 中定义的 `PluginDriver` 结构体。`PluginDriver` 是 Rspack 插件系统的核心执行引擎，负责管理和调用已注册的插件。

## 1. `PluginDriver` 结构体定义

```rust
pub struct PluginDriver {
  pub(crate) options: Arc<CompilerOptions>, // 共享编译选项
  pub plugins: Vec<Box<dyn Plugin>>,       // 存储所有插件实例
  pub resolver_factory: Arc<ResolverFactory>, // 共享解析器工厂
  pub registered_parser_and_generator_builder: HashMap<ModuleType, BoxedParserAndGeneratorBuilder>, // 注册的 Parser/Generator 构建器
  pub diagnostics: Arc<Mutex<Vec<Diagnostic>>>, // 收集插件产生的诊断信息 (线程安全)
}
```

*   **核心职责:** 管理插件列表 (`plugins`)，并在构建流程的各个阶段按顺序调用这些插件的钩子方法。
*   **插件存储:** 使用 `Vec<Box<dyn Plugin>>` 来存储不同类型的插件实例，利用了 Rust 的 trait object。
*   **Parser/Generator 注册表:** 持有从插件 `apply` 钩子收集来的 `ParserAndGeneratorBuilder` 映射，供 Rspack 核心在需要处理特定模块类型时查找并创建相应的解析器/生成器。
*   **诊断信息收集:** 提供一个线程安全的 (`Arc<Mutex<>>`) 容器来收集插件执行过程中产生的错误和警告。

## 2. 初始化 (`new()` 方法)

*   接收 `CompilerOptions`, 插件列表 (`Vec<Box<dyn Plugin>>`) 和 `ResolverFactory`。
*   **并行调用 `apply`:** 在初始化阶段，`PluginDriver` 会**并行**遍历插件列表，调用每个插件的 `apply` 方法。
*   **收集注册项:** 从每个插件的 `ApplyContext` 中收集通过 `register_parser_and_generator_builder` 注册的构建器，并存入 `registered_parser_and_generator_builder` 哈希表中。

## 3. 钩子调用机制

`PluginDriver` 为 `Plugin` trait 中定义的每个生命周期钩子都提供了一个对应的调用方法（如 `compilation()`, `make()`, `factorize()`, `process_assets()`, `done()` 等）。

*   **迭代执行:** 这些方法的核心逻辑是遍历 `self.plugins` 列表。
*   **顺序调用:** 对于大多数钩子，`PluginDriver` 会**按顺序**（插件在列表中的顺序）**串行**调用每个插件的相应钩子方法。这保证了插件执行的确定性。
*   **Bail Hook 支持:** 对于返回 `Result<Option<T>>` 的钩子（如 `factorize`, `module`, `render_chunk`），`PluginDriver` 会检查返回值。如果某个插件返回 `Ok(Some(T))`，`PluginDriver` 会立即停止调用后续插件的该钩子，并将该结果返回给调用者（通常是 `Compilation` 或 `Compiler`）。
*   **参数传递:** 负责将正确的 `PluginContext` 和特定于钩子的 `Args` 结构体传递给每个插件。
*   **错误处理:** 如果插件钩子返回 `Err`，`PluginDriver` 通常会停止执行并向上层传播错误。
*   **`process_assets` 分阶段执行:** `process_assets` 钩子被细分为多个阶段（如 `additional`, `pre_process`, `optimize_size` 等），`PluginDriver` 会按预定顺序依次调用所有插件的对应阶段方法。

## 4. 总结

*   `PluginDriver` 是 Rspack 插件系统的“指挥官”，确保插件在正确的时间以正确的顺序执行。
*   它通过迭代插件列表并调用其钩子方法来工作，支持串行执行和 Bail Hook 行为。
*   在初始化时通过调用 `apply` 钩子来收集插件注册的全局资源（如 Parser/Generator）。
*   `PluginDriver` 本身不直接修改构建状态，而是协调插件对 `Compilation` 对象进行操作。

**下一步:** 分析模块工厂 (`NormalModuleFactory` 和 `ContextModuleFactory`)，了解模块是如何被创建的。
