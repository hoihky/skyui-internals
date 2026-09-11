---
title: Chapter 25 — Attached Properties
order: 25
---

# Chapter 25: Attached Properties and Primitive Enhancement

Attached properties let you add capabilities to existing types without subclassing. SkyUI uses them for button loading states, tooltip configuration, responsive grid behavior, theme overrides, and mobile touch targets. This chapter explains the Avalonia registration mechanism, changed-handler wiring, storage semantics, and walks through the most important attached property classes with debugging guidance.

## What Is an Attached Property?

In Avalonia, an attached property is registered on a static helper class but can be set on any compatible `AvaloniaObject`:

```csharp
public static class SkyButtonProperties
{
    public static readonly AttachedProperty<bool> IsLoadingProperty =
        AvaloniaProperty.RegisterAttached<Button, bool>(
            "IsLoading",
            typeof(SkyButtonProperties),
            defaultValue: false);

    public static bool GetIsLoading(Button button) =>
        button.GetValue(IsLoadingProperty);

    public static void SetIsLoading(Button button, bool value) =>
        button.SetValue(IsLoadingProperty, value);
}
```

XAML usage:

```xml
<Button Classes="sky sky-primary"
        sky:SkyButtonProperties.IsLoading="{Binding IsSaving}"
        Content="Save" />
```

The property is stored on the `Button` instance but defined externally.

### Avalonia Concept: Property Inheritance

Attached properties do **not** automatically inherit down the visual tree unless registered with `inherits: true` in metadata. `SkyThemeProperties` on `Application` intentionally inherit so descendant controls read density and accent overrides. `IsLoading` on a `Button` does not propagate to child text blocks — each owner holds its own value.

Check metadata when a child does not see an attached value you set on a parent.

### RegisterAttached Owner Type

The generic parameter `RegisterAttached<Button, bool>` declares the **expected owner type** for class handlers and XAML assignability. Setting the property on a wrong type fails at compile time in strict XAML or silently stores on unexpected types if forced in code — stick to documented owner types.

## SkyButtonProperties.IsLoading

When `IsLoading` becomes true, the button should show a spinner and ignore clicks:

```csharp
static SkyButtonProperties()
{
    IsLoadingProperty.Changed.AddClassHandler<Button>(OnIsLoadingChanged);
}

private static void OnIsLoadingChanged(Button button, AvaloniaPropertyChangedEventArgs e)
{
    var isLoading = e.NewValue is true;
    button.Classes.Set("sky-loading", isLoading);
    button.IsEnabled = !isLoading;

    if (isLoading)
    {
        StoreOriginalContent(button);
        button.Content = CreateSpinner();
    }
    else
    {
        RestoreOriginalContent(button);
    }
}
```

### Storing Original Content

Attached property handlers cannot use instance fields on the static helper per button. Store original content in a weak dictionary or Avalonia's attached property storage:

```csharp
private static readonly AttachedProperty<object?> OriginalContentProperty =
    AvaloniaProperty.RegisterAttached<Button, object?>(
        "OriginalContent",
        typeof(SkyButtonProperties));

private static void StoreOriginalContent(Button button)
{
    if (button.GetValue(OriginalContentProperty) is null)
        button.SetValue(OriginalContentProperty, button.Content);
}

private static void RestoreOriginalContent(Button button)
{
    var original = button.GetValue(OriginalContentProperty);
    if (original is not null)
    {
        button.Content = original;
        button.SetValue(OriginalContentProperty, null);
    }
}
```

Using a private attached property avoids a static `Dictionary<Button, object>` that leaks if buttons are removed without clearing loading state.

### Design Decisions

1. **Content swap** — Original content is stored in attached data; a `SkyProgressRing` replaces it during loading
2. **Disable interaction** — `IsEnabled = false` prevents double-submit
3. **Style class** — `sky-loading` can adjust opacity and min-width in the theme
4. **Works on any Button** — No `SkyButton` subclass required

This is the canonical example of enhancing a primitive through attached properties.

### Debugging IsLoading

| Symptom | Fix |
|---------|-----|
| Button text never returns | `IsLoading` stuck true; restore path not hit |
| Spinner shows but button clickable | Missing `IsEnabled = !isLoading` |
| Content wrong after rapid toggle | Race — guard re-entrancy; store original only once |
| Min-width collapse | Theme should set min-width on `.sky-loading` |

## SkyThemeProperties

Theme attached properties live on `Application` (or any resource host):

