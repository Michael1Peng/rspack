# Rspack 源码分析：`rspack` Facade Crate

本文档分析了 `crates/rspack/src/lib.rs` 文件，这是 `rspack` crate 的入口点。该 crate 主要扮演着 Rspack 配置中心和内置插件管理器的角色。

## 1. `rspack()` 函数

这是从 Rust 代码层面创建 Rspack `Compiler` 实例的核心函数。

```rust
pub fn rspack(mut options: CompilerOptions, mut plugins: Vec<Box<dyn Plugin>>) -> Compiler {
  // 1. 添加一系列内置插件到 plugins 列表
  //    - 基础插件 (Asset, Json)
  //    - 运行时插件 (根据 options.target 选择)
  //    - HMR 插件 (如果 options.dev_server.hot)
  //    - Externals 插件
  //    - JavaScript 核心插件 (JsPlugin)
  //    - Devtool 插件 (根据 options.devtool)
  //    - Module/Chunk ID 分配插件 (根据 options.module_ids 等)
  //    - 其他优化插件 (RemoveEmptyChunksPlugin)

  // 2. 合并用户自定义插件
  plugins.append(&mut options.plugins);

  // 3. 使用最终的 options 和 plugins 列表创建 Compiler 实例
  Compiler::new(options, plugins)
}
```

**核心职责:**

*   **配置驱动的插件注册:** `rspack()` 函数的主要工作是根据传入的 `CompilerOptions`，实例化并注册一系列内置的 Rspack 插件。这些内置插件实现了 Rspack 的大部分核心功能。
*   **功能模块化:** 通过将核心功能（如资源处理、运行时代码生成、JavaScript 处理、SourceMap 生成、ID 分配、优化等）封装在不同的内置插件中，实现了良好的模块化。
*   **Target 适配:** 根据 `options.target.platform`（如 `Web`, `Node`）注册不同的运行时插件，以生成适用于目标环境的代码。
*   **用户插件集成:** 将用户在 `CompilerOptions.plugins` 中提供的自定义插件与内置插件合并。
*   **Compiler 实例化:** 最后，调用 `rspack_core::Compiler::new()`，将处理过的 `CompilerOptions` 和完整的插件列表传递给它，创建并返回 `Compiler` 实例。

## 2. `DevServer`

`rspack` crate 还提供了一个简单的 `DevServer` 功能：

*   `dev_server()` 函数：调用 `rspack()` 创建 `Compiler`，并将其包装在 `DevServer` 结构体中。
*   `DevServer::serve()` 方法：
    *   调用 `compiler.build()` 执行一次构建。
    *   使用 `warp` 库启动一个简单的 HTTP 服务器，用于托管构建输出目录（通常是 `dist`）。

## 3. 总结

`rspack` crate 作为 Rspack 的 Facade（外观）层，起到了承上启下的作用：

*   **对上 (用户/Node Binding):** 提供 `rspack()` 函数作为创建 `Compiler` 的入口。
*   **对下 (rspack_core):** 负责根据配置组装插件列表，并将最终的配置和插件传递给 `rspack_core::Compiler` 进行初始化。

这种设计体现了 Rspack "核心引擎 + 插件生态" 的架构思想。`rspack_core` 提供了通用的编译流程和钩子，而 `rspack` crate 则通过注册一系列内置插件来赋予这个流程具体的功能。

**下一步:** 分析 `node_binding` Crate，了解 Rspack 的 Rust 核心是如何通过 NAPI 暴露给 Node.js 环境，并与 JavaScript 世界进行交互的。
