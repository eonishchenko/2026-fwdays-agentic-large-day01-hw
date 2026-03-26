# System patterns: `packages/excalidraw/`

Цей документ фіксує **архітектурні патерни** пакету `packages/excalidraw/`: ключові модулі, state management та взаємозв’язки компонентів.

## Архітектурний огляд

Пакет організовано “плоско” (без `src/`): top-level модулі + директорії підсистем.

- **Публічна точка входу**: `packages/excalidraw/index.tsx`
- **Центральний оркестратор редактора**: `packages/excalidraw/components/App.tsx` (класовий React-компонент `App`)
- **UI шар**: `packages/excalidraw/components/`
- **Командний шар**: `packages/excalidraw/actions/`
- **Сцена/елементи/рендер**: `packages/excalidraw/scene/`, `packages/excalidraw/renderer/` + інтеграція з `@excalidraw/element`
- **Дані та (де)серіалізація**: `packages/excalidraw/data/`
- **Контексти/портали UI**: `packages/excalidraw/context/`
- **UI-специфічний атомарний стан**: `packages/excalidraw/editor-jotai.ts` (+ атоми поруч із компонентами)

## Key modules (що за що відповідає)

### `index.tsx` — Composition Root + Public API

- Збирає дерево провайдерів та монтує `App`.
- Експортує:
  - **компонент `Excalidraw`**,
  - частину UI-компонентів (`Sidebar`, `MainMenu`, `WelcomeScreen`, …),
  - утиліти/серіалізацію/експорт (`serializeAsJSON`, `exportToSvg`, …),
  - хуки для підписки на appState через API (`useExcalidrawStateValue`, `useOnExcalidrawStateChange`).

Усередині `ExcalidrawBase`:

- `EditorJotaiProvider store={editorJotaiStore}`
- `InitializeApp` (i18n/theme/init)
- `App` (основний редактор)

### `components/App.tsx` — Editor Kernel (центр системи)

`App` створює та зв’язує основні підсистеми:

- **`this.state`**: `AppState` (основний стан редактора в React state)
- **`this.scene = new Scene()`**: сцена (джерело істини для елементів)
- **`this.store = new Store(this)`**: store з `@excalidraw/element` для захоплення змін/дельт
- **`this.history = new History(this.store)`**: undo/redo поверх `StoreDelta/StoreSnapshot`
- **`this.actionManager = new ActionManager(...)`**: виконання команд (клавіші/UI/API)
- **`this.renderer = new Renderer(this.scene)`**: рендеринг сцени

Також `App` роздає **контексти**, через які UI та інтеграція отримують доступ до API/стану:

- `ExcalidrawAPIContext` / `ExcalidrawAPISetContext`
- `ExcalidrawAppStateContext` (значення `this.state`)
- `ExcalidrawSetAppStateContext` (setter `this.setAppState`)
- `ExcalidrawElementsContext` (поточні non-deleted elements)
- `ExcalidrawActionManagerContext` (`this.actionManager`)
- `ExcalidrawContainerContext` (container/id)

`LayerUI` є основним UI-композитом: отримує `canvas`, `appState`, `elements`, `actionManager`, `setAppState` та рендерить панелі/меню/діалоги/канваси.

### `actions/*` — Command pattern

- `actions/register.ts` тримає **глобальний список** зареєстрованих actions (функція `register()` додає у масив).
- `actions/manager.tsx` (`ActionManager`):
  - реєструє actions,
  - знаходить відповідну action для шорткатів (`handleKeyDown`),
  - виконує action (`executeAction`) передаючи `(elements, appState, value, app)`,
  - може рендерити action-панелі (`renderAction`) для UI.

Таким чином: **UI/шорткати/API** → `ActionManager` → `Action.perform()` → повертається `ActionResult` → `App` синхронізує (AppState/elements/store/history).

### `history.ts` — Undo/redo на дельтах Store

`History` працює з `StoreDelta/StoreSnapshot` із `@excalidraw/element`:

- при звичайних змінах **записує інверсію** дельти в undo stack (`record()`),
- `undo()` / `redo()` застосовує `HistoryDelta` до `(elements, appState)` та планує micro-action у `store` для коректного “capture/sync”.

### `appState.ts` — Default AppState + storage conf

- `getDefaultAppState()` задає значення за замовчуванням для великої кількості UI/interaction полів (tool, selection, zoom/scroll, snapping, dialogs, …).
- `APP_STATE_STORAGE_CONF` описує, які ключі стану зберігати для різних цілей (browser/export/server).

## State management (як влаштовано)

Система використовує **3 рівні стану**, кожен із власною зоною відповідальності:

### 1) Editor AppState (React state у `App`)

- `AppState` — це “UI/interaction state”: активний інструмент, selection, zoom/scroll, відкриті меню/діалоги, флаги режимів, тощо.
- Живе в `App` як `this.state`, роздається через контекст `ExcalidrawAppStateContext`.
- Оновлюється через `this.setAppState` (роздається як `ExcalidrawSetAppStateContext` та пропсом у `LayerUI`).

### 2) Scene/elements state (Scene + Store з `@excalidraw/element`)

- Елементи (фігури) не тримаються в React state; джерелом істини виступають `Scene`/`Store`.
- UI отримує масив елементів через `this.scene.getNonDeletedElements()` (контекст + пропси).
- Зміни елементів відбуваються через механізми `@excalidraw/element` (мутації/дельти/снапшоти), а `History` працює поверх цих дельт.

### 3) Editor-local UI atoms (Jotai)

`editor-jotai.ts`:

- створює ізольований scope (`createIsolation()` з `jotai-scope`) та окремий `editorJotaiStore`.
- використовується для дрібного/локального стану UI, який не хочуть тримати в `AppState` (попапи, панелі, тимчасові UI-фічі).
- у `App` є helper `updateEditorAtom(...)`, який робить `editorJotaiStore.set(atom, ...)` і форсить перерендер (через `triggerRender()`).

## Підписки на стан ззовні (host app)

Хуки `hooks/useAppStateValue.ts` працюють через **imperative API**:

- `useAppStateValue(selector)` підписується на `api.onStateChange(selector, cb)` і ререндерить компонент лише коли змінюється потрібний фрагмент.
- `useOnAppStateChange(selector, cb)` викликає callback без перерендеру.

Це дозволяє host apps працювати “поза деревом” `Excalidraw`, якщо вони обгорнуті в `ExcalidrawAPIProvider` (див. `index.tsx`).

## UI composition: контексти + “tunnels”

Окрім звичайних React Context, пакет використовує “тунелі” (`tunnel-rat`) в `context/tunnels.ts`:

- це патерн для рендерингу частин UI (меню/хінти/футер/тригери) в потрібні місця DOM/layout, не перев’язуючи компонентне дерево.
- у `TunnelsContextValue` також присутній окремий `tunnelsJotai` scope (тимчасове рішення “поки не буде store per editor instance”).

## Канонічні потоки

### Потік виконання команди (keyboard/UI/API)

`UI`/`keyboard`/`host API` → `ActionManager` → `Action.perform(elements, appState, value, app)` → `ActionResult` → `App` синхронізує:

- оновлює `AppState` (`setState` / `setAppState`)
- оновлює elements у `Scene/Store` (дельти/снапшоти)
- `History` записує/відкочує зміни
- `Renderer`/канваси перемальовуються

### Потік “дані ↔ редактор”

`data/*` (restore/reconcile/json/blob/library/…) ↔ `App` (initialData, load/save/export) ↔ `Scene/Store` + `AppState`.

