# Blazor UI/UX Engineering Guide

`ui-ux-pro-max` treats Blazor as a first-class implementation stack (`--stack blazor`)
that sits on top of the same visual design intelligence used for every other
framework. Blazor is not a visual style: styles, palettes, typography, UX rules and
anti-patterns are unchanged. What the Blazor stack adds is how to implement that
design correctly in the Microsoft .NET ecosystem.

```bash
python3 .claude/skills/ui-ux-pro-max/scripts/search.py "<query>" --stack blazor
python3 .claude/skills/ui-ux-pro-max/scripts/search.py "<query>" --design-system --framework blazor
python3 .claude/skills/ui-ux-pro-max/scripts/search.py "<query>" --design-system --tokens css
```

## 1. Application profiles

The first step of every Blazor task is identifying the model from the project, not
from the prompt. Search returns guidance tagged with `Applies To` so a server app
never receives WebAssembly advice, and a MAUI app never receives render modes.

| Model | How to recognize it | Design and engineering rules |
|-------|---------------------|------------------------------|
| **Blazor Web App** | .NET 8+ `Microsoft.NET.Sdk.Web` project with `Program.cs`, `App.razor`, `AddRazorComponents()` / `MapRazorComponents<App>()` | Static SSR by default; choose Interactive Server, Interactive WebAssembly or Interactive Auto per component/page with `@rendermode`; plan for prerendering and double initialization |
| **Blazor WebAssembly (standalone)** | `wwwroot/index.html`, `WebAssemblyHostBuilder`, no server render modes | Client rendering only; budget the download, design a first-load state, use browser storage and offline-aware UX |
| **Blazor Server (legacy, pre-.NET 8)** | `Startup.cs`, `_Host.cshtml`, `MapBlazorHub()` | SignalR circuit state, reconnection and connection-loss feedback, server-owned long-running work; do not add Web App render modes during a refresh |
| **.NET MAUI Blazor Hybrid** | MAUI `*.csproj` with `BlazorWebView` | Android, iOS, Windows, macOS; WebView rendering, safe areas, touch, soft keyboard, offline data, native navigation. **No render modes, no circuits** |

### Render modes (Blazor Web App, .NET 8+)

- **Static SSR** — no circuit, no WebAssembly payload. Use for marketing, docs,
  catalogs and any read-only page.
- **Interactive Server** — component state on the server over SignalR. Best for
  internal dashboards and admin workspaces; design for reconnection.
- **Interactive WebAssembly** — runs in the browser. Justify it with offline work,
  client-side computation, or server-free hosting; account for startup cost.
- **Interactive Auto** — server first, WebAssembly on later visits. Test both
  phases and keep client-safe service abstractions.
- **Prerendering** — interactive components are prerendered by default; do not
  call `IJSRuntime` or mutate the DOM during prerender.

Set the render mode on the smallest interactive boundary, not on `<Routes>` by
default:

```razor
@page "/dashboard"
@rendermode InteractiveServer
```

## 2. Mandatory workflow for an existing application

1. Inspect the solution, `*.csproj` `TargetFramework`, `Program.cs`, `App.razor`,
   `Routes.razor`, `_Imports.razor`.
2. Identify the app model, hosting model and render mode.
3. Read `PackageReference` entries and `wwwroot` styles to find the CSS framework
   and component library.
4. Read existing tokens, `app.css`, `.razor.css` files, layouts and shared
   components; reuse them.
5. Decide whether the surface is a public site or an internal business workspace.
6. Search the visual domains, then the `blazor` stack; generate a design system for
   new or system-wide work.
7. Implement with native Razor and the installed dependencies.
8. Verify responsiveness, accessibility, interaction, RTL and render-mode behavior.
9. Run `dotnet build` and the project's tests.
10. Preserve business logic, APIs, authentication, authorization and component
    contracts; prefer incremental changes to rewrites.

## 3. Component architecture

- One component per file, with `.razor.cs` for logic and `.razor.css` for styles.
- `[Parameter]` values are immutable inputs; derive local state in
  `OnParametersSet`/`OnParametersSetAsync`.
- Child-to-parent communication uses `EventCallback` / `EventCallback<T>`.
- Cross-cutting theme, density and direction travel through `CascadingValue`.
- Generic components use `@typeparam` and `RenderFragment<TItem>`.
- Use `@key` in every repeated list of components.
- Inject narrow interfaces with `@inject`; register services in `Program.cs`.
- Implement `IDisposable`/`IAsyncDisposable` for subscriptions, timers and JS
  module references.

