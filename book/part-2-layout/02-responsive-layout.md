---
title: Chapter 6 — Responsive Layout
order: 6
---

# Chapter 6: Responsive Layout with SkyResponsiveGrid and SkyGridLayout

Modern desktop applications are not fixed-width. Users resize windows, dock panels, and run apps on tablets. SkyUI provides responsive layout through `SkyResponsiveGrid` (a self-adjusting grid) and `SkyGridLayout` (attached properties that make any `Grid` responsive). This chapter explains both, the breakpoint system they share, and how width-driven layout connects to Avalonia's property system and visual tree lifecycle.

## Why Width-Driven Columns?

Fixed column counts break on small windows: tiles clip, horizontal scrollbars appear, or content stacks awkwardly. Fixed counts also waste space on ultrawide monitors — four narrow columns when eight would fit.

SkyUI responsive layout answers one question repeatedly: **given the current width of this container, how many equal columns should we use?** Child controls flow in row-major order; when column count changes, children reflow without explicit `Grid.Column` edits in page XAML.

This is container-query-style behavior implemented with Avalonia's `Bounds` property rather than window-level media queries. The grid reacts to **its own** width, not only the window — important when the grid lives inside a sidebar or split pane.

## The Breakpoint Model

`SkyGridBreakpoint` maps control width to a column count:

```csharp
public static int ResolveColumns(double width, SkyResponsiveColumnProfile? profile)
{
    profile ??= SkyResponsiveColumnProfile.Default;

    if (width >= profile.Xl) return profile.XlColumns;
    if (width >= profile.Lg) return profile.LgColumns;
    if (width >= profile.Md) return profile.MdColumns;
    if (width >= profile.Sm) return profile.SmColumns;
    return profile.XsColumns;
}
```

`SkyResponsiveColumnProfile` holds the breakpoint widths and corresponding column counts:

```csharp
public class SkyResponsiveColumnProfile
{
    public double Xs { get; set; } = 0;
    public int XsColumns { get; set; } = 1;

    public double Sm { get; set; } = 480;
    public int SmColumns { get; set; } = 2;

    public double Md { get; set; } = 768;
    public int MdColumns { get; set; } = 3;

    public double Lg { get; set; } = 1024;
    public int LgColumns { get; set; } = 4;

    public double Xl { get; set; } = 1440;
    public int XlColumns { get; set; } = 4;

    public static SkyResponsiveColumnProfile Default { get; } = new();
}
```

This is a mobile-first stepped scale. As the container grows wider, more columns become available. The profile is configurable per grid instance so a dashboard and a settings page can use different breakpoints.

### Reading the Breakpoint Table

| Width range | Default columns | Typical layout |
|-------------|-----------------|----------------|
| `< 480px` | 1 | Phone-narrow, stacked KPIs |
| `480–767px` | 2 | Small tablet, two-up tiles |
| `768–1023px` | 3 | Laptop partial width |
| `1024–1439px` | 4 | Full desktop |
| `≥ 1440px` | 4 | Wide desktop (same count, wider cells) |

The default profile caps at four columns on extra-large widths. Dashboards with eight small tiles can set `XlColumns="6"` or `"8"` on a custom profile without changing `ResolveColumns` logic.

## SkyResponsiveGrid

`SkyResponsiveGrid` subclasses `UniformGrid` and updates `Columns` whenever its width changes:

```csharp
public class SkyResponsiveGrid : UniformGrid
{
    public static readonly StyledProperty<SkyResponsiveColumnProfile?> ColumnProfileProperty =
        AvaloniaProperty.Register<SkyResponsiveGrid, SkyResponsiveColumnProfile?>(
            nameof(ColumnProfile));

    public SkyResponsiveGrid()
    {
        Classes.Add("sky");
        Classes.Add("sky-responsive-grid");
        ColumnSpacing = 16;
        RowSpacing = 16;
    }

    protected override void OnAttachedToVisualTree(VisualTreeAttachmentEventArgs e)
    {
        base.OnAttachedToVisualTree(e);
        UpdateColumns(Bounds.Width);
    }

    protected override void OnPropertyChanged(AvaloniaPropertyChangedEventArgs change)
    {
        base.OnPropertyChanged(change);

        if (change.Property == BoundsProperty)
            UpdateColumns(Bounds.Width);
        else if (change.Property == ColumnProfileProperty)
            UpdateColumns(Bounds.Width);
    }

    private void UpdateColumns(double width)
    {
        if (width <= 0)
            return;

        var columns = SkyGridBreakpoint.ResolveColumns(width, ColumnProfile);
        if (Columns != columns)
            Columns = columns;
    }
}
```

