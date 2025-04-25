# Rspack 源码分析：`loader_runner` Crate

本文档分析了 `crates/loader_runner` Crate，它负责执行 Rspack 中的 Loader 链，对模块源码进行转换。

## 1. 核心组件与数据结构

*   **`Loader` Trait:**
    *   定义了所有 Loader 必须实现的接口。
    *   核心方法是 `async fn run(&self, loader_context: &LoaderContext) -> Result<Option<TWithDiagnosticArray<LoaderResult>>>`。
    *   Loader 需要是线程安全的 (`Sync + Send`)。
*   **`LoaderContext`:**
    *   传递给 Loader `run` 方法的上下文信息。
    *   包含：输入源码 (`source`)、资源信息 (`resource`, `resource_path`, etc.)、上一个 Loader 的 SourceMap 和附加数据、编译器/编译上下文引用、以及用于收集文件系统依赖的集合。
*   **`LoaderResult`:**
    *   Loader `run` 方法的返回值。
    *   包含：处理后的内容 (`content`)、生成的 SourceMap、附加数据、以及该 Loader 收集到的文件系统依赖。
*   **`Content` Enum:**
    *   Loader 之间传递的数据类型，可以是 `String` 或 `Vec<u8>`，支持文本和二进制内容。
*   **`LoaderRunner`:**
    *   负责管理和执行整个 Loader 链。
    *   持有要处理的 `ResourceData` 和 Loader Runner 插件列表。
*   **`LoaderRunnerPlugin` Trait (定义在 `plugin.rs`):**
    *   允许插件介入 Loader Runner 的初始资源处理阶段 (`process_resource` 钩子)。

## 2. Loader 执行流程 (`LoaderRunner::run`)

`LoaderRunner` 的 `run` 方法是执行 Loader 链的核心：

1.  **获取初始上下文:** 调用 `get_loader_context()` 方法：
    *   调用 `process_resource()` 获取初始内容。`process_resource` 会先尝试调用 `LoaderRunnerPlugin` 的 `process_resource` 钩子，如果插件未处理，则从文件系统读取资源内容。
    *   创建包含初始内容、资源信息和空依赖集合的 `LoaderContext`。
2.  **反向迭代 Loader:** **关键：** Loader 链是**从后往前**执行的（`loaders.as_ref().iter().rev()`），这与 Webpack 的行为一致。
3.  **执行单个 Loader:** 对列表中的每个 Loader：
    *   调用其 `run()` 方法，传入当前的 `LoaderContext`。
4.  **更新上下文:**
    *   如果 Loader 返回 `Ok(Some(loader_result))`：
        *   从 `loader_result` 中提取处理后的 `content`, `source_map`, `additional_data` 和文件系统依赖。
        *   用这些新信息更新 `LoaderContext` 的相应字段。
        *   收集该 Loader 返回的诊断信息。
    *   如果 Loader 返回 `Ok(None)`：`LoaderContext` 保持不变，相当于跳过了该 Loader。
    *   如果 Loader 返回 `Err(...)`：中断执行并返回错误。
5.  **传递给下一个 Loader:** 更新后的 `LoaderContext` 被传递给 Loader 链中的下一个 Loader（即配置顺序上的前一个 Loader）。
6.  **返回最终结果:** 当所有 Loader 执行完毕后，`LoaderRunner::run` 返回：
    *   最后一个 Loader（即配置顺序上的第一个 Loader）产生的 `LoaderResult`（包含了最终处理结果和所有 Loader 收集的依赖信息）。
    *   所有 Loader 在执行过程中产生的诊断信息集合。

## 3. 总结

`loader_runner` Crate 实现了一个独立、健壮的 Loader 执行引擎，其设计与 Webpack 的 `loader-runner` 高度相似。它通过定义清晰的 `Loader` 接口、上下文 (`LoaderContext`) 和结果 (`LoaderResult`)，以及核心的 `LoaderRunner` 来管理执行流程。反向执行 Loader 链的机制确保了 Loader 的执行顺序符合预期。对异步 Loader 的支持以及插件扩展点（`LoaderRunnerPlugin`）使其更加灵活。`rspack_core` 中的 `NormalModule` 在其 `build` 阶段会使用这个 `LoaderRunner` 来处理模块源码。

**下一步:** 整理所有分析结果，绘制更详细的架构图，并撰写最终的总结报告。
