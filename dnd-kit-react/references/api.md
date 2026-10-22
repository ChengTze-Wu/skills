# @dnd-kit/react API reference (v0.5.x)

Condensed from https://dndkit.com (React section + core concepts). Read the section you need.

## Contents
- DragDropProvider
- useDraggable
- useDroppable
- useSortable
- DragOverlay
- useDragOperation / useDragDropMonitor / useDragDropManager
- Sensors
- Modifiers
- Collision detection
- Plugins (Feedback, Accessibility, AutoScroller, Debug)
- Sortable internals (optimistic sorting, type guards, helpers)

---

## DragDropProvider

```tsx
<DragDropProvider
  manager?            // custom DragDropManager instance
  sensors?            // Sensor[] | (defaults) => Sensor[]
  plugins?            // Plugin[] | (defaults) => Plugin[]
  modifiers?          // Modifier[] | (defaults) => Modifier[]
  onBeforeDragStart?  // (event, manager) => void   event.preventDefault() cancels
  onDragStart?
  onDragMove?         // preventable
  onDragOver?         // preventable: blocks plugin default (e.g. optimistic sorting) for this event
  onCollision?        // preventable: stops automatic target selection
  onDragEnd?          // event.canceled
>
```
Array → replaces defaults. Function → receives defaults to extend. Multiple providers on one page are independent.

## useDraggable(input) — `@dnd-kit/react`

Input:
- `id` (required, string | number) — unique within provider
- `type` — matched against droppables' `accept`
- `element`, `handle` — pass existing element/ref instead of using returned refs
- `disabled: boolean`
- `plugins: PluginDescriptor[]` — per-entity, e.g. `[Feedback.configure({feedback: 'clone'})]`
- `modifiers: Modifier[]` — override global ones
- `sensors: Sensor[]` — override global ones
- `alignment: {x: 'start'|'center'|'end', y: …}` — overlay placement
- `data: object` — available as `source.data` in events
- `effects: () => Effect[]` — advanced

Output: `ref`, `handleRef`, `isDragging`, `isDragSource`, `isDropping` (during drop animation), `draggable` (instance).

## useDroppable(input) — `@dnd-kit/react`

