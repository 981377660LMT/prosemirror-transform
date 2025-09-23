好的，我们来深入、详细地讲解 replace.ts 这个文件。这是 prosemirror-transform 中最复杂、最精巧、也是最核心的算法所在。它解决了一个根本性的问题：**如何将任意一段文档片段（`Slice`）智能地、符合模式（Schema）地插入到文档的任意一个“缺口”中？**

简单的 `doc.replace()` 只能在 `Slice` 完美匹配目标位置的 Schema 时才能成功。但在实际应用中，比如用户粘贴内容时，源和目标几乎总是不匹配的。`replace.ts` 中的算法就是为了解决这个“不匹配”的问题，它会尝试通过包裹、提升、填充缺失节点等一系列操作，来找到一个最合理的替换方案。

我们将这个文件分解为三个主要部分来讲解：

1.  **`replaceStep`**: 顶层入口，它会先尝试快速路径，如果不行，则启动 `Fitter` 算法。
2.  **`Fitter` 类**: 整个智能替换算法的核心引擎。
3.  **`replaceRange` / `deleteRange`**: 更高层的、面向用户意图的“智能”替换/删除 API。

---

### 第一部分：`replaceStep` - 替换的入口与守卫

```typescript
// ...existing code...
export function replaceStep(doc: Node, from: number, to = from, slice = Slice.empty): Step | null {
  if (from == to && !slice.size) return null

  let $from = doc.resolve(from),
    $to = doc.resolve(to)
  // Optimization -- avoid work if it's obvious that it's not needed.
  if (fitsTrivially($from, $to, slice)) return new ReplaceStep(from, to, slice)
  return new Fitter($from, $to, slice).fit()
}
// ...existing code...
```

这个函数是所有替换操作的起点。它的逻辑很简单：

1.  **无操作检查**: 如果替换范围为空且插入的 `slice` 也为空，则直接返回 `null`，表示这是一个无意义的操作。
2.  **快速路径优化 (`fitsTrivially`)**: 它首先检查这次替换是否“极其简单”。一个“极其简单”的替换需要满足：
    - `slice` 没有开放的边（`openStart` 和 `openEnd` 都为 0）。
    - `from` 和 `to` 在同一个父节点下，且深度相同。
    - 父节点允许在 `[from, to]` 的位置直接插入 `slice` 的内容。
      如果满足，就直接创建一个简单的 `ReplaceStep` 并返回。这覆盖了大量常见情况，如在段落内输入文本。
3.  **启动 `Fitter`**: 如果快速路径失败，说明这是一个复杂的替换。此时，它会创建一个 `Fitter` 类的实例，并调用其 `fit()` 方法来执行复杂的拟合算法。

---

### 第二部分：`Fitter` 类 - 智能替换的核心引擎

`Fitter` 是一个状态机，它的目标是将一个 `unplaced` 的 `Slice`，一点一点地“塞”进由 `$from` 和 `$to` 定义的文档缺口中。

可以把它想象成一个建筑工人，他需要把一个预制好的、多层的建筑模块（`unplaced` Slice）安装到一个已经建好的大楼的缺口（`[$from, $to]`）里。

#### 1. `Fitter` 的核心状态

- `$from`, `$to`: 描述了文档中的“缺口”。
- `unplaced: Slice`: 剩余的、还**未被放置**的 `Slice` 部分。算法的目标就是不断消耗它，直到它变为空。
- `frontier: {type, match}[]`: **这是最核心、最抽象的概念**。它代表了已经放置好的内容的**右侧开放边界**。它是一个栈，栈顶代表最内层的开放节点。它从 `$from` 的祖先节点链开始，随着内容的放置而不断变化。
- `placed: Fragment`: 已经成功**放置好**的内容片段。

#### 2. `fit()` 方法 - 主算法循环

`fit()` 方法是 `Fitter` 的主驱动循环，它的逻辑可以概括为：

```
只要 unplaced 不为空:
  1. 寻找一个可行的放置方案 (findFittable)
  2. 如果找到了:
     执行放置 (placeNodes)
  3. 如果没找到:
     a. 尝试将 unplaced 的内容“打开”一层 (openMore)，暴露其内部内容，然后重试。
     b. 如果无法再打开，就从 unplaced 的最外层丢弃一个节点 (dropNode)，然后重试。

当 unplaced 为空后:
  1. 检查是否需要移动 $to 后面的内联内容 (mustMoveInline)，这会产生 ReplaceAroundStep。
  2. 尝试将 frontier 与 $to 的结构对齐并闭合 (close)。
  3. 如果闭合成功，根据最终的 placed 内容和 openStart/openEnd，生成最终的 ReplaceStep 或 ReplaceAroundStep。
```

#### 3. 关键子方法解析

