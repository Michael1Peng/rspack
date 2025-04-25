# Rspack 源码分析：`rspack_core` 之 `Chunk`

本文档分析了 `crates/rspack_core/src/chunk.rs` 中定义的 `Chunk` 结构体。`Chunk` 是 Rspack 构建输出的基本单元，代表一组模块的集合，最终会生成一个或多个输出文件。

## 1. `Chunk` 结构体定义

```rust
pub struct Chunk {
  // --- 标识与命名 ---
  pub ukey: ChunkUkey,             // 唯一内部标识符
  pub name: Option<String>,        // 可选名称 (来自入口或 import())
  pub id: Option<String>,          // 最终输出 ID (运行时使用)
  pub ids: Vec<String>,            // (用途待确认)
  pub id_name_hints: HashSet<String>, // ID 生成提示

  // --- 内容与输出 ---
  pub files: HashSet<String>,      // 该 Chunk 生成的所有文件名
  pub content_hash: HashMap<SourceType, String>, // 按源码类型的内容哈希
  pub hash: Xxh3,                  // 中间哈希计算状态

  // --- 关系 ---
  pub groups: HashSet<ChunkGroupUkey>, // 所属的 ChunkGroup Ukey 集合

  // --- 属性与元数据 ---
  pub runtime: RuntimeSpec,        // 所属运行时环境
  pub kind: ChunkKind,             // Chunk 类型 (Normal / HotUpdate)
  pub chunk_reasons: Vec<String>,  // 创建原因 (调试用)
}

#[derive(Debug, PartialEq, Eq)]
pub enum ChunkKind {
  Normal,
  HotUpdate,
}
```

*   **核心职责:** 聚合模块，并代表最终输出文件（或文件组）的内容和元数据。
*   **唯一标识 (`ukey`):** 每个 Chunk 都有一个内部唯一的 `ChunkUkey`，用于在 `Compilation` 的 `chunk_by_ukey` 数据库中存储和查找。
*   **命名与 ID (`name`, `id`):** Chunk 可以有一个可选的 `name`（通常来自配置或代码注释），并在构建后期被分配一个最终的 `id`，用于在运行时加载和引用。
*   **文件 (`files`):** 一个 Chunk 可以对应多个输出文件（例如，`.js` 文件和对应的 `.js.map` SourceMap 文件）。`files` 字段记录了所有这些文件的名称。
*   **内容哈希 (`content_hash`):** 存储了根据 Chunk 内容计算出的哈希值，按不同的 `SourceType`（如 JavaScript, CSS）区分。这是实现长期缓存的关键。
*   **与 ChunkGroup 的关系 (`groups`):** 一个 Chunk 可以属于一个或多个 `ChunkGroup`。例如，一个共享模块组成的 Chunk 可能同时被多个入口点 ChunkGroup 引用。
*   **运行时 (`runtime`):** 指定了 Chunk 运行的目标环境。
*   **类型 (`kind`):** 分为普通 Chunk 和 HMR 更新 Chunk。

## 2. `Chunk` 的方法

`Chunk` 结构体提供了一系列方法来查询其属性和与其他图结构（`ChunkGraph`, `ChunkGroupByUkey`）交互：

*   **状态查询:** `can_be_initial()`, `is_only_initial()`, `has_entry_module()`, `has_runtime()` 等方法用于判断 Chunk 的特性（是否是初始加载、是否包含入口模块、是否是运行时 Chunk）。
*   **图遍历:** `get_all_referenced_chunks()`, `get_all_initial_chunks()`, `get_all_async_chunks()` 等方法用于查找与当前 Chunk 相关联的其他 Chunk（通过遍历其所属的 `ChunkGroup` 及其子关系）。
*   **关系管理:** `add_group()`, `split()`, `disconnect_from_groups()` 用于维护 Chunk 与 ChunkGroup 之间的关系。
*   **ID 与命名:** `expect_id()`, `name_for_filename_template()` 用于获取用于运行时或文件名模板的标识符。

## 3. 总结

`Chunk` 是 Rspack 将模块组织成可部署单元的核心数据结构。它不仅包含了自身的元数据（名称、ID、哈希、类型），还通过 `groups` 字段维护了与 `ChunkGroup` 的关联关系。`Compilation` 在 `seal` 阶段的核心任务之一就是根据 `ModuleGraph` 和优化策略来创建和组织这些 `Chunk`，并最终将它们渲染成输出文件。

**下一步:** 分析 `ChunkGroup` 数据模型，了解 Chunk 是如何被分组和管理的。
