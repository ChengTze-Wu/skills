# 設定：Sensors、Modifiers、碰撞偵測、外掛

## Sensors（@dnd-kit/dom）

預設：`PointerSensor`（滑鼠、觸控、筆）與 `KeyboardSensor`。

預設的 PointerSensor 行為：滑鼠在把手上立即啟動；觸控延遲 250ms、容許 5px 移動；其他指標 200ms 延遲、5px。

```ts
import {PointerSensor, KeyboardSensor, PointerActivationConstraints} from '@dnd-kit/dom';

PointerSensor.configure({
  // 陣列：任一條件成立就啟動
  activationConstraints: [
    new PointerActivationConstraints.Distance({value: 8}),
    new PointerActivationConstraints.Delay({value: 200, tolerance: 10}),
  ],
});

PointerSensor.configure({
  // 函式：依指標類型分流（'mouse' | 'touch' | 'pen'）
  activationConstraints(event, source) {
    return event.pointerType === 'touch'
      ? [new PointerActivationConstraints.Delay({value: 250, tolerance: 5})]
      : [new PointerActivationConstraints.Distance({value: 5})];
    // 回傳 undefined = 沒有限制，按下立刻開始（不會回到預設）
  },
  activatorElements: (source) => [source.element?.querySelector('[data-drag-handle]')],
  preventActivation: (event, source) =>
    event.target instanceof Element && event.target.closest('[data-no-drag]') !== null,
});

KeyboardSensor.configure({
  keyboardCodes: {
    start: ['Space', 'Enter'], cancel: ['Escape'], end: ['Space', 'Enter', 'Tab'],
    up: ['ArrowUp', 'w'], down: ['ArrowDown', 's'], left: ['ArrowLeft', 'a'], right: ['ArrowRight', 'd'],
  },
  offset: 20,           // 每次按鍵像素；Shift 乘 5。也可 {x, y}
  preventActivation: (event, source) => event.target instanceof HTMLInputElement,
});
```

- `preventActivation` 預設在 `<button>`、`<input>`、`<a>` 等互動元素上不啟動拖曳（除非是把手）。
- 按鍵值用 `KeyboardEvent.key`（`Enter`、`ArrowUp`、`w`），`Space` 有別名可用。
- 設在三層：manager 的 `sensors`（全域）、實體的 `sensors`（優先）、`Sensor.configure()`。
- 用陣列取代預設時別漏掉 `KeyboardSensor`，否則失去鍵盤無障礙。

## Modifiers

```ts
import {RestrictToHorizontalAxis, RestrictToVerticalAxis, SnapModifier} from '@dnd-kit/abstract/modifiers';
import {RestrictToWindow, RestrictToElement} from '@dnd-kit/dom/modifiers';

new DragDropManager({modifiers: (d) => [...d, RestrictToWindow]});

new Draggable({
  id, element,
  modifiers: [
    RestrictToElement.configure({element: containerEl}),   // 在 vanilla 可直接給元素；若元素之後才建立，用 () => el
    SnapModifier.configure({size: 20}),                    // 或 {x: 50, y: 25}
  ],
}, manager);
```

- 依序套用，先限制、後轉換。
- 單一元素的 modifiers 優先於全域。

自訂 modifier：

```ts
import {Modifier} from '@dnd-kit/abstract';
import type {Coordinates} from '@dnd-kit/geometry';

class SnapToGrid extends Modifier {
  apply(operation): Coordinates {
    if (this.disabled) return operation.transform;
    const {gridSize = 20} = this.options ?? {};
    const {transform} = operation;
    return {x: Math.round(transform.x / gridSize) * gridSize, y: Math.round(transform.y / gridSize) * gridSize};
  }
}
const snap = SnapToGrid.configure({gridSize: 10});
```

方法：`apply(operation)`、`enable()`、`disable()`、`isDisabled()`、`destroy()`；靜態 `configure(options)`。

## 碰撞偵測（@dnd-kit/collision）

設在每個 Droppable/Sortable 的 `collisionDetector`。

| 偵測器 | 說明 |
| --- | --- |
| `defaultCollisionDetection` | 先指標交集，沒有再用形狀交集 |
| `pointerIntersection` | 指標必須在目標內 |
| `shapeIntersection` | 重疊面積最大者 |
| `closestCenter` | 無交集時選中心最近的。卡片堆疊、格狀 |
| `closestCorners` | 四角平均距離最小 |
| `pointerDistance` | 指標到目標中心距離 |
| `directionBiased` | 只偵測拖曳方向上的目標 |

```ts
import {closestCenter} from '@dnd-kit/collision';
new Droppable({id, element, collisionDetector: closestCenter, collisionPriority: 2}, manager);
```

`CollisionPriority`（`@dnd-kit/abstract`）：`Lowest`、`Low`、`Normal`、`High`、`Highest`，也可直接給數字。巢狀容器讓內層優先。

自訂偵測器：

```ts
import type {CollisionDetector} from '@dnd-kit/abstract';
import {CollisionPriority, CollisionType} from '@dnd-kit/abstract';

const detector: CollisionDetector = ({dragOperation, droppable}) => {
  if (!droppable.shape) return null;
  return {id: droppable.id, value: 1, type: CollisionType.Collision, priority: CollisionPriority.Normal};
};
```

## 外掛（Plugins）

```ts
import {Feedback, AutoScroller, Accessibility} from '@dnd-kit/dom';
import type {DragStartEvent, DragEndEvent} from '@dnd-kit/dom';

new DragDropManager({
  plugins: (d) => [
    ...d,
    Feedback.configure({dropAnimation: {duration: 300, easing: 'ease'}}),  // 或 null 關閉
    AutoScroller.configure({acceleration: 15, threshold: {x: 0, y: 0.3}}),
    Accessibility.configure({
      announcements: {
        dragstart: ({operation: {source}}: DragStartEvent) => (source ? `開始拖曳 ${source.id}` : undefined),
        dragend: ({operation: {source, target}, canceled}: DragEndEvent) =>
          source ? (canceled ? '已取消' : `已放到 ${target?.id ?? '無'}`) : undefined,
      },
      screenReaderInstructions: {draggable: '按空白鍵拿起，方向鍵移動，再按空白鍵放下，Esc 取消。'},
    }),
  ],
});

// 關掉自動捲動
new DragDropManager({plugins: (d) => d.filter((p) => p !== AutoScroller)});
```

Feedback 的 `feedback` 選項可在單一實體上設定：`'default' | 'clone' | 'move' | 'none'`，也支援函式形式動態決定。`dropAnimation` 可為 `null`、`{duration, easing}` 或 `async ({element, translate}) => {...}`。另有 `keyboardTransition` 可調整或關閉鍵盤移動的 CSS transition。

自訂外掛請看官方 Plugins 文件（繼承 `Plugin` 類別，用 `configure()` 建立設定）。
