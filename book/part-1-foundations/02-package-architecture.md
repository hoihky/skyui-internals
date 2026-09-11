---
title: Chapter 2 — Package Architecture
order: 2
---

# Chapter 2: SkyUI Package Architecture

A UI toolkit that ships as one giant assembly becomes hard to version, hard to test, and hard to adopt incrementally. SkyUI splits responsibilities across NuGet packages so a minimal app can reference only essentials, while a data-heavy admin tool can add grids and filter editors without pulling diagram rendering.

This chapter explains each package, what it contains, how dependencies flow, and the architectural principles that keep the split maintainable.

## Package Dependency Graph

```
                    ┌─────────────────┐
                    │  SkyUI.Themes.Sky │
                    └────────┬────────┘
                             │ references
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
   ┌──────────┐        ┌──────────┐        ┌──────────┐
   │ SkyUI    │        │SkyUI.Fonts│       │SkyUI.Core │
   └────┬─────┘        └──────────┘        └──────────┘
        │
   ┌────┴─────┐
   ▼          ▼
SkyUI.Core  SkyUI.Icons

Optional (reference SkyUI, not each other):
   SkyUI.Data          SkyUI.Diagram
```

| Package | Target | Role |
|---------|--------|------|
| `SkyUI.Core` | `net10.0` | Tokens, motion, theme attached properties, applicators |
| `SkyUI.Fonts` | `net10.0` | Inter + Noto Sans SC typography resource dictionaries |
| `SkyUI.Icons` | `net10.0` | `SkyIcon`, `SkyFontIcon`, `SkyIconKind` geometry catalog |
| `SkyUI` | `net10.0` | Essentials controls — forms, layout, nav, shell, feedback |
| `SkyUI.Themes.Sky` | `net10.0` | ContentFirstDark: ControlThemes, styles, primitives preset |
| `SkyUI.Data` | `net10.0` | Virtual data grid, filter editor |
| `SkyUI.Diagram` | `net10.0` | `DiagramSurface` canvas control |

Nothing in `SkyUI.Core` references controls. That directionality prevents circular references: tokens are defined once and consumed everywhere.

## SkyUI.Core: Infrastructure Without UI Chrome

`SkyUI.Core` answers: *how does every control know what "accent color" or "16px padding" means, and how can an app change those at runtime?*

### SkyTokenKeys

Stable string constants for resource dictionary keys:

```csharp
public static class SkyTokenKeys
{
    public static class Brush
    {
        public const string Accent = "SkyAccentBrush";
        public const string Surface = "SkySurfaceBrush";
        // ...
    }
    public static class Spacing
    {
        public const string Space8Px = "SkySpace8Px";
        public const string Space16Px = "SkySpace16Px";
        // ...
    }
}
```

C# code and XAML both reference the same keys. When a token name changes, you update one constant and the compiler finds stale usages.

### SkyThemeProperties (Attached)

```xml
<Application sky:SkyThemeProperties.AccentOverride="#1ED760"
             sky:SkyThemeProperties.Density="Compact">
```

Changed handlers invoke:

- `SkyAccentOverrideApplicator` — injects overridden accent brushes into app resources
- `SkyDensityApplicator` — swaps comfortable vs compact spacing/height tokens

Because controls use `{DynamicResource SkyAccentBrush}`, accent override propagates without recompiling templates.

### SkyMotionAnimator

Shared enter/exit animations for `SkyDialogHost`, `SkySnackbarHost`, `SkySheetHost`, and `SkyNavigationView` content transitions. Centralizing motion prevents each overlay from inventing different durations and easing curves.

## SkyUI: Control Organization

Controls live under `src/SkyUI/Controls/` grouped by **domain**, not by inheritance:

```
Controls/
  Accordion/       SkyAccordion, SkyAccordionItem
  CheckedListBox/  CheckedListBox + adapter types
  Data/            SkyPagination*, SkyKpiTile, SkyLoadingOverlay
  Feedback/        SkyAlert, SkyDialogHost, SkySnackbarHost, ...
  Forms/           SkyFormField, SkyAutocomplete, SkyRadioGroup, ...
  Layout/          SkyCard, SkyDivider, SkyResponsiveGrid, SkyExpander
  Lists/           SkyVirtualTreeView
  Menus/           SkyMenuBar, SkyContextMenu, SkyMenuFlyout
  Mobile/          SkySafeArea, SkyActionSheet, SkyTouchTarget
  Navigation/      SkyNavigationView, SkyTabView, SkyBreadcrumb
  Pickers/         SkyDatePicker, SkyTimePicker, SkyCalendar
  Primitives/      SkyButtonProperties, SkyTooltip, SkyPopover
  Shell/           SkySplitView, SkyListDetailPage, SkyCommandBar, ...
  Timeline/        VideoTimeline
```

This structure maps to how product teams think about UI ("we need a form field" → look in `Forms/`) and mirrors the theme folder layout in `SkyUI.Themes.Sky/Themes/SkyDark/Controls/`.

## Four Control Inheritance Strategies

SkyUI consistently chooses one of four approaches:

### 1. TemplatedControl — custom visual + behavior

