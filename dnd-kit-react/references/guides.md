# 進階指南（React）

## 目錄

1. 手動管理排序 state
2. 與 React Query / SWR 一起用
3. 欄位也能排序（看板）
4. Sensors
5. Modifiers
6. 碰撞偵測
7. 回饋（Feedback）與 DragOverlay
8. 其他外掛

---

## 1. 手動管理排序 state

預設的樂觀排序（OptimisticSortingPlugin）會在拖曳中直接調整 DOM，所以只需處理 `onDragEnd`。

單一列表：

```tsx
import {DragDropProvider} from '@dnd-kit/react';
import {isSortable} from '@dnd-kit/react/sortable';

<DragDropProvider
  onDragEnd={(event) => {
    if (event.canceled) return;
    const {source} = event.operation;
    if (isSortable(source)) {
      const {initialIndex, index} = source;
      if (initialIndex !== index) {
        setItems((items) => {
          const next = [...items];
          const [removed] = next.splice(initialIndex, 1);
          next.splice(index, 0, removed);
          return next;
        });
      }
    }
  }}
>
```

多列表用 `initialGroup` / `group`：

```tsx
const snapshot = useRef(structuredClone(items));

<DragDropProvider
  onDragStart={() => { snapshot.current = structuredClone(items); }}
  onDragEnd={(event) => {
    if (event.canceled) { setItems(snapshot.current); return; }
    const {source} = event.operation;
    if (!isSortable(source)) return;
    const {initialIndex, index, initialGroup, group} = source;
    if (initialGroup == null || group == null) return;

    setItems((items) => {
      if (initialGroup === group) {
        const list = [...items[group]];
        const [removed] = list.splice(initialIndex, 1);
        list.splice(index, 0, removed);
        return {...items, [group]: list};
      }
      const from = [...items[initialGroup]];
      const [removed] = from.splice(initialIndex, 1);
      const to = [...items[group]];
      to.splice(index, 0, removed);
      return {...items, [initialGroup]: from, [group]: to};
    });
  }}
>
```

在 `onDragOver` 更新 state 時，要在 `onDragStart` 存快照，取消時還原。

想擋掉某次移動（例如不准拖進某個 group）：在 `onDragOver` 呼叫 `event.preventDefault()`，樂觀排序就會略過這次。

| | `move` helper | 手動 |
| --- | --- | --- |
| 設定 | 一行 | 較多，但完全可控 |
| 更新時機 | `onDragOver` 或 `onDragEnd` | 通常只在 `onDragEnd` |
| 資料結構 | 陣列、陣列的 Record | 任意 |

## 2. 與 React Query / SWR 一起用

問題：拖曳中 refetch 會讓資料取代掉已被樂觀移動的 DOM，出現重複項目。
做法：畫面一律用本地 state 渲染；只在沒有拖曳時才從伺服器資料同步。

```tsx
const {data: fetched} = useQuery({queryKey: ['items'], queryFn: fetchItems});
const [items, setItems] = useState(fetched ?? []);
const isDragging = useRef(false);

useEffect(() => {
  if (fetched && !isDragging.current) setItems(fetched);
}, [fetched]);

<DragDropProvider
  onDragStart={() => { isDragging.current = true; }}
  onDragEnd={(event) => {
    isDragging.current = false;
    if (event.canceled) { setItems(fetched ?? []); return; }
    setItems((items) => move(items, event));
    // 再把新順序送到伺服器
  }}
>
```

## 3. 欄位也能排序（看板）

欄位改用 `useSortable`，`type: 'column'`，項目維持 `type: 'item'`：

