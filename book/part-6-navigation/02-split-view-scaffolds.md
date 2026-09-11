---
title: Chapter 18 — SkySplitView and Page Scaffolds
order: 18
---

# Chapter 18: SkySplitView and Page Scaffolds

Individual shell controls solve one layout problem each. Page scaffolds compose them into complete page templates. This chapter covers `SkySplitView` as a master-detail primitive and `SkyListDetailPage` as a composite scaffold that assembles header, command bar, split view, status bar, and drawer into one control. Along the way we examine Avalonia template parts, pointer capture, pseudo-class state machines, and the composition patterns that keep large apps structurally consistent.

## SkySplitView: Master-Detail Layout

`SkySplitView` divides space between a pane (master) and a detail area. It supports inline and overlay display modes, left/right placement, and user-resizable pane width. It is the building block behind mail clients, file browsers, settings lists, and any screen where selecting an item in a narrow column reveals detail in a wider region.

### Avalonia Concept: TemplatedControl and Template Parts

`SkySplitView` inherits `TemplatedControl`, meaning its visual tree comes from a `ControlTheme` rather than inline XAML on the control class. Named elements in the theme are located via `[TemplatePart]` constants and `e.NameScope.Find` in `OnApplyTemplate`. If a part is missing, the control should degrade gracefully — null-check before attaching pointer handlers.

### Pseudo-Classes

```csharp
[PseudoClasses("open", "closed", "overlay", "inline", "left", "right", "resizing")]
public class SkySplitView : TemplatedControl
```

Seven pseudo-classes drive a matrix of visual states:

| Pseudo-class | Meaning |
|--------------|---------|
| `:open` / `:closed` | Pane is expanded or collapsed |
| `:inline` / `:overlay` | Pane shares layout space or floats over content |
| `:left` / `:right` | Pane placement |
| `:resizing` | User is dragging the resize grip |

Theme authors combine selectors for precise styling:

```xml
<Style Selector="SkySplitView:open:inline:left">
  <!-- inline left pane open: show grip, animate width -->
</Style>

<Style Selector="SkySplitView:open:overlay:right:resizing">
  <!-- suppress width transition while dragging -->
</Style>
```

When `:resizing` is active, disable CSS-like width transitions in the theme. Animating `OpenPaneLength` during drag feels laggy and fights the pointer.

### State Synchronization

```csharp
static SkySplitView()
{
    IsPaneOpenProperty.Changed.AddClassHandler<SkySplitView>((v, _) => v.ApplyPaneState());
    OpenPaneLengthProperty.Changed.AddClassHandler<SkySplitView>((v, _) => v.ApplyPaneState());
    DisplayModeProperty.Changed.AddClassHandler<SkySplitView>((v, _) => v.ApplyPaneState());
    PanePlacementProperty.Changed.AddClassHandler<SkySplitView>((v, _) => v.ApplyPaneState());
}

private void ApplyPaneState()
{
    PseudoClasses.Set(":open", IsPaneOpen);
    PseudoClasses.Set(":closed", !IsPaneOpen);
    PseudoClasses.Set(":inline", DisplayMode == SkySplitViewDisplayMode.Inline);
    PseudoClasses.Set(":overlay", DisplayMode == SkySplitViewDisplayMode.Overlay);
    PseudoClasses.Set(":left", PanePlacement == SkySplitViewPanePlacement.Left);
    PseudoClasses.Set(":right", PanePlacement == SkySplitViewPanePlacement.Right);

    UpdatePaneWidth();
    UpdateScrimVisibility();
}
```

Every property that affects layout calls into `ApplyPaneState`. This central method is the single place where pseudo-classes and pane dimensions are updated.

Centralizing state application avoids scattered `if` blocks that drift out of sync when a new property is added. When you add `CompactPaneLength` or `IsPanePinned`, register one more changed handler that calls `ApplyPaneState()` — do not patch pseudo-classes in individual property setters.

### Inline vs Overlay Semantics

| Mode | Layout | Best for |
|------|--------|----------|
| **Inline** | Pane consumes horizontal space; detail shrinks | Desktop, wide tablets, persistent lists |
| **Overlay** | Pane floats above detail; scrim dims background | Narrow widths, temporary filters, mobile |

