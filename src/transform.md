好的，我们来深入、详细地讲解 transform.ts 这个文件，特别是 `Transform` 类。这是开发者与 ProseMirror 变换系统交互的**主要入口和高级 API**。

在之前的讲解中，我们分析了 `Step`（原子变化）、`StepMap`（位置映射）和 `Mapping`（映射管道）。这些都是底层的、精密的构建块。而 `Transform` 类，就是将这些构建块封装起来，提供一个流畅、易于使用的“事务构建器”。

### 宏观定位：`Transform` 是什么？

你可以将一个 `Transform` 对象想象成：

1.  **一个事务构建器 (Transaction Builder)**: 它从一个初始的文档状态开始，然后你可以通过调用它的各种方法，一步步地向这个“事务”中添加修改。
2.  **一个变更记录器 (Change Recorder)**: 它精确地记录了你所做的每一个原子修改（`Step`），以及这些修改前后的文档状态（`docs`），还有所有这些修改累积起来的位置映射（`mapping`）。
3.  **一个流畅的 API (Fluent API)**: 它的大多数方法都返回 `this`（`Transform` 对象本身），这使得你可以将多个操作链接起来，写出像 `tr.delete(10, 12).insert(10, "new")` 这样清晰、连贯的代码。

最终，prosemirror-state 中的 `Transaction` 类会继承 `Transform`，为其添加选区管理等更多功能。因此，理解 `Transform` 就是理解 `Transaction` 的核心。

---

### 第一部分：`Transform` 的核心状态（内部属性）

一个 `Transform` 对象内部维护着四个至关重要的属性，它们共同描述了一个完整的变换过程：

```typescript
// ...existing code...
export class Transform {
  /// The steps in this transform.
  readonly steps: Step[] = []
  /// The documents before each of the steps.
  readonly docs: Node[] = []
  /// A mapping with the maps for each of the steps in this transform.
  readonly mapping: Mapping = new Mapping()

  constructor(
    public doc: Node
  ) {}
// ...existing code...
```

- **`doc: Node`**: 这是**当前**的文档状态。每当一个 `Step` 被成功应用，这个属性就会被更新为新的文档。它始终代表变换过程中的最新结果。
- **`steps: Step[]`**: 一个 `Step` 对象的数组，按照应用顺序记录了所有发生过的原子修改。这是整个变换的“配方”。
- **`docs: Node[]`**: 一个文档快照的数组。`docs[i]` 存储的是应用 `steps[i]` **之前**的文档状态。这个数组对于计算逆操作 (`invert`) 至关重要。
- **`mapping: Mapping`**: 一个 `Mapping` 对象，它包含了 `steps` 数组中每一个 `Step` 对应的 `StepMap`。这个 `mapping` 可以将任何一个在变换**开始前**的位置，映射到变换**结束后**的对应位置。

---

### 第二部分：核心机制 - `step()` 和 `addStep()`

所有 `Transform` 的魔法都归结于这两个方法。

- **`step(step: Step)`**: 这是向 `Transform` 添加一个原子修改的**公共**方法。

  1.  它调用 `this.maybeStep(step)` 尝试应用这个 `step`。
  2.  `maybeStep` 内部会执行 `step.apply(this.doc)`。
  3.  如果 `apply` 成功（返回的 `result.failed` 为假），`maybeStep` 就会调用 `this.addStep()` 来记录这次成功的修改。
  4.  如果 `apply` 失败，`step()` 方法会直接抛出一个 `TransformError`。

- **`addStep(step: Step, doc: Node)`**: 这是一个**内部**方法，是 `Transform` 状态更新的核心。当一个 `step` 被确认可以成功应用后，这个方法会被调用：
  1.  `this.docs.push(this.doc)`: 将**当前**的文档（即应用此 `step` 之前的状态）存入 `docs` 数组，以备后用（例如撤销）。
  2.  `this.steps.push(step)`: 将这个 `step` 本身存入 `steps` 数组。
  3.  `this.mapping.appendMap(step.getMap())`: 从 `step` 中获取它的 `StepMap`，并将其追加到 `Transform` 的主 `mapping` 中。
  4.  `this.doc = doc`: 用 `apply` 方法返回的**新文档**来更新 `Transform` 的 `doc` 属性。