```tsx
// Column.tsx
const {ref} = useSortable({
  id, index, type: 'column',
  collisionPriority: CollisionPriority.Low,
  accept: ['item', 'column'],
});

// App.tsx
const [columnOrder, setColumnOrder] = useState(() => Object.keys(items));

<DragDropProvider
  onDragStart={() => { previousItems.current = items; }}
  onDragOver={(event) => {
    if (event.operation.source?.type === 'column') return;
    setItems((items) => move(items, event));
  }}
  onDragEnd={(event) => {
    const {source} = event.operation;
    if (event.canceled) {
      if (source?.type === 'item') setItems(previousItems.current);
      return;
    }
    if (source?.type === 'column') setColumnOrder((cols) => move(cols, event));
  }}
>
  {columnOrder.map((column, columnIndex) => (
    <Column key={column} id={column} index={columnIndex}>
      {items[column].map((id, index) => <Item key={id} id={id} index={index} column={column} />)}
    </Column>
  ))}
</DragDropProvider>
```

欄位順序在 `onDragEnd` 才更新，拖曳中交給 dnd-kit 樂觀處理，可少很多 re-render。

## 4. Sensors

預設已含 `PointerSensor`（滑鼠、觸控、筆）與 `KeyboardSensor`。只有要改啟動方式時才設定。

三個層級：Provider 的 `sensors` prop（全域）、`useDraggable`/`useSortable` 的 `sensors`（單一元素，優先）、`Sensor.configure({...})`（建立設定好的 sensor）。

```tsx
import {PointerSensor, PointerActivationConstraints} from '@dnd-kit/dom';

<DragDropProvider
  sensors={(defaults) => [
    ...defaults.filter((s) => s !== PointerSensor),
    PointerSensor.configure({
      activationConstraints(event, source) {
        if (event.pointerType === 'touch') {
          return [new PointerActivationConstraints.Delay({value: 250, tolerance: 5})];
        }
        return [new PointerActivationConstraints.Distance({value: 5})];
      },
    }),
  ]}
>
```

- 可傳陣列（多個條件，任一成立就啟動）或函式（依 `event.pointerType` 分流）。
- 函式回傳 `undefined` 代表**沒有限制**，按下去立刻開始，不會回到預設。
- `preventActivation(event, source)` 回傳 `true` 就略過這次按下。預設在 `<button>`、`<input>`、`<a>` 等互動元素上不啟動拖曳。
- `activatorElements: (source) => Element[]`：啟動元素在拖曳元素之外時才需要。一般情況直接用 `handleRef`。

KeyboardSensor：

```tsx
KeyboardSensor.configure({
  keyboardCodes: {
    start: ['Space', 'Enter'], cancel: ['Escape'], end: ['Space', 'Enter', 'Tab'],
    up: ['ArrowUp', 'w'], down: ['ArrowDown', 's'], left: ['ArrowLeft', 'a'], right: ['ArrowRight', 'd'],
  },
  offset: 20,                 // 每次按鍵移動的像素；按 Shift 變 5 倍。也可 {x, y}
  preventActivation: (event, source) => event.target instanceof HTMLInputElement,
});
```

按鍵值用 `KeyboardEvent.key`（如 `Enter`、`ArrowUp`、`w`），不是 `code`。

## 5. Modifiers

```tsx
// 限制在容器內：element 必須是函式
RestrictToElement.configure({element: () => containerRef.current})

// 限制在視窗內
RestrictToWindow

// 限制單一軸
RestrictToVerticalAxis      // @dnd-kit/abstract/modifiers
RestrictToHorizontalAxis

// 對齊格線
SnapModifier.configure({size: 20})        // 或 {x: 50, y: 25}
```

- 依序套用，先限制、後轉換。
- 全域：`<DragDropProvider modifiers={[RestrictToWindow]}>`；單一元素：hook 的 `modifiers`（優先於全域）。
- 全域擴充預設用 `modifiers={(defaults) => [...defaults, RestrictToWindow]}`。

## 6. 碰撞偵測

`collisionDetector` 設在**每個 droppable / sortable**，不是全域。從 `@dnd-kit/collision` 匯入（已是 `@dnd-kit/react` 的相依套件）。