Switch modes at runtime when the control width crosses a threshold — the same adaptive pattern as `SkyNavigationView`:

```csharp
protected override void OnPropertyChanged(AvaloniaPropertyChangedEventArgs change)
{
    base.OnPropertyChanged(change);
    if (change.Property == BoundsProperty && AdaptiveDisplayMode)
    {
        DisplayMode = Bounds.Width < OverlayBreakpoint
            ? SkySplitViewDisplayMode.Overlay
            : SkySplitViewDisplayMode.Inline;
    }
}
```

### Resize Grip

When `IsPaneResizable` is true, pointer events on `PART_ResizeGrip` track horizontal drag:

```csharp
private void OnResizeGripPointerPressed(object sender, PointerPressedEventArgs e)
{
    isResizing = true;
    PseudoClasses.Set(":resizing", true);
    resizeStartX = e.GetPosition(root).X;
    resizeStartLength = OpenPaneLength;
    e.Pointer.Capture(resizeGrip);
}

private void OnResizeGripPointerMoved(object sender, PointerEventArgs e)
{
    if (!isResizing) return;
    var delta = e.GetPosition(root).X - resizeStartX;
    var newLength = PanePlacement == SkySplitViewPanePlacement.Left
        ? resizeStartLength + delta
        : resizeStartLength - delta;
    OpenPaneLength = Math.Clamp(newLength, MinPaneLength, MaxPaneLength);
}

private void OnResizeGripPointerReleased(object sender, PointerReleasedEventArgs e)
{
    isResizing = false;
    PseudoClasses.Set(":resizing", false);
    e.Pointer.Capture(null);
}
```

`OpenPaneLength` is clamped between `MinPaneLength` and `MaxPaneLength`. The `:resizing` pseudo-class can show a different grip color or disable transitions during drag.

### Avalonia Concept: Pointer Capture

Pointer capture routes all pointer events to the capturing element until release, even when the cursor leaves the control bounds. Without capture, fast drags "drop" the grip when the pointer outruns the hit target. Always capture on press and release on `PointerReleased` — including when the pointer is captured elsewhere due to platform quirks.

Use coordinates relative to a stable root (`root` panel) rather than the grip itself. Grip position changes as `OpenPaneLength` updates; delta math becomes wrong if you measure from a moving origin.

### Overlay Mode and Scrim

In overlay mode, the pane floats above the detail content. `PART_Scrim` is a semi-transparent overlay behind the pane that dismisses the pane on click:

```csharp
private void UpdateScrimVisibility()
{
    if (scrim is null) return;
    scrim.IsVisible = DisplayMode == SkySplitViewDisplayMode.Overlay && IsPaneOpen;
}

private void OnScrimPointerPressed(object sender, PointerPressedEventArgs e)
{
    if (e.Source == scrim)
        IsPaneOpen = false;
}
```

The scrim uses `SkyScrimBrush` from the token layer.

Set scrim `ZIndex` below the pane panel but above detail content. If hit testing fails, verify `IsHitTestVisible="True"` on the scrim when visible and that detail content is not intercepting events at a higher z-order.

### Pane Content and Detail Content

`SkySplitView` exposes two content properties:

```csharp
public static readonly StyledProperty<object?> PaneContentProperty = ...;
public static readonly StyledProperty<object?> DetailContentProperty = ...;
```

The template hosts them in separate presenters. Binding from a page:

```xml
<SkySplitView IsPaneOpen="{Binding IsPaneOpen}"
              OpenPaneLength="{Binding PaneWidth, Mode=TwoWay}"
              DisplayMode="Inline"
              PanePlacement="Left">
  <SkySplitView.PaneContent>
    <CheckedListBox ItemsSource="{Binding Items}"
                    SelectedItem="{Binding SelectedItem, Mode=TwoWay}" />
  </SkySplitView.PaneContent>
  <SkySplitView.DetailContent>
    <local:ItemDetailView DataContext="{Binding SelectedItem}" />
  </SkySplitView.DetailContent>
</SkySplitView>
```

When `SelectedItem` is null, show an empty state in detail — do not collapse the pane unless product design requires it.

### Usage

