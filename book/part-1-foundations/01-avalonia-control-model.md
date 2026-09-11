---
title: Chapter 1 — The Avalonia Control Model
order: 1
---

# Chapter 1: The Avalonia Control Model

Every SkyUI control is an Avalonia control first. Before studying `SkyFormField` or `CheckedListBox`, you need a working mental model of how Avalonia represents UI objects, how properties flow between data and visuals, and how templates turn a C# class into something the user can see and interact with.

This chapter builds that foundation. Later chapters refer back to these ideas constantly.

## Objects, Properties, and the Property System

In Avalonia, almost everything meaningful inherits `AvaloniaObject`. The central idea is the **styled property**: a named, typed slot whose value can come from a default, a style, a binding, or a direct `SetValue` call — with a well-defined precedence order.

When you write:

```csharp
public static readonly StyledProperty<string?> LabelProperty =
    AvaloniaProperty.Register<SkyFormField, string?>(nameof(Label));

public string? Label
{
    get => GetValue(LabelProperty);
    set => SetValue(LabelProperty, value);
}
```

you are doing four things at once:

1. **Registering** a global identity (`LabelProperty`) so the framework can route styles, bindings, and animations to this slot.
2. **Declaring metadata** — type, owner type, default value, binding mode, coercion, validation.
3. **Exposing a CLR wrapper** so C# and XAML can read and write the value naturally.
4. **Enabling change notification** — any change can trigger recalculation, pseudo-class updates, or layout.

### Property Value Precedence

Avalonia resolves the effective value of a styled property from multiple sources. From lowest to highest priority (simplified):

| Source | Example |
|--------|---------|
| Default value | Registered at `AvaloniaProperty.Register` |
| Inherited value | Font family flowing from parent |
| Style setters | `<Setter Property="Padding" Value="8" />` |
| Template bindings | `{TemplateBinding Background}` |
| Local value | `control.Label = "Name"` or XAML attribute |
| Animation | Running transition temporarily overriding value |

Understanding precedence matters when debugging: if your code sets `Background` but the theme style also sets it, the winner depends on whether the value is set locally or only through styling.

### Class Handlers vs Instance Handlers

SkyUI overwhelmingly uses **class handlers** registered in the static constructor:

```csharp
static SkyFormField()
{
    ErrorMessageProperty.Changed.AddClassHandler<SkyFormField>(
        (field, e) => field.SyncErrorState());
}
```

A class handler runs for **every** instance of `SkyFormField` when `ErrorMessage` changes. This is preferable to overriding `OnPropertyChanged` when:

- The reaction is specific to one property
- You do not need to intercept changes before the base class sees them
- You want to keep instance `OnPropertyChanged` thin

The event args (`AvaloniaPropertyChangedEventArgs`) expose `OldValue`, `NewValue`, and `Property`, which is useful when one handler serves multiple related properties.

## Direct Properties for Computed State

Some values should be readable in bindings but not writable from XAML. `SkyPaginationBase.PageCount` is derived from `TotalCount` and `PageSize`:

```csharp
public static readonly DirectProperty<SkyPaginationBase, int> PageCountProperty =
    AvaloniaProperty.RegisterDirect<SkyPaginationBase, int>(
        nameof(PageCount),
        control => control.PageCount);

private int pageCount;
public int PageCount => pageCount;
```

When paging inputs change, the control recalculates `pageCount` and calls `RaisePropertyChanged(PageCountProperty)`. Direct properties are the right tool for **projections** of internal state — never for primary user input.

## Attached Properties

Attached properties belong to a static helper class but can be set on other types:

```csharp
SkyButtonProperties.SetIsLoading(myButton, true);
```

```xml
<Button sky:SkyButtonProperties.IsLoading="{Binding IsSaving}" />
```

They are covered in depth in Chapter 25. For now, remember: attached properties extend existing types without subclassing.

## The Visual Tree and Logical Tree

Avalonia maintains two parallel hierarchies:

**Logical tree** — matches what you declared in XAML. Parent-child relationships used for resources, data context inheritance, and `IsLogicalAncestorOf` checks. When `SkyFormField` hosts a `TextBox` as content, the text box is a logical child of the field.

**Visual tree** — includes everything actually rendered, including objects created inside `ControlTemplate`. The `PART_Label` `TextBlock` inside `SkyFormField`'s template is in the visual tree but not the logical tree as a direct child of the field.

