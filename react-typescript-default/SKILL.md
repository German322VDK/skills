---
name: react-typescript-default
description: Corporate default standard for creating, reviewing, refactoring, or extending React + TypeScript frontend applications. Use for React architecture, Vite setup, Mantine UI, React Router, optional existing React Query usage, Zustand state, hooks, managers, page-based structure, public layer imports, aliases, styling, TypeScript rules, JSX formatting, component documentation, package scripts, and final validation.
---

# React TypeScript Default

Use this skill when creating, reviewing, refactoring, or extending React + TypeScript frontend applications.

Default stack:

- React
- TypeScript
- Vite
- Mantine
- React Router
- Zustand, only when shared client state is required

Prefer consistency with the existing project over introducing new patterns.

## React Principles

Prefer composition over inheritance. Prefer small focused components. Extract reusable UI only after duplication appears.

Avoid prop drilling when Context or Zustand is more appropriate.

Keep complex UI flows and business logic out of UI components. Use hooks only when React features are actually required; otherwise prefer plain TypeScript functions, managers, stores, or API clients.

Pages orchestrate features. Widgets are for reused or cross-page UI composition. Entities should not depend on pages.

## Project Structure

Use page-based architecture.

Preferred structure:

```text
src/
    app/
        router/
        theme/
        popup/
    pages/
    widgets/
    entities/
    dialogs/
    shared/
        api/
        hooks/
        lib/
        types/
        ui/
    assets/
        images/
            svg/
```

Prefer barrel exports:

```ts
import {
    MainPage,
    LoginPage,
    MetricsPage,
    TerminalPage,
    AccessesPage,
    TerminalLogsPage
} from '@Src/pages'
```

Entity folders may contain `api`, `converters`, `enums`, `models`, `store`, `types`, and `index.ts`. Do not put UI components into entities by default.

Preferred entity shape:

```text
entities/
    tasks/
        api/
            TasksApiClient.ts
            TasksApiClient.types.ts
        converters/
            TaskItemStatusConverter.ts
        enums/
            TaskItemStatus.ts
        models/
            TaskItem.ts
        store/
            TasksManager.ts
        index.ts
```

Export entity public API through `index.ts`. Do not import another entity's internal files directly when its public API exports the needed type or function.

Models go to `models`, enums to `enums`, converters and label maps to `converters`, API clients and API-only contracts to `api`, and stores / managers to `store`. Do not mix them in one folder when dedicated folders already exist. Use `Record<EnumOrUnion, string>` for display labels when appropriate.

Request and response types belong in the `api` folder when they are API-specific transport contracts. Put API-specific types in `TasksApiClient.types.ts`. Domain/frontend models belong in `models`; enums belong in `enums`. Do not put API-only DTOs such as `CreateTaskRequest` or `ChangeTaskStatusRequest` into `models`.

## Layer Responsibilities

- `app`: application bootstrap, router, theme, global providers, popup/dialog infrastructure.
- `pages`: page-level orchestration.
- `widgets`: reused cross-page UI blocks that compose entities and shared UI.
- `entities`: domain-specific frontend data and logic: API clients, converters, enums, models, stores, types, and public exports.
- `dialogs`: modal/dialog UI and dialog flows.
- `shared`: reusable infrastructure, UI, hooks, API utilities, types, and libraries.

Dependency direction:

- `app` may compose `pages`, `widgets`, and `shared`.
- `pages` may use `widgets`, `entities`, `dialogs`, and `shared`.
- `widgets` may use `entities` and `shared`.
- `entities` may use `shared`.
- `shared` must not depend on `pages`, `widgets`, `entities`, or `dialogs`.

Do not put business-specific logic into `shared` only because it is reused once.

Do not put UI components into `entities` by default. Entity UI is allowed only when the project already follows that style or the component is truly entity-specific and reused across pages/widgets. Prefer page-local components or widgets for UI.

## Pages

Pages should stay thin.

Pages orchestrate widgets, hooks, stores, dialogs, and navigation.

Pages may load initial data or call page-specific flows.

Do not put complex business logic directly into page components.

Move reusable page logic into manager/store functions or dedicated hooks only when React lifecycle or state is actually needed.

Do not create a widget just because a page became large. Widgets are for UI blocks reused across multiple pages or truly cross-page composition.

If UI belongs only to one page, split it into local page components inside the same page folder.

