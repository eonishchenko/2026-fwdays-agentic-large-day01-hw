# Decision Log

## 2026-03-26 - App.tsx state management analysis

### Context
- Requested analysis target: `packages/excalidraw/components/App.tsx`.
- Goal: describe state management approach, side effects, and lifecycle behavior.

### Findings
- `App` is a class component (`React.Component<AppProps, AppState>`) with explicit lifecycle methods.
- State management is hybrid:
  - UI/application state is stored in `this.state` (`AppState`).
  - Scene/domain data is managed via dedicated objects (`Scene`, `Store`, `History`, `Library`) rather than only React state.
  - `syncActionResult()` acts as the central state synchronization path by applying action results to scene, files, and app state, and scheduling store/history updates.
- State propagation happens through:
  - React Context providers (`ExcalidrawAppStateContext`, `ExcalidrawElementsContext`, API and manager contexts).
  - Callback props and emitters (`onChange`, `onScrollChange`, `onStateChange`, internal event bus emitters).

### Side Effects Identified
- Browser/DOM integrations:
  - Global and container event listeners (keyboard, pointer, wheel, paste/cut, drag/drop, resize, blur/focus, scroll, message, fullscreen, gestures).
  - `ResizeObserver` usage for layout recalculation.
  - Direct document style mutation (`document.documentElement.style.overscrollBehaviorX`).
  - `window.launchQueue` consumer setup for launch/share target scenarios.
- Async behavior:
  - Initial scene data restore from `initialData` (function or promise).
  - Library updates and font loading with async follow-up handlers.
- External notifications:
  - Lifecycle events to host (`onMount`, `onInitialize`, `onUnmount`, `onExcalidrawAPI`, `onChange`).
  - Internal lifecycle bus events (`editor:mount`, `editor:initialize`, `editor:unmount`).
- Cleanup side effects:
  - Destroying renderer/scene/library, clearing caches, stopping trails, removing listeners/subscriptions, and invalidating API usage after unmount.

### Lifecycle Summary
- Constructor:
  - Initializes default `AppState`, editor infrastructure objects, action manager, scene/store/history, and imperative API reference.
- `componentDidMount`:
  - Recreates API for safe lifecycle semantics.
  - Registers listeners/subscriptions and observers.
  - Initializes scene and dimensions (`updateDOMRect`, `initializeScene` path).
  - Emits mount callbacks/events and exposes API to host.
- `componentDidUpdate`:
  - Emits one-time initialize event when loading completes.
  - Flushes app-state observer, reacts to prop/state transitions, syncs UI flags, emits scroll/follow events, commits store snapshot, and emits `onChange` when not loading.
- `componentWillUnmount`:
  - Marks API as destroyed, emits unmount events, disconnects observers, removes listeners, destroys runtime objects, clears emitters/caches, and resets global side effects.

### Decision
- Treat `App.tsx` as an orchestration/root runtime component:
  - React state manages UI-facing app state.
  - Non-React domain services (`Scene`/`Store`/`History`) manage editor model, mutations, and persistence semantics.
  - Lifecycle methods are critical for side-effect registration and cleanup correctness.