| 偵測器 | 適用 |
| --- | --- |
| `defaultCollisionDetection`（預設） | 先 `pointerIntersection`，指標不在任何目標內才用 `shapeIntersection` |
| `pointerIntersection` | 指標必須在目標內，精準 |
| `shapeIntersection` | 重疊面積最大者勝 |
| `closestCenter` | 沒交集時退而選中心最近者。卡片堆疊、格狀 |
| `closestCorners` | 四角平均距離最小。直式列表邊緣較寬鬆 |
| `pointerDistance` | 以指標到各目標中心的距離 |
| `directionBiased` | 只偵測拖曳方向上的目標，避免在兩目標交界抖動 |

```tsx
useSortable({id, index, collisionDetector: closestCorners});
useDroppable({id, collisionPriority: CollisionPriority.Low}); // 重疊時卡片優先於欄位
```

自訂偵測器：

```ts
import type {CollisionDetector} from '@dnd-kit/abstract';
import {CollisionPriority, CollisionType} from '@dnd-kit/abstract';

const myDetector: CollisionDetector = ({dragOperation, droppable}) => {
  if (!droppable.shape) return null;
  return {id: droppable.id, value: 1, type: CollisionType.Collision, priority: CollisionPriority.Normal};
};
```

`value` 越高越優先；回傳 `null` 代表沒碰撞。

## 7. 回饋（Feedback）與 DragOverlay

Feedback 外掛預設啟用：把被拖元素升到瀏覽器 top layer，並處理放開動畫。

全域設定（用函式形式，才不會丟掉其他預設外掛）：

```tsx
<DragDropProvider plugins={(defaults) => [...defaults, Feedback.configure({dropAnimation: null})]}>
```

單一元素：

```tsx
useDraggable({id, plugins: [Feedback.configure({feedback: 'clone'})]});
```

`feedback` 模式：

| 值 | 行為 |
| --- | --- |
| `'default'` | 升到 top layer，隨指標移動 |
| `'clone'` | 原位留一份複本，原件移動 |
| `'move'` | 移動，不升 top layer、不留佔位 |
| `'none'` | 外掛不做任何視覺回饋；搭配自訂 `DragOverlay` 用 |

放開動畫：`dropAnimation: null` 關閉；`{duration: 300, easing: 'ease'}`；或 `async ({element, translate}) => { await element.animate(...).finished; }`。

放開時「跳兩次」：狀態更新和放開動畫打架。關掉 `dropAnimation`，或在 `onDragEnd` 用 `move`（會對齊樂觀位置）。

`DragOverlay` 也有自己的 `dropAnimation` prop（`null`、`{duration, easing}` 或函式）。

## 8. 其他外掛

預設外掛：`Accessibility`、`AutoScroller`、`Cursor`、`Feedback`、`PreventSelection`、`ScrollListener`、`Scroller`。

```tsx
// 自動捲動
AutoScroller.configure({acceleration: 15, threshold: {x: 0, y: 0.3}}) // 預設 25 與 {x: .2, y: .2}；某軸設 0 即停用

// 關掉自動捲動
plugins={(defaults) => defaults.filter((p) => p !== AutoScroller)}

// 無障礙：自訂朗讀文字（參數需標註事件型別，否則 strict 模式會報 implicit any）
// import type {DragStartEvent, DragEndEvent} from '@dnd-kit/react';
Accessibility.configure({
  announcements: {
    dragstart: ({operation: {source}}: DragStartEvent) => (source ? `開始拖曳 ${source.id}` : undefined),
    dragend: ({operation: {source, target}, canceled}: DragEndEvent) =>
      source ? (canceled ? '已取消' : `已放到 ${target?.id ?? '無'}`) : undefined,
  },
  screenReaderInstructions: {draggable: '按空白鍵拿起，方向鍵移動，再按空白鍵放下，Esc 取消。'},
  debounce: 500, // 只影響 dragover、dragmove
})
```

自訂的朗讀文字回傳 `undefined` 代表不朗讀。預設說明文字是英文，台灣站台建議自訂成中文。