Pages should be thin, but not empty wrappers around one widget. Avoid `TasksPage -> TasksWidget` when the widget is used only by that page.

Prefer local page components over unnecessary widgets.

Split large components into smaller local components with one clear responsibility. Do not keep forms, title, filters, list, pagination, and item cards in one component when splitting improves readability.

Preferred page folder example:

```text
pages/
    tasks/
        TasksPage.tsx
        components/
            TasksPageTitle.tsx
            TasksCreateForm.tsx
            TasksList.tsx
            TaskCard.tsx
```

Do not create `index.ts` inside page folders unless the page is explicitly intended to expose a public API. Public page exports belong only to the root `pages/index.ts`.

Bad:

```tsx
/** Страница управления задачами */
export const TasksPage = memo(() => (
    <TasksWidget/>
))
```

Good:

```tsx
/** Страница управления задачами */
export const TasksPage = memo(() => (
    <Container py={'xl'} size={'lg'}>
        <TasksPageTitle/>
        <TasksCreateForm/>
        <TasksList/>
    </Container>
))
```

## File Naming

Each file name should match what it exports: `TasksPage.tsx` exports `TasksPage`, `TasksApiClient.ts` exports `TasksApiClient`, `TaskItemStatus.ts` exports `TaskItemStatus`, `TaskItem.ts` exports `TaskItem`, and `DeviceConverter.ts` exports `DeviceConverter`.

Avoid vague file names such as `index2.ts`, `utils.ts`, `helpers.ts`, and `types.ts` when the file mixes unrelated types.

`index.ts` is allowed only as a public barrel export.

## Public Layer Imports

Never import files from another layer's internal folders. Public layers expose a public API through `index.ts`.

Use barrel exports for public layer boundaries. Avoid creating barrel exports in every nested folder without need.

Import from the public layer root when crossing layer boundaries.

Good:

```ts
import {
    MainPage,
    LoginPage,
    MetricsPage,
    TerminalPage
} from '@Src/pages'
```

Bad:

```ts
import { MainPage } from '@Pages/main/MainPage'
import { TerminalPage } from '@Pages/terminal/TerminalPage'
import { DeviceConverter } from '@Entities/device/converters/DeviceConverter'
```

Example public API:

```ts
export { MainPage } from './main/MainPage'
export { LoginPage } from './login/LoginPage'
export { TerminalPage } from './terminal/TerminalPage'
```

Allowed public layers:

- `pages`
- `widgets`
- `entities`
- `dialogs`
- `shared/ui`
- `app`

Internal imports inside one feature are allowed.

Good cross-layer imports:

```ts
import { TerminalLogsPage } from '@Src/pages'
import { DeviceConverter } from '@Entities/device'
```

## Routing

Use `react-router-dom`.

Rules:

- put `BrowserRouter` at the application root;
- centralize routes inside `@App/router`;
- keep pages inside `src/pages`;
- do not scatter route definitions;
- prefer existing navigation abstraction hooks such as `useAppNavigation`;
- do not call `useNavigate` directly everywhere if the project already has a navigation hook;
- expose reused route params through navigation hooks;
- keep route strings centralized when possible.

## Application Root

Compose providers in this order:

1. `QueryClientProvider`, only when the existing project uses React Query
2. `MantineProvider`
3. `Notifications`
4. `ModalsProvider`
5. `BrowserRouter`
6. `PopupRenderer`, if used
7. `AppRouter`

Do not add React Query providers by default. When reusing React Query, create `QueryClient` outside React components.

## Mantine

Mantine is mandatory. Prefer Mantine components before custom styling.

Use:

- `@mantine/core`
- `@mantine/hooks`
- `@mantine/modals`
- `@mantine/notifications`
- `@mantine/dates`
- `@tabler/icons-react`

Do not introduce another UI framework unless explicitly requested.

## Styling

Prefer Mantine props, Mantine theme, and CSS variables.

Use SCSS Modules only when justified:

- Mantine props are insufficient;
- pseudo-elements are required;
- complex selectors are required;
- animations require CSS.

Avoid inline styles, global CSS, and large SCSS files.

## TypeScript

Use strict mode.

Avoid `any`, `ts-ignore`, and weakened compiler settings.

Prefer explicit types, readonly collections, `type` for DTOs / unions, and `interface` for extensible contracts.

## Hooks

Do not create hooks by default.

Create a hook only when React features are required:

- React state;
- React lifecycle;
- React context;
- React Router hooks;
- subscribing to Zustand or reactive store state;
- memoized callbacks that are actually used by React components.

Prefer plain TypeScript functions, managers, stores, or API clients when React features are not needed.

Avoid wrapping every API call, converter, formatter, or store action into a hook. Prefer existing hooks before creating new ones.

Valid hook examples: `useAppNavigation`, `useLifeCycle`, `useSnackbar`, `usePopup`, and `useTerminalLogs` when it uses React lifecycle, state, navigation, or store subscriptions.

Use existing `useLifeCycle` in the established project style. Inspect the existing hook signature before using it and do not invent a different lifecycle API.

Preferred style when the project exposes lifecycle callbacks:

```ts
const { onMount } = useLifeCycle()

onMount(() => {
    TasksManager.loadTasks()
})
```

Avoid object-based lifecycle APIs unless the existing hook actually requires them:

```ts
useLifeCycle({
    onMount: () => {
        TasksManager.loadTasks()
    }
})
```

Lifecycle hook example:

```ts
/* Хук размонтирования элемента */
const useOnUnmount = (callback: () => void) => {
    const savedCallback = useRef<() => void>(callback)

    /* Обновление ссылки на колбек */
    useEffect(() => {
        savedCallback.current = callback
    })

    useEffect(() => {
        return () => savedCallback.current()
        // eslint-disable-next-line react-hooks/exhaustive-deps
    }, [])
}

/* Хук вмонтирования элемента */
const useOnMount = (callback: () => void) => {
    const savedCallback = useRef<() => void>(callback)

    /* Обновление ссылки на колбек */
    useEffect(() => {
        savedCallback.current = callback
    })

    useEffect(() => {
        savedCallback.current()
        // eslint-disable-next-line react-hooks/exhaustive-deps
    }, [])
}

/* Хук жизненного цикла */
export const useLifeCycle = () => ({ onUnmount: useOnUnmount, onMount: useOnMount })
```

Logic that usually should not be a hook: API clients, converters, enum label maps, URL param actions, pure formatters, plain manager methods, and store mutation methods.

Avoid `useEffect` for use cases and business flows. Do not use `useEffect` as a default lifecycle replacement. Prefer explicit lifecycle helpers such as `onMount` / `onUnmount` when the project has `useLifeCycle`.

`useEffect` is allowed for low-level React integration or domain-specific reactive effects when no clearer abstraction exists. Loading initial page data should usually go through `onMount`, a store/manager, or an existing page flow abstraction.

Do not be afraid to use `memo`, `useMemo`, and `useCallback`. Prefer `memo` for exported pages, page-local components, widgets, and reusable UI components. Use `useCallback` for handlers passed to child components or stored in dependencies. Use `useMemo` for non-trivial derived values or when value stability matters. Avoid meaningless memoization for simple local constants that are not passed down.

### Promise Awaiting

Use an existing `usePromiseAwaiter` as the default helper for local promise pending state:

```ts
/** Функция для ожидания результата выполнения Promise с флагом процесса выполнения */
export const usePromiseAwaiter = <TReturn = void, TArgs extends unknown[] = []>(
    func: (...args: TArgs) => Promise<TReturn>
): [isPending: boolean, run: (...args: TArgs) => Promise<TReturn>] => {
    const [isPending, setPending] = useState(false)

    const run = useCallback((...args: TArgs): Promise<TReturn> => {
            setPending(true)
            return func(...args).finally(() => setPending(false))
        },
        [func])

    return [isPending, run]
}
```

Prefer existing `usePromiseAwaiter` over custom `isLoading` state for imperative async actions triggered by UI. Do not use it for shared server state handled by stores/managers or existing project abstractions. Do not create another promise loading hook if `usePromiseAwaiter` already exists.

Always handle returned promises explicitly. Use `await` when calling async functions inside another async function. Do not leave floating promises or ignore promises silently.

Preferred:

```ts
const handleCreateAsync = useCallback(async () => {
    await TasksManager.createTaskAsync(request)
}, [request])
```

Avoid inside async functions:

```ts
const refreshAsync = async () => {
    TasksManager.loadTasksAsync()
}
```

Use:

```ts
const refreshAsync = async () => {
    await TasksManager.loadTasksAsync()
}
```

## State Management

Use `useState` only for local component UI state, such as an input value local to one component, opened/closed state local to one component, or a temporary UI-only flag.