**小结**: 每一次成功的变换，都是通过 `step()` -> `maybeStep()` -> `addStep()` 的流程，同步更新 `doc`, `steps`, `docs`, `mapping` 这四个核心属性，确保了变换过程的完整性和一致性。

---

### 第三部分：高级 API - 便捷的变换方法

`Transform` 类提供了大量便捷的方法，它们本质上都是“语法糖”，最终都会创建一到多个 `Step` 对象，并通过 `this.step()` 来应用它们。

我们可以将这些方法分为几类：

#### 1. 基础内容修改 (The Primitives)

这些方法提供了最直接、最精确的内容修改能力。

- **`replace(from, to, slice)`**: 这是所有内容修改的基础。它在内部调用 `replaceStep()` 辅助函数来创建一个 `ReplaceStep`，然后调用 `this.step()`。
- **`replaceWith(from, to, content)`**: `replace` 的一个变体，可以直接接收 `Node` 或 `Fragment` 作为内容。
- **`delete(from, to)`**: 相当于 `this.replace(from, to, Slice.empty)`。
- **`insert(pos, content)`**: 相当于 `this.replaceWith(pos, pos, content)`。

#### 2. “智能”内容修改 (The WYSIWYG Helpers)

这些方法在处理粘贴、拖拽等场景时更有用，因为它们不要求 `from` 和 `to` 是精确的，而是将其作为“提示”，并尝试寻找一个更符合用户预期的、结构上有效的替换方案。

- **`replaceRange(from, to, slice)`**
- **`replaceRangeWith(from, to, node)`**
- **`deleteRange(from, to)`**

**关键区别**: `replace` 会严格遵守 `from` 和 `to`，如果替换在结构上无效，它就会失败。而 `replaceRange` 则会尝试通过扩大范围、关闭开放节点等方式来“修复”替换，使其变得有效。例如，当粘贴一个列表项到段落中间时，`replaceRange` 可能会决定将整个段落都替换掉。

#### 3. 结构化操作

这些方法用于修改文档的树状结构，它们通常会创建复杂的 `ReplaceAroundStep`。

- **`lift(range, target)`**: 将 `range` 内的内容从其父节点中“提升”出来。
- **`join(pos)`**: 合并 `pos` 位置两侧的块级节点。
- **`wrap(range, wrappers)`**: 用 `wrappers` 将 `range` 内的内容包裹起来。
- **`split(pos, depth)`**: 在 `pos` 位置分割节点。
- **`setBlockType(from, to, type)`**: 改变 `[from, to]` 范围内文本块的类型（例如，从段落变为标题）。

#### 4. 属性和标记操作

这些方法直接创建 `AttrStep` 或 `MarkStep`。

- **`setNodeAttribute(pos, attr, value)`**: 修改单个节点的属性。
- **`setDocAttribute(attr, value)`**: 修改顶层 `doc` 节点的属性。
- **`addMark(from, to, mark)`**: 为一个范围添加标记。
- **`removeMark(from, to, mark)`**: 从一个范围移除标记。
- **`setNodeMarkup(pos, type, attrs, marks)`**: 一个强大的方法，可以一次性修改单个节点的类型、属性和标记。

### 总结

- `Transform` 是一个**状态机**，它从一个初始 `doc` 开始，通过一系列 `Step`，最终到达一个新的 `doc` 状态。
- 它的核心是 `addStep` 方法，该方法原子性地更新 `doc`, `steps`, `docs`, `mapping` 四个属性，保证了变换过程的**数据一致性**。
- 它提供的大量高级 API（如 `replaceRange`, `lift`, `wrap`）将创建 `Step` 的复杂逻辑封装起来，为开发者提供了**强大而便捷的工具**。
- 通过链式调用，`Transform` 使得构建复杂的文档修改变得**清晰易读**。

当你使用 `editor.dispatch(tr.split(10).addMark(10, 15, boldMark))` 时，你就是在创建一个 `Transform` 对象，向其中添加一个 `SplitStep` 和一个 `AddMarkStep`，然后将这个完整的、记录了所有变化的 `Transform`（即 `Transaction`）交给编辑器状态系统去处理。这就是 `Transform` 在 ProseMirror 生态系统中的核心作用。