```xml
<SkySplitView IsPaneOpen="True"
            OpenPaneLength="280"
            DisplayMode="Inline"
            PanePlacement="Left">
  <SkySplitView.PaneContent>
    <CheckedListBox ItemsSource="{Binding Items}" />
  </SkySplitView.PaneContent>
  <SkySplitView.DetailContent>
    <local:ItemDetailView DataContext="{Binding SelectedItem}" />
  </SkySplitView.DetailContent>
</SkySplitView>
```

### Debugging SkySplitView

| Issue | Check |
|-------|-------|
| Pane width stuck at zero | `IsPaneOpen` false or `:closed` styles set `Width=0` |
| Resize grip dead | `IsPaneResizable` false; handler not wired in `OnApplyTemplate` |
| Drag direction inverted | `PanePlacement` delta sign wrong for right-side panes |
| Scrim visible in inline mode | `UpdateScrimVisibility` guard on `DisplayMode` |
| Pointer stuck resizing | Missing `Capture(null)` on release/lost capture |

---

## SkyListDetailPage: Composition Scaffold

`SkyListDetailPage` is a high-level page template that composes multiple shell controls:

```
┌──────────────────────────────────────────────┐
│ SkyPageHeader (title, subtitle, back)      │
├──────────────────────────────────────────────┤
│ SkyCommandBar (primary/secondary commands)   │
├──────────────┬───────────────────────────────┤
│ SkySplitView │                               │
│  pane: list  │  detail: content              │
├──────────────┴───────────────────────────────┤
│ SkyStatusBar                                 │
└──────────────────────────────────────────────┘
  SkyDrawer (filter panel, slides in from right)
```

Rather than requiring every page to assemble these pieces manually, the scaffold exposes unified properties:

```csharp
public class SkyListDetailPage : TemplatedControl
{
    // Header
    public static readonly StyledProperty<string?> TitleProperty = ...;
    public static readonly StyledProperty<string?> SubtitleProperty = ...;
    public static readonly StyledProperty<ICommand?> BackCommandProperty = ...;

    // Split view
    public static readonly StyledProperty<object?> ListContentProperty = ...;
    public static readonly StyledProperty<object?> DetailContentProperty = ...;
    public static readonly StyledProperty<bool> IsPaneOpenProperty = ...;

    // Command bar
    public static readonly StyledProperty<IEnumerable?> PrimaryCommandsProperty = ...;
    public static readonly StyledProperty<IEnumerable?> SecondaryCommandsProperty = ...;

    // Status
    public static readonly StyledProperty<object?> StatusContentProperty = ...;

    // Drawer
    public static readonly StyledProperty<bool> IsFilterDrawerOpenProperty = ...;
    public static readonly StyledProperty<object?> FilterContentProperty = ...;
}
```

The template wires these properties to the child shell controls via `{TemplateBinding}`. Consumers set page-level properties; the scaffold distributes them to the right parts.

### Template Binding vs Named Part Wiring

Two approaches appear in scaffolds:

**Pure template binding** — child controls bind directly to scaffold properties. Simple, declarative, no code-behind.

**Code-behind forwarding** — `OnApplyTemplate` finds `PART_SplitView` and assigns properties in handlers. Useful when one scaffold property maps to multiple child properties (e.g., `IsFilterDrawerOpen` → `SkyDrawer.IsOpen` + command bar toggle state).

Prefer template binding until indirection is required — fewer moving parts at debug time.

### Why Composition Scaffolds Matter

Without scaffolds, every list-detail page in an app duplicates:

- Header layout and back button wiring
- Command bar placement
- Split view configuration
- Status bar content
- Filter drawer toggle

A scaffold encodes the **page pattern** once. Individual pages only provide content and commands.

### Example Page Consumption

```xml
<SkyListDetailPage Title="{Binding Title}"
                   Subtitle="{Binding Subtitle}"
                   BackCommand="{Binding GoBackCommand}"
                   IsPaneOpen="{Binding IsListVisible}"
                   IsFilterDrawerOpen="{Binding IsFilterOpen}"
                   PrimaryCommands="{Binding ToolbarCommands}">
  <SkyListDetailPage.ListContent>
    <SkyVirtualList ItemsSource="{Binding Items}"
                    SelectedItem="{Binding SelectedItem, Mode=TwoWay}" />
  </SkyListDetailPage.ListContent>
  <SkyListDetailPage.DetailContent>
    <local:DetailPanel DataContext="{Binding SelectedItem}" />
  </SkyListDetailPage.DetailContent>
  <SkyListDetailPage.FilterContent>
    <local:FilterPanel DataContext="{Binding Filter}" />
  </SkyListDetailPage.FilterContent>
</SkyListDetailPage>
```

