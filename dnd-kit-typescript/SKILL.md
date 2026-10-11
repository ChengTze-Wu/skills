---
name: dnd-kit-typescript
description: Build drag and drop and sortable lists in plain TypeScript/JavaScript (no framework) with the new dnd-kit core (@dnd-kit/dom, v0.5+). Use for DragDropManager, Draggable, Droppable, Sortable classes, monitor events, sensors, modifiers, plugins. React projects use dnd-kit-react instead. 用 dnd-kit 原生 TypeScript 做拖放與排序。
---

# dnd-kit（原生 TypeScript / JavaScript）

依據官方文件 dndkit.com（v0.5.0，2026-06）。核心套件是 `@dnd-kit/dom`，不綁任何框架。

React 專案請改用 `dnd-kit-react` skill（`@dnd-kit/react` 是這個核心外面的一層薄包裝，概念相同）。

## 不要用舊 API

舊版的 `@dnd-kit/core`、`@dnd-kit/sortable` 是 React 專用，API 完全不同。新程式碼一律用 `@dnd-kit/dom`。

## 安裝

```bash
npm install @dnd-kit/dom
npm install @dnd-kit/helpers   # 需要 move / arrayMove 時才裝
```

## 核心概念

| 概念 | 說明 |
| --- | --- |
| `DragDropManager` | 總指揮。協調所有拖曳、放置、sensors、plugins、modifiers，並提供事件 |
| `Draggable` | 可拖曳的元素 |
| `Droppable` | 可放置的目標 |
| `Sortable` | 同時是 Draggable 與 Droppable，用來排序 |

建立實體時把 `manager` 當第二個參數傳入，就會自動註冊。

## 基本配方

### 拖曳 + 放置

```ts
import {DragDropManager, Draggable, Droppable} from '@dnd-kit/dom';

const manager = new DragDropManager();
const app = document.getElementById('app')!;

const dragEl = document.createElement('button');
dragEl.textContent = 'draggable';
const draggable = new Draggable({id: 'draggable-button', element: dragEl}, manager);

const dropEl = document.createElement('div');
dropEl.className = 'droppable';
// 因為 effects 會參照 droppable 自己，需明確標註型別，否則 TS 會報循環參照
const droppable: Droppable = new Droppable({
  id: 'droppable-container',
  element: dropEl,
  // 響應式 effect：isDropTarget 變動時自動執行
  effects() {
    return [
      () => { dropEl.classList.toggle('active', droppable.isDropTarget); },
    ];
  },
}, manager);

manager.monitor.addEventListener('dragend', (event) => {
  const {operation, canceled} = event;
  if (canceled) return;                       // 被取消（如按 Esc）
  const {source, target} = operation;
  if (source && target?.id === droppable.id) {
    droppable.element?.append(source.element!);
  } else if (source) {
    app.prepend(source.element!);
  }
});

app.append(dragEl, dropEl);
```

`id` 在同一個 manager 內必須唯一。

### 排序

```ts
import {DragDropManager} from '@dnd-kit/dom';
import {Sortable, isSortable} from '@dnd-kit/dom/sortable';

const manager = new DragDropManager();
let items = ['Item 1', 'Item 2', 'Item 3', 'Item 4'];
const list = document.createElement('ul');

items.forEach((id, index) => {
  const element = document.createElement('li');
  element.textContent = id;
  new Sortable({id, index, element}, manager);   // index 必填
  list.append(element);
});

manager.monitor.addEventListener('dragend', (event) => {
  if (event.canceled) return;
  const {source} = event.operation;
  if (isSortable(source)) {
    const {initialIndex, index} = source;
    if (initialIndex !== index) {
      const next = [...items];
      const [removed] = next.splice(initialIndex, 1);
      next.splice(index, 0, removed);
      items = next;
    }
  }
});
```

DOM 順序在拖曳中已由預設的 `OptimisticSortingPlugin` 調整好，你只要在 `dragend` 同步自己的資料。也可以用 `move(items, event)`（`@dnd-kit/helpers`）。

### 多個列表

用 `group` 分組，同 group 的項目互相排序，跨 group 的移動也靠它：

```ts
list1.forEach((id, index) => new Sortable({id, index, group: 'list1', element: makeEl(id)}, manager));
list2.forEach((id, index) => new Sortable({id, index, group: 'list2', element: makeEl(id)}, manager));
```

`dragend` 時比較 `initialGroup` / `group`：相同就是同列表內重排，不同就是跨列表搬移（範例見 `references/api.md`）。

### 拖曳把手與類型

