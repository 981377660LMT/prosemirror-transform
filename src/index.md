好的，我们来系统性地、有逻辑地讲解 prosemirror-transform 这个模块，并为您规划一条清晰的代码阅读顺序。

prosemirror-transform 是 ProseMirror 架构的基石之一。它提供了一种**可追溯、可逆、可组合**的方式来描述和应用对文档的修改。理解它，就等于理解了 ProseMirror 中“变化”的本质，是实现历史记录（undo/redo）和协同编辑的基础。

### 宏观逻辑：从原子操作到复杂变换

想象一下编辑文档的过程，比如“将一段文字加粗”或“删除一个段落”。这些操作可以被分解为更小、更精确的原子步骤。prosemirror-transform 的核心思想正是如此：

1.  **原子性 (Atomicity)**: 任何对文档的修改都被定义为一个或多个**`Step`**（步骤）。一个 `Step` 是最小的、不可再分的、可序列化的修改单元（例如，在位置 `x` 到 `y` 替换内容）。
2.  **可追溯性 (Traceability)**: 每个 `Step` 都会生成一个 **`StepMap`**，它精确地描述了该步骤如何影响文档中的位置。这使得我们可以在新旧文档之间准确地映射（map）坐标。
3.  **可组合性 (Composability)**: 多个 `Step` 和 `StepMap` 可以被一个 **`Transform`** 对象串联起来，形成一个完整的、复杂的“事务 (Transaction)”。`Transform` 会维护一个 `Mapping` 对象，它是所有 `StepMap` 的集合，可以一次性完成从最初始文档到最终文档的位置映射。
4.  **高级封装 (High-Level API)**: 直接操作 `Step` 很繁琐。因此，模块提供了一系列高级辅助函数（如 `replace`, `split`, `lift`），它们在内部创建并应用合适的 `Step`，为开发者提供了便捷的 API。

---

### 核心组件与代码阅读顺序

根据上述逻辑，我为您规划了从底层到上层、由浅入深的阅读顺序。

#### 第 1 站：`Step` - 变化的基本单位

**目标**：理解什么是“原子修改”。

**阅读文件**：`step.ts`

`Step` 是一个抽象基类，它定义了所有具体修改步骤必须遵守的契约。

**关键点**：

- `apply(doc)`: 将此步骤应用到一个文档上，返回一个新的文档或一个失败结果 (`StepResult`)。这是 `Step` 的核心执行逻辑。
- `invert(doc)`: 接收**修改前**的文档，生成一个与当前步骤效果相反的新 `Step`。这是实现“撤销”功能的关键。
- `getMap()`: 返回一个 `StepMap`，用于位置映射。
- `map(mapping)`: 当其他变化发生时，如何“调整”当前 `Step` 的位置。这是实现“协同编辑”中操作变换（rebasing）的核心。
- `toJSON()` 和 `Step.fromJSON()`: `Step` 可以被序列化为 JSON，这使得它可以被存储或通过网络传输。`Step.jsonID` 用于注册不同类型的 `Step`。

**接下来，阅读具体的 `Step` 实现，以获得具象认知**：

- `replace_step.ts`: 这是最重要、最常见的 `Step`。
  - `ReplaceStep`: 在 `from` 到 `to` 的范围内，用给定的 `Slice` 替换内容。
  - `ReplaceAroundStep`: 更复杂的替换，它会“保留”被替换范围中间的一部分内容，并将其移动到新插入的 `Slice` 中。
- `attr_step.ts`: `AttrStep` 用于修改单个节点的属性。
- `mark_step.ts`: `AddMarkStep` 和 `RemoveMarkStep` 用于添加或移除范围内容的标记。

#### 第 2 站：`StepMap` & `Mapping` - 位置的追踪者

**目标**：理解当文档内容增删时，位置坐标是如何变化的。

**阅读文件**：`map.ts`

**关键点**：