```
Logical:   SkyFormField → TextBox (content)
Visual:    SkyFormField → [template root] → StackPanel → PART_Label, PART_InputHost → TextBox
```

`SkyFormField` subscribes to `OnAttachedToLogicalTree` to find descendant `InputElement`s and wire `LostFocus` for validation. That only works on the **logical** tree — template-internal elements that are not logically attached to the field would be missed if you walked the visual tree incorrectly.

### Visual vs Logical Tree APIs

| API | Tree | Use case |
|-----|------|----------|
| `this.GetVisualDescendants()` | Visual | Find all `SkyAccordionItem` instances under an accordion |
| `this.IsLogicalAncestorOf(child)` | Logical | Check if hosted input belongs to this form field |
| `OnAttachedToVisualTree` | Visual | Start layout-sensitive work (measure bounds) |
| `OnAttachedToLogicalTree` | Logical | Subscribe to content element events |

## TemplatedControl and ControlTemplate

`TemplatedControl` separates **behavior** (C#) from **structure** (XAML template in the theme). The control class declares an API and template contract; the theme provides the visual tree.

### Registering a ControlTheme

In `SkyUI.Themes.Sky`:

```xml
<ControlTheme x:Key="{x:Type controls:SkyFormField}"
              TargetType="controls:SkyFormField">
  <Setter Property="Template">
    <ControlTemplate TargetType="controls:SkyFormField">
      <StackPanel Spacing="4">
        <DockPanel>
          <TextBlock Name="PART_Label" DockPanel.Dock="Left" />
          <TextBlock Name="PART_RequiredIndicator" Text="*" DockPanel.Dock="Right" />
        </DockPanel>
        <TextBlock Name="PART_Hint" Opacity="0.7" />
        <ContentPresenter Name="PART_InputHost"
                          Content="{TemplateBinding Content}" />
        <TextBlock Name="PART_Error" IsVisible="False" />
      </StackPanel>
    </ControlTemplate>
  </Setter>
</ControlTheme>
```

Key bindings:

- `{TemplateBinding Property}` — one-way bind to a property on the **templated control**
- `{Binding}` inside template — uses `DataContext` of the templated control unless overridden
- `Name="PART_*"` — registers the element in the template name scope

### OnApplyTemplate Lifecycle

```csharp
protected override void OnApplyTemplate(TemplateAppliedEventArgs e)
{
    base.OnApplyTemplate(e);

    // 1. Unhook old parts (if re-applying)
    if (_closeButton is not null)
        _closeButton.Click -= OnCloseClick;

    // 2. Resolve new parts
    _closeButton = e.NameScope.Find(CloseButtonPartName) as Button;

    // 3. Hook events
    if (_closeButton is not null)
        _closeButton.Click += OnCloseClick;

    // 4. Push current state to parts
    SyncErrorState();
}
```

`OnApplyTemplate` runs:

- After the control is first loaded with a theme
- When the theme changes at runtime
- When a new `Template` is assigned in code

Always unhook old event handlers before replacing part references. Failure to do so causes duplicate handlers and subtle memory leaks.

### TemplatePart Metadata

```csharp
[TemplatePart(InputHostPartName, typeof(ContentPresenter))]
public class SkyFormField : TemplatedControl
{
    public const string InputHostPartName = "PART_InputHost";
}
```

`[TemplatePart]` is documentation and tooling support — Avalonia does not enforce that the part exists. SkyUI uses `public const string` part names consistently so theme authors and control authors share one contract.

## Pseudo-Classes

Pseudo-classes are boolean flags on a control that style selectors can target. They are not full properties — you do not bind to them from view models.

```csharp
[PseudoClasses("error")]
public class SkyFormField : TemplatedControl
{
    private void SyncErrorState() =>
        PseudoClasses.Set(":error", HasError);
}
```

```xml
<Style Selector="controls|SkyFormField.error /template/ ContentPresenter#PART_InputHost">
  <Setter Property="BorderBrush" Value="{DynamicResource SkyDangerBrush}" />
</Style>
```

Note the selector syntax: `controls|SkyFormField.error` matches the type with pseudo-class `error`. Selectors can drill into template children with `/template/`.

Pseudo-classes are ideal for **derived visual state** — error, expanded, resizing, compact — that consumers should not set directly.

## Style Classes (CSS Classes)

`control.Classes` is a set of string tags, similar to CSS classes:

```csharp
public SkyCard()
{
    Classes.Add("sky");
    Classes.Add("sky-card");
}

private void SyncHoverClass() =>
    Classes.Set("sky-card-hoverable", IsHoverable);
```

`Classes.Set(name, enabled)` adds or removes a single class atomically. Theme selectors use `.classname`:

```xml
<Style Selector="controls|SkyCard.sky-card-hoverable">
  <Setter Property="BoxShadow" Value="{DynamicResource SkyElevationMd}" />
</Style>
```

SkyUI uses style classes for **configurable variants** (primary button, outlined chip) and pseudo-classes for **runtime state** (error, open, resizing).

## Routed Events

Routed events bubble or tunnel through the visual tree:

```csharp
public static readonly RoutedEvent<RoutedEventArgs> DeleteRequestedEvent =
    RoutedEvent.Register<Chip, RoutedEventArgs>(
        nameof(DeleteRequested), RoutingStrategies.Bubble);

public event EventHandler<RoutedEventArgs>? DeleteRequested
{
    add => AddHandler(DeleteRequestedEvent, value);
    remove => RemoveHandler(DeleteRequestedEvent, value);
}

// Raising:
RaiseEvent(new RoutedEventArgs(DeleteRequestedEvent));
```

| Strategy | Direction | Typical use |
|----------|-----------|-------------|
| Bubble | Child → parent | Button click, item expanded |
| Tunnel | Parent → child | Rare in SkyUI |
| Direct | Source only | Internal handling |

`SkyAccordion` listens for bubbling `ExpandedEvent` from child items to enforce single-selection mode without tight coupling.

Set `e.Handled = true` on routed events when you want to stop further propagation — `Chip` does this on delete button click to prevent also firing `ChipClick`.

## Content Property and Child Collections

`[Content]` marks the default XAML child property:

```csharp
[Content]
public object? Content { get; set; }
```

```xml
<SkyFormField Label="Email">
  <TextBox />   <!-- becomes Content -->
</SkyFormField>
```

For collections, `[Content]` on `IList Items` allows:

```xml
<SkyNavigationView>
  <SkyNavigationViewItem Content="Home" />
  <SkyNavigationViewItem Content="Settings" />
</SkyNavigationView>
```

## Data Binding Modes

When registering input properties, set the default binding mode explicitly:

```csharp
AvaloniaProperty.Register<SkyAutocomplete, string?>(
    nameof(Text), defaultBindingMode: BindingMode.TwoWay);
```

| Mode | Behavior |
|------|----------|
| OneWay | Source → control |
| TwoWay | Source ↔ control |
| OneTime | Source → control once |
| OneWayToSource | Control → source only |

SkyUI uses TwoWay for user-editable values (`Text`, `SelectedItem`, `Value`, `IsOpen`) and OneWay for display-only projections unless the consumer overrides in XAML.

## Re-Owning Properties from Base Types

`SkySlider` does not reimplement slider math. It **re-owns** properties from `RangeBase` and `Slider`:

```csharp
public static readonly StyledProperty<double> ValueProperty =
    RangeBase.ValueProperty.AddOwner<SkySlider>();
```

`AddOwner<T>()` registers the same property on a derived type so styles and bindings target `SkySlider.Value` while Avalonia's built-in `Slider` inside the template can bind to the same property on the outer control via `{TemplateBinding Value}`.

This pattern keeps composite controls thin: outer API surface matches inner primitive behavior.

## Measure, Arrange, and Layout (Brief)

Even templated controls participate in layout. The panel system calls:

1. **Measure** — child reports desired size
2. **Arrange** — parent assigns final bounds

`SkyResponsiveGrid` overrides `OnPropertyChanged` for `BoundsProperty` because when the control's allocated width changes, column count must update. Custom `Panel` subclasses (Chapter 27) override `MeasureOverride` and `ArrangeOverride` directly.

## Summary Checklist for Control Authors

| Concept | SkyUI usage |
|---------|-------------|
| StyledProperty | All bindable API |
| DirectProperty | PageCount, computed flags |
| Class handler | Property → pseudo-class / class sync |
| OnApplyTemplate | Wire PART_* elements |
| Pseudo-classes | Error, expanded, open, compact |
| Style classes | sky-*, variant names |
| Routed events | Delete, close, selection changed |
| Logical tree hooks | Form field validation on focus loss |
| AddOwner | SkySlider value, SkyIcon foreground |

With this model in place, the next chapter maps how SkyUI organizes dozens of controls into composable packages.