Input:
- `id` (required)
- `element`
- `accept: Type | Type[] | (source: Draggable) => boolean` — omitted = accepts all
- `type` — category label; NOT consulted by accept rules (accept checks the draggable's type)
- `collisionDetector: (input) => Collision | null`
- `collisionPriority: number` (use `CollisionPriority.Lowest|Low|Normal|High|Highest` from `@dnd-kit/abstract`)
- `disabled`, `data`, `effects`

Output: `ref`, `isDropTarget`, `droppable`.

## useSortable(input) — `@dnd-kit/react/sortable`

Accepts everything from useDraggable + useDroppable, plus:
- `index` (required) — current position in its list
- `group` — list identifier for multi-list; items without group share one implicit group
- `transition: {duration?, easing?, idle?} | null` — `idle: true` animates index changes even when not dragging
- `target` — separate element to act as the droppable area
- `disabled: boolean | {draggable?: boolean, droppable?: boolean}`

Output: `ref`, `targetRef`, `sourceRef`, `handleRef`, `isDropTarget`, `isDragSource`, `isDragging`, `isDropping`, `sortable`.

`type` on a sortable = what *it* is when dragged; its own `accept` = what can be dropped on it.

## DragOverlay — `@dnd-kit/react`

Props: `children: ReactNode | (source: Draggable) => ReactNode`, `tag='div'`, `className`, `style`, `disabled: boolean | (source) => boolean`, `dropAnimation: undefined (250ms ease) | null | {duration, easing} | async ({element, feedbackElement, translate}) => void`.

Rules: one per provider; children only render during a drag; dnd kit hooks inside it do nothing.

## Monitoring hooks

- `useDragOperation()` → `{source, target}` (undefined when idle). Re-renders only on source/target change.
- `useDragDropMonitor({onBeforeDragStart, onDragStart, onDragMove, onDragOver, onCollision, onDragEnd})` — same events as provider, from any descendant. Must be inside a provider.
- `useDragDropManager()` → manager (type `DragDropManager | null`; outside a provider returns a shared default manager at runtime). For `manager.monitor.addEventListener('dragstart', fn)` (returns cleanup), registry access, imperative actions.

Event table:

| Event | Preventable | Data |
|---|---|---|
| beforeDragStart | yes | operation |
| dragStart | no | operation, nativeEvent |
| dragMove | yes | operation, to, by, nativeEvent |
| dragOver | yes | operation |
| collision | yes | collisions |
| dragEnd | no | operation, canceled, nativeEvent |

## Sensors — `@dnd-kit/dom`

Defaults: `PointerSensor`, `KeyboardSensor`. Configure at provider (`sensors`), per draggable (`sensors` option, takes precedence), via `Sensor.configure({...})`.

### PointerSensor.configure
- `activationConstraints: Constraint[] | (event, source) => Constraint[] | undefined`
  - `new PointerActivationConstraints.Distance({value: 8})`
  - `new PointerActivationConstraints.Delay({value: 200, tolerance: 10})` (tolerance number or `{x, y}`)
  - Multiple constraints: activates when ANY is satisfied.
  - Function returning `undefined` = no constraints (instant start), defaults are NOT restored.
  - Default behavior: mouse on handle → immediate; touch → 250ms delay / 5px tolerance; text inputs → 200ms / 0px; other → 200ms delay / 10px tolerance + 5px distance.
- `activatorElements: (source) => Element[]` — activator outside the source subtree (for an inner handle just use `handleRef`).
- `preventActivation: (event, source) => boolean` — default skips interactive elements (button, input, a…) that aren't the handle.

### KeyboardSensor.configure
- `keyboardCodes: {start, cancel, end, up, down, left, right}` — `KeyboardEvent.key` values (`'Enter'`, `'ArrowUp'`, `'w'`, alias `'Space'`). Defaults start/end with Space/Enter, cancel Escape.
- `offset: number | {x, y}` — px per arrow press (Shift ×5).
- `preventActivation: (event, source) => boolean`.

Replacing sensors with an array without `KeyboardSensor` removes keyboard accessibility.

## Modifiers

| Modifier | Import | Config |
|---|---|---|
| `RestrictToVerticalAxis` / `RestrictToHorizontalAxis` | `@dnd-kit/abstract/modifiers` | — |
| `SnapModifier` | `@dnd-kit/abstract/modifiers` | `.configure({size: 20 | {x, y}})` |
| `RestrictToElement` | `@dnd-kit/dom/modifiers` | `.configure({element: () => ref.current})` |
| `RestrictToWindow` | `@dnd-kit/dom/modifiers` | — |

Applied in array order. Global on provider or per hook (per hook wins).

## Collision detection — `@dnd-kit/collision`

Set per droppable/sortable via `collisionDetector`.
- `defaultCollisionDetection` — pointerIntersection, falling back to shapeIntersection. Used when none set.
- `pointerIntersection` — pointer must be inside the droppable rect. Precise zones.
- `shapeIntersection` — greatest overlap area. Large containers.
- `closestCenter` — default first; if no intersection, nearest center. Grids / card stacks.
- `closestCorners` — smallest average corner distance. Forgiving vertical lists.
- `pointerDistance` — nearest center to pointer.
- `directionBiased` — only targets in drag direction; reduces jitter between neighbors.

Custom:
```ts
import type {CollisionDetector} from '@dnd-kit/abstract';
import {CollisionPriority, CollisionType} from '@dnd-kit/abstract';
const detector: CollisionDetector = ({dragOperation, droppable}) => {
  if (!droppable.shape) return null;
  return {id: droppable.id, value: 1, type: CollisionType.Collision, priority: CollisionPriority.Normal};
};
```
Higher `value` wins; `collisionPriority` biases overlapping droppables.

## Plugins — `@dnd-kit/dom`

Default: Accessibility, AutoScroller, Cursor, Feedback, StyleInjector. Opt-in: Debug (visualizes shapes/collisions). Sortables additionally register `OptimisticSortingPlugin` and `SortableKeyboardPlugin` (`@dnd-kit/dom/sortable`).

Always extend with the function form so defaults survive:
```tsx
plugins={(defaults) => [...defaults, Feedback.configure({dropAnimation: null})]}
```

### Feedback
- `feedback: 'default' | 'clone' | 'move' | 'none'`
  - default: element promoted to top layer, follows pointer
  - clone: copy stays in place, original moves
  - move: moves without top-layer promotion or placeholder
  - none: no feedback (use with custom DragOverlay)
- `dropAnimation: null | {duration, easing} | async ({element, translate}) => {…}`
- also `rootElement`, `keyboardTransition` (see https://dndkit.com/extend/plugins/feedback)

### Accessibility
```tsx
Accessibility.configure({
  announcements: {
    dragstart({operation: {source}}) { if (!source) return; return `Picked up ${source.id}`; },
    dragover({operation: {source, target}}) { if (!source || !target) return; return `${source.id} is over ${target.id}`; },
    dragend({operation: {source, target}, canceled}) {
      if (!source) return;
      return canceled ? 'Drag canceled' : `Dropped ${source.id} on ${target?.id ?? 'nothing'}`;
    },
  },
  // screenReaderInstructions: {draggable: '…'}, debounce: ms (dragover/dragmove only)
})
```
Sets ARIA attributes on draggables automatically.

## Sortable internals

**Optimistic sorting:** reorders DOM during drag without React renders. Consequences: `source` and `target` point to the same element; use `source.index` / `initialIndex` / `group` / `initialGroup`. Optimistic moves auto-revert on cancel. `event.preventDefault()` in `onDragOver` skips the optimistic update for that event (e.g. to forbid moving into a locked column). To disable entirely: `useSortable({..., plugins: [SortableKeyboardPlugin]})` (omit OptimisticSortingPlugin) — then source/target differ and you must reorder in `onDragOver` yourself.

**Type guards** (`@dnd-kit/react/sortable`):
```ts
if (isSortable(source)) { source.index; source.initialIndex; source.group; source.initialGroup; }
if (isSortableOperation(operation)) { operation.source.initialIndex; operation.target.index; }
```

**Manual reorder without helpers:**
```ts
onDragEnd={(event) => {
  if (event.canceled) return;
  const {source} = event.operation;
  if (!isSortable(source)) return;
  const {initialIndex, index, initialGroup, group} = source;
  // same group: splice(initialIndex,1) then splice(index,0,item)
  // cross group: remove from items[initialGroup][initialIndex], insert at items[group][index]
}}
```

**Helpers** (`@dnd-kit/helpers`):
- `move(items, event)` / `swap(items, event)` — items is `Id[]`, `{id}[]`, or `Record<group, Id[] | {id}[]>`. Works with dragover or dragend events; returns unchanged items if canceled or no target. Record keys are matched as strings against group/droppable ids.
- `arrayMove(array, from, to)` / `arraySwap(array, from, to)` — plain index helpers.

| | `move` | manual |
|---|---|---|
| Code | one line | more |
| When | onDragOver (multi-list) or onDragEnd | onDragEnd |
| Data shape | arrays / record of arrays | anything |