```razor
@typeparam TItem
@foreach (var item in Items)
{
    <Row @key="RowKey(item)" Item="item" OnSelect="OnSelect" />
}
```

## 4. Razor syntax and lifecycle

Use native Blazor syntax only — `@page`, `@using`, `@inject`, `@bind`/`@bind-Value`,
`@onclick`, `@onchange`, `@oninput`, `@if`, `@foreach`, `@key`, `@ref`,
`@typeparam`, `@code`, `@rendermode`. Never emit React hooks, JSX, Vue directives or
Angular templates inside `.razor` files.

Lifecycle rules:

- `OnInitialized`/`OnInitializedAsync` — first initialization; may run twice with
  prerendering.
- `OnParametersSet`/`OnParametersSetAsync` — parameter changes and derived state.
- `OnAfterRender`/`OnAfterRenderAsync` — DOM and JavaScript interop, guarded by
  `firstRender` for setup.
- `StateHasChanged` — only when state changes outside an event handler.
- `ShouldRender` — opt out of expensive rerenders.
- `IDisposable`/`IAsyncDisposable` — release event handlers and JS references.

## 5. Styling, tokens and CSS isolation

Inspect the project before recommending a CSS framework. Blazor templates ship
Bootstrap; enterprise projects often add MudBlazor or Radzen; never assume
Tailwind, Bootstrap or anything else is installed.

- Global styles and design tokens: `wwwroot/css/tokens.css` + `wwwroot/app.css`.
- Component styles: `Component.razor.css` (CSS isolation). Use `::deep` only for
  child markup you own.
- Consume tokens through CSS custom properties, never repeated hex values.
- Themes switch with a `data-theme` attribute on `<html>` and token overrides.
- Use Flexbox, Grid and container queries; define breakpoints once.
- Respect `prefers-reduced-motion`, `forced-colors` and high contrast.

```css
/* Dashboard.razor.css */
.dashboard {
    display: grid;
    gap: var(--space-md, 16px);
    padding-inline: var(--space-lg, 24px);
    color: var(--color-foreground);
}

.card {
    background: var(--color-card);
    border: 1px solid var(--color-border);
    border-inline-start: 3px solid var(--color-primary);
    border-radius: 12px;
}
```

`--tokens css` emits the palette roles (`--color-*`) and the font stacks
(`--font-*`). Spacing steps (`--space-*`) are exported only when a density dial
is set, hence the fallbacks above. Radii are not exported at all: keep them as
values here or define `--radius-*` yourself in `app.css`.

Link the token file once, before `app.css`:

```html
<link rel="stylesheet" href="css/tokens.css" />
```

## 6. Forms and validation

Use `EditForm` with a model, the built-in input components, and the validation
system:

```razor
<EditForm Model="_model" OnValidSubmit="SaveAsync">
    <DataAnnotationsValidator />
    <ValidationSummary />

    <label for="email">Email</label>
    <InputText id="email" @bind-Value="_model.Email" aria-invalid="@context.GetValidationMessages().Any()" />
    <ValidationMessage For="() => _model.Email" />

    <button type="submit" disabled="@_isSaving">
        @(_isSaving ? "Saving..." : "Save")
    </button>
</EditForm>
```

- `InputText`, `InputNumber`, `InputSelect`, `InputCheckbox`, `InputDate`.
- `DataAnnotationsValidator`, `ValidationMessage`, `ValidationSummary`; custom
  rules via `ValidationAttribute` or `IValidatableObject`.
- Disable and label busy controls during submit; never allow double submits.
- Preserve entered values after a failed save; show a retryable error.
- `NavigationLock` guards unsaved changes for internal navigation.
- Associate errors with fields (`aria-invalid`, message elements) and move focus to
  the summary on submit.

## 7. Navigation and application structure

- `Router` + `RouteView` + `MainLayout`; nested layouts via `@layout`.
- `NavLink` with `NavLinkMatch.All`/`Prefix` for active state.
- `NavigationManager` for programmatic navigation (`forceLoad` only for full
  reloads).
- `AuthorizeView`/`AuthorizeRouteView` for role-based navigation; enforce the same
  rule on the server.
- Provide `NotFound` and `NotAuthorized` experiences inside the shell.
- Breadcrumbs for deep hierarchies; collapsible drawer navigation with focus
  management below 1024px.
- Static SSR pages navigate with full requests — do not assume an interactive
  circuit on every page.

## 8. Component libraries

