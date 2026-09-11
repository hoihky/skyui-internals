---
title: Chapter 22 — SkyAlert, SkyProgressRing, and SkySkeleton
order: 22
---

# Chapter 22: SkyAlert, SkyProgressRing, and SkySkeleton

Feedback controls communicate status without blocking the entire application. This chapter covers inline alerts, circular progress indicators, and skeleton placeholders — three controls that share `SkyFeedbackVariant` semantics but serve different UX roles.

## SkyFeedbackVariant: Shared Semantic Coloring

```csharp
public enum SkyFeedbackVariant
{
    Neutral, Info, Warning, Danger, Success
}
```

Used by `SkyAlert`, `SkyBanner`, `SkySnackbarBar`, and message box content. Each variant maps to brush tokens in the theme:

| Variant | Typical brush |
|---------|---------------|
| Info | `SkyInfoBrush` |
| Warning | `SkyWarningBrush` |
| Danger | `SkyDangerBrush` |
| Success | `SkyAccentBrush` |
| Neutral | `SkySurfaceElevatedBrush` |

One enum ensures a "warning" looks identical whether it appears in an alert bar or a snackbar toast.

---

## SkyAlert: Inline Dismissible Message

```csharp
public class SkyAlert : TemplatedControl
{
    public static readonly StyledProperty<string?> TitleProperty = ...;
    public static readonly StyledProperty<string?> MessageProperty = ...;
    public static readonly StyledProperty<SkyFeedbackVariant> VariantProperty = ...;
    public static readonly StyledProperty<bool> IsCloseableProperty = ...;

    public static readonly RoutedEvent<RoutedEventArgs> CloseRequestedEvent = ...;
}
```

### Variant as Style Class

```csharp
private void SyncVariantClass()
{
    foreach (var name in Enum.GetNames<SkyFeedbackVariant>())
        Classes.Remove($"sky-feedback-{name.ToLowerInvariant()}");

    Classes.Add($"sky-feedback-{Variant.ToString().ToLowerInvariant()}");
}
```

Unlike pseudo-classes, SkyUI uses **style classes** for feedback variants because they combine with other classes (`sky-alert`). Theme selector example:

```xml
<Style Selector="controls|SkyAlert.sky-feedback-danger">
  <Setter Property="BorderBrush" Value="{DynamicResource SkyDangerBrush}" />
</Style>
```

Removing all variant classes before adding the current one prevents stale classes when `Variant` changes from `Info` to `Danger`.

### Close Button Wiring

```csharp
protected override void OnApplyTemplate(TemplateAppliedEventArgs e)
{
    base.OnApplyTemplate(e);
    SyncVariantClass();

    if (_closeButton is not null)
        _closeButton.Click -= OnCloseClick;

    _closeButton = e.NameScope.Find(CloseButtonPartName) as Button;
    if (_closeButton is not null)
        _closeButton.Click += OnCloseClick;
}

private void OnCloseClick(object? sender, RoutedEventArgs e) =>
    RaiseEvent(new RoutedEventArgs(CloseRequestedEvent));
```

The alert does not remove itself — the parent handles `CloseRequested` and removes the control or hides it via view model. This keeps the control stateless regarding persistence.

### Usage

```xml
<SkyAlert Title="Connection lost"
          Message="Retrying in 30 seconds..."
          Variant="Warning"
          IsCloseable="True"
          CloseRequested="OnAlertDismissed" />
```

`SkyMessageBox` reuses `SkyAlert` as dialog body content — composition across feedback layers.

---

## SkyProgressRing: Determinate and Indeterminate

```csharp
public class SkyProgressRing : TemplatedControl
{
    public static readonly StyledProperty<double> ValueProperty = ...;
    public static readonly StyledProperty<double> MinimumProperty = ...;
    public static readonly StyledProperty<double> MaximumProperty = ...;
    public static readonly StyledProperty<bool> IsIndeterminateProperty = ...;
    public static readonly StyledProperty<double> StrokeThicknessProperty = ...;
}
```

### Default Size Overrides

```csharp
static SkyProgressRing()
{
    WidthProperty.OverrideDefaultValue<SkyProgressRing>(40);
    HeightProperty.OverrideDefaultValue<SkyProgressRing>(40);
    MinWidthProperty.OverrideDefaultValue<SkyProgressRing>(24);
    MinHeightProperty.OverrideDefaultValue<SkyProgressRing>(24);
}
```

