# 03 — Project: Kanban Board (Drag and Drop)

> **"Drag and drop looks like a UI feature, but underneath it's a state synchronization problem: the visual position during a drag, the optimistic committed position after drop, and the eventually-consistent server position all need to agree — and they need to agree even when the user drags fast, drops on an invalid target, or the network request fails after the drop already looked successful."**

This project guide builds a Trello-style Kanban board: draggable cards across columns, draggable column reordering, optimistic updates with rollback, real-time collaboration awareness, and keyboard-accessible drag-and-drop as a first-class requirement, not an afterthought.

---

## 📚 What You'll Build

A Kanban board with: multiple columns containing draggable cards, drag-and-drop reordering within and across columns, column reordering, optimistic persistence with rollback on failure, multi-user awareness (see who else is viewing/editing), and full keyboard accessibility for drag operations.

---

## Requirements

```
FUNCTIONAL:
  - Columns containing cards; cards draggable within and between columns
  - Columns themselves are draggable (reorderable)
  - Card details: title, description, labels, assignee, due date
  - Optimistic reordering: UI updates instantly, persists in the background
  - Undo/redo for accidental moves

NON-FUNCTIONAL:
  - Drag-and-drop must be keyboard-accessible (not just mouse/touch)
  - Reordering 100+ cards across 10+ columns must stay performant
  - Network failure during a drag operation must not corrupt board state
  - Works correctly with concurrent edits from multiple users
```

---

## Architecture Overview

```
COMPONENT TREE:
  <Board>
    <BoardHeader />
    <ColumnList>                  (draggable container of columns)
      <Column>                     (draggable, contains cards)
        <ColumnHeader />
        <CardList>                  (droppable area for cards)
          <Card />                   (draggable)
        <AddCardButton />
      <AddColumnButton />
    <CardDetailModal />            (opened when a card is clicked)

STATE MODEL:
  Board: { id, columns: Column[] }
  Column: { id, title, cardIds: string[], position: number }
  Card: { id, title, description, columnId, position: number, ... }

  KEY DESIGN DECISION: cards reference their column via columnId, and
  ordering within a column is tracked via a `position` field (fractional
  indexing — see Step 2) rather than array index, so reordering doesn't
  require renumbering every other card.
```

---

## Step 1 — Choosing a Drag-and-Drop Approach

```
OPTIONS:
  1. Native HTML5 Drag and Drop API
     Pros: no library, browser-native
     Cons: notoriously inconsistent across browsers, poor mobile/touch
           support, awkward to make accessible, difficult custom drag previews

  2. Pointer-event-based custom implementation
     Pros: full control, can be made accessible, works on touch
     Cons: significant implementation effort (hit testing, auto-scroll, etc.)

  3. A dedicated library: @dnd-kit, react-beautiful-dnd (maintenance mode),
     or Pragmatic drag and drop (Atlassian)
     Pros: handles touch, accessibility, virtualization compatibility,
           edge cases already solved
     Cons: dependency, learning curve for the library's mental model

DECISION FOR THIS PROJECT: @dnd-kit
  Reasons: actively maintained, built-in keyboard support, sensors
  architecture supports both mouse and touch, works well with virtualized
  lists, and has first-class accessibility announcements.
```

---

## Step 2 — Fractional Indexing for Card Positions

The trickiest data modeling decision: how do you represent "this card's position within its column" in a way that supports efficient reordering?

