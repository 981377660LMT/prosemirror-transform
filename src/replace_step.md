好的，我们来深入、详细地讲解 replace_step.ts 这个文件。这是 prosemirror-transform 中最核心、最基本的文件，因为它定义了所有**内容修改**的基础。几乎你在编辑器里做的每一个涉及内容增删的操作（打字、删除、粘贴、拖拽）最终都会被分解成这里的 `Step`。

这个文件定义了两种 `Step`：

1.  **`ReplaceStep`**: 最基础的“替换”操作。可以看作是文档级别的 `splice`，即删除一个范围的内容，并在此处插入新的内容。
2.  **`ReplaceAroundStep`**: 一种更高级、更强大的替换。它在替换一个范围的同时，会**保留**该范围内部的一小块“峡谷 (gap)”内容，并将其移动到新插入的内容中。

---

### 第一部分：`ReplaceStep` - 内容修改的基石

`ReplaceStep` 是 ProseMirror 中最常见的 `Step`。它代表了一个简单的意图：**在文档的 `[from, to]` 位置，删除原有内容，并插入 `slice` 的内容。**

#### 1. 构造函数 `constructor(...)`

```typescript
// ...existing code...
  constructor(
    readonly from: number,
    readonly to: number,
    readonly slice: Slice,
    readonly structure = false
  ) {
// ...existing code...
```

- `from: number`: 替换范围的起始位置。
- `to: number`: 替换范围的结束位置。
- `slice: Slice`: 要插入的内容。`Slice` 是一个非常重要的概念，它代表一片文档的“切片”，可以有“开放”的开始和结束边，这使得它可以智能地与被切开的文档结构进行拼接。
- `structure: boolean`: 一个内部使用的、非常重要的标志。当它为 `true` 时，这个 `Step` 只应该在 `[from, to]` 范围是“空的结构”（即只包含节点的闭合和开启标记，没有实际内容）时才能成功应用。这是一个**安全锁**，主要用于协同编辑。当一个用户的操作（比如“分割段落”）被重基（rebase）时，这个标志可以防止它意外地覆盖掉其他用户在此期间插入的文本。

#### 2. 核心方法实现

- **`apply(doc: Node)`**:

  - 它首先检查 `this.structure` 安全锁。如果为 `true`，它会调用 `contentBetween()` 辅助函数（稍后讲解）来验证 `[from, to]` 之间没有实际内容。如果验证失败，则返回 `StepResult.fail`。
  - 如果安全检查通过，它就直接调用 `StepResult.fromReplace(doc, this.from, this.to, this.slice)`。这个便捷函数会尝试执行 `doc.replace()`，并自动处理可能抛出的 `ReplaceError`，返回一个 `StepResult`。

- **`getMap()`**:

  - 返回一个 `new StepMap([this.from, this.to - this.from, this.slice.size])`。
  - 这是 `StepMap` 最基础的形式，一个包含三个数字的数组：`[替换起始点, 删除的长度, 插入的长度]`。这个 `StepMap` 对象包含了足够的信息，可以将任何一个在旧文档中的位置，映射到新文档中的对应位置。

- **`invert(doc: Node)`**:

  - 这是 `Step` 设计精妙之处的完美体现。如何撤销一次替换？答案是**执行一次反向的替换**。
  - 它返回一个新的 `ReplaceStep`：
    - `from`: 替换的起始点不变，还是 `this.from`。
    - `to`: 替换的结束点是 `this.from + this.slice.size`，即新插入内容的结尾。
    - `slice`: 要插入的内容是**原始文档在 `[from, to]` 范围内的切片**，即 `doc.slice(this.from, this.to)`。
  - 简单来说，就是用被删除的旧内容，去替换掉新插入的内容。

- **`map(mapping: Mappable)`**:

  - 这是协同编辑的核心。它计算当另一个变化 (`mapping`) 发生后，当前的 `ReplaceStep` 应该如何调整。
  - 它分别映射 `from` 和 `to` 的位置。`mapResult` 返回的对象包含新的位置 `pos` 和一个 `deletedAcross` 标志。
  - 如果 `deletedAcross` 同时为 `true`，意味着整个替换范围都被 `mapping` 删除了，那么这个 `Step` 就没有存在的意义了，返回 `null`。
  - 否则，它返回一个新的、位置被调整过的 `ReplaceStep`。

