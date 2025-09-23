好的，我们来深入、详细地讲解 structure.ts 这个文件。

如果说 `replace.ts` 是 ProseMirror 的“智能内容填充引擎”，那么 structure.ts 就是它的“**结构化操作工具箱**”。这个文件提供了一系列高级的、面向开发者意图的函数，用于执行所有常见的文档**结构性**修改，例如提升、包裹、分割、合并和改变节点类型。

这些函数几乎都是 `Transform` 上的便捷方法的具体实现。它们的核心工作模式是：接收一个 `Transform` 对象和一些参数，然后经过复杂的计算，最终创建并向 `tr` 中添加一个或多个原子性的 `Step`（通常是 `ReplaceAroundStep` 或 `ReplaceStep`）。

我们将这个文件按功能分为三大类来讲解：

1.  **垂直结构操作**: `lift` 和 `wrap`，用于在文档树中上下移动内容。
2.  **节点类型与属性修改**: `setBlockType` 和 `setNodeMarkup`，用于原地改变节点的“外壳”。
3.  **水平结构操作**: `split` 和 `join`，用于分割或合并相邻的节点。

---

### 第一部分：垂直结构操作 (`lift` 和 `wrap`)

#### `lift(tr, range, target)` - 提升内容

- **意图**: 将 `range` 内的内容从其父节点中“提升”出来，使其成为更高层级节点的直接子节点。典型的例子就是将一个列表项（`<li>`）提升为一个段落（`<p>`），或者将一个被 `<blockquote>` 包裹的段落提升出来。
- **实现策略**: 这是 `ReplaceAroundStep` 的一个经典应用。
  1.  它计算出需要被“切开”的父节点。
  2.  它创建一个 `Slice`，这个 `Slice` 的内容是由被切开的父节点的“前半部分”和“后半部分”组成的。例如，提升 `<ul><li>item</li></ul>` 中的 `item`，这个 `Slice` 的内容就是 `<ul>...</ul>`（一个开放的 `ul`）和 `<ul>...</ul>`（另一个开放的 `ul`）。
  3.  它创建一个 `ReplaceAroundStep`，用这个包含“分裂的父节点”的 `Slice`，去替换掉包含原始父节点的更大范围，同时将 `range` 内的内容（“峡谷”）保留并移动到 `Slice` 的“洞”中。
- **`liftTarget(range)`**: 这是一个辅助函数，用于在执行 `lift` 之前，检查并找到一个有效的目标深度。它回答了“这段内容可以被提升到哪里？”这个问题。

#### `wrap(tr, range, wrappers)` - 包裹内容

- **意图**: 用一组 `wrappers`（例如 `[{type: blockquoteType}, {type: paragraphType}]`）将 `range` 内的内容包裹起来。
- **实现策略**: 同样是 `ReplaceAroundStep` 的完美应用，但比 `lift` 更直观。
  1.  它创建一个 `Slice`，其内容就是由 `wrappers` 构成的嵌套结构，中间留有一个“洞”。
  2.  它创建一个 `ReplaceAroundStep`，用这个 `Slice` 替换掉 `range` 的范围，同时将 `range` 内的原始内容（“峡谷”）移动到 `Slice` 的“洞”里。
- **`findWrapping(range, nodeType)`**: 这是一个非常重要的辅助函数，用于在执行 `wrap` 之前，找到一个有效的包裹方案。它会检查 `range` 的外部和内部，计算出需要添加哪些父节点和子节点才能使包裹合法。

---

### 第二部分：节点类型与属性修改 (`setNodeMarkup` 等)

这是您当前选择的焦点。这类函数用于改变一个已有节点的“身份”，而不改变其内部内容。

#### `setNodeMarkup(tr, pos, type, attrs, marks)` - 节点变身术

