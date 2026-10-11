# 進階情境配方

以下範例都通過 TypeScript 型別檢查，但**沒有在瀏覽器實測過拖曳**。上線前請實際拖一次。

## 目錄

1. 格狀排序 + 放大預覽（DragOverlay）
2. 虛擬化長列表
3. 樹狀巢狀排序（拖進資料夾）
4. 看板 + 伺服器同步（React Query）
5. 為什麼這些情境要特別注意

---

## 1. 格狀排序 + 放大預覽

格狀不用特別設定，`useSortable` 會依元素位置自動處理，直接用預設碰撞偵測即可。

重點：**`useSortable` 的 `plugins` 傳陣列會取代它的預設外掛**（`SortableKeyboardPlugin`、`OptimisticSortingPlugin`）。要加 Feedback 設定時，用函式形式保留預設：

```tsx
import {useState} from 'react';
import {DragDropProvider, DragOverlay} from '@dnd-kit/react';
import {useSortable} from '@dnd-kit/react/sortable';
import {move} from '@dnd-kit/helpers';
import {Feedback} from '@dnd-kit/dom';

function SortableCard({photo, index}: {photo: Photo; index: number}) {
  const {ref, isDragging} = useSortable({
    id: photo.id,
    index,
    // feedback: 'none' = 外掛不移動原元素，視覺交給 DragOverlay
    plugins: (defaults) => [...defaults, Feedback.configure({feedback: 'none'})],
  });
  return (
    <div ref={ref} style={{opacity: isDragging ? 0.3 : 1}}>
      <Card photo={photo} />
    </div>
  );
}

export function Gallery({initial}: {initial: Photo[]}) {
  const [photos, setPhotos] = useState(initial);
  const byId = new Map(photos.map((p) => [String(p.id), p]));

  return (
    <DragDropProvider
      onDragEnd={(event) => {
        if (event.canceled) return;
        setPhotos((items) => move(items, event));
      }}
    >
      <div style={{display: 'grid', gridTemplateColumns: 'repeat(4, 1fr)', gap: 16}}>
        {photos.map((p, i) => <SortableCard key={p.id} photo={p} index={i} />)}
      </div>
      <DragOverlay dropAnimation={null}>
        {(source) => {
          const photo = byId.get(String(source.id));
          return photo ? <Card photo={photo} style={{opacity: 0.7, transform: 'scale(1.1)'}} /> : null;
        }}
      </DragOverlay>
    </DragDropProvider>
  );
}
```

- `DragOverlay` 的 `source.id` 型別是 `UniqueIdentifier`，查表時用 `String(source.id)`。
- `<img>` 要加 `draggable={false}`，避免瀏覽器原生圖片拖曳搶走手勢。
- 不想用 `feedback: 'none'`、只想要預設的浮起效果，就不需要 DragOverlay 也不需要設定 plugins。

## 2. 虛擬化長列表（@tanstack/react-virtual）

三個要點：

1. `useSortable` 的 `index` 要傳**完整陣列的位置**（`virtualItem.index`），不是畫面上第幾列。
2. **被拖的那一列要保持 mounted**。捲動時它若被虛擬化卸載，拖曳會中斷。用 `rangeExtractor` 把它補進去。
3. 列的位置用 `top`（或 `translateY` 放在外層），**不要把 `transform` 放在 `useSortable` 的那個元素上**，那是 dnd-kit 做位移用的。

```tsx
import {useCallback, useRef, useState} from 'react';
import {DragDropProvider} from '@dnd-kit/react';
import type {DragStartEvent, DragEndEvent} from '@dnd-kit/react';
import {isSortable, useSortable} from '@dnd-kit/react/sortable';
import {move} from '@dnd-kit/helpers';
import {defaultRangeExtractor, useVirtualizer} from '@tanstack/react-virtual';
import type {Range} from '@tanstack/react-virtual';

function Row({todo, index, start, size}: {todo: Todo; index: number; start: number; size: number}) {
  const {ref, handleRef, isDragging} = useSortable({id: todo.id, index});
  return (
    <div ref={ref} style={{position: 'absolute', top: start, left: 0, right: 0, height: size,
        background: isDragging ? '#eef4ff' : '#fff'}}>
      <button ref={handleRef}>⠿</button> {todo.title}
    </div>
  );
}

export function VirtualSortableList({initial}: {initial: Todo[]}) {
  const [todos, setTodos] = useState(initial);
  const parentRef = useRef<HTMLDivElement>(null);
  const activeIndex = useRef<number | null>(null);   // 用 ref，rangeExtractor 才讀得到最新值

  const rangeExtractor = useCallback((range: Range) => {
    const indexes = defaultRangeExtractor(range);
    const active = activeIndex.current;
    if (active != null && !indexes.includes(active)) {
      indexes.push(active);
      indexes.sort((a, b) => a - b);
    }
    return indexes;
  }, []);

  const virtualizer = useVirtualizer({
    count: todos.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48,
    overscan: 8,
    rangeExtractor,
  });

  return (
    <DragDropProvider
      onDragStart={(e: DragStartEvent) => {
        const {source} = e.operation;
        if (isSortable(source)) { activeIndex.current = source.initialIndex; virtualizer.measure(); }
      }}
      onDragEnd={(e: DragEndEvent) => {
        activeIndex.current = null;
        setTodos((current) => move(current, e));   // 對完整陣列操作，未 mount 的列不受影響
      }}
    >
      <div ref={parentRef} style={{height: 600, overflow: 'auto'}}>
        <div style={{height: virtualizer.getTotalSize(), position: 'relative'}}>
          {virtualizer.getVirtualItems().map((vi) => (
            <Row key={todos[vi.index].id} todo={todos[vi.index]} index={vi.index} start={vi.start} size={vi.size} />
          ))}
        </div>
      </div>
    </DragDropProvider>
  );
}
```