- **`StepMap`**: 代表**单个** `Step` 引起的位置变化。它非常轻量，通常只存储几个数字（例如，`[from, oldSize, newSize]`）。它的 `map` 方法可以告诉你一个旧坐标在新文档中的位置。
- **`Mapping`**: 代表**一整个 `Transform`**（即一系列 `Step`）引起的位置变化。
  - 它内部持有一个 `StepMap` 数组 (`maps`)。
  - 它的 `map` 方法会依次调用内部所有 `StepMap` 的 `map` 方法，像流水线一样，最终计算出从最开始的文档到最终文档的位置映射。
  - `getMirror` 和 `appendMappingInverted`: 这些是为协同编辑中的“rebase”操作设计的复杂功能，用于处理互为逆操作的步骤，可以先大致了解其作用。

#### 第 3 站：`Transform` - 变化的指挥官

**目标**：理解如何将多个 `Step` 组合成一个完整的变换。

**阅读文件**：`transform.ts`

`Transform` 类是开发者最常直接交互的对象。它以一个初始文档开始，通过一系列方法调用来累积 `Step`。

**关键点**：

- `steps: Step[]`: 存储了本次变换中所有的原子步骤。
- `docs: Node[]`: 存储了每一步执行前的文档状态，用于 `invert`。
- `mapping: Mapping`: 包含了所有已执行 `Step` 的 `StepMap`，用于位置映射。
- `step(step: Step)`: 这是向 `Transform` 中添加新步骤的核心方法。它会：
  1.  调用 `step.apply()` 生成新文档。
  2.  将旧文档、新 `Step` 分别存入 `docs` 和 `steps` 数组。
  3.  调用 `step.getMap()` 并将其添加到 `mapping` 中。
  4.  更新 `tr.doc` 为新文档。

#### 第 4 站：高级 API - 便捷的操作封装

**目标**：理解 ProseMirror 提供的便捷 API 是如何工作的。

直接调用 `tr.step(new ReplaceStep(...))` 太过繁琐。因此，prosemirror-transform 提供了大量封装好的高级方法，它们最终都会调用 `tr.step()`。

**阅读顺序**：

1.  **`replace.ts`**: 这是最核心的变换逻辑。

    - `replaceStep()`: 这是所有替换操作的入口。它会创建一个 `Fitter` 对象来执行复杂的“适配”逻辑。
    - `Fitter` 类: 这是 replace.ts 的精华。当你想插入一个 `Slice` 时，文档的结构可能会被破坏。`Fitter` 的任务就是通过一系列智能算法（如 `findFittable`），计算出如何通过闭合节点、打开节点、注入节点 (`fillBefore`) 或包裹节点 (`findWrapping`) 等方式，以最“自然”的方式将 `Slice` 融入文档，并生成最终的 `ReplaceStep` 或 `ReplaceAroundStep`。
    - 阅读 `Transform.prototype.replace` (`transform.ts`)，你会发现它只是 `replaceStep` 的一个简单封装。`delete` 和 `insert` 又是 `replace` 的封装。

2.  **`structure.ts`**: 提供了一系列用于修改文档结构的高级函数。

    - `lift`: 将一段内容从其父节点中“提升”出来。
    - `wrap`: 用新的父节点“包裹”一段内容。
    - `split`: 在指定位置“分割”一个节点。
    - `join`: “连接”两个相邻的节点。
    - `setBlockType`: 改变一个块级节点的类型（例如，`p` -> `h1`）。
    - 这些函数内部都会计算并创建正确的 `ReplaceStep` 或 `ReplaceAroundStep` 来实现效果。

3.  **`mark.ts`**:
    - `addMark` 和 `removeMark`: 遍历指定范围，创建一系列 `AddMarkStep` 和 `RemoveMarkStep`。

### 总结与回顾

- **起点**: `index.ts` - 查看模块导出了哪些核心类和函数。
- **核心抽象**: `step.ts` -> `map.ts` -> `transform.ts`
- **核心实现**: `replace_step.ts` -> `replace.ts` (尤其是 `Fitter` 类)
- **功能扩展**: `structure.ts`, `mark.ts`, `attr_step.ts`

遵循这个顺序，您将能够清晰地理解 prosemirror-transform 是如何从底层的原子操作一步步构建起一个强大、可靠且富有表现力的文档变换系统的。
