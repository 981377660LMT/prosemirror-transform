好的，我们来深入、详细地讲解 map.ts 这个文件。这是 prosemirror-transform 模块的灵魂，也是 ProseMirror 实现协同编辑（Operational Transformation, OT）和稳健历史记录的魔法核心。

这个文件的目标是解决一个根本性问题：**当文档内容发生变化（增删）后，如何可靠地计算出任意一个旧的位置（position）在新文档中应该在哪里？**

我们将按照从基础概念到复杂实现的顺序来讲解。

---

### 第一部分：`Mappable` 和 `MapResult` - 接口与结果

#### `Mappable` 接口

```typescript
// ...existing code...
export interface Mappable {
  map: (pos: number, assoc?: number) => number
  mapResult: (pos: number, assoc?: number) => MapResult
}
// ...existing code...
```

- 这是一个**契约**。它规定任何“可映射”的对象，都必须提供两个方法：
  - `map`: 只返回映射后的新位置（一个数字）。
  - `mapResult`: 返回一个 `MapResult` 对象，它不仅包含新位置，还带有**额外的上下文信息**。

#### `MapResult` 类

```typescript
// ...existing code...
export class MapResult {
  constructor(readonly pos: number, readonly delInfo: number, readonly recover: number | null) {}
  // ...
}
// ...existing code...
```

这是 `map` 操作的“富信息”返回结果。

- `pos`: 映射后的新位置。
- `delInfo`: 一个**位掩码 (bitmask)**，用二进制位记录了位置周围的内容是否被删除。这是极其重要的信息。
  - `deletedBefore`: 旧位置**前面**的字符被删除了吗？
  - `deletedAfter`: 旧位置**后面**的字符被删除了吗？
  - `deletedAcross`: 旧位置**前后**的字符都被删除了吗？（即位置本身在一个被删除的范围内）
- `recover`: 一个“恢复值”，用于 `Mapping` 类中处理复杂的重基（rebasing）场景。我们稍后会重点讲它。

---

### 第二部分：`StepMap` - 单个步骤的位置映射

`StepMap` 是 `Mappable` 的最基本实现。**一个 `StepMap` 精确地描述了单个 `Step`（如 `ReplaceStep`）所造成的位置变化。**

#### 1. 核心数据结构 `ranges`

```typescript
// ...existing code...
  constructor(
    readonly ranges: readonly number[],
    readonly inverted = false
  ) {
// ...existing code...
```

- `ranges`: 一个只读的数字数组，是 `StepMap` 的核心数据。它以**每 3 个数字为一组**的形式存储变化：`[start, oldSize, newSize]`。
  - `[2, 0, 5]` 表示：在位置 2，删除了 0 个字符，插入了 5 个字符（即一次插入）。
  - `[5, 3, 0]` 表示：在位置 5，删除了 3 个字符，插入了 0 个字符（即一次删除）。
  - `[10, 4, 2]` 表示：在位置 10，删除了 4 个字符，插入了 2 个字符（即一次替换）。

#### 2. 核心方法 `_map(pos, assoc, simple)`

这是 `StepMap` 的心脏。它接收一个旧位置 `pos`，并计算出新位置。

- `assoc` (association): 关联性。一个非常重要的概念。当内容在一个位置被插入时，紧贴着这个位置的光标应该如何移动？
  - `assoc = 1` (默认, 右关联): 光标被推到插入内容的**右边**。
  - `assoc = -1` (左关联): 光标留在原地，在插入内容的**左边**。
- **工作流程**:
  1.  它遍历 `ranges` 数组中的每一个 `[start, oldSize, newSize]` 变化。
  2.  `diff` 变量累积了到目前为止文档长度的总变化。
  3.  如果 `pos` 在当前变化范围 `start` 的**前面**，则跳过，继续累加 `diff`。
  4.  如果 `pos` 在当前变化范围 `[start, start + oldSize]` 的**内部**：
      - 它会根据 `pos` 是在范围的开头、结尾还是中间，以及 `assoc` 的值，来精确计算 `delInfo` 位掩码。
      - 它会计算出最终的位置。例如，对于一次插入 `[10, 0, 5]`，一个在 `10` 的右关联光标会被映射到 `15`。
      - 它会生成一个 `recover` 值，这个值编码了当前 `range` 的索引和 `pos` 在这个 `range` 内的偏移量。这就像一个“面包屑”，为以后可能的“撤销映射”留下线索。
  5.  如果 `pos` 在所有变化范围的**后面**，那么它的新位置就是 `pos + diff`。

#### 3. `invert()` 方法

- 这是一个极其精巧的设计。它**不**重新计算 `ranges`，只是简单地返回一个 `new StepMap(this.ranges, !this.inverted)`。
- 它通过翻转 `inverted` 标志，使得 `_map` 方法内部的 `oldIndex` 和 `newIndex` 交换。这意味着，同一个 `StepMap` 对象，通过 `invert()` 就可以用来进行**双向映射**（从旧到新，或从新到旧），非常高效。