```csharp
public static class SkyThemeProperties
{
    public static readonly AttachedProperty<Color?> AccentOverrideProperty = ...;
    public static readonly AttachedProperty<SkyDensity> DensityProperty = ...;
}
```

```xml
<Application sky:SkyThemeProperties.AccentOverride="#1ED760"
             sky:SkyThemeProperties.Density="Compact">
```

Changed handlers invoke applicators:

```csharp
private static void OnAccentOverrideChanged(
    AvaloniaObject sender, AvaloniaPropertyChangedEventArgs e)
{
    if (sender is Application app && e.NewValue is Color color)
        SkyAccentOverrideApplicator.Apply(app, color);
    else if (sender is Application app2 && e.NewValue is null)
        SkyAccentOverrideApplicator.Clear(app2);
}
```

The applicator injects overridden brush values into the resource dictionary. All controls using `{DynamicResource SkyAccentBrush}` pick up the new color.

### DynamicResource vs StaticResource

Overrides must target resources read via `DynamicResource`. `{StaticResource}` resolves once at load — accent changes will not propagate. Audit theme templates when debugging "accent override does nothing."

### Density Propagation

`SkyDensity.Compact` adjusts padding tokens, row heights, and font sizes through merged resource dictionaries the applicator swaps. Set density once on `Application` rather than per-page unless a specific screen requires comfort mode.

## SkyGridLayout

Attached properties on `Grid` enable responsive column behavior:

```csharp
public static readonly AttachedProperty<bool> IsResponsiveProperty =
    AvaloniaProperty.RegisterAttached<Grid, bool>("IsResponsive");

public static readonly AttachedProperty<SkyResponsiveColumnProfile?> ColumnProfileProperty = ...;
public static readonly AttachedProperty<bool> AutoPlaceChildrenProperty = ...;
```

When `IsResponsive` is set on a grid, the changed handler subscribes to bounds changes and recalculates column definitions. This was covered in Chapter 6; the attached property form is the alternative to using `SkyResponsiveGrid`.

### Subscription Lifecycle

```csharp
private static void OnIsResponsiveChanged(Grid grid, AvaloniaPropertyChangedEventArgs e)
{
    if (e.NewValue is true)
    {
        grid.GetObservable(BoundsProperty).Subscribe(grid, OnBoundsChanged);
        RecalculateColumns(grid);
    }
    else
    {
        // unsubscribe — use composite disposable stored on grid via attached property
    }
}
```

Failing to unsubscribe leaks subscriptions when grids leave the tree. Store `IDisposable` on a private attached property and dispose when `IsResponsive` becomes false or the grid is detached.

### AutoPlaceChildren

When true, the handler assigns `Grid.Column` and `Grid.Row` on direct children based on column count — useful for form fields that should reflow without rewriting XAML Grid indices at every breakpoint.

## SkyTooltipProperties

Configures tooltip delay and placement on any control:

```csharp
public static readonly AttachedProperty<int> ShowDelayProperty = ...;
public static readonly AttachedProperty<PlacementMode> PlacementProperty = ...;
```

A changed handler on the attached properties configures the `ToolTip` instance on the target control:

```csharp
private static void OnShowDelayChanged(Control control, AvaloniaPropertyChangedEventArgs e)
{
    ToolTip.SetShowDelay(control, (int)(e.NewValue ?? 400));
}

private static void OnPlacementChanged(Control control, AvaloniaPropertyChangedEventArgs e)
{
    if (e.NewValue is PlacementMode mode)
        ToolTip.SetPlacement(control, mode);
}
```

Prefer wrapping Avalonia's built-in `ToolTip.*` attached properties when possible — SkyUI adds semantic defaults aligned with shell timing tokens (longer delay on touch-first layouts).

Set `ToolTip.Tip` in XAML as usual; Sky properties only adjust behavior:

```xml
<Button Content="Save"
        ToolTip.Tip="Save changes"
        sky:SkyTooltipProperties.ShowDelay="600"
        sky:SkyTooltipProperties.Placement="Top" />
```

## SkyTouchTarget

Mobile primitives enforce minimum 44-pixel touch targets:

```csharp
public static readonly AttachedProperty<bool> EnsureMinSizeProperty =
    AvaloniaProperty.RegisterAttached<Control, bool>(
        "EnsureMinSize",
        typeof(SkyTouchTarget),
        defaultValue: true);

private static void OnEnsureMinSizeChanged(Control control, AvaloniaPropertyChangedEventArgs e)
{
    if (e.NewValue is true)
    {
        control.MinWidth = Math.Max(control.MinWidth, 44);
        control.MinHeight = Math.Max(control.MinHeight, 44);
    }
}
```

