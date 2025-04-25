# Rspack 源码分析：`rspack_core` 之 `NormalModuleFactory`

本文档分析了 `crates/rspack_core/src/normal_module_factory.rs` 中定义的 `NormalModuleFactory`。这是负责创建 Rspack 中“普通”模块（通常指需要解析、加载、转换和分析依赖的模块，如 JavaScript、CSS 等）的核心工厂。

## 1. `NormalModuleFactory` 结构体

```rust
#[derive(Debug)]
pub struct NormalModuleFactory {
  context: NormalModuleFactoryContext, // 模块创建上下文
  plugin_driver: SharedPluginDriver,   // 共享插件驱动
  cache: Arc<Cache>,                 // 共享缓存
}

#[derive(Debug, Clone)]
pub struct NormalModuleFactoryContext {
  pub original_resource_path: Option<PathBuf>, // 发起请求的模块路径 (Option)
  pub module_type: Option<ModuleType>,        // 预设的模块类型 (Option)
  pub side_effects: Option<bool>,             // 预设的副作用标记 (Option)
  pub options: Arc<CompilerOptions>,          // 共享编译器选项
  pub lazy_visit_modules: std::collections::HashSet<String>, // 懒编译模块列表
  pub issuer: String,                         // 发起请求的模块标识符字符串
}
```

*   **职责:** 根据依赖请求 (`ModuleDependency`) 和上下文信息，解析模块路径，确定模块类型和处理规则，最终创建相应的模块实例（主要是 `NormalModule`）。
*   **上下文 (`context`):** 包含了创建模块所需的关键信息，如发起者（issuer）、编译器选项等。
*   **依赖:** 依赖插件驱动 (`plugin_driver`) 来调用钩子，依赖缓存 (`cache`) 来加速解析过程。

## 2. 核心创建流程 (`factorize` 和 `factorize_normal_module`)

`NormalModuleFactory` 的核心逻辑在 `factorize` 和 `factorize_normal_module` 方法中：

1.  **`factorize()` (入口):**
    *   **尝试插件 `factorize` 钩子:** 首先调用插件的 `factorize` 钩子。如果某个插件返回了模块结果 (`Ok(Some(ModuleFactoryResult))`），则直接使用该结果。这允许插件完全控制某些特定模块（如 `ExternalModule`）的创建过程。
    *   **调用内部逻辑:** 如果没有插件处理，则调用 `factorize_normal_module()`。

2.  **`factorize_normal_module()` (核心实现):**
    *   **解析 (Resolve):**
        *   根据依赖信息（请求字符串、发起者路径、依赖类型等）构造 `ResolveArgs`。
        *   **缓存优先:** 尝试从缓存 (`cache.resolve_module_occasion`) 中获取解析结果。
        *   **执行解析:** 如果缓存未命中，调用 `resolve()` 函数（可能在 `resolve.rs` 中）执行实际的路径解析。此过程涉及文件系统查找、`resolve` 配置应用，并可能触发插件的解析相关钩子。
        *   **处理结果:**
            *   **成功:** 获得包含完整路径、查询参数等的 `ResourceData`。
            *   **忽略:** 创建一个 `RawModule` 表示被忽略的模块（如 `externals`）。
            *   **失败:** 创建一个 `MissingModule` 表示未找到的模块，并记录错误。
    *   **匹配规则 (Match Rules):**
        *   调用 `calculate_module_rules()`，根据解析得到的 `ResourceData`（主要是资源路径）和发起者（issuer），匹配 `CompilerOptions.module.rules` 中定义的规则。
    *   **确定模块属性:**
        *   调用 `calculate_module_type()`，根据匹配到的规则和文件扩展名，确定最终的 `ModuleType`。
        *   调用 `calculate_resolve_options()`，合并规则中定义的 `resolve` 选项。
        *   调用 `calculate_parser_and_generator_options()`，合并规则中定义的 `parser` 和 `generator` 选项。
    *   **获取 Parser/Generator:**
        *   根据确定的 `ModuleType`，从 `plugin_driver.registered_parser_and_generator_builder` 中查找对应的构建器函数。
        *   调用构建器函数，创建该模块类型所需的 `ParserAndGenerator` 实例。
    *   **创建 `NormalModule` 实例:**
        *   使用解析得到的资源信息、确定的模块类型、获取到的 Parser/Generator、合并后的选项等，创建一个 `NormalModule` 对象。
    *   **尝试插件 `module` 钩子:**
        *   调用插件的 `module` 钩子，传入新创建的 `NormalModule`。插件可以在此钩子中修改模块属性，甚至完全替换成另一种类型的模块。
    *   **返回结果:**
        *   将最终的模块实例（可能是 `NormalModule` 或被插件替换后的模块）包装在 `ModuleFactoryResult` 中。
        *   同时附带解析过程中收集到的文件依赖 (`file_dependencies`) 和缺失依赖 (`missing_dependencies`) 信息，用于后续的缓存和监听。

## 3. 总结

*   `NormalModuleFactory` 是 Rspack 模块创建流程中的关键一环，负责将一个依赖请求转化为一个具体的模块实例。
*   它整合了模块解析、规则匹配、类型确定、选项合并、Parser/Generator 获取等多个步骤。
*   插件系统在模块创建过程中扮演重要角色，可以通过 `factorize` 和 `module` 钩子深度介入。
*   工厂的设计体现了配置驱动（依赖 `CompilerOptions`）和缓存优先的原则。

**下一步:** 分析 `ContextModuleFactory`，了解用于处理上下文依赖（如 `require.context`）的模块是如何创建的。
