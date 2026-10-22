# Legacy @dnd-kit/core and migration

Use this when the project already depends on `@dnd-kit/core` / `@dnd-kit/sortable` / `@dnd-kit/utilities` / `@dnd-kit/modifiers`.

- **Maintaining legacy code:** write in the legacy style (section A) so the codebase stays consistent. Don't import from `@dnd-kit/react` in the same tree.
- **Migrating:** follow section B. Migrate a whole provider tree at once — legacy and new hooks can't share a context.

Official guide: https://dndkit.com/react/guides/migration · Legacy docs: https://dndkit.com/legacy/introduction/getting-started

## A. Legacy API cheat sheet

```tsx
import {DndContext, DragOverlay, closestCenter, KeyboardSensor, PointerSensor,
        useSensor, useSensors, useDraggable, useDroppable, type DragEndEvent} from '@dnd-kit/core';
import {SortableContext, arrayMove, sortableKeyboardCoordinates, useSortable,
        verticalListSortingStrategy} from '@dnd-kit/sortable';
import {CSS} from '@dnd-kit/utilities';

function SortableItem({id}: {id: string}) {
  const {attributes, listeners, setNodeRef, transform, transition, isDragging} = useSortable({id});
  return (
    <li ref={setNodeRef} {...attributes} {...listeners}
        style={{transform: CSS.Transform.toString(transform), transition, opacity: isDragging ? 0.5 : 1}}>
      {id}
    </li>
  );
}

export function List() {
  const [items, setItems] = useState(['a', 'b', 'c']);
  const sensors = useSensors(
    useSensor(PointerSensor, {activationConstraint: {distance: 5}}),
    useSensor(KeyboardSensor, {coordinateGetter: sortableKeyboardCoordinates}),
  );
  function handleDragEnd({active, over}: DragEndEvent) {
    if (over && active.id !== over.id) {
      setItems((items) => arrayMove(items, items.indexOf(active.id as string), items.indexOf(over.id as string)));
    }
  }
  return (
    <DndContext sensors={sensors} collisionDetection={closestCenter} onDragEnd={handleDragEnd}>
      <SortableContext items={items} strategy={verticalListSortingStrategy}>
        <ul>{items.map((id) => <SortableItem key={id} id={id} />)}</ul>
      </SortableContext>
    </DndContext>
  );
}
```
Legacy notes:
- `SortableContext items` must be the ids in rendered order.
- Strategies: `verticalListSortingStrategy`, `horizontalListSortingStrategy`, `rectSortingStrategy` (grids), `rectSwappingStrategy`.
- Multi-container: one `SortableContext` per container, move items between containers in `onDragOver`, finalize in `onDragEnd`; `onDragCancel` to revert.
- `DragOverlay` needs you to track `activeId` in state yourself.
- Use `CSS.Translate.toString` instead of `CSS.Transform` when you don't want scale distortion for different-sized items.
- Touch: `TouchSensor` with `{activationConstraint: {delay: 250, tolerance: 5}}`, plus `touch-action: none` on handles.
- Modifiers live in `@dnd-kit/modifiers` (`restrictToVerticalAxis`, `restrictToParentElement`, `restrictToWindowEdges`, `createSnapModifier`).

## B. Migration steps

1. Dependencies: remove `@dnd-kit/core`, `@dnd-kit/sortable`, `@dnd-kit/utilities` (and `@dnd-kit/modifiers`); add `@dnd-kit/react`, `@dnd-kit/helpers`.
2. `DndContext` → `DragDropProvider`. Handlers get `(event, manager)`; `active` → `event.operation.source`, `over` → `event.operation.target`. `onDragCancel` → `if (event.canceled)` inside `onDragEnd`.
3. `useDraggable` / `useDroppable`: replace `setNodeRef` + `{...listeners}` + `{...attributes}` + `transform` style with just `ref`. `isOver` → `isDropTarget`.
4. `useSortable`: import from `@dnd-kit/react/sortable`, **add `index`**, drop `transform`/`transition` styles. Delete `SortableContext` and strategies. For multi-container add `group`.
5. `arrayMove(items, oldIndex, newIndex)` in onDragEnd → `move(items, event)` (because with optimistic sorting `over.id === active.id`, the old `indexOf(over.id)` logic breaks).
6. `DragOverlay`: drop `activeId` state; use `<DragOverlay>{(source) => <Item id={source.id} />}</DragOverlay>`.
7. Sensors: delete `useSensors` unless customized. `MouseSensor` + `TouchSensor` → one `PointerSensor` with `activationConstraints(event) { event.pointerType === 'touch' ? … : … }`. `activationConstraint: {distance: 5}` → `[new PointerActivationConstraints.Distance({value: 5})]`; `{delay, tolerance}` → `Delay({value, tolerance})`.
8. Collision: `collisionDetection` prop on the context → `collisionDetector` option on each droppable/sortable. `pointerWithin` → `pointerIntersection`; `closestCenter`/`closestCorners` from `@dnd-kit/collision`.
9. Modifiers: `restrictToParentElement` → `RestrictToElement.configure({element: () => ref.current})` (`@dnd-kit/dom/modifiers`); `restrictToWindowEdges` → `RestrictToWindow`; `restrictToVerticalAxis`/`HorizontalAxis` → `RestrictToVerticalAxis`/`RestrictToHorizontalAxis` (`@dnd-kit/abstract/modifiers`); `createSnapModifier(n)` → `SnapModifier.configure({size: n})`.
10. Remove `touch-action` hacks only after testing — the new PointerSensor handles touch delay by default.

| Legacy | New |
|---|---|
| `DndContext` | `DragDropProvider` |
| `active` / `over` | `operation.source` / `operation.target` |
| `onDragCancel` | `event.canceled` in `onDragEnd` |
| `useSensor(s)` | `sensors` prop (defaults built in) |
| `MouseSensor`, `TouchSensor`, `PointerSensor` | `PointerSensor` (`@dnd-kit/dom`) |
| `KeyboardSensor` | `KeyboardSensor` (`@dnd-kit/dom`) |
| `collisionDetection` on context | `collisionDetector` per droppable |
| `pointerWithin` | `pointerIntersection` |
| `SortableContext`, strategies | not needed |
| `arrayMove` | `move` (`@dnd-kit/helpers`) |
| `isOver` | `isDropTarget` |
| `data.current` | `data` |