---

### 第三部分：`Mapping` - 多个 `StepMap` 的管道

`Mapping` 是 `Mappable` 的另一个实现。它代表了一系列 `StepMap` 的**有序集合**，通常对应一个完整的 `Transaction`（包含多个 `Step`）。

#### 1. 简单映射

如果 `Mapping` 中没有“镜像”信息（`mirror` 数组），它的 `map` 方法非常简单：

```typescript
// ...existing code...
  map(pos: number, assoc = 1) {
    // ...
    for (let i = this.from; i < this.to; i++)
      pos = this._maps[i].map(pos, assoc)
    return pos
  }
// ...existing code...
```

它就是一个管道，将 `pos` 依次穿过每一个 `StepMap`，得到最终结果。

#### 2. 复杂映射与 `mirror` - 您关注的焦点

这是整个文件最复杂、也是最关键的部分，专门用于处理**重基 (rebasing)**。

**问题场景**:

- 用户 A 做了修改 `A`。
- 同时，用户 B 做了修改 `B`。
- 当用户 A 接收到用户 B 的修改 `B` 时，他不能直接应用自己的 `A`，因为文档已经变了。他需要计算出一个在 `B` 之后的新版本 `A'`。这个过程就是 `A rebase B`。
- 在某些情况下，`A` 和 `B` 可能是互为逆操作的（例如，A 插入文本，B 删除了同一段文本）。直接按顺序映射会导致信息丢失。

**`mirror` 的解决方案**:
`mirror` 数组存储成对的索引，表示 `this.maps` 数组中哪两个 `StepMap` 是互为逆操作的。

现在我们来看您高亮的代码：`Mapping._map` 方法。

```typescript
// ...existing code...
  _map(pos: number, assoc: number, simple: boolean) {
    let delInfo = 0

    for (let i = this.from; i < this.to; i++) {
      let map = this._maps[i], result = map.mapResult(pos, assoc)
      // 1. 检查是否可以“跳过”
      if (result.recover != null) {
        let corr = this.getMirror(i) // 2. 查找当前步骤的镜像步骤
        if (corr != null && corr > i && corr < this.to) {
          // 3. 如果找到了一个未来的镜像步骤
          i = corr // 4. 跳过中间的所有步骤，直接快进到镜像步骤
          pos = this._maps[corr].recover(result.recover) // 5. 使用“面包屑”恢复位置
          continue // 6. 继续下一轮循环
        }
      }

      delInfo |= result.delInfo
      pos = result.pos
    }

    return simple ? pos : new MapResult(pos, delInfo, null)
  }
// ...existing code...
```

**深入讲解**:

1.  循环遍历每一个 `StepMap`。在 `i` 这一步，它先用 `map.mapResult(pos, assoc)` 计算出初步的映射结果 `result`。
2.  **检查“面包屑”**: 它检查 `result.recover` 是否存在。只有当位置 `pos` 落在 `StepMap` 的某个变化范围内部时，`recover` 值才会被创建。
3.  **查找镜像**: 如果 `recover` 存在，它就调用 `this.getMirror(i)` 查找在 `maps` 数组中，是否存在一个与第 `i` 个 `StepMap` 互为逆操作的 `StepMap`。
4.  **执行“跳跃”**: 如果找到了一个**在当前步骤之后**的镜像步骤 `corr`，魔法就发生了。这意味着从 `i` 到 `corr` 之间的所有步骤，实际上是在一个“先做再撤销”的块里。对于落在变化范围内的位置，这些中间步骤的累积效应是零。因此，它可以安全地**跳过**所有这些中间步骤。
5.  **位置恢复**: 它不是简单地把 `i` 设为 `corr`，而是用 `this._maps[corr].recover(result.recover)` 来精确地恢复位置。`recover` 方法利用 `result.recover` 中编码的“面包屑”（原始偏移量），在 `corr` 这个逆操作的 `StepMap` 上，直接计算出位置应该在 `corr` 步骤**之后**的哪里。这避免了累积映射可能产生的浮点误差或关联性问题，实现了**无损恢复**。
6.  **继续循环**: `continue` 语句使循环从 `corr + 1` 步继续，大大提高了效率和准确性。
7.  **常规路径**: 如果没有发生“跳跃”，它就正常地累积 `delInfo` 并更新 `pos`。

### 总结

- map.ts 是 ProseMirror 实现 OT 算法的核心组件。
- `StepMap` 使用 `[start, oldSize, newSize]` 格式高效地描述单个原子变化，并通过 `invert()` 支持双向映射。
- `Mapping` 将多个 `StepMap` 串联成一个管道。
- **`Mapping._map` 中的 `mirror` 和 `recover` 机制是整个设计的精髓**。它使得 `Mapping` 能够识别出“先做后撤销”的操作对，并“跳过”它们之间的映射，从而无损地、高效地处理复杂的重基场景，这是构建健壮协同编辑系统的关键。
