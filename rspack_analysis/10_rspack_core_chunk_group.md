# Rspack 源码分析：`rspack_core` 之 `ChunkGroup`

本文档分析了 `crates/rspack_core/src/chunk_group.rs` 中定义的 `ChunkGroup` 结构体。`ChunkGroup` 用于组织和管理逻辑上相关的 `Chunk` 集合。

## 1. `ChunkGroup` 结构体定义

```rust
#[derive(Debug)]
pub struct ChunkGroup {
  pub ukey: ChunkGroupUkey,             // 唯一内部标识符
  pub chunks: Vec<ChunkUkey>,          // 直接包含的 Chunk Ukey 列表 (有序)
  // 模块遍历顺序索引
  pub(crate) module_pre_order_indices: IdentifierMap<usize>,
  pub(crate) module_post_order_indices: IdentifierMap<usize>,
  // 父子关系
  pub(crate) parents: HashSet<ChunkGroupUkey>,  // 父 ChunkGroup Ukey 集合
  pub(crate) children: HashSet<ChunkGroupUkey>, // 子 ChunkGroup Ukey 集合
  // 类型与运行时
  pub(crate) kind: ChunkGroupKind,        // 组类型 (Entrypoint / Normal)
  pub(crate) runtime: RuntimeSpec,         // 所属运行时环境
  // 内部计数器
  pub(crate) next_pre_order_index: usize,
  pub(crate) next_post_order_index: usize,
  // Entrypoint 特有字段
  pub(crate) runtime_chunk: Option<ChunkUkey>, // 运行时代码所在的 Chunk Ukey
  pub(crate) entry_point_chunk: Option<ChunkUkey>, // 入口点代码所在的 Chunk Ukey
}

#[derive(Debug, PartialEq, Eq)]
pub enum ChunkGroupKind {
  Entrypoint, // 代表入口点
  Normal,     // 代表异步加载点或代码分割点
}
```

*   **核心职责:** 将相关的 `Chunk` 组织在一起，并维护组之间的父子关系，以表示加载依赖。
*   **唯一标识 (`ukey`):** 每个 `ChunkGroup` 都有一个内部唯一的 `ChunkGroupUkey`。
*   **包含 Chunks (`chunks`):** 一个 `ChunkGroup` 直接包含一个或多个 `Chunk`。`Vec` 类型可能暗示了组内 Chunk 的顺序性。
*   **父子关系 (`parents`, `children`):** `ChunkGroup` 之间可以形成有向无环图（DAG），表示加载关系。例如，一个入口点 `ChunkGroup` 可能有多个子 `ChunkGroup`，代表它异步加载的模块。
*   **类型 (`kind`):**
    *   `Entrypoint`: 代表应用程序的入口点，通常对应 `CompilerOptions.entry` 中的一项。
    *   `Normal`: 代表通过动态导入 (`import()`) 或代码分割（如 SplitChunksPlugin）创建的非入口 Chunk 集合。
*   **入口点属性 (`runtime_chunk`, `entry_point_chunk`):** 对于 `Entrypoint` 类型的组，会明确指定哪个 `Chunk` 包含了该入口的运行时代码，哪个 `Chunk` 包含了入口模块本身的代码。
*   **模块顺序 (`module_*_order_indices`):** 存储了该组内模块的遍历顺序信息，可能用于代码生成时的排序。

## 2. `ChunkGroup` 的方法

`ChunkGroup` 提供的方法主要用于管理其包含的 Chunk、维护父子关系以及查询自身属性：

*   **关系管理:** `connect_chunk()`, `unshift_chunk()`, `insert_chunk()`, `remove_chunk()` 用于添加、移除和排序组内的 Chunk，并同步更新 Chunk 的 `groups` 字段。`add_parent()`, `add_child()` (可能在 `ChunkGraph` 中实现) 用于建立组间关系。
*   **属性查询:** `is_initial()` 判断是否为入口点组。`get_runtime_chunk()`, `get_entry_point_chunk()` 获取入口点组的关键 Chunk。
*   **图遍历:** `ancestors()` 递归查找所有父级组。`Chunk` 上的 `get_all_*_chunks()` 方法会利用 `ChunkGroup` 的父子关系来查找相关联的 Chunk。
*   **文件获取:** `get_files()` 获取该组内所有 Chunk 生成的文件列表。

## 3. 总结

`ChunkGroup` 是 Rspack 组织代码块（Chunk）的关键结构。它不仅将相关的 Chunk 묶在一起，更重要的是通过父子关系 (`parents`, `children`) 建立了 Chunk 之间的加载依赖图。这使得 Rspack 能够理解入口点如何加载初始 Chunk，以及这些 Chunk 又如何按需加载异步 Chunk。`Entrypoint` 类型的 `ChunkGroup` 具有特殊地位，定义了应用程序的起点和运行时 Chunk。

**下一步:** 回到分析计划，开始分析 `rspack` 这个 Facade Crate，了解它是如何初始化 `Compiler` 并协调 `rspack_core` 和插件的。
