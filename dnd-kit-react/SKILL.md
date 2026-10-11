---
name: dnd-kit-react
description: Build drag and drop, sortable lists and kanban boards in React + TypeScript with the new dnd-kit (@dnd-kit/react, v0.5+). Use for DragDropProvider, useDraggable, useDroppable, useSortable, DragOverlay, move helper, sensors, modifiers, collision detection, or migrating from legacy @dnd-kit/core. 用 dnd-kit 在 React 做拖放、排序、看板。
---

# dnd-kit（React + TypeScript）

依據官方文件 dndkit.com/react（v0.5.0，2026-06）。

## 先看這個：不要用舊 API

新版套件是 `@dnd-kit/react`。下列是**舊版** `@dnd-kit/core` 的東西，新專案不要寫：

| 舊（不要用） | 新 |
| --- | --- |
| `DndContext` | `DragDropProvider` |
| `SortableContext`、排序策略 | 不需要，`useSortable` 自動處理 |
| `useSortable` from `@dnd-kit/sortable` | `useSortable` from `@dnd-kit/react/sortable` |
| `setNodeRef` / `listeners` / `attributes` | 只要 `ref` |
| `transform` + `CSS.Transform.toString` | 不需要，外掛自動處理 |
| `active` / `over` | `event.operation.source` / `.target` |
| `onDragCancel` | 在 `onDragEnd` 檢查 `event.canceled` |
| `arrayMove` | `move`（`@dnd-kit/helpers`） |
| `MouseSensor` / `TouchSensor` | `PointerSensor` |
| `collisionDetection` 放在 context | `collisionDetector` 放在 droppable / sortable |

使用者若提到舊 API，先問或確認專案實際裝的是哪一版，再決定要不要遷移。完整對照見 `references/migration.md`。

## 安裝

```bash
npm install @dnd-kit/react
npm install @dnd-kit/helpers   # 做排序才需要（move 工具）
```

`@dnd-kit/dom`、`@dnd-kit/abstract` 會自動裝。peer：React 18 或 19。

## 匯入位置

```ts
import {DragDropProvider, DragOverlay, useDraggable, useDroppable,
  useDragOperation, useDragDropMonitor, useDragDropManager} from '@dnd-kit/react';
import type {DragEndEvent, DragOverEvent, DragStartEvent} from '@dnd-kit/react';
import {useSortable, isSortable, isSortableOperation} from '@dnd-kit/react/sortable';
import {move} from '@dnd-kit/helpers';
import {PointerSensor, KeyboardSensor, PointerActivationConstraints, Feedback,
  AutoScroller, Accessibility} from '@dnd-kit/dom';
import {RestrictToWindow, RestrictToElement} from '@dnd-kit/dom/modifiers';
import {RestrictToVerticalAxis, RestrictToHorizontalAxis, SnapModifier} from '@dnd-kit/abstract/modifiers';
import {CollisionPriority} from '@dnd-kit/abstract';
import {closestCenter, closestCorners, pointerIntersection} from '@dnd-kit/collision';
```

## 基本配方

### 拖曳 + 放置

```tsx
import {useState} from 'react';
import {DragDropProvider, useDraggable, useDroppable} from '@dnd-kit/react';

function Draggable() {
  const {ref, isDragging} = useDraggable({id: 'draggable'});
  return <button ref={ref} data-dragging={isDragging}>Drag me</button>;
}

function Droppable({id, children}: {id: string; children?: React.ReactNode}) {
  const {ref, isDropTarget} = useDroppable({id});
  return <div ref={ref} data-over={isDropTarget}>{children}</div>;
}

export function App() {
  const [dropped, setDropped] = useState<string | undefined>();
  return (
    <DragDropProvider
      onDragEnd={(event) => {
        if (event.canceled) return;
        setDropped(event.operation.target?.id as string | undefined);
      }}
    >
      {dropped == null && <Draggable />}
      <Droppable id="zone">{dropped === 'zone' && <Draggable />}</Droppable>
    </DragDropProvider>
  );
}
```

重點：`id` 在同一個 `DragDropProvider` 內必須唯一；`ref` 一定要綁到實際 DOM 元素。

### 單一排序列表

```tsx
import {useState} from 'react';
import {DragDropProvider} from '@dnd-kit/react';
import {useSortable} from '@dnd-kit/react/sortable';
import {move} from '@dnd-kit/helpers';

function Item({id, index}: {id: string; index: number}) {
  const {ref, isDragging} = useSortable({id, index});
  return <li ref={ref} data-dragging={isDragging}>{id}</li>;
}

export function List() {
  const [items, setItems] = useState(['A', 'B', 'C', 'D']);
  return (
    <DragDropProvider onDragEnd={(event) => setItems((items) => move(items, event))}>
      <ul>{items.map((id, index) => <Item key={id} id={id} index={index} />)}</ul>
    </DragDropProvider>
  );
}
```

`move(items, event)` 會自己處理取消（canceled）的情況，回傳新陣列。`key` 要用穩定的 id，不要用 index。

### 多個列表（看板）