Default dimensions ensure the ring is usable without explicit Width/Height in XAML — important when `SkyButtonProperties.IsLoading` injects a ring into a button.

### Indeterminate Animation

```csharp
protected override void OnPropertyChanged(AvaloniaPropertyChangedEventArgs change)
{
    base.OnPropertyChanged(change);
    if (change.Property == IsIndeterminateProperty)
        UpdateSpinTimer();
    else if (change.Property is StyledProperty<double> &&
             change.Property.Name is nameof(Value) or nameof(Minimum) or nameof(Maximum))
        InvalidateVisual();
}

private void UpdateSpinTimer()
{
    if (IsIndeterminate)
    {
        _spinTimer = new DispatcherTimer { Interval = TimeSpan.FromMilliseconds(16) };
        _spinTimer.Tick += (_, _) =>
        {
            _spinAngle = (_spinAngle + 6) % 360;
            InvalidateVisual();
        };
        _spinTimer.Start();
    }
    else
    {
        _spinTimer?.Stop();
        _spinTimer = null;
    }
}
```

Indeterminate mode uses `DispatcherTimer` + `InvalidateVisual()` to rotate an arc. Determinate mode draws an arc proportional to `(Value - Minimum) / (Maximum - Minimum)`.

### Custom Render

`SkyProgressRing` overrides rendering in `Render(DrawingContext context)` (implementation in source) rather than using only XAML shapes — this gives precise arc geometry and animation performance.

### SkyButtonProperties.IsLoading Integration

When `IsLoading` attaches to a `Button`:

```csharp
button.Content = new SkyProgressRing
{
    IsIndeterminate = true,
    Width = 20,
    Height = 20
};
```

The ring becomes the button's content until loading completes. Understanding `SkyProgressRing` internals explains button loading behavior.

---

## SkySkeleton: Loading Placeholder

```csharp
public class SkySkeleton : TemplatedControl
{
    public static readonly StyledProperty<bool> IsActiveProperty =
        AvaloniaProperty.Register<SkySkeleton, bool>(nameof(IsActive), true);

    static SkySkeleton()
    {
        WidthProperty.OverrideDefaultValue<SkySkeleton>(120);
        HeightProperty.OverrideDefaultValue<SkySkeleton>(16);
        MinHeightProperty.OverrideDefaultValue<SkySkeleton>(8);
        CornerRadiusProperty.OverrideDefaultValue<SkySkeleton>(new CornerRadius(6));
    }
}
```

### Skeleton vs Overlay vs Progress

| Control | Blocks input | Shows structure | Typical scenario |
|---------|--------------|-----------------|------------------|
| `SkySkeleton` | No | Yes (gray shapes) | Initial list/card load |
| `SkyLoadingOverlay` | Optional | No (spinner over content) | Refresh existing view |
| `SkyProgressRing` | No | No | Inline spinner in button or row |

### Theme Animation

The C# class only exposes `IsActive`. The theme applies a pulsing opacity animation:

```xml
<Style Selector="controls|SkySkeleton">
  <Style.Animations>
    <Animation Duration="0:0:1.2" IterationCount="Infinite">
      <KeyFrame Cue="0%"><Setter Property="Opacity" Value="0.4" /></KeyFrame>
      <KeyFrame Cue="50%"><Setter Property="Opacity" Value="1.0" /></KeyFrame>
      <KeyFrame Cue="100%"><Setter Property="Opacity" Value="0.4" /></KeyFrame>
    </Animation>
  </Style.Animations>
</Style>
```

Separating animation to XAML lets designers tune timing without recompiling C#.

### Composing Skeleton Layouts

```xml
<StackPanel Spacing="12" IsVisible="{Binding IsLoading}">
  <SkySkeleton Width="200" Height="24" />
  <SkySkeleton Width="320" Height="16" />
  <SkySkeleton Width="280" Height="16" />
</StackPanel>

<local:ContentView IsVisible="{Binding IsLoading, Converter={x:Static BoolConverters.Not}}"
                   DataContext="{Binding Data}" />
```

Match skeleton shapes to final content layout to minimize visual jump when data arrives.

## Summary

Feedback controls share variant semantics but implement distinct UX patterns: alerts for messages, progress rings for ongoing work, skeletons for structural placeholders. Variant sync via style classes and `InvalidateVisual` for custom render are the key implementation techniques to study.