### Key Implementation Details

1. **Bounds watching** — `OnPropertyChanged` listens for `BoundsProperty` changes. This fires on every resize without manual `SizeChanged` event subscription. `Bounds` is a styled property on `Visual`; when layout assigns a new size, your handler runs.

2. **Visual tree attachment** — `OnAttachedToVisualTree` triggers the first column calculation once the control has a visual parent and valid layout slot. Before attachment, bounds may be zero or stale.

3. **Guard against zero width** — During initial layout, bounds may be zero. Updating columns to zero would break `UniformGrid` layout, so the method returns early.

4. **Conditional assignment** — `if (Columns != columns)` avoids redundant layout passes when the column count has not actually changed.

5. **Default spacing** — Constructor sets `ColumnSpacing` and `RowSpacing` to 16, matching the `SkySpace16Px` token. Theme styles can override spacing via selectors on `sky-responsive-grid`.

### UniformGrid Behavior

`UniformGrid` assigns each child the same cell size and fills row-major: item 0 column 0, item 1 column 1, …, item N wraps to the next row. SkyResponsiveGrid does not implement custom placement — it only changes **how many** columns exist. For explicit positioning, use `SkyGridLayout` on a standard `Grid` or set `Grid.Column` manually.

### Usage

```xml
<SkyResponsiveGrid>
  <SkyKpiTile Title="Users" Value="12,450" />
  <SkyKpiTile Title="Revenue" Value="$84.2k" />
  <SkyKpiTile Title="Churn" Value="2.1%" />
  <SkyKpiTile Title="NPS" Value="72" />
</SkyResponsiveGrid>
```

At narrow widths, tiles stack in one column. As the window widens, they reflow into two, three, or four columns automatically. Each `SkyKpiTile` is a logical and visual child of the grid.

### Custom Profile

```xml
<SkyResponsiveGrid>
  <SkyResponsiveGrid.ColumnProfile>
    <SkyResponsiveColumnProfile Sm="600" SmColumns="2"
                                Md="900" MdColumns="3"
                                Lg="1200" LgColumns="4"
                                Xl="1600" XlColumns="6" />
  </SkyResponsiveGrid.ColumnProfile>
  <!-- children -->
</SkyResponsiveGrid>
```

Profiles are plain CLR objects stored on a styled property — you can define them inline in XAML, in resources, or assign from code when a view model exposes layout preferences.

### Step-by-Step: Dashboard Page

**Goal:** Analytics dashboard with KPI row and detail cards below.

1. **Page root** — `ScrollViewer` or `SkyPageScaffold` content area (see navigation chapters)
2. **KPI section:**

```xml
<SkyResponsiveGrid Margin="{DynamicResource SkySpace16Px}">
  <SkyKpiTile Title="Active users" Value="{Binding ActiveUsers}" />
  <SkyKpiTile Title="MRR" Value="{Binding Mrr}" />
  <SkyKpiTile Title="Tickets" Value="{Binding OpenTickets}" />
</SkyResponsiveGrid>
```

3. **Section break** — optional `SkyDivider Text="Details"`
4. **Detail cards in second grid** with tighter profile if cards need minimum width:

```xml
<SkyResponsiveGrid ColumnProfile="{StaticResource TwoColumnProfile}">
  <SkyCard Header="Sales">...</SkyCard>
  <SkyCard Header="Support">...</SkyCard>
</SkyResponsiveGrid>
```

