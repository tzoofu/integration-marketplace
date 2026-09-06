# Gesture Swipe Card Engine

- **category**: other
- **provider**: internal (custom)
- **reusable**: yes — a pointer-event-based drag/swipe hook with deadzone, axis-lock, iOS bounce suppression, and fly-out/snap-back animation is a portable primitive for any card-swipe UI.
- **docs**: n/a — internal pattern built on the standard Pointer Events / Touch Events web APIs (https://developer.mozilla.org/en-US/docs/Web/API/Pointer_events), no external library.

## Overview
A shared pointer-event-based drag/swipe engine powering both a Tinder-style full-screen swipe deck (4-direction: left/right/up/down) and inline swipe-to-set-status gestures on list-view cards (horizontal only), unified behind one `useSwipe()` hook parameterized by axis mode.

## Playbook

### Prerequisites
- React (or any framework supporting hooks-equivalent local state + refs).
- No external gesture library needed — built entirely on native Pointer Events (`onPointerDown/Move/Up/Cancel`, `setPointerCapture`) plus one native (non-passive) `touchmove` listener for a Safari-specific bounce fix.

### Setup steps
1. Extract pure gesture math (threshold, deadzone, direction-from-delta) into a tiny standalone module with no React/DOM dependency — makes it trivially unit-testable and shareable.
2. Build one `useSwipe(options)` hook parameterized by `axis: "both" | "horizontal"` — "both" for a 4-direction full-screen deck, "horizontal" for an axis-locked inline card (so vertical page scroll still works, backed by CSS `touch-action: pan-y`).
3. Track a small state machine: `pos {x,y}` (live drag offset), `dragging`, `animating`, a `moved` ref (whether the pointer has exceeded a small deadzone — used to distinguish a tap from a drag so click-through still works), and a re-entrancy `locked` ref so a second gesture can't start mid fly-out animation.
4. In horizontal mode, don't commit to an axis until the pointer has moved past the deadzone — then lock to whichever axis had the larger delta; only call `setPointerCapture` once the horizontal axis is confirmed (so an accidental horizontal jitter during a vertical scroll doesn't hijack it).
5. Add a native (non-passive) `touchmove` listener via a callback ref (not a plain object ref) that calls `preventDefault()` once a drag has started — CSS `touch-action` alone doesn't reliably suppress iOS's elastic bounce mid-drag on every device; a callback ref is required if the underlying DOM node gets swapped (e.g. a card keyed by id that remounts after each swipe).
6. Implement `fly(direction, onDone)` (programmatic commit: animate to an off-screen position over N ms, then reset position and call `onDone` — used by button clicks/keyboard shortcuts, not just drag-release) and `snapBack()` (animate back to rest without committing — used for gestures that only "peek" rather than dismiss, e.g. a swipe-up that opens a detail view).
7. Wire keyboard-arrow equivalents to the same `commit(direction)` function the drag-release path uses, so the feature is keyboard-accessible for free.
8. For a full-screen deck, lock page scroll (`overflow: hidden` on `html`/`body`, plus `overscroll-behavior: none`) for the deck's entire lifetime — not just on mobile — if the deck's own vertical drag doubles as a swipe gesture; always reset scroll to top on mount so a stale scroll offset from another view doesn't get trapped once scroll is locked.

### Core pattern
```ts
// swipe-math.ts — pure, framework-agnostic gesture math
export type SwipeDirection = "right" | "left" | "up" | "down";
export const SWIPE_THRESHOLD = 90;
export const SWIPE_DEADZONE = 8;

export function directionFromDelta(dx: number, dy: number): SwipeDirection {
  if (Math.abs(dx) >= Math.abs(dy)) return dx > 0 ? "right" : "left";
  return dy < 0 ? "up" : "down";
}
```