Server communication should be implemented through API clients, stores/managers, and existing project abstractions. Do not add React Query by default.

Use Zustand only for shared client state or page flow state that must be shared outside one component. If an existing project already uses React Query for server state, reuse it instead of replacing it.

Do not use `useState` for shared state, page flow state, or state that must be accessed outside one component.

For page-level or cross-component state, prefer a non-reactive Zustand vanilla store / manager. Components subscribe reactively only to the parts they need with `useStoreWithEqualityFn`.

Store actions should live in the store/manager, not inside UI components, when the logic is reusable or page-level.

State that should usually be store/manager state: selected filters, current page number, active entity id, shared loading/error state, and busy row id used across several local components.

## Stores And Managers

Use Zustand vanilla stores with `createStore` when state must be usable outside React.

Expose stores and actions through manager objects when this matches the project style. Store objects may expose `$store` plus explicit methods.

Use Immer `produce(...)` for complex immutable updates when useful.

Use Zustand for shared client/UI state. Follow the existing project convention for server state instead of introducing a parallel state solution.

A manager may orchestrate API calls and store updates when this matches the project style. Prefer a manager for page/feature use cases that combine API calls, loading state, error state, store updates, and refresh after successful actions.

Keep API clients thin: they only perform HTTP requests and return typed results. Keep UI components thin: components call manager methods and subscribe to store state. Do not put API response parsing or React hooks into managers.

Use `useStoreWithEqualityFn` for Zustand vanilla store subscriptions.

Reactive read example:

```ts
const tasks = useStoreWithEqualityFn(TasksManager.$store, s => s.tasks)
const isLoading = useStoreWithEqualityFn(TasksManager.$store, s => s.isLoading)
const errorMessage = useStoreWithEqualityFn(TasksManager.$store, s => s.errorMessage)
```

Non-reactive read example:

```ts
const divisionIds = CurrentLocationManager.divisionIds
```

Use reactive reads only when the component must re-render after state changes.

Inline selectors are preferred for simple state reads. Extract selector functions only when selector logic is reused or non-trivial. Do not duplicate selector functions for every field by default.

Avoid unnecessary selector functions:

```ts
const selectTasks = (state: ReturnType<typeof TasksManager.$store.getState>) => state.tasks
const tasks = useStore(TasksManager.$store, selectTasks)
```

## API

API clients are plain objects with arrow function methods, not hooks.

API clients belong in `shared/api` or entity/feature-specific `api` folders.

Keep API response types in `ApiClient.types.ts`. Keep request models in `models` when they are part of the domain/feature model.

Request and response types belong in the `api` folder when they are API-specific contracts. Put API-specific types in `TasksApiClient.types.ts`. Do not put API request/response DTOs into `models` if they are only transport contracts.

Do not create `index.ts` inside `api` folders unless it is explicitly needed as a public API boundary.

Prefer typed requests and responses.

Server communication should go through API clients, stores/managers, or existing project abstractions.

Do not put React Query inside API clients.

Do not generate `useMutation` by default. If an existing project already uses React Query, reuse it instead of replacing it.

Do not mix large request orchestration, local form state, error state, and table rendering inside one huge component. Extract use-case orchestration into local page functions, a page store/manager, or dedicated hooks only when React features are needed.

API clients should perform HTTP requests, serialize request bodies, deserialize responses, and return typed results. They should not contain duplicated response parsing, business logic, React state, React hooks, or React Query logic.

Common HTTP response handling must live in shared reusable API infrastructure, not inside every API client. Extract helpers such as `readJsonResponse`, `readEmptyResponse`, `readProblemMessage`, and `toFailResponse` to `shared/api`.

Preferred `shared/api` shape:

```text
shared/
    api/
        ApiResult.ts
        HttpResponseReader.ts
        ProblemDetailsConverter.ts
        Fetch.ts
```

`Fetch.ts` is optional and should be added only when a reusable fetch wrapper is actually needed. Reuse existing shared helpers for ProblemDetails, ApiResult, response parsing, or fetch wrappers. Never duplicate HTTP infrastructure.

API clients should return typed discriminated unions when the project uses result unions:

```ts
type GetDeviceResponse =
    | { status: 'ok', data: DeviceDto }
    | { status: 'fail', message: string }
```

UI and components handle `status === 'fail'` explicitly.

Do not throw for expected API errors such as `400` / `500` responses when the project uses result unions.

Catch unexpected client errors and convert them to `{ status: 'fail', message }`.