每個項目設 `group`、`type`、`accept`；每欄用 `useDroppable`，這樣空欄也能放。欄位要用 `CollisionPriority.Low`，讓項目優先被偵測到。

```tsx
// Item.tsx
export function Item({id, index, column}: {id: string; index: number; column: string}) {
  const {ref, isDragging} = useSortable({id, index, group: column, type: 'item', accept: 'item'});
  return <button ref={ref} data-dragging={isDragging}>{id}</button>;
}

// Column.tsx
import {useDroppable} from '@dnd-kit/react';
import {CollisionPriority} from '@dnd-kit/abstract';

export function Column({id, children}: {id: string; children: React.ReactNode}) {
  const {ref, isDropTarget} = useDroppable({
    id, type: 'column', accept: 'item', collisionPriority: CollisionPriority.Low,
  });
  return <div ref={ref} data-over={isDropTarget}>{children}</div>;
}

// App.tsx
const [items, setItems] = useState<Record<string, string[]>>({A: ['A0', 'A1'], B: ['B0'], C: []});
const previous = useRef(items);

<DragDropProvider
  onDragStart={() => { previous.current = items; }}
  onDragOver={(event) => setItems((items) => move(items, event))}
  onDragEnd={(event) => { if (event.canceled) setItems(previous.current); }}
>
  {Object.entries(items).map(([column, ids]) => (
    <Column key={column} id={column}>
      {ids.map((id, index) => <Item key={id} id={id} index={index} column={column} />)}
    </Column>
  ))}
</DragDropProvider>
```

在 `onDragOver` 更新 state 時，一定要在 `onDragStart` 存快照，取消時還原。欄位本身也要能排序的話，見 `references/guides.md`。

### 拖曳時顯示不同的畫面（DragOverlay）

```tsx
<DragDropProvider>
  <Draggable id="foo" />
  <Draggable id="bar" />
  <DragOverlay>{(source) => <div>Dragging {String(source.id)}</div>}</DragOverlay>
</DragDropProvider>
```

每個 `DragDropProvider` 只放**一個** `DragOverlay`。要搭配 `Feedback.configure({feedback: 'none'})` 隱藏原本元素的拖曳效果時，見 `references/guides.md`。overlay 裡面的 hook 不會生效。

### 拖曳把手

```tsx
const {ref, handleRef} = useDraggable({id});
return <div ref={ref}>內容<button ref={handleRef}>⠿</button></div>;
```

`useSortable` 也有 `handleRef`。接上後，只有把手能開始拖。

## 容易出錯的地方

1. **排序時 `source` 和 `target` 是同一個元素**（預設的 optimistic sorting）。不要比對 `source.id === target.id` 來判斷移動了什麼。要用 `isSortable(source)` 後讀 `initialIndex`、`index`、`initialGroup`、`group`。
2. 取消拖曳（按 Esc）要檢查 `event.canceled`。
3. `plugins`、`sensors`、`modifiers` 傳**陣列會取代預設**，傳 `(defaults) => [...defaults, X]` 才是擴充。取代預設時漏掉 `KeyboardSensor` 會失去鍵盤無障礙。
4. `RestrictToElement` 的 `element` 要傳函式 `() => ref.current`，不要傳 `ref.current`（第一次 render 是 `null`，會靜默失效）。
5. 搭配 React Query / SWR：refetch 時可能出現重複項目。拖曳中不要同步伺服器資料到本地 state，見 `references/guides.md`。
6. 放開後有「跳兩次」：用 `move` 處理，或 `Feedback.configure({dropAnimation: null})`。
7. 編輯器若提示型別找不到：事件型別從 `@dnd-kit/react` 匯入（`DragEndEvent` 等）；排序型別守衛從 `@dnd-kit/react/sortable` 匯入。
8. Next.js / RSC：用到 hook 的檔案加 `'use client'`。

## 常用 API 速查

`useDraggable({id, type?, handle?, disabled?, modifiers?, sensors?, plugins?, data?, alignment?})` → `{ref, handleRef, isDragging, isDragSource, isDropping, draggable}`

`useDroppable({id, accept?, type?, collisionDetector?, collisionPriority?, disabled?, data?})` → `{ref, isDropTarget, droppable}`

`useSortable({id, index, group?, type?, accept?, transition?, disabled?, handle?, collisionDetector?, ...})` → `{ref, handleRef, targetRef, sourceRef, isDragging, isDropTarget, isDragSource, isDropping}`

`DragDropProvider` 事件：`onBeforeDragStart`（可 `preventDefault`）、`onDragStart`、`onDragMove`、`onDragOver`、`onDragEnd`、`onCollision`。另有 `manager`、`sensors`、`plugins`、`modifiers`。

`useDragOperation()` → `{source, target}`（只在來源或目標改變時重新 render）。

更多細節：

- `references/api.md`：hooks、元件、事件物件完整欄位
- `references/guides.md`：sensors、modifiers、碰撞偵測、回饋、外掛、狀態管理、多列表進階
- `references/migration.md`：從 `@dnd-kit/core` 遷移的對照
