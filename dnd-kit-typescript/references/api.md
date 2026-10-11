# API 參考（@dnd-kit/dom v0.5）

## DragDropManager

```ts
import {DragDropManager} from '@dnd-kit/dom';
const manager = new DragDropManager({sensors?, plugins?, modifiers?});
```

三個選項都可傳陣列（取代預設）或函式 `(defaults) => [...]`（擴充預設）。

屬性：

- `registry`：`draggables`、`droppables`、`plugins`、`sensors`、`modifiers`
- `dragOperation`：`source`、`target`、`position`、`status`、`canceled`
- `monitor`：`addEventListener`、`removeEventListener`
- `renderer`：和 React 這類非同步 renderer 整合用

方法：`destroy()`（取消註冊所有實體、清理外掛/sensors/modifiers、移除所有監聽）。

註冊：

```ts
const cleanup = manager.registry.register(draggable);  // 手動註冊
cleanup();                                              // 或 manager.registry.unregister(draggable)
new Draggable({id, element, register: false}, manager); // 不自動註冊
```

## Draggable

```ts
import {Draggable} from '@dnd-kit/dom';
const draggable = new Draggable(input, manager);
```

| 選項 | 說明 |
| --- | --- |
| `id` | 必填，`string \| number`，manager 內唯一 |
| `element` | 要拖的 DOM 元素 |
| `handle` | 把手元素。省略則整個元素可拖 |
| `type` | 限制可放到哪些 droppable（配合 `accept`） |
| `disabled` | 暫時停用 |
| `modifiers`、`sensors` | 此元素專用，優先於全域 |
| `plugins` | `PluginDescriptor[]`，如 `[Feedback.configure({feedback: 'clone'})]` |
| `alignment` | `{x: 'start'\|'center'\|'end', y: ...}` |
| `data` | 附加資料 |
| `effects` | `() => Effect[]`，進階：響應式副作用，`destroy()` 時自動清理 |

屬性：`id`、`element`、`handle`、`type`、`disabled`、`isDragging`、`isDropping`。
方法：`register()`、`unregister()`、`destroy()`。

`feedback` 模式：`'default'`、`'clone'`（原位留複本）、`'move'`（不升 top layer、無佔位）、`'none'`（不做視覺回饋，自己處理）。

## Droppable

```ts
import {Droppable} from '@dnd-kit/dom';
const droppable = new Droppable(input, manager);
```

| 選項 | 說明 |
| --- | --- |
| `id`、`element` | 同上 |
| `accept` | `Type \| Type[] \| (draggable) => boolean`。省略則全收 |
| `type` | 分類用。`accept` 只比對拖曳者的 `type` |
| `collisionDetector` | 預設 `defaultCollisionDetection` |
| `collisionPriority` | 數字，越大越優先 |
| `disabled`、`data`、`effects` | 同上 |

屬性：`id`、`element`、`disabled`、`isDropTarget`、`shape`。
方法：`accepts(draggable)`、`refreshShape()`、`register()`、`unregister()`、`destroy()`。

## Sortable

```ts
import {Sortable, isSortable, isSortableOperation} from '@dnd-kit/dom/sortable';
```

| 選項 | 說明 |
| --- | --- |
| `id`、`index` | 必填 |
| `element` | 要排序的元素 |
| `group` | 分組，同組互相排序 |
| `handle`、`target` | 把手；另指定的放置目標元素 |
| `transition` | `{duration?: number (預設 250), easing?: string, idle?: boolean (預設 false)}` 或 `null` |
| `disabled` | `boolean` 或 `{draggable?: boolean; droppable?: boolean}` |
| `type`、`accept` | 限制可一起排序的類型 |
| `modifiers`、`sensors`、`plugins`、`data` | 同 Draggable |

屬性：`index`、`group`、`isDragging`、`isDropTarget`、`disabled`、`element`、`target`。
方法：`register()`、`unregister()`、`destroy()`、`accepts(draggable)`、`refreshShape()`。

拖曳來源（sortable）的位置屬性，經 `isSortable(source)` 縮小型別後可用：

| 屬性 | 意義 |
| --- | --- |
| `index` | 目前位置（拖曳中會被外掛更新） |
| `initialIndex` | 開始拖曳時的位置 |
| `group` | 目前所在組 |
| `initialGroup` | 開始拖曳時所在組 |

`isSortableOperation(operation)`：一次把 `source` 與 `target` 都縮小為 sortable。

### 樂觀排序（OptimisticSortingPlugin）

每個 Sortable 預設啟用。拖曳經過其他項目時：

1. 直接移動 DOM 元素。
2. 更新受影響項目的 `index`（與 `group`）。
3. 把 drop target 設成拖曳來源自己（`manager.actions.setDropTarget(source.id)`）。

所以拖曳中 `source` 與 `target` 是同一個。

- 單次事件不想讓它排序：在 `dragover` 呼叫 `event.preventDefault()`。
- 完全關掉、由自己的 state 控制：

```ts
import {SortableKeyboardPlugin} from '@dnd-kit/dom/sortable';
new Sortable({id, index, element, plugins: [SortableKeyboardPlugin]}, manager); // 不含 OptimisticSortingPlugin
```

關掉之後 `source` 與 `target` 是不同元素，可以直接比對 id，但要自己在 `dragover` 處理重排，畫面才順。

### 多列表 dragend 範例

```ts
manager.monitor.addEventListener('dragend', (event) => {
  if (event.canceled) return;
  const {source} = event.operation;
  if (!isSortable(source)) return;

  const {initialIndex, index, initialGroup, group} = source;
  if (initialGroup == null || group == null) return;

  if (initialGroup === group) {
    const list = [...items[group]];
    const [removed] = list.splice(initialIndex, 1);
    list.splice(index, 0, removed);
    items = {...items, [group]: list};
  } else {
    const from = [...items[initialGroup]];
    const [removed] = from.splice(initialIndex, 1);
    const to = [...items[group]];
    to.splice(index, 0, removed);
    items = {...items, [initialGroup]: from, [group]: to};
  }
});
```

## helpers（@dnd-kit/helpers）

```ts
import {move, swap, arrayMove, arraySwap} from '@dnd-kit/helpers';
items = move(items, event);   // event 為 dragover 或 dragend
```

`items` 可以是 `id[]`、`{id}[]` 或 `Record<string, (id | {id})[]>`。取消時回傳原資料。
