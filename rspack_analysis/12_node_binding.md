# Rspack 源码分析：`node_binding` Crate

本文档分析了 `crates/node_binding/src/lib.rs` 文件，该 Crate 负责使用 NAPI (Node-API) 将 Rspack 的 Rust 核心功能暴露给 Node.js 环境。

## 1. 核心职责与技术

*   **桥接 Rust 与 Node.js:** 作为 Rust 核心 (`rspack`, `rspack_core`) 与 JavaScript/Node.js 世界之间的桥梁。
*   **NAPI:** 使用 `napi` 和 `napi_derive` crate 来实现 Node.js Addon。
*   **暴露 `Rspack` 类:** 将 Rust 的 `Compiler` 功能封装成一个名为 `Rspack` 的类，暴露给 Node.js 使用。

## 2. 全局 `Compiler` 管理

*   **`static COMPILERS`:** 使用 `once_cell::sync::Lazy` 和自定义的 `SingleThreadedHashMap` (基于 `dashmap::DashMap`) 来全局存储 `Compiler` 实例。每个实例通过唯一的 `CompilerId` (u32) 访问。
*   **`SingleThreadedHashMap`:** 这是一个自定义的 `DashMap` 包装器，其 `borrow_mut`, `borrow`, `insert_if_vacant`, `remove` 等方法被标记为 `unsafe`。注释强调这些方法**只应在创建映射的线程（通常是 Node.js 主线程）上调用**。这是为了避免在多编译器场景下使用 `Mutex<HashMap>` 可能导致的死锁，同时适应 NAPI 和 Node.js 的单线程模型。这是一种需要开发者理解其假设的务实方案。
*   **`static COMPILER_ID`:** 原子计数器，用于生成唯一的 `CompilerId`。

## 3. `Rspack` NAPI 类

这是暴露给 Node.js 的主要接口。

*   **`new()` (Constructor):**
    *   接收 JS 传入的 `RawOptions` 和可选的 `JsHooks`。
    *   **选项规范化:** 调用 `rspack_binding_options::normalize_bundle_options` 将 JS 选项转换为 Rust 的 `CompilerOptions`。
    *   **JS 钩子适配:** 创建 `JsHooksAdapter` (如果提供了 `JsHooks`)，并将其作为 Rust 插件添加到 `CompilerOptions` 中。
    *   **JS Loader 适配:** 处理 `JsLoaderAdapter`，调用 `unref` 防止阻塞事件循环。
    *   **创建 Rust Compiler:** 调用 `rspack::rspack()` 创建核心 `Compiler` 实例。
    *   **存储 Compiler:** 将 `Compiler` 实例存入全局 `COMPILERS` 映射，并关联一个新生成的 `CompilerId`。
    *   返回包含 `CompilerId` 的 `Rspack` 实例给 Node.js。
*   **`build()` / `rebuild()`:**
    *   提供异步的构建和重新构建接口。
    *   使用 `callbackify` 将 Rust 的 `async` 操作转换为 Node.js 回调风格。
    *   通过 `unsafe { COMPILERS.borrow_mut(...) }` 获取对应的 Rust `Compiler` 实例并调用其 `build()` 或 `rebuild()` 方法。
    *   包含防止递归调用导致死锁的警告。
*   **`unsafe_last_compilation()`:**
    *   允许 JS 端访问最后一次构建的 `Compilation` 状态。
    *   将 Rust 的 `&mut Compilation` 包装成 `JsCompilation` 对象传递给 JS 回调。
    *   方法名和文档都强调了在 JS 端缓存 `JsCompilation` 的危险性（悬垂指针）。
*   **`drop()`:**
    *   提供给 JS 端调用的销毁方法。
    *   通过 `unsafe { COMPILERS.remove(...) }` 从全局映射中移除并销毁对应的 Rust `Compiler` 实例，释放内存。

## 4. 桥接机制

*   **选项转换:** `rspack_binding_options` crate 负责处理复杂的 JS 选项到 Rust 结构的转换和规范化。
*   **插件/钩子适配:** `JsHooksAdapter` (在 `plugins` 模块) 实现了 Rust 的 `Plugin` trait，它内部持有对 JS 钩子函数的引用（通过 NAPI 的 `ThreadsafeFunction`），当 Rust 的 `PluginDriver` 调用钩子时，`JsHooksAdapter` 会调用相应的 JS 函数。
*   **Loader 适配:** `JsLoaderAdapter` (可能在 `plugins` 或 `rspack_binding_options` 中定义) 实现了 Rust 的 `Loader` trait，允许在 Rust 的 Loader 执行流程中调用 JS Loader 函数。
*   **值转换:** `js_values` 模块负责在 Rust 结构（如 `Compilation`, `Stats`, `Asset` 等）和暴露给 JS 的对象 (`JsCompilation`, `JsStats`, `JsAsset` 等）之间进行转换。

## 5. 总结

`node_binding` crate 是 Rspack 能够作为 Node.js 工具使用的关键。它巧妙地利用 NAPI 将高性能的 Rust 核心编译能力暴露出来，同时通过适配器模式整合了 JavaScript 的插件和 Loader 生态。全局 `Compiler` 管理方案虽然引入了 `unsafe` 和线程安全假设，但解决了 NAPI 环境下的潜在死锁问题。开发者在使用此绑定时需要注意其文档中关于并发和生命周期的警告。

**下一步:** 分析 `loader_runner` Crate，了解 Loader 的执行流程。