```typescript
// ❌ NAIVE APPROACH: integer positions, renumber on every move
// Moving a card to position 2 requires updating EVERY card after it
// (position 2 → 3, position 3 → 4, etc.) — expensive and creates many
// conflicting updates in a multi-user scenario

// ✅ FRACTIONAL INDEXING: positions are floating-point numbers,
// inserting between two cards just needs the average of their positions
function getPositionBetween(
  before: number | null,
  after: number | null,
): number {
  if (before === null && after === null) return 1000; // first card in an empty column
  if (before === null) return after! / 2; // insert at the very start
  if (after === null) return before + 1000; // insert at the very end
  return (before + after) / 2; // insert between two cards
}

// Example: cards at positions [1000, 2000, 3000]
// Inserting between 1000 and 2000: getPositionBetween(1000, 2000) = 1500
// New order: [1000, 1500, 2000, 3000] — only ONE record needed updating

// LIMITATION: after MANY insertions between the same two positions,
// floating-point precision can be exhausted (positions converge).
// Production systems (Figma, Linear) use a more robust approach:
// base-62 STRING fractional indices, which have effectively unlimited
// precision (you can always insert a character to get a value between
// any two strings). For this project, numeric fractional indexing is
// sufficient at typical Kanban board scale, but it's worth knowing the
// string-based approach exists for systems with very high edit frequency.

function moveCard(
  card: Card,
  newColumnId: string,
  beforeCard: Card | null,
  afterCard: Card | null,
) {
  const newPosition = getPositionBetween(
    beforeCard?.position ?? null,
    afterCard?.position ?? null,
  );
  return { ...card, columnId: newColumnId, position: newPosition };
}
```

**Key decision:** fractional indexing means moving a single card only requires updating that ONE card's record (new `position` and possibly new `columnId`) — not renumbering the entire column. This matters enormously for both performance (fewer writes) and multi-user conflict reduction (fewer records being concurrently modified).

---

## Step 3 — Optimistic Drag-and-Drop with Rollback

```typescript
function useDragAndDrop(board: Board) {
  const queryClient = useQueryClient();

  const moveCardMutation = useMutation({
    mutationFn: (params: {
      cardId: string;
      newColumnId: string;
      newPosition: number;
    }) => boardApi.moveCard(params),

    // Optimistic update: apply immediately, before server confirms
    onMutate: async ({ cardId, newColumnId, newPosition }) => {
      await queryClient.cancelQueries({ queryKey: ["board", board.id] });

      const previousBoard = queryClient.getQueryData(["board", board.id]);

      queryClient.setQueryData(["board", board.id], (old: Board) => ({
        ...old,
        cards: old.cards.map((c) =>
          c.id === cardId
            ? { ...c, columnId: newColumnId, position: newPosition }
            : c,
        ),
      }));

      return { previousBoard }; // saved for rollback
    },

    // Rollback on failure
    onError: (err, variables, context) => {
      if (context?.previousBoard) {
        queryClient.setQueryData(["board", board.id], context.previousBoard);
      }
      toast.error("Failed to move card — reverted");
    },

    // Always refetch after settling, to reconcile with server truth
    // (in case other users made concurrent changes)
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ["board", board.id] });
    },
  });

  function handleDragEnd(event: DragEndEvent) {
    const { active, over } = event;
    if (!over) return; // dropped outside any valid target

    const cardId = active.id as string;
    const newColumnId = over.data.current?.columnId;
    const { beforeCard, afterCard } = computeNeighbors(over);

    moveCardMutation.mutate({
      cardId,
      newColumnId,
      newPosition: getPositionBetween(
        beforeCard?.position,
        afterCard?.position,
      ),
    });
  }

  return { handleDragEnd };
}
```

**Key decision:** the `onMutate` / `onError` / `onSettled` triad from TanStack Query is the canonical pattern for optimistic updates with safe rollback. `onMutate` snapshots the previous state BEFORE applying the optimistic change, `onError` restores that exact snapshot if the server rejects the change, and `onSettled` always refetches afterward — this last step is important because it reconciles with whatever the server's true state is, which matters when other users have made concurrent changes to the same board.

---

## Step 4 — Keyboard Accessibility for Drag and Drop