Used when the control needs named template parts, non-trivial interaction, or state machines.

Examples: `SkyFormField`, `SkyDialogHost`, `CheckedListBox`, `Chip`, `SkyNavigationView`.

### 2. ContentControl / ItemsControl — composition containers

Used when structure is "host children" with optional chrome.

Examples: `SkyCard` (ContentControl), `SkyAccordion` (ItemsControl), `Badge` (ContentControl).

### 3. Primitive subclass — thin wrapper

Used when Avalonia's built-in control already does the work; SkyUI only adds style classes and small API extensions.

```csharp
public class SkySearchBox : TextBox
{
    public SkySearchBox() => Classes.Add("sky-search");
}

public class SkyDatePicker : CalendarDatePicker
{
    public SkyDatePicker()
    {
        Classes.Add("sky");
        Classes.Add("sky-date-picker");
    }
}
```

### 4. Panel subclass — custom layout/rendering

Used when template-based layout is insufficient.

Example: `DiagramSurface` positions nodes at absolute coordinates and draws edges in `Render`.

## Dual Consumption: Named Controls vs Styled Primitives

SkyUI deliberately supports two authoring styles:

**Named controls** for behavior-rich components:

```xml
<SkyFormField Label="Email">
  <TextBox Watermark="you@example.com" />
</SkyFormField>
```

**Styled primitives** for simple visual variants:

```xml
<Button Classes="sky sky-primary" Content="Save" />
<ComboBox Classes="sky" ItemsSource="{Binding Items}" />
```

`SkyThemeClasses` centralizes class name strings:

```csharp
public const string Sky = "sky";
public const string Primary = "sky-primary";
public const string Outlined = "sky-outlined";
public const string Search = "sky-search";
```

The dual model avoids an explosion of `SkyPrimaryButton`, `SkyOutlinedButton` subclasses. Subclass only when behavior differs, not when color differs.

## Theme Package: Behavior/Appearance Split

| Layer | Location | Contains |
|-------|----------|----------|
| Behavior | `SkyUI` C# | Properties, events, validation, adapters |
| Appearance | `SkyUI.Themes.Sky` XAML | ControlTheme templates, Style selectors |

Bootstrapping in consumer `App.axaml`:

```xml
<Application.Styles>
  <FluentTheme />
  <StyleInclude Source="avares://SkyUI.Themes.Sky/Themes/SkyTheme.axaml" />
</Application.Styles>
```

For data controls, include the extension theme:

```xml
<StyleInclude Source="avares://SkyUI.Data/Themes/SkyTheme.WithData.axaml" />
```

### Two-File Theme Convention

| File | Purpose |
|------|---------|
| `Forms.axaml` | `ControlTheme` keyed by `{x:Type ...}` |
| `Forms.Styles.axaml` | Pseudo-class and variant `Style` selectors |

Primitives that style Avalonia types without subclassing live in `SkyPreset.Primitives.axaml`.

## Optional Packages in Detail

### SkyUI.Data

- `SkyVirtualDataGrid` — windowed row access via `IVirtualGridDataSource`
- `FilterEditor` — hierarchical filter tree with SQL preview
- Separate theme merge: apps without grids do not load grid templates

### SkyUI.Diagram

- `DiagramSurface` : `Panel`
- Pluggable `IEdgePathComputer`, `IDiagramNodePresenterFactory`, `IDiagramSceneHitTester`
- Demonstrates SkyUI beyond templated controls

## XAML Namespace

`XmlnsDefinition` maps CLR namespaces to `https://skyui.dev`:

```xml
xmlns:sky="https://skyui.dev"
```

Types from `SkyUI`, `SkyUI.Core`, `SkyUI.Data`, and `SkyUI.Diagram` resolve through one XML namespace prefix.

## Samples, Demo, and Tests

| Project | Purpose |
|---------|---------|
| `SkyUI.Demo` | Desktop gallery — one page per control category |
| `samples/SettingsApp` | Navigation shell, theme/density switching |
| `samples/CrudListDetail` | List-detail + virtual grid MVVM |
| `tests/SkyUI.UnitTests` | Token math, pagination, validators |
| `tests/SkyUI.HeadlessTests` | Template apply, binding, interaction without GPU |

Expected workflow for a new control: implement C# → add theme → demo page → headless test.

## Design Principles

1. **Separation of behavior and appearance** — C# never sets `Background = Brushes.Gray`
2. **Progressive disclosure** — optional Data/Diagram packages
3. **Token-driven visuals** — semantic brush names, 8px spacing scale
4. **Composition over monoliths** — `SkyListDetailPage` assembles shell primitives
5. **Pluggable adapters** — CheckedListBox, VirtualGrid, FilterEditor, Diagram
6. **Consistent overlay pattern** — host at root + static facade + motion animator

## Summary

SkyUI's package boundaries encode architectural intent. Core owns tokens and motion; Essentials owns controls; Themes owns visuals; Data and Diagram own specialized domains. When you add a feature, place it in the shallowest package that can express it — do not put grid logic in `SkyUI` if only data apps need it.

Next: how design tokens connect C# constants to theme XAML.
