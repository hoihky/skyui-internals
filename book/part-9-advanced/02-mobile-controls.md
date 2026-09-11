---
title: Chapter 28 — Mobile Primitives
order: 28
---

# Chapter 28: Mobile Primitives — SkySafeArea and SkyActionSheet

SkyUI's mobile support is early but establishes patterns for safe-area insets, touch targets, and bottom action sheets. This chapter explains platform service integration through Avalonia's `TopLevel` APIs.

## SkySafeArea: Device Notch and Home Indicator

Modern phones have non-rectangular screens. Content must pad away from notches, status bars, and home indicators.

```csharp
public class SkySafeArea : ContentControl
{
    public static readonly StyledProperty<bool> ApplyTopProperty = ...;    // default true
    public static readonly StyledProperty<bool> ApplyBottomProperty = ...;
    public static readonly StyledProperty<bool> ApplyLeftProperty = ...;
    public static readonly StyledProperty<bool> ApplyRightProperty = ...;
}
```

### Platform Integration

```csharp
protected override void OnAttachedToVisualTree(VisualTreeAttachmentEventArgs e)
{
    base.OnAttachedToVisualTree(e);
    var topLevel = TopLevel.GetTopLevel(this);
    if (topLevel?.InsetsManager is { } insets)
    {
        insets.SafeAreaChanged += OnSafeAreaChanged;
        ApplyInsets(insets.SafeAreaPadding);
    }
}

private void ApplyInsets(Thickness safeArea)
{
    Padding = new Thickness(
        ApplyLeft ? safeArea.Left : 0,
        ApplyTop ? safeArea.Top : 0,
        ApplyRight ? safeArea.Right : 0,
        ApplyBottom ? safeArea.Bottom : 0);
}
```

### Avalonia Concept: IInsetsManager

`IInsetsManager` is Avalonia's cross-platform abstraction for display cutouts. On iOS it maps to safe area layout guides; on Android to window insets; on desktop it typically returns zero padding.

`SafeAreaChanged` fires on rotation, split-screen, or keyboard appearance — the handler re-applies padding.

### Detach on Removal

```csharp
protected override void OnDetachedFromVisualTree(VisualTreeAttachmentEventArgs e)
{
    DetachInsets();
    base.OnDetachedFromVisualTree(e);
}
```

Always unsubscribe platform events on detach to prevent callbacks on disposed controls.

### Usage

```xml
<SkySafeArea>
  <Grid RowDefinitions="Auto,*">
    <SkyPageHeader Title="Settings" />
    <ScrollViewer Grid.Row="1">
      <!-- content avoids notch and home bar -->
    </ScrollViewer>
  </Grid>
</SkySafeArea>
```

Disable individual edges when a parent already handles that inset:

```xml
<SkySafeArea ApplyTop="False">
  <!-- top safe area handled by system status bar overlay -->
</SkySafeArea>
```

## SkyKeyboardInset

Sibling control that pads bottom content when the on-screen keyboard occludes the view. Uses `IInputPane` from `TopLevel`:

```csharp
var inputPane = TopLevel.GetTopLevel(this)?.InputPane;
inputPane!.OccludedRectChanged += (_, _) =>
    Padding = new Thickness(0, 0, 0, inputPane.OccludedRect.Height);
```

Combine `SkySafeArea` + `SkyKeyboardInset` for full mobile layout handling.

## SkyTouchTarget: 44px Minimum

```csharp
public static class SkyTouchTarget
{
    public static readonly AttachedProperty<bool> EnsureMinSizeProperty =
        AvaloniaProperty.RegisterAttached<Control, bool>("EnsureMinSize", defaultValue: true);
}
```

When true, sets `MinWidth` and `MinHeight` to 44 (Apple HIG / Material touch target guideline) without changing visual icon size inside the button.

```xml
<Button Classes="sky" Width="32" Height="32"
        sky:SkyTouchTarget.EnsureMinSize="True" />
```

The button's **layout** hit area expands to 44×44; content can remain smaller.

---

## SkyActionSheet: Bottom Sheet Menu

Mobile apps use action sheets for contextual choices (Share, Delete, Cancel).

### Static Facade over SkySheetHost

```csharp
public static class SkyActionSheet
{
    private static SkySheetHost? attachedHost;

    public static void Attach(SkySheetHost host) => attachedHost = host;

    public static async Task<SkyActionSheetResult> ShowAsync(
        IReadOnlyList<SkyActionSheetItem> items,
        string? title = null,
        SkySheetHost? host = null)
    {
        // Build button column UI, show sheet, await tap
    }
}
```

Same **Attach + ShowAsync** pattern as `SkyMessageBox`.

### SkyActionSheetItem Model

```csharp
public class SkyActionSheetItem
{
    public string Title { get; set; } = "";
    public bool IsDestructive { get; set; }
    public bool IsCancel { get; set; }
}
```

Destructive actions render in danger color; cancel item is visually separated at the bottom (iOS pattern).

### Application Setup

```xml
<Panel>
  <local:MainView />
  <SkySheetHost x:Name="SheetHost" />
</Panel>
```

```csharp
SkyActionSheet.Attach(SheetHost);
```

```csharp
var result = await SkyActionSheet.ShowAsync(new[]
{
    new SkyActionSheetItem { Title = "Share" },
    new SkyActionSheetItem { Title = "Delete", IsDestructive = true },
    new SkyActionSheetItem { Title = "Cancel", IsCancel = true },
}, title: "Playlist options");

if (result.SelectedIndex == 1)
    await DeletePlaylistAsync();
```

