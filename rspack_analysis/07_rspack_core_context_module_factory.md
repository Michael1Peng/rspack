# Rspack 源码分析：`rspack_core` 之 `ContextModuleFactory`

本文档分析了 `crates/rspack_core/src/context_module_factory.rs` 中定义的 `ContextModuleFactory`。这个工厂专门用于处理 Webpack 中的上下文依赖（Context Dependency），例如由 `require.context()` 产生的依赖。

## 1. `ContextModuleFactory` 结构体

```rust
pub struct ContextModuleFactory {
  plugin_driver: SharedPluginDriver, // 共享插件驱动
  cache: Arc<Cache>,                 // 共享缓存
}
```

*   **职责:** 根据上下文依赖请求，解析目标目录，并创建一个 `ContextModule` 实例。
*   **结构简单:** 相较于 `NormalModuleFactory`，其结构更简单，主要依赖插件驱动和缓存。

## 2. 核心创建流程 (`resolve` 方法)

`ContextModuleFactory` 的核心逻辑在其 `resolve` 方法中实现（由 `ModuleFactory::create` 调用）：

1.  **解析目录 (Resolve Directory):**
    *   构造 `ResolveArgs`，关键设置是 `resolve_to_context: true`，指示解析器查找并返回目录信息，而不是具体文件。`importer` 通常为 `None`。
    *   **缓存优先:** 尝试从缓存 (`cache.resolve_module_occasion`) 获取解析结果。
    *   **执行解析:** 调用 `resolve()` 函数执行目录路径解析。

2.  **处理解析结果:**
    *   **成功 (`Ok(ResolveResult::Info)`):**
        *   获取到代表目标目录的 `ResolveInfo`。
        *   **创建 `ContextModule`:** 使用解析到的目录路径和从依赖对象 (`data.dependency.options()`) 中获取的上下文选项（如递归标志、匹配正则等）创建一个 `ContextModule` 实例。
    *   **忽略 (`Ok(ResolveResult::Ignored)`):**
        *   创建一个 `RawModule` 来表示该上下文被忽略。
    *   **失败 (`Err(ResolveError)`):**
        *   创建一个 `MissingModule` 来表示找不到指定的上下文目录。

3.  **返回结果:**
    *   将创建的模块实例 (`ContextModule`, `RawModule`, 或 `MissingModule`) 包装在 `ModuleFactoryResult` 中返回。
    *   附带解析过程中收集的文件依赖和缺失依赖信息。

## 3. 与 `NormalModuleFactory` 的对比

*   **目标不同:** `NormalModuleFactory` 解析并创建代表单个文件的模块（如 `.js`, `.css`），而 `ContextModuleFactory` 解析目录并创建代表该目录上下文的 `ContextModule`。
*   **解析配置:** `ContextModuleFactory` 使用 `resolve_to_context: true` 进行解析。
*   **产出模块:** 主要产出 `ContextModule`，而 `NormalModuleFactory` 主要产出 `NormalModule`。
*   **复杂度:** `ContextModuleFactory` 的逻辑相对简单，不涉及模块规则匹配、Parser/Generator 选择等。

## 4. `ContextModule` 的作用

`ContextModuleFactory` 创建的 `ContextModule` 实例在后续的构建阶段（主要是 `build` 过程）会执行以下操作：

*   读取其解析到的目录。
*   根据依赖选项（递归、正则）查找目录下的文件。
*   为所有匹配的文件创建新的模块依赖（通常是 `ContextElementDependency`）。
*   这些新的依赖会被 `Compilation` 的 `make` 流程捕获，并触发 `NormalModuleFactory`（或其他工厂）来创建这些文件的实际模块。

## 5. 总结

`ContextModuleFactory` 是 Rspack 处理 `require.context` 等动态上下文导入需求的关键组件。它通过解析目录并创建 `ContextModule`，将对一个目录的动态请求转换为了对该目录下多个具体文件模块的依赖，从而将动态性纳入到了静态分析的框架中。

**下一步:** 分析 Rspack 的核心数据模型，如 `Module`, `Chunk`, `ChunkGroup` 等。