```ts
new Draggable({id: 'a', element, handle: handleEl}, manager);   // 只有 handle 能開始拖
new Draggable({id: 'a', element, type: 'item'}, manager);        // 只能放到 accept 此 type 的目標
new Droppable({id: 'z', element: zone, accept: 'item'}, manager);
new Droppable({id: 'z', element: zone, accept: ['item', 'card']}, manager);
new Droppable({id: 'z', element: zone, accept: (d) => d.type === 'item' && d.data.category === 'fruit'}, manager);
```

## 容易出錯的地方

1. **排序時 `source` 與 `target` 是同一個元素**（預設樂觀排序）。不能比對 id 判斷移動了什麼，要用 `isSortable(source)` 讀 `initialIndex`、`index`、`initialGroup`、`group`。
2. 一定要處理 `event.canceled`。
3. `manager` 的 `plugins`、`sensors`、`modifiers`：傳陣列是**取代預設**，傳 `(defaults) => [...defaults, X]` 才是擴充。
4. `Draggable`/`Droppable`/`Sortable` 的 `element` 建構時可不給，但**必須設定後才會運作**。
5. 元素移除時要清理：呼叫實體的 `destroy()`，不用時整個 manager 呼叫 `manager.destroy()`，避免殘留監聽。
6. 不想自動註冊：傳 `register: false`，之後手動 `manager.registry.register(entity)`（回傳 cleanup）。
7. 事件名稱是**小寫**：`beforedragstart`、`dragstart`、`dragmove`、`dragover`、`collision`、`dragend`。（React 版的 prop 才是 `onDragEnd` 駝峰。）
8. 在框架（Vue、Svelte、Solid）裡用，改用各自的官方套件（`@dnd-kit/vue` 等）。

## 事件（manager.monitor）

```ts
manager.monitor.addEventListener('beforedragstart', (e) => { if (blocked(e.operation.source)) e.preventDefault(); });
manager.monitor.addEventListener('dragstart', (e) => {});
manager.monitor.addEventListener('dragmove', (e) => { e.operation.position.current; });
manager.monitor.addEventListener('dragover', (e) => { /* 可 preventDefault 擋掉外掛行為 */ });
manager.monitor.addEventListener('collision', (e) => { e.collisions; });
manager.monitor.addEventListener('dragend', (e) => { e.canceled; e.operation.source; e.operation.target; });
```

`addEventListener` 會回傳 cleanup 函式。

型別可從 `@dnd-kit/dom` 匯入：`DragStartEvent`、`DragMoveEvent`、`DragOverEvent`、`DragEndEvent`、`BeforeDragStartEvent`、`CollisionEvent`、`DraggableInput`、`DroppableInput`、`Sensors`。`CollisionDetector` 型別從 `@dnd-kit/collision` 或 `@dnd-kit/abstract` 匯入。排序相關從 `@dnd-kit/dom/sortable`：`SortableInput`、`SortableTransition`、`SortableDisabled`。

## 設定摘要

```ts
import {DragDropManager, PointerSensor, KeyboardSensor, PointerActivationConstraints,
  Feedback, AutoScroller, Accessibility} from '@dnd-kit/dom';
import {RestrictToWindow, RestrictToElement} from '@dnd-kit/dom/modifiers';
import {RestrictToVerticalAxis, SnapModifier} from '@dnd-kit/abstract/modifiers';

const manager = new DragDropManager({
  sensors: (d) => [
    ...d.filter((s) => s !== PointerSensor),
    PointerSensor.configure({
      activationConstraints: [new PointerActivationConstraints.Distance({value: 8})],
    }),
  ],
  plugins: (d) => [...d, Feedback.configure({dropAnimation: null})],
  modifiers: (d) => [...d, RestrictToWindow],
});
```

預設值：sensors 為 `PointerSensor` 與 `KeyboardSensor`；plugins 為 `Accessibility`、`AutoScroller`、`Cursor`、`Feedback`、`PreventSelection`、`ScrollListener`、`Scroller`。

單一元素可覆寫：`new Draggable({..., modifiers, sensors, plugins: [Feedback.configure({feedback: 'clone'})]}, manager)`。

碰撞偵測設在每個 Droppable/Sortable：`collisionDetector: closestCenter`（從 `@dnd-kit/collision` 匯入）；重疊時用 `collisionPriority`（數字，越大越優先）。

更多細節與完整選項：

- `references/api.md`：Draggable、Droppable、Sortable、Manager 的完整欄位、屬性、方法
- `references/config.md`：sensors、modifiers（含自訂）、碰撞偵測、外掛
