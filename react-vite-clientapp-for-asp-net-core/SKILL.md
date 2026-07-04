---
name: react-vite-clientapp-for-asp-net-core
description: Add or update a React + TypeScript + Vite frontend hosted from an ASP.NET Core API project. Use when a .NET backend needs a ClientApp under src/{ProjectName}.Api/ClientApp, ASP.NET Core SPA hosting with Proggmatic.SpaServices.Vite, Vite dev-server proxying, or production SPA file serving from the API project.
---

# React Vite ClientApp For ASP.NET Core

## Purpose

Use this skill when a project needs a React frontend hosted from an ASP.NET Core API project.

The frontend must live inside the API project:

```text
src/{ProjectName}.Api/ClientApp
```

Keep `ClientApp` visible in Rider and include it as ordinary project content when needed.

## Existing Projects

When React/Vite already exists:

- inspect the existing package manager and config first;
- reuse existing conventions;
- do not recreate the app;
- update only what is required;
- do not introduce a second frontend structure;
- do not rewrite the whole frontend.

Reuse the existing package manager when the project already uses one.

## Modern Tooling

Always prefer the latest stable official tooling.

When creating a new `ClientApp`:

- use the latest stable React;
- use the latest stable React DOM;
- use the latest stable TypeScript;
- use the latest stable Vite;
- use the latest stable `@vitejs/plugin-react`;
- use the latest stable ESLint packages.

Do not intentionally install outdated package versions.

Do not hardcode package versions unless the existing project already uses fixed versions.

Never downgrade package versions. If a package already exists and a newer compatible stable version is available, prefer the newer version unless the existing project has an explicit compatibility constraint.

When creating a new React project, use the official Vite React TypeScript template.

Prefer:

```bash
yarn create vite ClientApp --template react-ts
```

Use the current official equivalent if the Vite CLI changes.

Install all dependencies required by the current official template. Then install only project-specific packages that are missing, such as:

- `vite-plugin-eslint`
- `vite-plugin-svgr`
- `rimraf`
- `sass` when SCSS is used

Avoid manually recreating the Vite template.

## ASP.NET Core SPA Hosting

Use `Proggmatic.SpaServices.Vite`.

Configure SPA hosting only in `{ProjectName}.Api`. Do not put frontend hosting logic into App, Domain, or Infrastructure.

SPA hosting must remain inside `Program.cs`. Do not move SPA hosting into extension methods unless explicitly requested.

In `Program.cs`, read `UiHosting` once before `UseSpa` and use this pattern:

```csharp
using Proggmatic.SpaServices.Vite;

var uiHosting = Environment.GetEnvironmentVariable("UiHosting");
ArgumentException.ThrowIfNullOrWhiteSpace(uiHosting, "Не задана переменная окружения UiHosting");

if (uiHosting == "SpaFiles")
    app.UseSpaStaticFiles();

app.UseSpa(spa =>
{
    if (uiHosting == "SpaFiles")
        spa.Options.SourcePath = "ClientApp/build";

    if (uiHosting == "ViteDevServer")
    {
        spa.Options.SourcePath = "ClientApp";
        spa.Options.DevServerPort = 3002;

        spa.UseProxyToSpaDevelopmentServer("http://localhost:3002");
        spa.UseViteServer("start");
    }
});
```

Rules:

- `UiHosting=SpaFiles` means ASP.NET Core serves built frontend files from `ClientApp/build`.
- `UiHosting=ViteDevServer` means ASP.NET Core proxies to Vite on port `3002`.
- Read `UiHosting` once before `UseSpa`.
- Do not hardcode SPA hosting mode.
- Fail fast when `UiHosting` is missing.
- If `UiHosting == "SpaFiles"`, call `app.UseSpaStaticFiles()` before `app.UseSpa(...)`.
- If `UiHosting == "SpaFiles"`, set `spa.Options.SourcePath = "ClientApp/build"`.
- If `UiHosting == "ViteDevServer"`, set `spa.Options.SourcePath = "ClientApp"` and `spa.Options.DevServerPort = 3002`.
- For `ViteDevServer`, call `spa.UseProxyToSpaDevelopmentServer("http://localhost:3002")` and `spa.UseViteServer("start")`.
- Do not call `UseSpaStaticFiles()` for `ViteDevServer`.
- Keep SPA setup in `{ProjectName}.Api`.
- Support both production and development hosting.