This is applied to small icon buttons in mobile layouts to meet accessibility guidelines without changing the visual icon size.

### Visual vs Logical Target

Enlarging `MinWidth`/`MinHeight` expands the hit area and layout slot — icon may look small inside a larger box. Alternative: keep visual size, pad with transparent margin via theme when `EnsureMinSize` is true. SkyUI uses min size for simplicity; custom themes can switch to padding selectors.

Apply selectively — global default `true` on all controls wastes space on desktop:

```xml
<Button Classes="sky-icon"
        sky:SkyTouchTarget.EnsureMinSize="True" />
```

## Advanced: Registering Metadata

Attached properties support the same metadata as styled properties:

```csharp
public static readonly AttachedProperty<bool> IsLoadingProperty =
    AvaloniaProperty.RegisterAttached<Button, bool>(
        "IsLoading",
        typeof(SkyButtonProperties),
        defaultValue: false,
        inherits: false);
```

Optional coerce and validate callbacks catch invalid values at set time rather than in the changed handler:

```csharp
private static bool ValidateDelay(int delay) => delay >= 0;
```

## When to Use Attached Properties vs Subclassing

| Situation | Use |
|-----------|-----|
| Add one behavior to an existing primitive | Attached property |
| Change default styling only | Style class |
| New visual structure with template parts | Subclass or TemplatedControl |
| Cross-cutting concern on many types | Attached property |
| Fundamentally different interaction model | Subclass |

### Anti-Patterns

- **God attached class** — one static class with twenty unrelated properties; split by concern
- **Hidden state without cleanup** — subscriptions, timers, original content not restored
- **Behavior on `AvaloniaObject`** — handlers become no-ops on wrong types; prefer typed owners
- **Duplicating styled property** — if Avalonia already exposes `Grid.Row`, do not wrap unless adding semantics

### Attached Property Checklist

1. Create a static class (not a control)
2. Register with `RegisterAttached<TOwner, TValue>`
3. Provide `Get*` and `Set*` static methods
4. Register a changed handler in the static constructor
5. Document the XAML namespace prefix for consumers
6. Clean up subscriptions and stored state when property clears or control detaches
7. Add XML doc with owner type and default value

## Registering Changed Handlers on Attached Properties

```csharp
static SkyButtonProperties()
{
    IsLoadingProperty.Changed.AddClassHandler<Button>(OnIsLoadingChanged);
}
```

Note the type parameter: `AddClassHandler<Button>` means the handler receives `Button` as the sender. The handler only fires when the property is set on a `Button` instance.

For properties attached to `Control` or `AvaloniaObject`, use the corresponding type parameter:

```csharp
IsResponsiveProperty.Changed.AddClassHandler<Grid>(OnIsResponsiveChanged);
```

### Global Changed Handlers

You can subscribe without a class handler:

```csharp
IsLoadingProperty.Changed.Subscribe(args =>
{
    if (args.Sender is Button b) OnIsLoadingChanged(b, args);
});
```

Class handlers are preferred in SkyUI — less allocation, clearer owner typing.

## Testing Attached Properties

Unit tests construct a `Button`, call `SetIsLoading`, assert `Classes.Contains("sky-loading")`, then clear and assert content restored. Headless Avalonia test hosts materialize enough tree for property system to run.

For theme properties, attach a test `Application`, set accent override, assert resource dictionary contains expected brush key.

## Debugging Attached Properties in DevTools

Avalonia DevTools property grid shows attached values under the owner node. If XAML set fails silently, verify:

- XML namespace maps to the static class assembly
- Property name spelling matches registration name
- Owner type matches — `Grid` property on `StackPanel` ignored

## Summary

Attached properties are SkyUI's primary tool for extending Avalonia primitives without proliferating subclasses. `IsLoading` on buttons, accent override on applications, responsive behavior on grids, and touch targets on mobile controls all follow the same pattern: register, get/set, changed handler, XAML-accessible, cleanup on clear.

Combined with style classes (Chapter 3) and templated controls (Chapter 4), attached properties complete the three mechanisms SkyUI uses to build a cohesive toolkit on top of Avalonia's primitives. Reach for attached properties when behavior is orthogonal to control type; reach for subclasses when the visual tree itself must change.