5. **Resize window** — observe column count steps at breakpoints without changing XAML
6. **Verify** spacing tokens on margins match grid `ColumnSpacing` for aligned rhythm

## SkyGridLayout: Attached Properties

Sometimes you already have a `Grid` and want responsive behavior without replacing it with `SkyResponsiveGrid`. `SkyGridLayout` provides attached properties:

```csharp
public static class SkyGridLayout
{
    public static readonly AttachedProperty<bool> IsResponsiveProperty =
        AvaloniaProperty.RegisterAttached<Grid, bool>("IsResponsive");

    public static readonly AttachedProperty<SkyResponsiveColumnProfile?> ColumnProfileProperty =
        AvaloniaProperty.RegisterAttached<Grid, SkyResponsiveColumnProfile?>("ColumnProfile");

    public static readonly AttachedProperty<bool> AutoPlaceChildrenProperty =
        AvaloniaProperty.RegisterAttached<Grid, bool>("AutoPlaceChildren");
}
```

When `IsResponsive` is true on a `Grid`, the attached property changed handler subscribes to bounds changes and updates `Grid.ColumnDefinitions` to create equal-width `*` columns based on the resolved column count.

`AutoPlaceChildren` optionally assigns `Grid.Column` and `Grid.Row` to children in row-major order, which is convenient for dashboard tiles that do not need explicit grid positioning.

### How Attached Property Handlers Wire the Visual Tree

On `IsResponsive` changed to `true`:

1. Handler resolves the target `Grid`
2. Subscribes to `Bounds` changes (same signal `SkyResponsiveGrid` uses)
3. Calls shared `SkyGridBreakpoint.ResolveColumns`
4. Rebuilds `ColumnDefinitions` with N equal star columns
5. If `AutoPlaceChildren` is true, walks **logical children** and sets `Grid.Row` / `Grid.Column`

Using attached properties keeps page markup on `Grid` — no new control type — while centralizing breakpoint math in one helper class.

### Usage

```xml
<Grid sky:SkyGridLayout.IsResponsive="True"
      sky:SkyGridLayout.AutoPlaceChildren="True"
      ColumnSpacing="{DynamicResource SkySpace16Px}"
      RowSpacing="{DynamicResource SkySpace16Px}">
  <SkyKpiTile Title="Users" Value="12,450" />
  <SkyKpiTile Title="Revenue" Value="$84.2k" />
  <SkyKpiTile Title="Churn" Value="2.1%" />
  <SkyKpiTile Title="NPS" Value="72" />
</Grid>
```

Mix explicit and auto placement by leaving `AutoPlaceChildren` false and setting `Grid.Column` on special items (hero tile spanning two columns requires a span API or nested grid — SkyGridLayout auto-place does not merge cells).

### When to Prefer Each API

| Scenario | Recommendation |
|----------|----------------|
| New dashboard section, uniform tiles | `SkyResponsiveGrid` |
| Existing page already structured as `Grid` | `SkyGridLayout` attached props |
| Mixed column spans in same row | Plain `Grid` + manual columns; responsive optional |
| Nested inside `SkyCard` content | Either — both respect container width |

## Design Pattern: Subclass vs Attached Property

SkyUI offers both approaches deliberately:

| Approach | Best for |
|----------|----------|
| `SkyResponsiveGrid` | New layouts where you control the container type |
| `SkyGridLayout` attached props | Existing grids, mixed layouts, XAML that already uses `Grid` |

Attached properties follow the same pattern as `SkyThemeProperties` and `SkyButtonProperties`: extend existing types without subclassing. Breakpoint resolution stays in `SkyGridBreakpoint.ResolveColumns` — a single function used by both paths.

## Property System and Layout Interaction

`Columns` on `UniformGrid` and `ColumnDefinitions` on `Grid` are styled properties. When responsive logic updates them:

1. Property change invalidates measure
2. Parent layout pass re-runs
3. Children receive new available widths
4. `Bounds` on children may change, but column count should stabilize after one pass