- **`merge(other: Step)`**:

  - 一个性能优化。它尝试将两个相邻的、可以合并的 `ReplaceStep` 合并成一个。
  - 例如，用户连续输入 "a" 和 "b"，会产生两个 `ReplaceStep`。如果这两个 `Step` 的边界能够完美衔接（即第一个的结束点是第二个的开始点），并且它们的 `slice` 都没有开放的边，那么它们就可以被合并成一个插入 "ab" 的 `ReplaceStep`。这有助于减少历史记录栈的深度。

- **`toJSON()` / `fromJSON()`**:
  - 标准的序列化和反序列化逻辑，将 `Step` 的所有属性转换成 JSON 格式，或从 JSON 对象中重建 `Step`。

---

### 第二部分：`ReplaceAroundStep` - 结构化操作的利器

`ReplaceAroundStep` 远比 `ReplaceStep` 复杂，但它能实现一些非常强大的结构化操作，例如**包裹 (wrapping)** 和 **提升 (lifting)**。

想象一下，你想把一段文字用 `<blockquote>` 包裹起来。这个操作可以被描述为：删除这段文字，然后插入一个包含 `<blockquote>` 和这段文字的新结构。但更精确的描述是：**用一个包含 `<blockquote>` 的“模板”来替换掉这段文字周围的结构，同时将这段文字本身移动到模板的“洞”里。** 这就是 `ReplaceAroundStep` 所做的事情。

#### 1. 构造函数 `constructor(...)`

它有更多的参数来描述这个复杂的操作：

- `from`, `to`: 整个被替换的大范围的起止点。
- `gapFrom`, `gapTo`: 在 `[from, to]` 内部，需要被**保留并移动**的“峡谷”内容的起止点。
- `slice`: 要插入的“模板”切片。这个 `slice` 必须有一个“洞”，即它的内容中某处是空的。
- `insert: number`: “洞”在 `slice` 内部的起始位置。`[gapFrom, gapTo]` 的内容将被移动到这里。
- `structure`: 与 `ReplaceStep` 中的含义相同，一个安全锁。

#### 2. 核心方法实现

- **`apply(doc: Node)`**:

  1.  执行 `structure` 安全检查，确保 `[from, gapFrom]` 和 `[gapTo, to]` 这两个“外壳”部分没有实际内容。
  2.  从原始文档中提取“峡谷”内容：`let gap = doc.slice(this.gapFrom, this.gapTo)`。
  3.  将 `gap` 内容插入到“模板” `slice` 的“洞”中：`this.slice.insertAt(this.insert, gap.content)`。
  4.  用这个填充了内容的新 `slice`，对原始文档的 `[from, to]` 范围进行一次标准的替换。

- **`getMap()`**:

  - 它的 `StepMap` 更复杂，是一个包含六个数字的数组，精确地描述了 `[from, gapFrom]`, `[gapTo, to]` 被替换掉，以及 `[gapFrom, gapTo]` 被移动的过程。

- **`invert(doc: Node)`**:

  - 同样是创建一个反向的 `ReplaceAroundStep`。逻辑非常复杂，本质上是将被包裹的内容“解包”，并用原始的外部结构替换掉新的外部结构。

- **`map(mapping: Mappable)`**:
  - 比 `ReplaceStep` 的 `map` 更复杂，因为它需要同时映射 `from`, `to`, `gapFrom`, `gapTo` 四个位置，并确保它们在映射后仍然保持正确的相对关系。

---

### 第三部分：`contentBetween()` 辅助函数

这个函数是 `structure` 安全锁的具体实现。它检查一个范围 `[from, to]` 是否只包含“结构性”的节点闭合和开启，而没有“实质性”的内容（如文本节点或叶子节点）。

它通过从 `from` 点开始，模拟光标在文档树中向 `to` 点移动，如果一路上只遇到节点的边界（即从一个节点的末尾跳到下一个节点的开头），则认为是安全的；一旦遇到任何叶子节点或文本，就立即返回 `true`，表示“有内容”，不安全。

### 总结

- **`ReplaceStep` 是万物基础**。所有简单的增删改都由它完成。它的核心是 `[from, to, slice]`。
- **`ReplaceAroundStep` 是结构化魔术师**。所有涉及“包裹”、“提升”、“分割时保持父节点”等复杂结构变换的操作，都由它完成。它的核心是 `[from, to]` 的大范围和 `[gapFrom, gapTo]` 的保留“峡谷”。
- **`structure` 标志是协同编辑的守护神**，它防止了重基后的 `Step` 意外地破坏用户数据。
- 这两个 `Step` 的设计，完美地展示了 ProseMirror 如何将所有可能的文档修改，都抽象为可序列化、可逆、可映射的原子操作，为整个框架的稳定性、可扩展性和协同能力奠定了坚实的基础。