```jsx
import {
  DndContext,
  KeyboardSensor,
  PointerSensor,
  useSensor,
  useSensors,
} from "@dnd-kit/core";
import { sortableKeyboardCoordinates } from "@dnd-kit/sortable";

function Board({ board }) {
  const sensors = useSensors(
    useSensor(PointerSensor, {
      activationConstraint: { distance: 8 }, // avoid triggering drag on simple clicks
    }),
    useSensor(KeyboardSensor, {
      coordinateGetter: sortableKeyboardCoordinates,
    }),
  );

  return (
    <DndContext
      sensors={sensors}
      onDragStart={handleDragStart}
      onDragEnd={handleDragEnd}
      accessibility={{
        announcements: {
          onDragStart({ active }) {
            return `Picked up card ${active.data.current?.title}.`;
          },
          onDragOver({ active, over }) {
            if (over) {
              return `Card ${active.data.current?.title} is over column ${over.data.current?.columnTitle}.`;
            }
          },
          onDragEnd({ active, over }) {
            if (over) {
              return `Card ${active.data.current?.title} was dropped in column ${over.data.current?.columnTitle}.`;
            }
            return `Card ${active.data.current?.title} was dropped.`;
          },
          onDragCancel({ active }) {
            return `Dragging was cancelled. Card ${active.data.current?.title} was returned to its original position.`;
          },
        },
      }}
    >
      {/* Board content */}
    </DndContext>
  );
}
```

```
KEYBOARD INTERACTION MODEL (with @dnd-kit's KeyboardSensor):
  Tab: focus a draggable card
  Space/Enter: "pick up" the focused card (enters dragging mode)
  Arrow keys: move the card between valid drop positions
  Space/Enter again: "drop" the card at the current position
  Escape: cancel the drag, return card to its original position

This mirrors the WAI-ARIA design pattern for drag-and-drop, and the
`announcements` configuration above ensures screen reader users get
verbal feedback at each stage — without this, drag-and-drop is
completely unusable for screen reader users, since visual-only feedback
(the dragged card visually moving) provides no information to them.
```

**Key decision:** keyboard accessibility for drag-and-drop is not a "nice to have" add-on — for users who cannot use a mouse (motor impairments, screen reader users, keyboard-only power users), it's the ONLY way to interact with the board at all. Treating it as a first-class requirement (built into the sensor configuration from day one) is far less work than retrofitting it after the visual drag-and-drop is already built around assumptions that only work for pointer input.

---

## Step 5 — Multi-User Awareness (Avoiding Edit Conflicts)

```typescript
function useBoardPresence(boardId: string) {
  const [activeUsers, setActiveUsers] = useState<Map<string, PresenceInfo>>(
    new Map(),
  );

  useEffect(() => {
    const channel = new BroadcastChannel(`board-${boardId}`); // same-origin tabs
    // For true cross-device: use a WebSocket connection instead

    function broadcastPresence() {
      channel.postMessage({
        type: "presence",
        userId: currentUserId,
        userName: currentUserName,
        timestamp: Date.now(),
      });
    }

    channel.onmessage = (e) => {
      if (e.data.type === "presence") {
        setActiveUsers((prev) => new Map(prev).set(e.data.userId, e.data));
      }
      if (e.data.type === "editing_card") {
        // Show a visual indicator that another user is currently editing this card
        setEditingIndicators((prev) =>
          new Map(prev).set(e.data.cardId, e.data.userName),
        );
      }
    };

    broadcastPresence();
    const interval = setInterval(broadcastPresence, 10_000); // heartbeat

    return () => {
      clearInterval(interval);
      channel.close();
    };
  }, [boardId]);

  // Prune stale presence entries (user closed tab without a clean disconnect)
  useEffect(() => {
    const pruneInterval = setInterval(() => {
      setActiveUsers((prev) => {
        const next = new Map(prev);
        const staleThreshold = Date.now() - 30_000;
        next.forEach((info, userId) => {
          if (info.timestamp < staleThreshold) next.delete(userId);
        });
        return next;
      });
    }, 10_000);
    return () => clearInterval(pruneInterval);
  }, []);

  return activeUsers;
}
```