```ts
// useSwipe.ts — shared drag/swipe hook (axis "both" = 4-way deck, "horizontal" = axis-locked card)
export interface UseSwipeOptions {
  axis?: "both" | "horizontal";
  threshold?: number;
  deadzone?: number;
  leaveMs: number;        // fly-out animation duration
  leaveDistance: number;  // fly-out travel distance
  allowLeft?: boolean;    // horizontal mode: clamp drag to one side
  allowRight?: boolean;
  enabled?: boolean;
  onCommit(dir: SwipeDirection): void;
}

export function useSwipe(opts: UseSwipeOptions) {
  const axis = opts.axis ?? "both";
  const threshold = opts.threshold ?? SWIPE_THRESHOLD;
  const deadzone = opts.deadzone ?? SWIPE_DEADZONE;

  const [pos, setPos] = useState({ x: 0, y: 0 });
  const [dragging, setDragging] = useState(false);
  const [animating, setAnimating] = useState(false);
  const startRef = useRef<{ x: number; y: number } | null>(null);
  const movedRef = useRef(false);
  const axisRef = useRef<null | "h" | "v">(null);
  const lockRef = useRef(false);

  // Native non-passive touchmove listener — CSS touch-action alone isn't
  // reliable enough to kill iOS elastic bounce mid-drag on every device.
  // A callback ref (not useEffect) so it rebinds when the node is swapped.
  const ref = useCallback((el: HTMLElement | null) => {
    if (!el) return;
    function onTouchMove(e: TouchEvent) { if (movedRef.current) e.preventDefault(); }
    el.addEventListener("touchmove", onTouchMove, { passive: false });
    return () => el.removeEventListener("touchmove", onTouchMove);
  }, []);

  function fly(dir: SwipeDirection, onDone?: () => void) {
    if (lockRef.current) return;
    lockRef.current = true;
    const d = opts.leaveDistance;
    const target = dir === "right" ? { x: d, y: 0 } : dir === "left" ? { x: -d, y: 0 }
      : dir === "up" ? { x: 0, y: -d } : { x: 0, y: d };
    setAnimating(true);
    setPos(target);
    setTimeout(() => { onDone?.(); setPos({ x: 0, y: 0 }); setAnimating(false); lockRef.current = false; }, opts.leaveMs);
  }

  function snapBack() { setAnimating(true); setPos({ x: 0, y: 0 }); }

  function onPointerDown(e: React.PointerEvent) {
    if (lockRef.current) return;
    startRef.current = { x: e.clientX, y: e.clientY };
    movedRef.current = false;
    axisRef.current = null;
    if (axis === "both") e.currentTarget.setPointerCapture(e.pointerId);
  }

  function onPointerMove(e: React.PointerEvent) {
    if (!startRef.current || lockRef.current) return;
    const dx = e.clientX - startRef.current.x;
    const dy = e.clientY - startRef.current.y;
    if (axis === "horizontal") {
      if (axisRef.current === null) {
        if (Math.abs(dx) < deadzone && Math.abs(dy) < deadzone) return;
        axisRef.current = Math.abs(dx) > Math.abs(dy) ? "h" : "v";
        if (axisRef.current === "h") { movedRef.current = true; setDragging(true); e.currentTarget.setPointerCapture(e.pointerId); }
      }
      if (axisRef.current === "h") {
        let clamped = dx;
        if (opts.allowLeft === false) clamped = Math.max(0, clamped);
        if (opts.allowRight === false) clamped = Math.min(0, clamped);
        setPos({ x: clamped, y: 0 });
      }
      return;
    }
    if (Math.abs(dx) > deadzone || Math.abs(dy) > deadzone) { movedRef.current = true; setDragging(true); }
    setPos({ x: dx, y: dy });
  }

  function onPointerUp() {
    if (!startRef.current) return;
    startRef.current = null;
    setDragging(false);
    if (axis === "horizontal") {
      const wasHorizontal = axisRef.current === "h";
      axisRef.current = null;
      if (wasHorizontal && Math.abs(pos.x) > threshold) opts.onCommit(pos.x > 0 ? "right" : "left");
      else setPos({ x: 0, y: 0 });
      return;
    }
    if (Math.abs(pos.x) > threshold || Math.abs(pos.y) > threshold) opts.onCommit(directionFromDelta(pos.x, pos.y));
    else snapBack();
  }

  return {
    pos, dragging, animating,
    moved: () => movedRef.current,
    progress: Math.min(1, Math.max(Math.abs(pos.x), Math.abs(pos.y)) / threshold),
    handlers: { onPointerDown, onPointerMove, onPointerUp, onPointerCancel: onPointerUp },
    fly, snapBack, locked: () => lockRef.current, ref,
  };
}
```

### Env vars
none — purely client-side gesture code, no external configuration.

### Gotchas
- CSS `touch-action` (`pan-y`/`none`) alone doesn't reliably stop iOS's elastic scroll bounce mid-drag on every device — pair it with a native, non-passive `touchmove` listener that calls `preventDefault()` once a drag is confirmed.
- Use a callback ref (not a plain `useRef` + `useEffect`) to attach that native listener if the swiped element can remount under a new key (e.g. a deck's top card keyed by item id) — a plain effect-based ref won't rebind to the new DOM node.
- Don't set pointer capture immediately on `pointerdown` in axis-locked (horizontal) mode — wait until the drag has cleared the deadzone AND resolved to the horizontal axis, or you'll hijack vertical page-scroll gestures that start on the same element.
- Guard every animation entry point (`fly`) with a re-entrancy lock — without it, rapid repeat triggers (double-tap on an action button, or overlapping keyboard + drag input) can start two fly-out animations on the same card and corrupt the position state.
- If a full-screen deck locks page scroll, always force-reset scroll position to top on mount — a leftover scroll offset from a different view combined with a scroll lock otherwise permanently strands the page mid-scroll with no way to recover.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. The adopter shares one gesture hook between a full-screen
4-direction swipe deck (used to set a record's status via left/right/up/down commit actions) and
an axis-locked, horizontal-only inline card swipe, including keyboard-arrow equivalents and a
persistent scroll lock for the full-screen deck.