The guard `if (Columns != columns)` prevents infinite measure loops that would occur if every bounds notification reset columns even when unchanged.

## Relationship to Navigation Breakpoints

`SkyBreakpoint` in the navigation package uses similar width thresholds for switching `SkyNavigationView` between expanded sidebar, compact sidebar, and bottom navigation. The grid breakpoints and navigation breakpoints are separate profiles because layout density and navigation chrome have different requirements. A wide window might show four grid columns but still use compact sidebar if the user prefers it.

Do not assume one profile fits all subsystems. Copy numeric thresholds only when product design explicitly aligns navigation mode with content columns.

## Building Your Own Responsive Control

To add responsive behavior to a custom control:

1. Define a column profile or breakpoint table
2. Override `OnPropertyChanged` and watch `BoundsProperty`
3. Recalculate layout parameters in a single `UpdateLayout(double width)` method
4. Guard against zero bounds and redundant updates
5. Expose the profile as a `StyledProperty` so consumers can customize breakpoints
6. Hook `OnAttachedToVisualTree` for the initial layout after visual parent exists

Example extension: a responsive `WrapPanel` substitute that sets `ItemWidth` based on resolved columns:

```csharp
private void UpdateLayout(double width)
{
    if (width <= 0) return;

    var columns = SkyGridBreakpoint.ResolveColumns(width, ColumnProfile);
    var spacing = ColumnSpacing;
    var itemWidth = (width - spacing * (columns - 1)) / columns;

    if (Math.Abs(ItemWidth - itemWidth) > 0.5)
        ItemWidth = itemWidth;
}
```

Reuse `SkyGridBreakpoint` rather than duplicating threshold logic.

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Responsive grid inside horizontal `StackPanel` | Grid width unconstrained; columns never increase | Wrap in grid row with `*` width or use `HorizontalAlignment="Stretch"` |
| Expect cell spanning on `SkyResponsiveGrid` | Items cannot span columns | Use `Grid` + `SkyGridLayout` or manual `Grid` |
| Custom profile with descending breakpoints | Wrong column count at some widths | Ensure Sm < Md < Lg < Xl |
| `AutoPlaceChildren` after manual columns | Children jump position on resize | Disable auto-place or manage all positions in code |
| Watch window width instead of control bounds | Sidebar open does not reflow content | Watch `BoundsProperty` on the grid itself |
| Zero spacing with many columns | Tiles touch | Set `ColumnSpacing` / `RowSpacing` to token values |

## Debugging Tips

**Column count stuck at 1.** Inspect actual `Bounds.Width` of the responsive control in dev tools. If width is always small, parent layout is not allocating horizontal space.

**Flicker on resize.** Column count may oscillate if bounds hover near a breakpoint. Widen dead band only if product requires it — usually fix parent layout instead.

**Children overlap after profile change.** `AutoPlaceChildren` may need to re-run when column count changes; verify attached property handler re-places children on profile updates.

**Responsive grid empty on first paint.** Early `Bounds.Width` is zero — expected. If layout never updates, check that the control reached the visual tree (`OnAttachedToVisualTree` path).

**Different behavior between `SkyResponsiveGrid` and `SkyGridLayout`.** Confirm both use the same `ColumnProfile` instance or equivalent values; inline profiles that look similar may differ by one pixel threshold.

## Summary

Responsive layout in SkyUI is width-driven column resolution. `SkyResponsiveGrid` packages the behavior as a drop-in `UniformGrid` replacement for tile dashboards and KPI rows. `SkyGridLayout` brings the same behavior to ordinary `Grid` elements through attached properties, preserving existing markup while sharing `SkyGridBreakpoint.ResolveColumns` as the single source of breakpoint logic.

Watch `Bounds`, guard zero width, avoid redundant column updates, and treat container width — not window width — as the signal. Combined with layout primitives like `SkyCard` and `SkyDivider` from the previous chapter, you can build pages that reflow cleanly from narrow phone-like windows to wide multi-monitor desktops without duplicating XAML per form factor.
