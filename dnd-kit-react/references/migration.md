# 從 @dnd-kit/core 遷移到 @dnd-kit/react

## 套件

```diff
- "@dnd-kit/core", "@dnd-kit/sortable", "@dnd-kit/utilities"
+ "@dnd-kit/react", "@dnd-kit/helpers"
```

## 對照表

| 舊 | 新 |
| --- | --- |
| `DndContext` | `DragDropProvider` |
| `active` / `over` | `event.operation.source` / `.target` |
| `active.id` | `event.operation.source.id` |
| `onDragCancel` | `onDragEnd` 內檢查 `event.canceled` |
| `useSensor` / `useSensors` | 預設內建；用 Provider 的 `sensors` prop 調整 |
| `MouseSensor`、`TouchSensor`、`PointerSensor` | `PointerSensor`（`@dnd-kit/dom`）。要區分輸入類型就在 `activationConstraints` 函式裡看 `event.pointerType` |
| `KeyboardSensor` | `KeyboardSensor`（`@dnd-kit/dom`） |
| `collisionDetection`（在 `DndContext`） | `collisionDetector`（在 `useDroppable` / `useSortable`） |
| `pointerWithin` | `pointerIntersection`（`@dnd-kit/collision`） |
| `closestCenter`、`closestCorners` | 同名，來自 `@dnd-kit/collision` |
| `restrictToParentElement` | `RestrictToElement`（`@dnd-kit/dom/modifiers`） |
| `restrictToWindowEdges` | `RestrictToWindow`（`@dnd-kit/dom/modifiers`） |
| `restrictToVerticalAxis` / `restrictToHorizontalAxis` | `RestrictToVerticalAxis` / `RestrictToHorizontalAxis`（`@dnd-kit/abstract/modifiers`） |
| `createSnapModifier` | `SnapModifier`（`@dnd-kit/abstract/modifiers`） |
| `SortableContext` | 不需要 |
| `verticalListSortingStrategy` 等 | 不需要 |
| `arrayMove` | `move`（`@dnd-kit/helpers`） |
| `useSortable` from `@dnd-kit/sortable` | `useSortable` from `@dnd-kit/react/sortable` |
| `DragOverlay` + 自己存 `activeId` | `<DragOverlay>{(source) => ...}</DragOverlay>`，不用自己存 state |
| `isOver` | `isDropTarget` |

## 前後對照

### Provider

```tsx
// 舊
<DndContext
  onDragEnd={({active, over}) => { if (over) console.log(active.id, over.id); }}
  onDragCancel={({active}) => console.log('cancel', active.id)}
/>

// 新
<DragDropProvider
  onDragEnd={(event) => {
    const {source, target} = event.operation;
    if (event.canceled) return;           // 取代 onDragCancel
    if (target) console.log(source.id, target.id);
  }}
/>
```

### Draggable

```tsx
// 舊
const {attributes, listeners, setNodeRef, transform} = useDraggable({id});
<div ref={setNodeRef} {...listeners} {...attributes}
     style={{transform: CSS.Transform.toString(transform)}} />

// 新
const {ref, isDragging} = useDraggable({id});
<div ref={ref} className={isDragging ? 'dragging' : ''} />
```

### Droppable

```tsx
// 舊
const {setNodeRef, isOver} = useDroppable({id});
// 新
const {ref, isDropTarget} = useDroppable({id});
```

### Sortable

```tsx
// 舊
<SortableContext items={items} strategy={verticalListSortingStrategy}>
  {items.map((id) => <Item key={id} id={id} />)}
</SortableContext>
// Item 內：useSortable({id}) + transform + transition + listeners + attributes

// 新（不需 SortableContext，但要傳 index）
{items.map((id, index) => <Item key={id} id={id} index={index} />)}
// Item 內：const {ref} = useSortable({id, index});
// Provider：onDragEnd={(event) => setItems((items) => move(items, event))}
```

### DragOverlay

```tsx
// 舊
{activeId ? <Item id={activeId} /> : null}
// 新
<DragOverlay>{(source) => <Item id={source.id} />}</DragOverlay>
```

## 遷移步驟建議

1. 換套件。
2. 全部 `DndContext` → `DragDropProvider`，改事件欄位。
3. 每個 `useDraggable`/`useDroppable` 改成只用 `ref`，移除 `listeners`、`attributes`、`transform` 與 `CSS`。
4. 排序：移除 `SortableContext` 與策略，`useSortable` 加 `index`，用 `move` 取代 `arrayMove`。
5. sensors、modifiers、collision 依上表改寫。
6. 注意排序時 `source` 與 `target` 在樂觀排序下是同一個，不能再用「兩者 id 不同」判斷有沒有移動。