Preferred API client style:

```ts
/** API-клиент задач */
export const TasksApiClient = {
    getTasks,
    createTask,
    changeTaskStatus,
    deleteTask
}

const getTasks = async (signal?: AbortSignal): Promise<TaskListResponse> => {
    const response = await fetch('/tasks', { signal })
    return HttpResponseReader.readJsonResponse(response)
}
```

Response parsing belongs to shared API infrastructure. API clients should only describe endpoint-specific requests.

## Code Style

Use single quotes, no semicolons, functional components, named exports, early return, and readable names.

Use arrow functions by default. All local functions, exported functions, API client functions, helpers, handlers, and API helper functions should be arrow functions.

Avoid function declarations unless required by framework constraints or hoisting is intentionally needed.

Use arrow functions for shared API helpers such as `readJsonResponse`, `readEmptyResponse`, `readProblemMessage`, and `toFailResponse`.

Named async functions must use the `Async` suffix. Apply this to API client methods, manager methods, store actions, helpers, and exported async functions.

Preferred:

```ts
const loadTasksAsync = async (): Promise<void> => {
    ...
}

const refreshTasksAsync = async (): Promise<void> => {
    ...
}

/** API-клиент задач */
export const TasksApiClient = {
    getTasksAsync,
    createTaskAsync,
    deleteTaskAsync
}

const getTasksAsync = async (signal?: AbortSignal): Promise<TaskListResponse> => {
    ...
}
```

Avoid:

```ts
const loadTasks = async (): Promise<void> => {
    ...
}

const getTasks = async (): Promise<TaskListResponse> => {
    ...
}
```

Do not add `Async` suffix to inline lambdas passed directly to React hooks or callbacks, React component names, or hook names unless the hook itself is intentionally named that way.

Avoid class components, default exports, huge components, unnecessary abstractions, and meaningless `useEffect`, `useMemo`, or `useCallback`.

Do not add `void` before async calls by default. Prefer direct calls when the surrounding handler may ignore the returned promise:

```ts
const handleDelete = useCallback((taskId: string) => {
    TasksManager.deleteTaskAsync(taskId)
}, [])
```

Use `void` only when the existing ESLint configuration requires it or when intentionally marking a floating promise. Do not add `void` mechanically.

If ESLint requires explicit marking for fire-and-forget promises:

```ts
const handleDelete = useCallback((taskId: string) => {
    void TasksManager.deleteTaskAsync(taskId)
}, [])
```

Bad:

```ts
async function getTasks(signal?: AbortSignal): Promise<TaskListResponse> {
    ...
}
```

Good:

```ts
const getTasks = async (signal?: AbortSignal): Promise<TaskListResponse> => {
    ...
}
```

Bad:

```ts
function handleDelete(taskId: string) {
    ...
}
```

Good:

```ts
const handleDelete = useCallback((taskId: string) => {
    ...
}, [])
```

## JSX Style

Prefer explicit braces:

```tsx
<Button variant={'filled'}>
    {'Сохранить'}
</Button>

<Title children={`Логи: ${name ?? ''}`}/>
```

Prefer self-closing tags:

```tsx
<AppRouter/>
<LoadingView/>
```

Avoid overly strict null checks when an ordinary truthy check is enough:

```tsx
{errorMessage && (
    <Alert color={'red'} title={'Ошибка'}>
        {errorMessage}
    </Alert>
)}
```

Use strict checks only when empty string, zero, false, or null have different UI meaning.

## Component Formatting

Use short JSDoc comments in Russian for exported components, exported managers, exported API clients, exported enums, enum members, exported models/types, public functions, props types, and each prop. Comment public manager methods when they are exported individually or when the manager is exported as an object. Do not use XML-style comments in frontend code.

For local component props, prefer `type Props`.

Keep comments concise and business/UI oriented.

Do not comment obvious private local variables.

Example:

```tsx
/** Страница логов терминалов */
export const TerminalLogsPage = memo(() => {
    ...
})
```

Props:

```tsx
/** Пропсы заголовка страницы логов терминала */
type Props = {
    /** Название терминала */
    name?: string

    /** IP-адрес терминала */
    ipAddress?: string
}
```

Prefer `memo` for pages and widgets when appropriate.

For memoized components with props, use generic `memo<Props>` style. Keep destructuring multiline when there are several props and keep the props type near the component.

Preferred:

```tsx
export const TasksList = memo<Props>(({
    tasks,
    isLoading,
    busyTaskId,
    onStatusChange,
    onDelete
}) => (
    ...
))
```

Avoid:

```tsx
export const TasksList = memo(({ tasks, isLoading, busyTaskId, onStatusChange, onDelete }: Props) => (
    ...
))
```

Props for local components:

```tsx
/** Пропсы формы создания задачи */
type Props = {
    /** Признак выполнения запроса */
    isLoading: boolean

    /** Обработчик создания задачи */
    onCreate: (request: CreateTaskRequest) => Promise<void>
}
```

## Vite

Use `vite.config.ts`.

Keep aliases synchronized between `vite.config.ts` and `tsconfig.path.json`.

Default aliases:

- `@Src`
- `@App`
- `@Pages`
- `@Dialogs`
- `@Widgets`
- `@Entities`
- `@Layout`
- `@Ui`
- `@Svg`
- `@Shared`

Default development port: `3002`.

## TypeScript Configuration

Use `tsconfig.json` and `tsconfig.path.json`.

`tsconfig.json` extends `tsconfig.path.json`.

Keep strict mode enabled.

## Package Json

Default scripts:

```json
{
  "scripts": {
    "start": "vite",
    "build": "vite build",
    "clear": "npm cache clean --force && rimraf ./node_modules",
    "lint": "eslint ./src/**/*.*",
    "lint:fix": "eslint --fix ./src/**/*.*",
    "lint:error": "eslint ./src/**/*.* --quiet",
    "check-types": "tsc --project tsconfig.json --noEmit"
  }
}
```

Use latest stable package versions.

Reuse the existing package manager.

## Existing Projects

Before making changes:

- inspect project structure;
- reuse existing architecture;
- reuse aliases;
- reuse routing;
- reuse stores;
- reuse API layer;
- do not introduce parallel implementations.

Before creating new widgets, hooks, stores, managers, API clients, converters, navigation helpers, model folders, enum folders, page-local components, or UI abstractions, inspect the existing folder style, search for an existing implementation, reuse existing conventions, and extend existing code instead of creating a parallel abstraction.

Before creating shared API helpers for ProblemDetails, ApiResult, response parsing, or fetch wrappers, search the project and reuse existing HTTP infrastructure.

Prefer the project style over this default when the existing project is consistent.

## Final Validation

Before completing a task, verify:

- type check passes;
- lint passes;
- build passes;
- aliases are synchronized;
- no Vite demo artifacts remain;
- Mantine is used where appropriate;
- no unnecessary SCSS was introduced;
- public layer imports use `index.ts`;
- page folder `index.ts` files are not introduced unless a page-level public API is intentional;
- widgets are used only when reused or cross-page composition is justified;
- page-specific UI is split into local page components, not unnecessary widgets;
- entities do not contain UI unless the existing project style allows it;
- models, enums, converters, API, and store files are in correct folders;
- API-only request/response types live in the `api` folder;
- no unnecessary `index.ts` was created inside `api`;
- file names match exported names;
- all functions are arrow functions unless there is a clear exception;
- named async functions use the `Async` suffix;
- inline lambdas passed to hooks/callbacks are not renamed just to add `Async`;
- promises returned inside async functions are awaited;
- no floating promises were introduced unless intentionally marked and allowed by ESLint;
- `useState` is used only for local component UI state;
- shared or page-level state uses Zustand store/manager when needed;
- managers may orchestrate API calls and store updates, but API clients remain thin;
- Zustand store subscriptions use `useStoreWithEqualityFn` with inline selectors for simple reads;
- selector functions are extracted only when reused or non-trivial;
- existing `useLifeCycle` signature is respected;
- `void` is not added before async calls unless required;
- `usePromiseAwaiter` is reused for local promise pending state;
- memoization is used where appropriate;
- `useEffect` is not used for ordinary use cases;
- exported components, public functions, public types, and props are documented in Russian;
- exported enums and enum members are documented in Russian;
- exported managers and API clients are documented in Russian;
- JSX conditional rendering is not overly strict without reason;
- memoized components with props use `memo<Props>`;
- hooks were introduced only when React features are required;
- API clients are not implemented as hooks;
- React Query was not introduced by default;
- HTTP response parsing is centralized in shared API infrastructure;
- reactive Zustand reads are used only when re-rendering is required;
- layer dependency direction is preserved;
- project style remains consistent.