- **`findFittable()`**: 这是算法的“眼睛”。它在 `unplaced` 的开放边界上寻找一个节点（或一组节点），看它是否能被放置到 `frontier` 的某个层级上。

  - 它分两轮（`pass`）进行：
    - **Pass 1**: 只尝试直接匹配。即 `unplaced` 的节点是否能直接放入 `frontier` 的某个容器中。它也允许通过 `fillBefore` 填充一些必要的节点来达成匹配。
    - **Pass 2**: 如果直接匹配失败，它会尝试寻找一组**包裹节点** (`findWrapping`)。即，是否可以通过给 `unplaced` 的节点套上几层父节点（比如 `<li><p>...</p></li>`），来让它匹配 `frontier`？
  - 它返回一个 `Fittable` 对象，描述了“从 `unplaced` 的 `sliceDepth` 层，移动到 `frontier` 的 `frontierDepth` 层”的方案。

- **`placeNodes(fittable)`**: 这是算法的“手”。当 `findFittable` 找到方案后，`placeNodes` 负责执行它。

  1.  它首先根据 `frontierDepth` 闭合 `frontier` 中多余的层级。
  2.  如果需要包裹，它会创建并打开新的包裹节点，更新 `frontier`。
  3.  它从 `unplaced` 中取出要放置的节点，将它们添加到 `placed` 片段中，并更新 `frontier` 的 `match` 状态。
  4.  最后，它更新 `unplaced`，移除已经被放置掉的内容。

- **`openMore()` 和 `dropNode()`**: 这是算法的“妥协”策略。当 `findFittable` 找不到任何方案时：

  - `openMore()` 会增加 `unplaced` 的 `openStart` 值，相当于把 `unplaced` 的外壳“剥掉”一层，露出里面的内容，以便下一轮 `findFittable` 可以尝试匹配这些内部内容。
  - `dropNode()` 是最后的手段。如果 `openMore` 也失败了，说明 `unplaced` 的最外层节点无论如何也放不进去，只能将它丢弃。

- **`close()`**: 当所有内容都放置完毕后，`close` 方法负责处理 `placed` 内容的右边界和 `$to` 之间的“缝隙”。它会尝试填充所有必要的闭合标签，使得 `placed` 的右边界能够与 `$to` 的结构无缝对接。

这个过程极其复杂，但它保证了 ProseMirror 能够以一种可预测且符合 Schema 的方式，处理几乎所有粘贴和替换场景。

---

### 第三部分：`replaceRange` 和 `deleteRange` - 更高层的智能 API

这些函数是 `Transform` 上的便捷方法，它们在 `Fitter` 的基础上，增加了更多面向用户意图的逻辑。

#### `replaceRange(tr, from, to, slice)`

`replaceRange` 的目标是处理粘贴等场景，它认为 `from` 和 `to` 只是“提示”，而不是绝对的边界。

- **核心逻辑**: 它会分析 `[$from, $to]` 覆盖了哪些深度的父节点（`coveredDepths`）。然后，它会尝试将 `slice` 插入到这些不同深度的父节点中，寻找一个最佳的匹配。
- **“扩展”替换**: 如果 `slice` 的内容（比如一个 `<li>`）无法放入当前段落，但可以放入外层的 `<ul>`，`replaceRange` 可能会决定将替换范围**扩展**到整个段落，用新的 `<li>` 替换掉它。
- **选择最佳方案**: 它有一套复杂的启发式规则，用于在多种可能的替换方案中选择一个“看起来最正确”的。

#### `deleteRange(tr, from, to)`

`deleteRange` 也是一个“智能”删除。

- **核心逻辑**: 当你删除一个范围时，如果这个范围恰好完整地覆盖了一个或多个父节点，`deleteRange` 会倾向于将整个父节点删除，而不是留下一个空的、不合法的“外壳”。
- **示例**: 在 `<blockquote><p>abc</p></blockquote>` 中选中 "abc" 并删除，`deleteRange` 可能会决定将整个 `<blockquote>` 都删除，因为删除后剩下的 `<blockquote><p></p></blockquote>` 可能不是一个理想的状态。

### 总结

- replace.ts 是 ProseMirror 的“智能大脑”，它让编辑器能够优雅地处理不匹配的替换操作。
- `Fitter` 类是这个大脑的核心，它通过一个复杂的“寻找-放置-妥协”循环，将任意 `Slice` 适配到文档的任意位置。
- `frontier` 是 `Fitter` 中最关键的数据结构，它追踪着替换过程中的“右侧开放边界”。
- `replaceRange` 和 `deleteRange` 在 `Fitter` 的基础上，增加了更多启发式规则，使得替换和删除操作更符合用户的直觉和预期，是实现 WYSIWYG（所见即所得）体验的关键。

找到具有 1 个许可证类型的类似代码