**Key decision:** presence uses a heartbeat (broadcast every 10s) plus a pruning mechanism (remove entries older than 30s) rather than relying solely on an explicit "user left" event — this handles the common case where a user simply closes their browser tab without the application getting a chance to clean up, which is the norm rather than the exception in real usage.

---

## Step 6 — Undo/Redo for Card Moves

```typescript
interface MoveAction {
  cardId: string;
  fromColumnId: string;
  fromPosition: number;
  toColumnId: string;
  toPosition: number;
}

function useUndoableMoves() {
  const [history, setHistory] = useState<MoveAction[]>([]);
  const [redoStack, setRedoStack] = useState<MoveAction[]>([]);
  const { moveCard } = useDragAndDrop();

  function recordMove(action: MoveAction) {
    setHistory((prev) => [...prev, action]);
    setRedoStack([]); // clear redo stack on new action
  }

  function undo() {
    const lastAction = history[history.length - 1];
    if (!lastAction) return;

    moveCard({
      cardId: lastAction.cardId,
      newColumnId: lastAction.fromColumnId,
      newPosition: lastAction.fromPosition,
    });

    setHistory((prev) => prev.slice(0, -1));
    setRedoStack((prev) => [...prev, lastAction]);
  }

  function redo() {
    const nextAction = redoStack[redoStack.length - 1];
    if (!nextAction) return;

    moveCard({
      cardId: nextAction.cardId,
      newColumnId: nextAction.toColumnId,
      newPosition: nextAction.toPosition,
    });

    setRedoStack((prev) => prev.slice(0, -1));
    setHistory((prev) => [...prev, nextAction]);
  }

  // Keyboard shortcuts
  useEffect(() => {
    function handleKeyDown(e: KeyboardEvent) {
      if ((e.metaKey || e.ctrlKey) && e.key === "z") {
        e.preventDefault();
        e.shiftKey ? redo() : undo();
      }
    }
    document.addEventListener("keydown", handleKeyDown);
    return () => document.removeEventListener("keydown", handleKeyDown);
  }, [history, redoStack]);

  return {
    undo,
    redo,
    canUndo: history.length > 0,
    canRedo: redoStack.length > 0,
  };
}
```

---

## Performance Checklist

```
☐ Drag preview uses transform (compositor-only) for smooth dragging,
  not top/left position updates
☐ Large boards (100+ cards): consider virtualizing column contents
☐ Fractional indexing avoids renumbering on every reorder
☐ Optimistic updates make drag-and-drop feel instant regardless of
  network latency
☐ Drag sensors configured with activation distance to avoid accidental
  drags on simple clicks
```

## Accessibility Checklist

```
☐ Full keyboard operability: pick up, move, drop, cancel
☐ Screen reader announcements at each stage of the drag operation
☐ Focus management: focus returns to the moved card after drop
☐ Color is not the only indicator for card labels/priority (use icons/text too)
☐ Sufficient color contrast for all interactive elements
```

---

## Extension Ideas

```
- Swimlanes (group cards by assignee or priority, crossing column boundaries)
- Card templates and checklists
- Activity log / audit trail per card
- Filters (by assignee, label, due date) that hide non-matching cards
- WIP (work-in-progress) limits per column with visual warnings
- Board templates for quick board creation
- Export board to CSV/JSON
```

---

## 🔗 Related Topics

- [`patterns/05-compound-components.md`](../patterns/05-compound-components.md) — Compound component patterns useful for Card/Column composition
- [`animations/03-compositor-animations.md`](../animations/03-compositor-animations.md) — Smooth drag animations
- [`anti-patterns/01-prop-drilling.md`](../anti-patterns/01-prop-drilling.md) — State management across the board hierarchy

---

<div align="center">

**Next:** [`projects/04-markdown-editor.md`](./04-markdown-editor.md) →

</div>
