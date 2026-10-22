---
name: dnd-kit-react
description: Build drag-and-drop UIs in React with dnd kit (the new @dnd-kit/react package, plus migrating from or maintaining legacy @dnd-kit/core / @dnd-kit/sortable). Use this skill whenever the user wants draggable items, drop zones, sortable/reorderable lists, kanban or trello boards, drag handles, drag overlays, sortable grids, reordering table rows or columns, file/card drag between containers, or mentions dnd-kit, dndkit, DndContext, DragDropProvider, useDraggable, useDroppable, useSortable, SortableContext, arrayMove — even if they just say "make this list draggable" or "let users reorder these".
---

# dnd kit for React

dnd kit has two generations. Getting this right first matters more than anything else, because the APIs look similar but are incompatible:

| | **New (default)** | **Legacy** |
|---|---|---|
| Packages | `@dnd-kit/react`, `@dnd-kit/helpers` (+ `@dnd-kit/dom`, `@dnd-kit/abstract`, `@dnd-kit/collision` pulled in automatically) | `@dnd-kit/core`, `@dnd-kit/sortable`, `@dnd-kit/utilities`, `@dnd-kit/modifiers` |
| Provider | `<DragDropProvider>` | `<DndContext>` |
| Hooks return | `ref` (+ `handleRef`, `isDragging`…) | `setNodeRef`, `listeners`, `attributes`, `transform` |
| Sortable | `useSortable({id, index})` from `@dnd-kit/react/sortable` — no context wrapper | `SortableContext` + strategies + `arrayMove` |

**Decide which one to use:**
1. Look at `package.json` (and existing imports) in the user's project.
2. Nothing installed → use the **new** API.
3. Legacy `@dnd-kit/core` already in use → keep writing legacy-style code so the codebase stays consistent, and mention in one line that a migration path exists. Only migrate if the user asks. Read `references/legacy-and-migration.md` for both cases.
4. Never mix the two in one provider tree — hooks from one won't see the other's context.

The rest of this file is the new API. Docs: https://dndkit.com/react/quickstart

## Install

```bash
npm install @dnd-kit/react @dnd-kit/helpers   # helpers only needed for sortable `move`
```
Peer deps: React 18 or 19. The package ships `'use client'`, but in Next.js App Router the component that renders `<DragDropProvider>` with event handlers must itself be a Client Component (`'use client'` at top), because functions can't be passed from Server Components.

## Mental model

