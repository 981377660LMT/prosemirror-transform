好的，我们来深入、详细地讲解 mark.ts 这个文件。

这个文件与 `mark_step.ts` 密切相关，但处于一个更高的抽象层次。`mark_step.ts` 定义了**原子性**的 `AddMarkStep` 和 `RemoveMarkStep`。而 mark.ts 则提供了**智能的、面向开发者意图的**高级 API，这些 API 会经过复杂的计算，最终生成一系列原子性的 `MarkStep` 来完成任务。

这个文件主要包含三个函数：

1.  **`addMark(tr, from, to, mark)`**: 为一个范围添加标记。
2.  **`removeMark(tr, from, to, mark)`**: 从一个范围移除标记。
3.  **`clearIncompatible(tr, pos, parentType)`**: 一个非常特殊的工具，用于在节点内容被移动到新的父节点时，清理掉所有不兼容的标记和节点。

---

### 第一部分：`addMark` - 智能的标记添加

这是您当前选择的焦点。乍一看，`addMark` 的实现似乎比预期的要复杂得多。为什么不直接 `tr.step(new AddMarkStep(from, to, mark))` 就行了呢？

答案在于 ProseMirror `Mark` 系统的两个核心规则：

1.  **唯一性**: 一个节点不能拥有两个完全相同的标记。
2.  **排他性 (`excludes`)**: 标记可以在其 `MarkSpec` 中定义排他性。例如，一个 `link` 标记通常会定义 `excludes: "_"`，意味着它会排除所有其他 `link` 标记。你不能同时拥有两个链接。

`addMark` 函数必须优雅地处理这些规则。它的工作不仅仅是“添加”，更准确地说是“**确保一个范围拥有指定的标记**”，这可能需要先移除与之冲突的旧标记。

#### `addMark` 的实现策略：

1.  **遍历节点**: `tr.doc.nodesBetween(from, to, ...)` 遍历 `[from, to]` 范围内的所有节点。它只关心内联节点 (`node.isInline`)。

2.  **检查与计算**: 对于每个内联节点，它会：

    - 检查新 `mark` 是否已经存在，以及父节点是否允许此 `mark` 类型。
    - `let newSet = mark.addToSet(marks)`: **这是最关键的一步**。它调用 `Mark` 对象的 `addToSet` 方法，这个方法会自动处理排他性规则。如果 `marks` 中有一个与新 `mark` 冲突的旧 `link`，`addToSet` 返回的新集合 `newSet` 将只包含新 `link`，而没有旧的。

3.  **生成 `Step`**:

    - **移除冲突标记**: `if (!marks[i].isInSet(newSet))` 这个条件判断的是：“这个旧标记在新的标记集合里消失了吗？” 如果是，就意味着它与新 `mark` 冲突了，必须被移除。因此，代码会创建一个 `RemoveMarkStep` 并放入 `removed` 数组。
    - **添加新标记**: 代码为新 `mark` 创建一个 `AddMarkStep` 并放入 `added` 数组。

4.  **`Step` 合并优化**:

    - `if (removing && removing.to == start ...)` 和 `if (adding && adding.to == start ...)` 这两段是重要的性能优化。
    - `nodesBetween` 会逐个访问相邻的文本节点。如果两个相邻的文本节点都需要执行完全相同的标记操作（例如，都移除 `strong` 标记），这段代码不会创建两个独立的 `RemoveMarkStep`，而是将前一个 `Step` 的 `to` 属性扩展到当前节点的末尾。
    - 这使得最终生成的 `Step` 数量更少、范围更大，这不仅提升了性能，也让撤销/重做操作更符合用户预期（一次撤销就能取消整个加粗操作，而不是一个字一个字地取消）。

5.  **应用 `Step`**:
    - `removed.forEach(s => tr.step(s))`
    - `added.forEach(s => tr.step(s))`
    - 它**先应用所有移除步骤，再应用所有添加步骤**。这个顺序是重要的，可以避免中间状态出现问题。

**小结**: `addMark` 是一个智能函数，它通过计算标记集合的变化，自动处理标记的排他性，并优化生成的 `Step`，最终实现一个健壮且高效的“添加标记”操作。

---

### 第二部分：`removeMark` - 灵活的标记移除

`removeMark` 函数同样非常灵活，它的 `mark` 参数可以接收三种类型的值：

- `Mark` 实例: 精确移除这一个标记。
- `MarkType` 实例: 移除所有此类型的标记（例如，移除所有 `strong` 标记，无论其属性如何）。
- `null` 或 `undefined`: 移除范围内的**所有**标记。

#### `removeMark` 的实现策略：

1.  **遍历与匹配**: 它同样使用 `nodesBetween` 遍历范围内的内联节点。
2.  **收集待移除项**: 根据传入的 `mark` 参数的类型，它会构建一个 `toRemove` 数组，包含当前节点上所有需要被移除的 `Mark` 对象。
3.  **合并 `Step`**: 与 `addMark` 类似，它也有一个复杂的合并逻辑。
    - 它使用一个 `matched` 数组来存储正在构建中的 `RemoveMarkStep` 的信息（`{style, from, to, step}`）。
    - `step` 变量用于追踪它正在访问第几个内联节点。
    - 当它发现当前节点需要移除的标记 `style` 与上一个节点（`m.step == step - 1`）需要移除的标记相同时，它不会创建新条目，而是更新 `matched` 中已有条目的 `to` 和 `step`，从而实现 `Step` 的合并。
4.  **应用 `Step`**: 遍历 `matched` 数组，为每个合并好的条目创建一个 `RemoveMarkStep` 并应用到 `tr` 上。

---

### 第三部分：`clearIncompatible` - 内容净化器

这是一个更特殊、更底层的工具，通常在执行“剪切-粘贴”或“拖拽”等涉及到将节点内容移动到**不同类型父节点**下的操作时被内部调用。

- **意图**: 假设你要将 `<h1>` 里的内容（例如 `<strong>abc</strong>`）移动到一个 `<code>` 块中。`<code>` 块可能不允许 `<strong>` 标记。`clearIncompatible` 的作用就是“净化”这段内容，使其符合新家（`<code>` 块）的规则。
- **实现策略**:
  1.  它遍历 `pos` 位置节点的所有子节点。
  2.  **节点类型检查**: `allowed = match.matchType(child.type)` 检查子节点的类型是否被新父节点允许。如果不允许，就创建一个 `ReplaceStep` 将其整个删除。
  3.  **标记检查**: `if (!parentType.allowsMarkType(child.marks[j].type))` 检查子节点上的标记是否被新父节点允许。如果不允许，就创建一个 `RemoveMarkStep` 将其移除。
  4.  **换行符处理**: 如果新父节点不是预格式化文本（`whitespace != "pre"`），它还会将所有硬换行符替换为空格。
  5.  **填充缺失节点**: `if (!match.validEnd)` 检查在处理完所有子节点后，新父节点的内容是否满足其 Schema。如果不满足（例如，一个 `table_row` 必须包含 `table_cell`，但内容里没有），它会自动填充必要的空节点。

### 总结

- mark.ts 提供的是比 `mark_step.ts` 更高层、更智能的 API。
- `addMark` 和 `removeMark` 的核心复杂性在于**处理标记的排他性规则**和**对生成的 `Step` 进行合并优化**，以提供更好、更高效的变换。
- `clearIncompatible` 是一个强大的“内容净化”工具，它保证了在复杂的结构变换中，内容能够被安全、合法地迁移到新的父节点下，是 ProseMirror 健壮性的重要体现。