## Launch Settings

When integrating `ClientApp` with ASP.NET Core, update `launchSettings.json`.

Development profiles should contain:

```json
"environmentVariables": {
    "UiHosting": "ViteDevServer"
}
```

Rules:

- Automatically add the `UiHosting` environment variable if it is missing.
- Preserve all existing environment variables.
- Do not overwrite unrelated settings.
- Only add `UiHosting` when the ASP.NET Core project hosts a React `ClientApp`.

## Creating ClientApp

If `ClientApp` does not exist:

- create React + TypeScript + Vite app inside `ClientApp` using the official Vite React TypeScript template;
- use `yarn` unless the existing project already uses another package manager;
- remove default Vite demo clutter;
- create a minimal production-ready structure;
- ensure `yarn build` outputs to `ClientApp/build`;
- ensure the dev server runs on port `3002`;
- prefer `yarn.lock` for new `ClientApp`;
- do not generate `package-lock.json` when using `yarn`.

Preferred structure:

```text
ClientApp/
  src/
    app/
    assets/
      images/
        svg/
    dialogs/
    entities/
    layout/
    pages/
    shared/
      ui/
    widgets/
    main.tsx
    vite-env.d.ts
  index.html
  package.json
  tsconfig.json
  vite.config.ts
```

## Vite Config

Use `vite.config.ts`.

Base config:

```ts
import { AliasOptions, ConfigEnv, defineConfig, PluginOption, splitVendorChunkPlugin } from 'vite'
import react from '@vitejs/plugin-react'
import svgr from 'vite-plugin-svgr'
import path from 'node:path'
import eslintPlugin from 'vite-plugin-eslint'

export default defineConfig((env) => ({
    plugins: getPlugins(env),
    build: {
        outDir: './build',
        minify: true,
        cssCodeSplit: true
    },
    server: {
        port: 3002
    },
    esbuild: {
        sourcemap: false,
        treeShaking: true
    },
    resolve: {
        alias: getAliases()
    },
    optimizeDeps: {
        exclude: ['xlsx']
    }
}))

function getPlugins(configEnv: ConfigEnv): PluginOption[] {
    return [
        svgr(),
        react(),
        eslintPlugin({
            emitWarning: configEnv.command === 'serve',
            useEslintrc: true,
            cache: false,
            emitError: true,
            failOnError: true
        }),
        splitVendorChunkPlugin()
    ]
}

function getAliases(): AliasOptions {
    return {
        '@Src': path.resolve(__dirname, 'src/'),
        '@App': path.resolve(__dirname, 'src/app/'),
        '@Dialogs': path.resolve(__dirname, 'src/dialogs/'),
        '@Pages': path.resolve(__dirname, 'src/pages/'),
        '@Svg': path.resolve(__dirname, 'src/assets/images/svg/'),
        '@Ui': path.resolve(__dirname, 'src/shared/ui/'),
        '@Entities': path.resolve(__dirname, 'src/entities/'),
        '@Widgets': path.resolve(__dirname, 'src/widgets/'),
        '@Layout': path.resolve(__dirname, 'src/layout/'),
        '@Shared': path.resolve(__dirname, 'src/shared/')
    }
}
```

Use camelCase for local TypeScript functions such as `getPlugins` and `getAliases`.

When introducing path aliases, always update both:

- `vite.config.ts`;
- `tsconfig.json`.

Keep aliases synchronized between Vite and TypeScript.

## Vite Environment Types

Create or update `ClientApp/src/vite-env.d.ts`.

It must support:

- `vite/client`;
- `gif`, `jpg`, `jpeg`, `png`;
- `svg` as URL and `ReactComponent`;
- CSS modules;
- SCSS modules.

Use this content as the baseline:

```ts
/// <reference types="vite/client" />

declare module '*.gif' {
    const src: string
    export default src
}

declare module '*.jpg' {
    const src: string
    export default src
}

declare module '*.jpeg' {
    const src: string
    export default src
}

declare module '*.png' {
    const src: string
    export default src
}

declare module '*.svg' {
    import * as React from 'react'
    export const ReactComponent: React.FunctionComponent<React.SVGProps<SVGSVGElement> & { title?: string }>
    const src: string
    export default src
}

declare module '*.module.css' {
    const classes: { readonly [key: string]: string }
    export default classes
}

declare module '*.module.scss' {
    const classes: { readonly [key: string]: string }
    export default classes
}
```

## TypeScript And React Rules

Use:

- React;
- TypeScript;
- Vite;
- functional components;
- CSS Modules / SCSS Modules;
- path aliases from `vite.config.ts`.

Avoid:

- class components;
- `any`;
- inline styles by default;
- huge components;
- unnecessary `useEffect`;
- unnecessary `useMemo` / `useCallback`;
- default Vite demo code.

Prefer:

- explicit types;
- small components;
- feature/page-oriented folders;
- readable names;
- stable imports through aliases.

## Styling

Use SCSS Modules by default.

Prefer `.module.scss`.

Avoid plain CSS unless explicitly requested.

## Package Manager

Use `yarn` instead of `npm` for new React/Vite `ClientApp`.

Rules:

- If an existing project already uses another package manager, reuse the existing package manager.
- Do not mix `npm`, `yarn`, and `pnpm` in one `ClientApp`.
- Prefer `yarn.lock` for new `ClientApp`.
- Do not generate `package-lock.json` when using `yarn`.
- Never downgrade package versions.
- If a package already exists and a newer compatible stable version is available, prefer the newer version unless the existing project has an explicit compatibility constraint.

## Package Json Scripts

Use this default scripts section:

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

Notes:

- Keep `start` as `vite`.
- Keep `build` as `vite build`.
- Do not use `tsc && vite build` in `build`; type checking belongs to `check-types`.
- Add `rimraf` as a dev dependency because `clear` uses it.

Install runtime dependencies:

- `react`
- `react-dom`

Install dev dependencies:

- `typescript`
- `vite`
- `@vitejs/plugin-react`
- `vite-plugin-svgr`
- `vite-plugin-eslint`
- `eslint`
- `@types/react`
- `@types/react-dom`
- `rimraf`
- `sass` when SCSS modules are used

## ASP.NET Project File

Ensure `ClientApp` is not accidentally excluded from Rider/project view.

If needed, add project file entries so Rider displays `ClientApp`, but do not include `node_modules`, `build`, `dist`, or Vite cache as compile/content artifacts.

Exclude from project output and source control:

- `ClientApp/node_modules`
- `ClientApp/build`
- `ClientApp/dist`
- `ClientApp/.vite`

## Gitignore

Ensure `.gitignore` excludes:

```text
node_modules/
ClientApp/node_modules/
ClientApp/build/
ClientApp/dist/
ClientApp/.vite/
```

Do not ignore source files inside `ClientApp/src`.

## Validation

After adding or updating `ClientApp`, run:

```bash
yarn install
yarn check-types
yarn lint
yarn build
dotnet build
```

If tests exist, run:

```bash
dotnet test
```

Do not finish until every command succeeds.

Do not leave the project in a non-building state.

## Completion Checklist

Before completing the task, verify that:

- `launchSettings.json` contains `UiHosting=ViteDevServer`;
- `Program.cs` supports both `SpaFiles` and `ViteDevServer`;
- `ClientApp` builds successfully;
- TypeScript compiles successfully;
- ESLint passes;
- `dotnet build` succeeds;
- aliases are synchronized between `vite.config.ts` and `tsconfig.json`;
- no obsolete package versions were intentionally installed.

## Completion Output

When completing a task, summarize:

- where `ClientApp` was created;
- how SPA hosting is configured;
- which `UiHosting` values are supported;
- which commands to run for frontend dev/build;
- whether `dotnet build`, `yarn build`, `yarn check-types`, `yarn lint`, and tests passed.
