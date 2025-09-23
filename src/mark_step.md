好的，我们来深入、详细地讲解 mark_step.ts 这个文件。

这个文件定义了 ProseMirror 中用于**添加或移除标记 (Mark)** 的所有原子操作 (`Step`)。标记是应用在内联内容上的样式或元数据，例如加粗 (`<strong>`)、斜体 (`<em>`) 或链接 (`<a>`)。

### 核心思想：修改标记 = 替换节点

在深入代码之前，理解一个核心概念至关重要：在 ProseMirror 的数据模型中，标记不是独立存在的实体，而是**内联节点（如 `text` 或 `image` 节点）的一个属性**（即 `node.marks` 数组）。

因此，你**不能**“凭空”给一段文本添加标记。要为一个节点添加标记，你必须：

1.  创建一个该节点的**新副本**。
2.  将新标记添加到新副本的 `marks` 数组中。
3.  用这个新副本**替换**掉文档中的旧节点。

mark_step.ts 中所有的 `Step` 都遵循这个“**创建副本并替换**”的模式。这就是为什么它们的 `apply` 方法最终都依赖于 `StepResult.fromReplace`。

这个文件定义了两种类型的标记操作：

1.  **范围操作 (`AddMarkStep`, `RemoveMarkStep`)**: 对一个 `[from, to]` 范围内的所有内联内容进行操作。
2.  **单点操作 (`AddNodeMarkStep`, `RemoveNodeMarkStep`)**: 只对 `pos` 位置的单个节点进行操作。

---

### 第一部分：范围操作 (`AddMarkStep` / `RemoveMarkStep`)

这是最常见的用例，例如用户选中一段文本后点击“加粗”按钮。

#### `AddMarkStep` - 为一个范围添加标记

- **`apply(doc: Node)`**:

  1.  `doc.slice(this.from, this.to)`: 首先，它将要修改的范围从文档中“切”出来，得到一个 `Slice` 对象。
  2.  `mapFragment(...)`: 这是核心逻辑。它调用 `mapFragment` 辅助函数，递归地遍历切片中的所有内容。
  3.  `node.mark(this.mark.addToSet(node.marks))`: 对于遍历到的每一个内联节点，它都会调用 `node.mark()` 来创建一个**新的节点副本**，这个副本的 `marks` 数组中包含了新添加的 `this.mark`。`addToSet` 确保了标记不会被重复添加。
  4.  `new Slice(...)`: 用这些被修改过的新节点，重新组装成一个新的 `Slice`。
  5.  `StepResult.fromReplace(...)`: 最后，用这个包含新标记内容的新 `Slice`，去替换掉文档中 `[from, to]` 范围的旧内容。

- **`invert(): Step`**:

  - 非常直观：添加一个标记的逆操作，就是移除同一个标记。它直接返回一个 `new RemoveMarkStep(this.from, this.to, this.mark)`。

- **`merge(other: Step)`**:
  - 一个重要的性能优化。如果两个 `AddMarkStep` 操作的是同一个标记，并且它们的范围重叠或相邻，就将它们合并成一个更大的 `AddMarkStep`。这可以减少历史记录栈中的步骤数量，使用户在连续加粗时，一次撤销就能全部复原。

#### `RemoveMarkStep` - 从一个范围移除标记

它的逻辑与 `AddMarkStep` 几乎完全相同，唯一的区别在于 `mapFragment` 中调用的函数是 `node.mark(this.mark.removeFromSet(node.marks))`，即从节点的 `marks` 数组中移除指定的标记。它的 `invert()` 方法自然就是返回一个 `AddMarkStep`。

---

### 第二部分：单点操作 (`AddNodeMarkStep` / `RemoveNodeMarkStep`)

这部分代码是您当前选择的焦点。这类操作不常见，主要用于处理那些被视为单个“原子”单位的节点，例如一个图片节点或一个特殊的占位符节点。你不能只标记图片的一部分，只能标记整个图片。

