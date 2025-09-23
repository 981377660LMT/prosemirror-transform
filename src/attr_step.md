好的，我们来深入、详细地讲解 attr_step.ts 这个文件。

这个文件定义了 ProseMirror 中用于**修改节点属性 (Attribute)** 的原子操作 (`Step`)。节点属性是附加在节点上的元数据，例如 `heading` 节点的 `level` 属性，或者 `image` 节点的 `src` 属性。

### 核心思想：修改属性 = 替换节点

与 `mark_step.ts` 类似，理解这个文件的关键在于掌握 ProseMirror 的核心设计原则：**数据模型的不可变性 (immutability)**。

在 ProseMirror 中，节点对象是不可变的。你不能直接修改一个已有节点的属性，比如 `node.attrs.level = 2`。要改变一个节点的属性，你必须：

1.  创建一个该节点的**新副本**。
2.  在创建新副本时，传入新的属性对象。
3.  用这个新副本**替换**掉文档中的旧节点。

`AttrStep` 完美地遵循了这个“**创建副本并替换**”的模式。

这个文件定义了两种 `Step`，分别处理两种不同的情况：

1.  **`AttrStep`**: 修改文档中**任意位置**的单个节点的属性。
2.  **`DocAttrStep`**: 一个特例，专门用于修改**顶层 `doc` 节点**的属性。

---

### 第一部分：`AttrStep` - 修改任意节点的属性

这是您当前选择的焦点。`AttrStep` 用于执行诸如“将一个标题从 `h2` 变为 `h3`”（即修改 `level` 属性）或“更改一张图片的 URL”（即修改 `src` 属性）之类的操作。

#### 1. 构造函数 `constructor(...)`

```typescript
// ...existing code...
  constructor(
    readonly pos: number,
    readonly attr: string,
    readonly value: any
  ) {
// ...existing code...
```

- `pos: number`: 目标节点在文档中的起始位置。
- `attr: string`: 要修改的属性的名称，例如 `"level"`。
- `value: any`: 该属性的新值，例如 `3`。

#### 2. 核心方法实现

- **`apply(doc: Node)`**:
  这是 `AttrStep` 的核心执行逻辑，完美地展示了“替换模式”。

  1.  `let node = doc.nodeAt(this.pos)`: 首先，获取 `pos` 位置的目标节点。
  2.  `let attrs = Object.create(null); ...`: 创建一个新的、干净的属性对象。它通过遍历旧节点的 `node.attrs` 来复制所有旧属性。这是一个安全的做法，避免了直接修改可能被冻结的旧属性对象。
  3.  `attrs[this.attr] = this.value`: 在这个**新的**属性对象上，设置或更新指定的属性值。
  4.  `let updated = node.type.create(attrs, null, node.marks)`: 使用节点的原始类型 (`node.type`)，传入**新的** `attrs` 对象，以及原始的 `marks`，来创建一个全新的节点副本 (`updated`)。
  5.  `return StepResult.fromReplace(...)`: **这是最关键的一步**。它执行了一次替换操作，范围是 `[pos, pos + 1]`（即单个节点的范围），用一个只包含 `updated` 节点的新 `Slice` 来替换旧节点。这与 `AddNodeMarkStep` 的实现模式完全相同，再次强调了 ProseMirror 中“修改即替换”的统一模型。

- **`getMap()`**:

  - 返回 `StepMap.empty`。
  - **为什么？** 因为这个操作虽然替换了一个节点，但新旧节点的长度都是 1。它没有增加或删除任何内容，也没有改变文档的整体长度。因此，文档中任何位置的坐标在操作前后都没有发生变化。这是一个纯粹的“原地”属性修改，所以它不需要位置映射。

- **`invert(doc: Node)`**:

  - 如何撤销一次属性修改？答案是**将该属性设置回它的原始值**。
  - 它返回一个新的 `AttrStep`，其 `pos` 和 `attr` 与当前步骤相同，但 `value` 是从**变化前**的文档中获取的原始值：`doc.nodeAt(this.pos)!.attrs[this.attr]`。

- **`map(mapping: Mappable)`**:

  - 协同编辑的核心。它需要计算当其他变化 (`mapping`) 发生后，这个 `AttrStep` 应该如何调整。
  - `let pos = mapping.mapResult(this.pos, 1)`: 它映射目标节点的位置。
  - `return pos.deletedAfter ? null : ...`: 如果 `mapping` 删除了这个节点，那么这个 `Step` 就没有意义了，返回 `null`。
  - 否则，返回一个新的、位置更新过的 `AttrStep`。

- **`toJSON()` / `fromJSON()`**:
  - 标准的序列化和反序列化逻辑，用于网络传输或持久化。

---

### 第二部分：`DocAttrStep` - 修改顶层文档节点的属性

这是一个特殊情况。`doc` 节点是所有节点的根，它本身也可以有属性。例如，你可能想在 `doc` 节点上存储一个 `version` 或 `title` 属性。

#### 为什么需要一个单独的 `Step`？

因为你**不能替换 `doc` 节点**。`doc` 节点是整个文档本身。修改它的属性需要一种不同的机制。

- **`apply(doc: Node)`**:

  1.  与 `AttrStep` 类似，它创建一个新的属性对象 `attrs` 并更新指定的属性。
  2.  **关键区别**：它调用 `doc.type.create(attrs, doc.content, doc.marks)` 来创建一个**全新的 `doc` 节点**。注意，它传入了原始的 `doc.content`，因为内容本身没有改变。
  3.  `return StepResult.ok(updated)`: 它直接返回一个包含新 `doc` 节点的成功结果，而不是通过 `fromReplace`。

- **`getMap()`**:

  - 同样返回 `StepMap.empty`，因为文档内容和长度没有变化。

- **`invert(doc: Node)`**:

  - 与 `AttrStep` 逻辑相同，返回一个将属性设置回原始值的 `DocAttrStep`。

- **`map(mapping: Mappable)`**:
  - 返回 `this`。
  - **为什么？** 因为 `doc` 节点的属性修改与文档内部的位置变化无关。无论其他用户在文档中如何增删内容，修改 `doc` 节点的 `title` 属性这个操作本身永远不会被“删除”或“移动”。它始终是有效的，因此不需要映射。

### 总结

1.  **统一的替换模型**: `AttrStep` 再次强化了 ProseMirror 的核心模式：对节点的任何修改（无论是标记还是属性）都是通过创建新节点副本来实现的，并通过一次微小的 `ReplaceStep` 来完成替换。
2.  **无位置变化的 `Step`**: `AttrStep` 和 `DocAttrStep` 是很好的例子，说明了有些 `Step` 虽然改变了文档状态，但并不影响文档的坐标系统，因此它们的 `getMap()` 返回 `StepMap.empty`。
3.  **特殊情况的处理**: `DocAttrStep` 的存在表明了 ProseMirror 设计的严谨性。它为 `doc` 节点这个无法被替换的特殊情况提供了专门的、逻辑正确的处理方式。
