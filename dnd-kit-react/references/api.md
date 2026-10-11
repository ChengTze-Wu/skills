# API 參考（@dnd-kit/react v0.5）

## useDraggable

```ts
const {ref, handleRef, isDragSource, isDragging, isDropping, draggable} = useDraggable(input);
```

| 選項 | 型別 | 說明 |
| --- | --- | --- |
| `id` | `string \| number` | 必填，Provider 內唯一 |
| `type` | `string \| number \| symbol` | 只能放到 `accept` 此 type 的 droppable |
| `element` | `Element \| Ref` | 已有元素參照時可代替 `ref` |
| `handle` | `Element \| Ref` | 已有把手參照時可代替 `handleRef` |
| `disabled` | `boolean` | 停用拖曳 |
| `modifiers` | `Modifier[]` | 優先於全域 modifiers |
| `sensors` | `Sensors[]` | 優先於全域 sensors |
| `plugins` | `PluginDescriptor[]` | 例：`Feedback.configure({feedback: 'clone'})` |
| `alignment` | `{x: 'start'\|'center'\|'end'; y: ...}` | Feedback 外掛算位置用 |
| `data` | `Record<string, any>` | 在事件、modifier 等地方取用 |

`isDropping`：放開後的動畫期間為 true，可用來調整樣式。
`sensors` 傳行內陣列在 v0.5 已修正不會每次 render 重設。

## useDroppable

```ts
const {ref, isDropTarget, droppable} = useDroppable(input);
```

| 選項 | 說明 |
| --- | --- |
| `id` | 必填 |
| `element` | 代替 `ref` |
| `accept` | `Type \| Type[] \| (source: Draggable) => boolean`。省略則全收 |
| `type` | 分類用。`accept` 只比對**拖曳者**的 `type`，不看 droppable 自己的 `type` |
| `collisionDetector` | 見碰撞偵測 |
| `collisionPriority` | 數字，越大越優先。可用 `CollisionPriority.Lowest/Low/Normal/High/Highest` |
| `disabled` | 停用放置 |
| `data` | 附加資料 |

## useSortable

必填 `id`、`index`。也接受 useDraggable 與 useDroppable 的選項，另外有：

| 選項 | 說明 |
| --- | --- |
| `group` | 同 group 的項目互相排序（多列表） |
| `transition` | `{duration?, easing?, idle?}` 或 `null`。預設 250ms；`idle` 控制沒在拖曳時 index 變動是否做動畫 |
| `target` | 另指定當作放置目標的元素（代替 `targetRef`） |
| `disabled` | `boolean` 或 `{draggable?: boolean; droppable?: boolean}`。只停用拖曳或只停用放置 |

回傳：`ref`（同時當來源與目標）、`sourceRef`、`targetRef`、`handleRef`、`isDragging`、`isDragSource`、`isDropTarget`、`isDropping`。

型別守衛（從 `@dnd-kit/react/sortable` 匯入）：

```ts
isSortable(source)            // 之後可讀 index、initialIndex、group、initialGroup
isSortableOperation(operation) // source 與 target 都縮小為 sortable
```

## DragDropProvider

Props：`children`、`manager?`、`sensors?`、`plugins?`、`modifiers?`、以及事件：

```ts
onBeforeDragStart(event, manager)  // event.preventDefault() 可取消
onDragStart(event, manager)
onDragMove(event, manager)
onDragOver(event, manager)         // event.preventDefault() 可擋掉外掛的預設行為（如樂觀排序）
onDragEnd(event, manager)
onCollision(event, manager)        // event.preventDefault() 可擋掉自動選目標
```

事件物件（以 `onDragEnd` 為例）：

```ts
event.operation.source    // 被拖的 Draggable，可能為 null
event.operation.target    // 放置目標 Droppable，可能為 null
event.operation.position  // {current: {x, y}, initial: {x, y}}
event.operation.status
event.canceled            // 被取消（例如按 Esc）
event.nativeEvent
```

多個 Provider 彼此獨立，不同 Provider 內的項目不能互放。

## DragOverlay

| Prop | 說明 |
| --- | --- |
| `children` | 節點，或 `(source) => ReactNode` |
| `tag` | 外層元素，預設 `'div'` |
| `disabled` | `boolean` 或 `(source) => boolean` |
| `dropAnimation` | `undefined` 預設 250ms；`null` 關閉；`{duration, easing}`；或 `async ({element, feedbackElement, translate}) => {}` |
| `className`、`style` | 外層元素 |

每個 Provider 只放一個。只在拖曳進行中才會 render children。

## useDragOperation / useDragDropMonitor / useDragDropManager

```ts
const {source, target} = useDragOperation();   // 反應式；只在 source 或 target 變才重 render

useDragDropMonitor({onBeforeDragStart, onDragStart, onDragMove, onDragOver, onDragEnd, onCollision});
// 必須在 DragDropProvider 內的元件使用

const manager = useDragDropManager();           // 型別 DragDropManager | null，要做 null 檢查
manager.monitor.addEventListener('dragstart', handler); // 回傳 cleanup 函式
```

事件一覽：

| 事件 | 可 preventDefault | 資料 |
| --- | --- | --- |
| `beforeDragStart` | 是 | `operation` |
| `dragStart` | 否 | `operation`, `nativeEvent` |
| `dragMove` | 是 | `operation`, `to`, `by`, `nativeEvent` |
| `dragOver` | 是 | `operation` |
| `collision` | 是 | `collisions` |
| `dragEnd` | 否 | `operation`, `canceled`, `nativeEvent` |

只需要「現在拖什麼、在誰上面」用 `useDragOperation`。需要指標位置等逐次事件才用 `useDragDropMonitor`。

## move / swap（@dnd-kit/helpers）

```ts
move(items, event)   // 把拖曳的項目移到新位置，回傳新資料
swap(items, event)   // 兩者互換位置
arrayMove(array, from, to)
arraySwap(array, from, to)
```

- `items` 可以是 `id[]`、`{id}[]`，或 `Record<string, (id | {id})[]>`（多列表）。
- 以 `item === id` 或 `item.id === id` 比對。ID 是算出來的、對不上時，會退而使用 sortable 的 `index`/`initialIndex`。
- 取消拖曳時回傳原資料。可在 `onDragOver` 或 `onDragEnd` 使用。
- 資料結構太特殊（不是陣列或陣列的 Record）就改手動管理，見 `guides.md`。