The page XAML stays short. Visual consistency comes from the scaffold theme, not copy-pasted grid definitions.

### SkyFormPage and SkySettingsPage

Similar scaffolds exist for other page types:

- `SkyFormPage` — centered form layout with header and action buttons
- `SkySettingsPage` — sectioned settings with navigation sidebar

Each follows the same principle: compose existing shell controls, expose a simplified property surface, and let the theme template handle layout.

`SkySettingsPage` often nests a compact `SkyNavigationView` or `ListBox` in the pane and a `ScrollViewer` with settings sections in the detail region — same split metaphor, different content shape.

## SkyDrawer

Filter and settings panels use `SkyDrawer`, a slide-in panel with overlay:

```csharp
[TemplatePart(OverlayPartName, typeof(Panel))]
[TemplatePart(PanelPartName, typeof(Panel))]
[TemplatePart(CloseButtonPartName, typeof(Button))]
public class SkyDrawer : TemplatedControl
{
    public static readonly StyledProperty<bool> IsOpenProperty = ...;
    public static readonly StyledProperty<SkyDrawerPlacement> PlacementProperty = ...;
    public static readonly StyledProperty<object?> ContentProperty = ...;
}
```

`SkyListDetailPage` embeds a `SkyDrawer` for filter panels. The `IsFilterDrawerOpen` property on the page maps to `SkyDrawer.IsOpen`.

Drawers differ from `SkySplitView` overlay panes in intent: drawers are transient task panels (filters, pickers); split panes are structural navigation regions. Do not nest a drawer inside a resize grip without careful z-order — pointer capture conflicts are common.

Animate drawer open/close with the same `SkyMotionAnimator` slide transitions used elsewhere. Match duration tokens so filter panels feel consistent with sheets and dialogs.

## Avalonia Concept: Logical vs Visual Tree in Scaffolds

Scaffolds deepen the visual tree. `{Binding}` paths from a page into deeply nested template children sometimes break if you forget `DataContext` inheritance stops at template boundaries. Content injected via `ListContent` / `DetailContent` properties inherits the page's data context automatically when assigned to `ContentPresenter` slots — explicit `DataContext="{Binding SelectedItem}"` on detail content remains the page author's responsibility.

Use `x:DataType` on page roots for compiled bindings; scaffold templates should not hard-code view model types.

## Design Principle: Composition Over Monoliths

SkyUI shell architecture follows a clear hierarchy:

1. **Primitives** — `SkySplitView`, `SkyPageHeader`, `SkyCommandBar`
2. **Scaffolds** — `SkyListDetailPage`, `SkyFormPage`
3. **Application pages** — content plugged into scaffolds

When you build a page layout system:

- Keep primitives focused on one layout concern
- Use pseudo-classes for state that affects styling
- Build scaffolds that compose primitives through template binding
- Expose a reduced property API on scaffolds so page authors think in page terms, not layout terms
- Test primitives in isolation before composing scaffolds — split view resize bugs are easier to find without drawer and command bar in the way

## Scaffold Debugging Tips

1. **Missing header commands** — verify `PrimaryCommands` implements `IEnumerable` of command models the command bar template expects
2. **Back button inert** — `BackCommand` null; routed commands need `CanExecute` true
3. **Filter drawer won't open** — two-way bind `IsFilterDrawerOpen`; check drawer part names in theme
4. **Detail empty but list selects** — detail `DataContext` not bound to selected item
5. **Theme regression after scaffold change** — run UI snapshot tests on one primitive, one scaffold, one full page

## Summary

`SkySplitView` is the workhorse master-detail primitive with resize, overlay, and placement support. Page scaffolds like `SkyListDetailPage` demonstrate how to compose multiple shell controls into reusable page templates, reducing boilerplate and enforcing consistent application structure. Master pointer capture, pseudo-class matrices, and template binding — those three Avalonia skills transfer directly to any custom shell you build.