Detect the installed library from `.csproj`; never assume one, never add a second
library for a single component, and verify parameters against the installed
version.

| Library | Theming entry point |
|---------|---------------------|
| MudBlazor | `MudThemeProvider` + `MudTheme` (`PaletteLight`/`PaletteDark`) — fill the palette that matches the generated mode: `PaletteDark` for a dark system, `PaletteLight` for a light one |
| Radzen Blazor | `RadzenTheme` + Radzen CSS variables (`--rz-primary`, ...) |
| Microsoft Fluent UI Blazor | `FluentDesignTheme` and Fluent design tokens |
| Syncfusion Blazor | vendor theme package for the installed version |
| Telerik UI for Blazor | vendor theme / `TelerikRootComponent` setup |
| DevExpress Blazor | vendor theme stylesheet for the installed version |
| Blazorise | provider registered with one design system |

Native HTML elements styled by the token system remain a valid option.

## 9. Enterprise application design

Internal business applications differ from public sites:

- Workspaces, not landing pages: toolbar, filters, grid, detail panel.
- Information-dense dashboards with clear hierarchy; one primary metric per region.
- Data grids with sorting, filtering, server-side paging, saved views and removable
  filter chips.
- Bulk actions that display scope and require confirmation.
- Master-detail panels instead of losing filter state on navigation.
- Status with text + icon + color, never color alone.
- Role- and permission-aware navigation and actions, enforced server-side.
- Audit/activity context with localized timestamps.
- Loading skeletons, empty states and retryable error states for every data surface.
- Grids that adapt: column priority, card rows or horizontal scroll with a sticky
  key column — never a shrunk desktop grid.

## 10. Responsive design

Verify **375, 390, 768, 1024, 1280 and 1440px**. Touch targets are at least 44×44px
and never hover-only. Dialogs become full-screen sheets and sidebars become
overlays below the tablet breakpoint. Restore focus to the trigger on close.

## 11. Accessibility (WCAG 2.2 AA)

- Semantic HTML first; ARIA only to fill gaps.
- Visible focus (`:focus-visible`), focus not obscured by sticky headers.
- Keyboard operation for menus, comboboxes and grids; dialog focus trapping and
  restoration.
- Accessible validation: field association, `aria-invalid`, summary focus.
- Async feedback via `aria-live`/`role="status"`.
- No color-only status; no emoji as icons; use the project's SVG icon set.
- Reduced motion and forced-colors support.

## 12. JavaScript interop

Prefer C# and Blazor. Use `IJSRuntime` only for browser APIs or existing JS
libraries:

- Import ES modules with `IJSObjectReference` and dispose them.
- Call interop from `OnAfterRenderAsync(firstRender)`; never during prerender.
- Never mutate Blazor-owned DOM nodes; return measurements to C# instead.

## 13. Performance

- `Virtualize<TItem>` for large collections.
- `@key`, stable parameters and `ShouldRender` to avoid unnecessary rerenders.
- Match the render mode to cost and latency: Static SSR < Interactive Server <
  Interactive WebAssembly < Interactive Auto for payload.
- Stream async data and render skeletons per region.
- Trim and lazy-load WebAssembly assemblies; keep the client dependency graph small.
- Dispose event handlers and interop references to avoid circuit/page leaks.

## 14. RTL and internationalization

- `dir` and `lang` on `<html>`; never flip components with `scaleX(-1)`.
- Logical CSS properties (`padding-inline`, `margin-inline-start`,
  `border-inline-start`, `inset-inline`, `text-align: start`).
- Mirror directional icons (arrows, chevrons, progress) but not logos or media
  controls.
- Format dates and numbers with `CultureInfo.CurrentUICulture`; support Persian,
  Arabic and Hebrew calendars and digits.
- Isolate mixed-direction content with `<bdi>`.

```razor
<html lang="fa" dir="rtl">
```

## 15. Testing

- bUnit for component tests: rendering, parameters, events, validation, permission
  states, empty states.
- End-to-end tests for each render mode actually used; a passing `dotnet build` does
  not prove interactivity works.
- Run `dotnet build` and the project's existing test suite after the change.

## 16. Known limitations

- The dataset curates guidance, not generated application code; APIs must still be
  verified against the installed package version.
- Legacy pre-.NET 8 Blazor Server rows are intentionally limited to migration and
  circuit-model differences, not full legacy feature coverage.
- Component-library guidance covers theming and common components; it does not
  enumerate every vendor API.
- `--framework blazor` annotates a generated system; it does not scaffold a project,
  install packages, or modify a `.csproj`.