- `<DragDropProvider>` creates a drag-drop manager for everything inside it. Ids must be unique **within one provider**. Separate providers = separate worlds (items can't cross).
- Hooks are thin wrappers around classes (`Draggable`, `Droppable`, `Sortable`). They return a **callback `ref`** — attach it to the DOM element. No spreading of `listeners`/`attributes`, no `transform` style: the library moves the element itself (the Feedback plugin promotes it to the top layer during drag).
- Sensible defaults are already on: PointerSensor (mouse/touch/pen) + KeyboardSensor; plugins Accessibility, AutoScroller, Cursor, Feedback, StyleInjector. You rarely need to configure sensors/plugins.
- Your React state is the source of truth. dnd kit tells you what happened through events; you update state.

## Import map

```ts
import {DragDropProvider, DragOverlay, useDraggable, useDroppable,
        useDragOperation, useDragDropMonitor, useDragDropManager} from '@dnd-kit/react';
import {useSortable, isSortable, isSortableOperation} from '@dnd-kit/react/sortable';
import {move, swap, arrayMove, arraySwap} from '@dnd-kit/helpers';
import {PointerSensor, KeyboardSensor, PointerActivationConstraints,
        Feedback, Accessibility, AutoScroller} from '@dnd-kit/dom';
import {RestrictToElement, RestrictToWindow} from '@dnd-kit/dom/modifiers';
import {RestrictToVerticalAxis, RestrictToHorizontalAxis, SnapModifier} from '@dnd-kit/abstract/modifiers';
import {CollisionPriority} from '@dnd-kit/abstract';
import {closestCenter, closestCorners, pointerIntersection, shapeIntersection,
        pointerDistance, directionBiased} from '@dnd-kit/collision';
```
These subpath imports are easy to get wrong (e.g. axis modifiers are in `abstract`, element/window ones in `dom`). Use this table rather than guessing.

## Recipes

### 1. Draggable into drop zones

```tsx
import {useState} from 'react';
import {DragDropProvider, useDraggable, useDroppable} from '@dnd-kit/react';

function Draggable({id}: {id: string}) {
  const {ref, isDragging} = useDraggable({id});
  return <button ref={ref} data-dragging={isDragging}>Drag me</button>;
}

function Droppable({id, children}: {id: string; children?: React.ReactNode}) {
  const {ref, isDropTarget} = useDroppable({id});
  return <div ref={ref} className={isDropTarget ? 'zone active' : 'zone'}>{children}</div>;
}

export default function App() {
  const [parent, setParent] = useState<string | null>(null);
  const item = <Draggable id="item" />;
  return (
    <DragDropProvider
      onDragEnd={(event) => {
        if (event.canceled) return;
        setParent((event.operation.target?.id as string) ?? null);
      }}
    >
      {parent === null && item}
      {['A', 'B', 'C'].map((id) => (
        <Droppable key={id} id={id}>{parent === id ? item : `Zone ${id}`}</Droppable>
      ))}
    </DragDropProvider>
  );
}
```

### 2. Single sortable list

```tsx
import {useState} from 'react';
import {DragDropProvider} from '@dnd-kit/react';
import {useSortable} from '@dnd-kit/react/sortable';
import {move} from '@dnd-kit/helpers';

function Item({id, index}: {id: string; index: number}) {
  const {ref, isDragging} = useSortable({id, index});
  return <li ref={ref} data-dragging={isDragging}>{id}</li>;
}

export default function List() {
  const [items, setItems] = useState(['a', 'b', 'c', 'd']);
  return (
    <DragDropProvider onDragEnd={(event) => setItems((items) => move(items, event))}>
      <ul>{items.map((id, index) => <Item key={id} id={id} index={index} />)}</ul>
    </DragDropProvider>
  );
}
```
- `key` must be the stable id, never the array index — otherwise React reuses the wrong DOM node after reorder.
- `move` matches items by `item === id` or `item.id === id`, so arrays of objects with an `id` field work directly. It also returns the original array on a canceled drag.
- Grid / horizontal layouts need no "strategy": sorting adapts to layout automatically.

### 3. Multiple lists / kanban

Items get `group` (their column) and `type`/`accept` so items and columns don't mix. Columns must be droppable themselves, or you can't drop into an **empty** column.

```tsx
import {useRef, useState} from 'react';
import {DragDropProvider, useDroppable} from '@dnd-kit/react';
import {useSortable} from '@dnd-kit/react/sortable';
import {CollisionPriority} from '@dnd-kit/abstract';
import {move} from '@dnd-kit/helpers';

function Item({id, index, column}: {id: string; index: number; column: string}) {
  const {ref, isDragging} = useSortable({id, index, group: column, type: 'item', accept: 'item'});
  return <div ref={ref} data-dragging={isDragging}>{id}</div>;
}

function Column({id, children}: {id: string; children: React.ReactNode}) {
  const {ref, isDropTarget} = useDroppable({
    id, type: 'column', accept: 'item',
    collisionPriority: CollisionPriority.Low, // items inside win over the column
  });
  return <div ref={ref} data-over={isDropTarget}>{children}</div>;
}

export default function Board() {
  const [items, setItems] = useState<Record<string, string[]>>({
    todo: ['t1', 't2'], doing: ['d1'], done: [],
  });
  const snapshot = useRef(items);

  return (
    <DragDropProvider
      onDragStart={() => { snapshot.current = items; }}
      onDragOver={(event) => setItems((items) => move(items, event))}
      onDragEnd={(event) => { if (event.canceled) setItems(snapshot.current); }}
    >
      {Object.entries(items).map(([column, ids]) => (
        <Column key={column} id={column}>
          {ids.map((id, index) => <Item key={id} id={id} index={index} column={column} />)}
        </Column>
      ))}
    </DragDropProvider>
  );
}
```
Why `onDragOver` here: cross-list moves need React to re-render the item into the new column during the drag. Because you're changing state mid-drag, snapshot in `onDragStart` and restore on cancel (Esc).

**Sortable columns too:** make `Column` use `useSortable({id, index, type: 'column', accept: ['item', 'column'], collisionPriority: CollisionPriority.Low})`, keep a `columnOrder` state, skip column drags in `onDragOver` (`if (event.operation.source?.type === 'column') return;`) and in `onDragEnd` do `setColumnOrder((c) => move(c, event))` when the source type is `'column'` and not canceled.

### 4. Drag handle

```tsx
const {ref, handleRef} = useSortable({id, index}); // same for useDraggable
return (
  <li ref={ref}>
    <span ref={handleRef} aria-label="Drag">⋮⋮</span>
    {label}
    <button onClick={remove}>Delete</button>
  </li>
);
```
Once `handleRef` is connected, only the handle starts a drag. Even without a handle, clicks on interactive children (`button`, `input`, `a`…) don't start a drag by default.

### 5. DragOverlay (custom preview while dragging)

```tsx
<DragDropProvider>
  {/* draggables… */}
  <DragOverlay>{(source) => <Card id={source.id} preview />}</DragOverlay>
</DragDropProvider>
```
Render it **once per provider** (use the function child to vary content by `source`). Don't call dnd kit hooks inside the overlay content — they have no effect there, so render a plain presentational component. `dropAnimation={null}` disables the return animation; `{duration, easing}` tunes it.

### 6. Reactive drag state elsewhere in the tree

`useDragOperation()` → `{source, target}`; re-renders only when they change (not on every pointer move). Use it for "Dragging X over Y" banners or disabling unrelated UI. For per-move updates use `useDragDropMonitor({onDragMove…})`.

## Event shape

All handlers receive `(event, manager)`:
```ts
event.operation.source        // Draggable | null   (legacy: active)
event.operation.target        // Droppable | null   (legacy: over)
event.operation.source?.data  // whatever you passed as `data` to the hook
event.operation.position      // {current: {x,y}, initial: {x,y}}
event.canceled                // dragEnd only; replaces legacy onDragCancel
```
Events: `onBeforeDragStart` (call `event.preventDefault()` to block), `onDragStart`, `onDragMove`, `onDragOver`, `onCollision`, `onDragEnd`.

## Pitfalls that cause most bugs

1. **Sortable: `source` and `target` are the same element.** Optimistic sorting (on by default) reorders the DOM during the drag, so by `onDragEnd` the target is the source itself. Don't compute moves from `source.id`/`target.id`. Use `move(items, event)` or `isSortable(source)` → `source.initialIndex` / `source.index` (and `initialGroup` / `group` for multi-list).
2. **Single list: update in `onDragEnd`, not `onDragOver`.** Optimistic sorting already gives visual feedback; updating every dragover just causes extra renders. Multi-list is the exception (recipe 3).
3. **Snapshot + revert on cancel** whenever you update state before `onDragEnd`.
4. **`plugins` / `sensors` / `modifiers` as a plain array replace the defaults.** That silently drops keyboard support, auto-scroll, accessibility. Prefer the function form: `plugins={(defaults) => [...defaults, Feedback.configure({dropAnimation: null})]}`. To reconfigure a default sensor: `sensors={(d) => [...d.filter((s) => s !== PointerSensor), PointerSensor.configure({...})]}`.
5. **`RestrictToElement` with a ref:** pass `element: () => containerRef.current`, not `containerRef.current` (that's `null` on first render, and the modifier silently does nothing).
6. **Server data (React Query/SWR) + sortable → duplicate items.** Render from local state; only sync from fetched data when not dragging (track with a ref set in `onDragStart`/`onDragEnd`).
7. **Double snap on drop:** your state update fights the drop animation. Use `move` in `onDragEnd`, or disable the animation via `Feedback.configure({dropAnimation: null})`.
8. **Empty column can't receive items:** the column needs `useDroppable` (recipe 3).
9. **Nested/overlapping droppables pick the wrong target:** lower the container's `collisionPriority` or set a different `collisionDetector` on that droppable (detectors are per droppable, not global).
10. **Don't port legacy habits:** no `{...listeners}`, `{...attributes}`, `CSS.Transform.toString(transform)`, `SortableContext`, or sorting strategies in new-API code.

## Common configuration (short form)

- Touch vs mouse activation:
  ```tsx
  PointerSensor.configure({
    activationConstraints(event) {
      return event.pointerType === 'touch'
        ? [new PointerActivationConstraints.Delay({value: 250, tolerance: 5})]
        : [new PointerActivationConstraints.Distance({value: 5})];
    },
  })
  ```
  Defaults already differ by pointer type (touch: 250 ms delay); only override if asked.
- Lock to an axis: `modifiers: [RestrictToVerticalAxis]` on the hook. Snap: `SnapModifier.configure({size: 20})`.
- Clone left behind while dragging: `plugins: [Feedback.configure({feedback: 'clone'})]` on the hook. With a custom `DragOverlay`, the source can use `feedback: 'none'`.
- Restrict which items a zone accepts: `type` on the draggable, `accept` (type, array, or `(source) => boolean`) on the droppable.
- Disable: `disabled: true`; sortable also takes `disabled: {draggable: true}` or `{droppable: true}`.
- Localized screen-reader announcements: `Accessibility.configure({announcements: {dragstart, dragover, dragend}})` via the plugins function form.

For full option lists (sensors, modifiers, collision detectors, feedback, accessibility, all hook props) read `references/api.md`.

## Before handing back code

- Every `useDraggable`/`useDroppable`/`useSortable` is rendered inside a `DragDropProvider`.
- Ids are unique within the provider and `key`s are stable ids.
- Sortable items receive the correct current `index`.
- Cancel is handled (`event.canceled`).
- Imports match the import map above.
- If the user has TypeScript, `event.operation.target?.id` is `UniqueIdentifier` (string | number) — cast or narrow when storing it in typed state.