- **意图**: 改变 `pos` 位置节点的类型、属性和/或标记。例如，将一个二级标题（`<h2 level=2>`）变为三级标题（`<h3 level=3>`），或者给一个图片节点添加一个 `class` 属性。
- **实现策略**:
  1.  `let node = tr.doc.nodeAt(pos)`: 首先获取目标节点。
  2.  `let newNode = type.create(attrs, null, marks || node.marks)`: 创建一个**新的节点外壳**。这个新节点拥有新的类型、属性和标记，但其内容（第二个参数）暂时为 `null`。
  3.  **分支处理**:
      - **叶子节点 (`node.isLeaf`)**: 如果是叶子节点（如 `image` 或 `horizontal_rule`），它没有内部内容需要保留。因此，直接用一个简单的 `tr.replaceWith` 就可以用新节点替换旧节点。
      - **非叶子节点**: 这是更复杂的情况。我们不能简单地替换，因为需要**保留**旧节点的 `node.content`。这正是 `ReplaceAroundStep` 的用武之地。
        - `tr.step(new ReplaceAroundStep(pos, pos + node.nodeSize, pos + 1, pos + node.nodeSize - 1, ...))`
        - **`from`, `to`**: `pos` 到 `pos + node.nodeSize`，即整个旧节点的范围。
        - **`gapFrom`, `gapTo`**: `pos + 1` 到 `pos + node.nodeSize - 1`，这精确地选中了旧节点的**内部内容**作为“峡谷”。
        - **`slice`**: `new Slice(Fragment.from(newNode), 0, 0)`，这是一个只包含新节点“外壳”的 `Slice`。注意，它的 `openStart` 和 `openEnd` 都是 0，但因为 `newNode` 本身是一个非叶子节点，它天然就有一个可以容纳内容的“洞”。
        - **`insert`**: `1`，表示将“峡谷”内容插入到新 `Slice` 中 `newNode` 的起始内容位置（即偏移量 1 的位置）。
        - **`structure`**: `true`，这是一个安全标志，确保这个操作不会意外覆盖任何内容。

#### `setBlockType(...)`

- **意图**: 这是一个更具体的函数，专门用于改变一个范围内的文本块（Textblock）的类型，例如将多个段落一次性全变成标题。
- **实现策略**: 它会遍历范围内的所有文本块，对于每一个需要改变的块，它内部调用的核心逻辑与 `setNodeMarkup` 完全相同，即使用 `ReplaceAroundStep` 来替换节点外壳，保留内部的内联内容。

---

### 第三部分：水平结构操作 (`split` 和 `join`)

#### `split(tr, pos, depth, typesAfter)` - 分割节点

- **意图**: 在 `pos` 位置进行分割。最常见的例子是在段落中间按回车，将其一分为二。`depth` 参数允许同时分割多个层级的父节点（例如在嵌套列表中按回车）。
- **实现策略**:
  1.  它首先计算出需要被分割的节点，并根据 `typesAfter` 参数确定分割后右侧新节点的类型。
  2.  它创建一个 `Slice`，这个 `Slice` 的内容是**两个开放的、并列的节点结构**。例如，分割 `<p>abcdef</p>`，`Slice` 的内容会像 `<p>abc</p><p>def</p>`，但这两个 `<p>` 都是“开放”的。
  3.  它创建一个 `ReplaceStep`，在 `pos` 位置，用这个包含两个开放结构的 `Slice` 进行一次**长度为 0 的替换**（即一次插入）。ProseMirror 的 `replace` 机制会自动处理开放 `Slice` 的拼接，从而实现分割效果。
- **`canSplit(...)`**: 对应的检查函数，用于判断在某个位置进行分割是否合法。

#### `join(tr, pos, depth)` - 合并节点

- **意图**: 合并 `pos` 两侧的两个可合并的节点。最常见的例子是在一个段落的开头按“删除”键，将其与前一个段落合并。
- **实现策略**:
  1.  它找到 `pos` 两侧的两个节点。
  2.  它创建一个 `ReplaceStep`，删除这两个节点之间的“边界”。例如，在 `<p>a</p><p>b</p>` 之间，这个边界就是 `</p><p>`，长度为 2。
  3.  删除这个边界后，ProseMirror 的 `replace` 机制会发现 `a` 和 `b` 现在处于同一个父节点下，并自动将它们的内容合并。
- **`canJoin(doc, pos)`**: 对应的检查函数，用于判断 `pos` 两侧的节点是否可以合并。

### 总结

- structure.ts 是 ProseMirror 变换系统的高级 API 层，它将复杂的结构操作封装成易于理解和使用的函数。
- 这些函数的核心武器是 **`ReplaceAroundStep`**，它通过“替换外壳，保留内容”的模式，优雅地实现了 `lift`、`wrap`、`setNodeMarkup` 等操作。
- 对于 `split` 和 `join`，则巧妙地利用了 `ReplaceStep` 配合开放的 `Slice` 或删除节点边界来实现。
- 每个操作函数（如 `lift`）几乎都有一个对应的检查函数（如 `liftTarget`），这体现了 ProseMirror 命令系统（在 prosemirror-commands 中实现）的通用模式：**先检查可行性，再执行操作**。