- `key` 用穩定的 id，不要用 `vi.key` 或 index。
- 只在 `onDragEnd` 更新資料即可；樂觀排序在拖曳中已處理畫面。
- 自訂捲動容器時，`AutoScroller` 能不能順利捲動要實際測。

## 3. 樹狀巢狀排序（拖進資料夾）

排序時 `source` 與 `target` 是同一個，所以**不能用 target 判斷「放進哪個資料夾」**。常見解法：

1. 把「可見的節點」攤平成**一條**扁平清單，全部用同一個 `group` 的 `useSortable`。
2. 拖曳結束時，用 `source.index`（它落在哪一列）加上**水平位移**（`position.current.x - position.initial.x`）換算縮排層級：往右拖＝放進上一個資料夾，往左拖＝移出一層。
3. 資料本身保持巢狀（`children`），用函式把節點搬到新的父層。
4. 拖曳資料夾時，暫時隱藏它的子孫列，避免把資料夾拖進自己裡面。

```tsx
const [tree, setTree] = useState<TreeNode[]>(initial);
const [activeId, setActiveId] = useState<string | null>(null);
const snapshot = useRef(tree);
const rows = flatten(tree, activeId);          // activeId 的子孫被隱藏

<DragDropProvider
  onDragStart={(e) => { snapshot.current = tree; setActiveId(String(e.operation.source?.id ?? '')); }}
  onDragEnd={(e) => {
    setActiveId(null);
    if (e.canceled) { setTree(snapshot.current); return; }
    const {source, position} = e.operation;
    if (!isSortable(source)) return;
    const id = String(source.id);
    const without = rows.filter((r) => r.node.id !== id);
    const dx = position.current.x - position.initial.x;
    const {parentId, afterId} = projectDrop(without, source.index, dx, INDENT);
    setTree((t) => moveNode(t, id, parentId, afterId));
  }}
>
  <ul>{rows.map((row, i) => <TreeRow key={row.node.id} row={row} index={i} />)}</ul>
</DragDropProvider>

// TreeRow：useSortable({id, index, group: 'tree', type: 'row', accept: 'row'})，縮排用 paddingLeft: depth * INDENT
```

`flatten`、`moveNode`、`projectDrop` 是你自己的資料運算（不是 dnd-kit 的 API）：

- `flatten(tree, hiddenSubtreeOf)`：依 `expanded` 攤平成 `{node, depth, parentId}[]`。
- `projectDrop(rowsWithoutDragged, insertAt, offsetX, indent)`：前一列是資料夾時，最大層級是它的層級加一；層級被限制在「下一列」與「前一列」之間；回傳 `{parentId, afterId}`。
- `moveNode(tree, id, parentId, afterId)`：從舊位置移除再插入新父層，並展開目標資料夾。

拖曳中隱藏子孫列會改變渲染的列數，是否干擾進行中的拖曳要實測。

## 4. 看板 + 伺服器同步（React Query）

基本結構見 `guides.md` 第 2、3 節。補充三個容易漏的點：

1. **mutation 進行中也要暫停同步**，不只拖曳中。否則剛放開、請求還沒完成時 refetch 回來，會把新順序蓋回舊的。
2. `onDragEnd` 裡呼叫 `mutate` 時，`setState` 的更新是非同步的，要從 `ref` 讀最新的欄位與順序（`latest.current`），不要讀閉包裡的 state。
3. 樂觀更新用 `onMutate` 先 `cancelQueries` 再 `setQueryData`，`onSettled` 才 `invalidateQueries`。

```tsx
const dragging = useRef(false);
const latest = useRef({columns, columnOrder});
latest.current = {columns, columnOrder};

const mutation = useMutation({
  mutationFn: saveLayout,
  onMutate: async (layout) => {
    await queryClient.cancelQueries({queryKey: ['board']});
    queryClient.setQueryData(['board'], (old) => (old ? {...old, ...layout} : old));
  },
  onSettled: () => queryClient.invalidateQueries({queryKey: ['board']}),
});

useEffect(() => {
  if (!data || dragging.current || mutation.isPending) return;   // 兩個條件都要擋
  setColumns(data.columns);
  setColumnOrder(data.columnOrder);
}, [data, mutation.isPending]);
```

卡片資料（標題等）放在另一個以 id 查詢的 map，欄位 state 只存 id 陣列，這樣 `move` 才能直接用。

## 5. 為什麼這些情境要特別注意

| 現象 | 原因 |
| --- | --- |
| 排序時 `source.id === target.id` | 預設的樂觀排序把 drop target 設成來源自己 |
| 單一元素設了 `plugins` 後排序變怪 | `useSortable` 的 `plugins` 傳陣列會取代預設外掛，要用 `(defaults) => [...defaults, ...]` |
| 虛擬化列拖著拖著消失 | 被拖的列捲出範圍被卸載，要用 `rangeExtractor` 保留 |
| 放開後順序跳回去 | 拖曳或 mutation 進行中，伺服器資料蓋掉本地 state |
