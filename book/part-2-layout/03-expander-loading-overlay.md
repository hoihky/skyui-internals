---
title: Chapter 7 — SkyExpander and SkyLoadingOverlay
order: 7
---

# Chapter 7: SkyExpander and SkyLoadingOverlay

Part II continues with two controls at opposite ends of the complexity spectrum: a **primitive subclass** that adds almost no code (`SkyExpander`), and a **content overlay** that manages hit-testing during async work (`SkyLoadingOverlay`).

## SkyExpander: When Subclassing Is Enough

Avalonia ships `Expander` — a header that toggles content visibility. SkyUI does not reimplement expand/collapse logic. It subclasses and applies theme classes:

```csharp
public class SkyExpander : Expander
{
    public SkyExpander()
    {
        Classes.Add("sky");
        Classes.Add("sky-expander");
    }
}
```

### Why This Is Valid Architecture

Before writing a `TemplatedControl`, ask:

1. Does the built-in control already expose the right API? **Yes** — `IsExpanded`, `Header`, content.
2. Can styling alone achieve the desired look? **Yes** — theme targets `Expander.sky-expander`.
3. Is there new behavior? **No** — no validation, no adapters, no custom parts.

If all three answers favor the primitive, subclass + style class is the correct choice. You avoid maintaining duplicate template parts and inherit bug fixes from Avalonia upstream.

### Theme Responsibilities

The theme template for `Expander.sky-expander` typically adjusts:

- Chevron icon rotation when expanded (via `^:expanded` selector on Avalonia's pseudo-class)
- Header padding from `SkySpace12Px`
- Content area background from `SkySurfaceBrush`
- Corner radius and border from card tokens

The C# class stays frozen; visual iteration happens entirely in XAML.

### Usage

```xml
<SkyExpander Header="Advanced options" IsExpanded="False">
  <StackPanel Spacing="8">
    <CheckBox Content="Enable debug logging" />
    <CheckBox Content="Send anonymous telemetry" />
  </StackPanel>
</SkyExpander>
```

### Lesson

Not every control in a toolkit needs custom C#. SkyUI's catalog mixes full implementations with thin wrappers deliberately. When auditing your own codebase, flag wrappers that grew unnecessary logic — they may have been candidates for style-only customization.

---

## SkyLoadingOverlay: Blocking Async Operations

`SkyLoadingOverlay` lives in `src/SkyUI/Controls/Data/` because it is commonly paired with data views, though it works over any content.

### Base Class: ContentControl

It inherits `ContentControl` — the overlay wraps existing page content:

```xml
<SkyLoadingOverlay IsLoading="{Binding IsRefreshing}"
                   LoadingText="Loading records...">
  <SkyVirtualDataGrid DataSource="{Binding Source}" />
</SkyLoadingOverlay>
```

When `IsLoading` is false, only the child grid is visible. When true, a semi-transparent scrim and optional spinner appear above the child.

### Properties

```csharp
public class SkyLoadingOverlay : ContentControl
{
    public static readonly StyledProperty<bool> IsLoadingProperty = ...;
    public static readonly StyledProperty<string?> LoadingTextProperty = ...;
    public static readonly StyledProperty<bool> BlocksInputProperty =
        ... defaultValue: true);
}
```

| Property | Role |
|----------|------|
| `IsLoading` | Shows/hides overlay chrome |
| `LoadingText` | Optional message below spinner |
| `BlocksInput` | When true, pointer events do not reach children |

### Hit-Testing Model

This is the critical implementation detail:

```csharp
private void SyncLoadingState()
{
    var loading = IsLoading;
    Classes.Set("sky-loading-overlay-active", loading);

    if (_overlay is not null)
    {
        _overlay.IsVisible = loading;
        _overlay.IsHitTestVisible = loading && BlocksInput;
    }
}
```

Avalonia's input system routes pointer events to the topmost element that has `IsHitTestVisible = true`. If the overlay panel is visible but not hit-test visible, users can interact with content beneath — useful for background refresh indicators.

When `BlocksInput` is true (default), the overlay captures all clicks, preventing double-submit or navigation during save operations.

### Constructor and Classes

```csharp
public SkyLoadingOverlay()
{
    Classes.Add("sky");
    Classes.Add("sky-loading-overlay");
}
```

The `sky-loading-overlay-active` class is toggled in code; theme styles increase scrim opacity and show the progress ring when active.

### Template Structure

```xml
<ControlTheme TargetType="controls:SkyLoadingOverlay">
  <Setter Property="Template">
    <ControlTemplate>
      <Grid>
        <ContentPresenter Content="{TemplateBinding Content}" />
        <Panel Name="PART_Overlay"
               Background="{DynamicResource SkyScrimBrush}"
               IsVisible="False">
          <StackPanel HorizontalAlignment="Center" VerticalAlignment="Center">
            <SkyProgressRing IsIndeterminate="True" />
            <TextBlock Text="{TemplateBinding LoadingText}" />
          </StackPanel>
        </Panel>
      </Grid>
    </ControlTemplate>
  </Setter>
</ControlTheme>
```

The child content and overlay are siblings in a `Grid`. The overlay covers the full area when visible.

### View Model Integration

```csharp
public async Task RefreshAsync()
{
    IsRefreshing = true;
    try
    {
        Records = await _api.FetchAllAsync();
    }
    finally
    {
        IsRefreshing = false;
    }
}
```

```xml
<SkyLoadingOverlay IsLoading="{Binding IsRefreshing}"
                   LoadingText="Fetching data..."
                   BlocksInput="True">
  <!-- page content -->
</SkyLoadingOverlay>
```

Always clear `IsLoading` in `finally` so a failed request does not leave the UI permanently blocked.

### Comparison with SkySkeleton

| Control | Use when |
|---------|----------|
| `SkyLoadingOverlay` | Blocking wait over existing layout; user should not interact |
| `SkySkeleton` | Placeholder shapes while structure is known but data is not yet loaded |

Skeletons suit initial page load; overlays suit refresh-in-place.

## Summary

`SkyExpander` demonstrates the **minimal subclass** pattern. `SkyLoadingOverlay` demonstrates **content overlay** with explicit hit-test control — a pattern you will reuse for any "please wait" UI over existing content.