### SkySheetHost Animation

Uses `SkyMotionAnimator` translate-from-bottom enter/exit — parallel to dialog fade-scale but directionally appropriate for mobile sheets.

## Mobile vs Desktop Control Choice

| Desktop | Mobile |
|---------|--------|
| `SkyContextMenu` | `SkyActionSheet` |
| `SkyDialogHost` | `SkySheetHost` for non-blocking choices |
| Standard padding | `SkySafeArea` + `SkyKeyboardInset` |
| 32px buttons | `SkyTouchTarget.EnsureMinSize` |

## Deep Dive: SkySafeArea Implementation Details

### Why Visual Tree Attachment Matters

`IInsetsManager` is only available after the control attaches to a `TopLevel` (window). `OnAttachedToVisualTree` is the correct hook — not the constructor:

```csharp
protected override void OnAttachedToVisualTree(VisualTreeAttachmentEventArgs e)
{
    base.OnAttachedToVisualTree(e);
    var topLevel = TopLevel.GetTopLevel(this);
    _insetsManager = topLevel?.InsetsManager;
    if (_insetsManager is null) return;

    _insetsManager.SafeAreaChanged += OnSafeAreaChanged;
    ApplyInsets(_insetsManager.SafeAreaPadding);
}
```

On iOS simulators and notched devices, `SafeAreaPadding` returns non-zero top/bottom. Desktop returns `0` — the same XAML works everywhere.

### Per-Edge Toggles

```csharp
Padding = new Thickness(
    ApplyLeft ? safeArea.Left : 0,
    ApplyTop ? safeArea.Top : 0,
    ApplyRight ? safeArea.Right : 0,
    ApplyBottom ? safeArea.Bottom : 0);
```

Use `ApplyTop="False"` when a full-bleed header image extends under the status bar but body content should still respect safe area via a nested `SkySafeArea`.

### Nesting Safe Areas

Avoid double-padding:

```xml
<!-- Wrong: double top inset -->
<SkySafeArea>
  <SkySafeArea>...</SkySafeArea>
</SkySafeArea>

<!-- Right: outer handles horizontal, inner handles bottom only -->
<SkySafeArea ApplyBottom="False">
  <Grid RowDefinitions="Auto,*">
    <SkyPageHeader />  <!-- extends under status bar -->
    <SkySafeArea Grid.Row="1" ApplyTop="False">
      <ScrollViewer>...</ScrollViewer>
    </SkySafeArea>
  </Grid>
</SkySafeArea>
```

---

## Deep Dive: SkyKeyboardInset

When the on-screen keyboard opens, `IInputPane.OccludedRect` reports the covered region:

```csharp
private void OnOccludedRectChanged(object? sender, EventArgs e)
{
    var pane = TopLevel.GetTopLevel(this)?.InputPane;
    if (pane is null) return;
    var bottom = pane.OccludedRect.Bottom - pane.OccludedRect.Top;
    Padding = new Thickness(0, 0, 0, Math.Max(0, bottom));
}
```

Place `SkyKeyboardInset` around the page root so the focused `TextBox` scrolls above the keyboard. Pair with `ScrollViewer` `BringIntoViewOnFocusChanged` on inputs.

---

## Deep Dive: SkyActionSheet Implementation Flow

`ShowAsync` builds UI dynamically:

1. Create vertical `StackPanel` of buttons from `SkyActionSheetItem` list
2. Mark destructive item with danger brush class
3. Separate cancel item at bottom with gap
4. Assign panel to `SkySheetHost.SheetContent`
5. Set `IsOpen = true` — animator slides up
6. `TaskCompletionSource` completes when user taps an item or dismisses

```csharp
private static Task<SkyActionSheetResult> ShowOnHostAsync(...)
{
    var tcs = new TaskCompletionSource<SkyActionSheetResult>();
    foreach (var item in items)
    {
        var btn = new Button { Content = item.Title };
        if (item.IsDestructive)
            btn.Classes.Add("sky-danger");
        btn.Click += (_, _) =>
        {
            host.IsOpen = false;
            tcs.TrySetResult(new SkyActionSheetResult(index));
        };
        panel.Children.Add(btn);
    }
    host.IsOpen = true;
    return tcs.Task;
}
```

`SemaphoreSlim` optional — only one sheet at a time, matching dialog gate pattern.

### SkyActionSheetItem Semantics

| Flag | Behavior |
|------|----------|
| `IsDestructive` | Red text — delete account, remove file |
| `IsCancel` | Separated visually; dismisses without action index |

iOS Human Interface Guidelines inspire the layout; Android Material bottom sheets are similar.

---

## SkyTouchTarget in Practice

```xml
<Button Width="24" Height="24"
        sky:SkyTouchTarget.EnsureMinSize="True">
  <SkyIcon Kind="More" Size="Small" />
</Button>
```

Visual remains 24px; hit target expands to 44px via `MinWidth`/`MinHeight` coercion. Theme may add transparent padding — check `SkyTouchTarget` changed handler for exact behavior.

---

## Testing Mobile Layouts Without Devices

- Avalonia iOS/Android simulators report safe area insets
- Desktop: insets are zero — test padding logic with mocked `IInsetsManager` in unit tests
- Resize window in mobile demo project to verify `SkyNavigationView` bottom mode

## Summary

Mobile primitives integrate **platform services through TopLevel** — safe area, keyboard occlusion, touch targets, and action sheets. The architecture mirrors desktop overlays (host + static facade + motion) while respecting platform inset APIs.