#### `AddNodeMarkStep` - 为单个节点添加标记

- **`constructor(readonly pos: number, readonly mark: Mark)`**:

  - 它只接收一个 `pos`，即目标节点在文档中的起始位置。

- **`apply(doc: Node)`**:

  1.  `let node = doc.nodeAt(this.pos)`: 获取 `pos` 位置的节点。
  2.  `let updated = node.type.create(node.attrs, null, this.mark.addToSet(node.marks))`: 这是核心。它创建一个与旧节点类型、属性都相同的新节点副本 (`updated`)，但 `marks` 数组是加入了新标记后的版本。
  3.  `return StepResult.fromReplace(doc, this.pos, this.pos + 1, ...)`: **这是最精妙的部分**。它执行了一次替换，范围是 `[pos, pos + 1]`（即单个节点的范围），用一个只包含 `updated` 节点的新 `Slice` 来替换。这再次印证了“修改标记 = 替换节点”的原则。

- **`invert(doc: Node): Step`**:

  - 这里的逻辑比 `AddMarkStep` 的 `invert` 要复杂一些，因为它需要处理一些边缘情况。
  - 理想情况下，添加一个标记的逆操作是移除它，所以它最终会 `return new RemoveNodeMarkStep(this.pos, this.mark)`。
  - 但它首先会检查一种特殊情况：如果添加这个标记实际上没有改变节点的 `marks` 集合（例如，因为另一个等价的标记已经存在），那么逆操作就不应该是移除它，而应该是添加一个在原始 `marks` 集合中但不在新集合中的标记。这是一个非常细致的正确性保证，但在绝大多数情况下，你可以认为 `invert` 就是返回一个 `RemoveNodeMarkStep`。

- **`map(mapping: Mappable): Step | null`**:
  - 它映射 `pos` 的位置。`mapResult` 的第二个参数 `1` 表示关联性偏向右边。
  - 如果 `pos.deletedAfter` 为 `true`，意味着这个节点被 `mapping` 删除了，所以返回 `null`。
  - 否则，返回一个新的、位置更新过的 `AddNodeMarkStep`。

#### `RemoveNodeMarkStep` - 从单个节点移除标记

- **`apply(doc: Node)`**:

  - 逻辑与 `AddNodeMarkStep.apply` 完全对称，只是在创建 `updated` 节点时调用的是 `this.mark.removeFromSet(node.marks)`。

- **`invert(doc: Node): Step`**:
  - 移除一个标记的逆操作就是添加同一个标记。
  - 它会检查 `this.mark` 是否真的存在于节点的 `marks` 集合中。如果不存在（可能因为这个 `Step` 是通过 `map` 产生的，而原始标记已被其他操作改变），那么它的逆操作就是它自身（一个无效的移除操作），所以返回 `this`。
  - 否则，返回 `new AddNodeMarkStep(this.pos, this.mark)`。

### 总结

1.  **统一的替换模型**: 所有四种 `MarkStep` 的本质都是 `ReplaceStep` 的一种特例。它们通过创建一个新的、`marks` 属性被修改过的节点（或节点片段），来替换掉文档中的旧版本。
2.  **范围 vs. 单点**: `Add/RemoveMarkStep` 通过 `mapFragment` 遍历一个范围，适用于文本等连续内容。`Add/RemoveNodeMarkStep` 直接操作单个节点，适用于图片等原子节点。
3.  **可逆性与可映射性**: 每个 `Step` 都精确地定义了它的 `invert` 和 `map` 方法，这使得基于标记的操作能够无缝地集成到 ProseMirror 的撤销历史和协同编辑体系中。
4.  **细节与健壮性**: `invert` 和 `merge` 方法中的各种检查，展示了 ProseMirror 在处理各种边缘情况、保证数据一致性和提供良好用户体验方面的严谨性。
