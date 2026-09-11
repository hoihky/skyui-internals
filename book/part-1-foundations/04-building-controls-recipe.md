---
title: Chapter 4 — The Control-Building Recipe
order: 4
---

# Chapter 4: The Control-Building Recipe

This chapter distills the patterns from Part I into a repeatable recipe. Follow these steps whenever you add a new control to SkyUI or build a custom control in your own Avalonia application using SkyUI conventions. Each step connects to Avalonia's property system, template model, and the visual/logical tree split introduced in Chapter 1.

## Step 1: Choose the Base Class

Ask three questions:

1. Does the control need a custom visual tree with named parts? → `TemplatedControl`
2. Does it host a single child or a collection? → `ContentControl` or `ItemsControl`
3. Can styling alone achieve the goal? → Subclass an Avalonia primitive or use style classes

Examples from SkyUI:

```csharp
// Complex custom chrome with named parts
public class SkyFormField : TemplatedControl { }

// Simple container with header/footer
public class SkyCard : ContentControl { }

// Collection with custom item containers
public class SkyAccordion : ItemsControl { }

// Thin styling wrapper — no custom template in C#
public class SkySearchBox : TextBox
{
    public SkySearchBox() => Classes.Add("sky-search");
}
```

| Base class | Visual tree | When to choose |
|------------|-------------|----------------|
| `TemplatedControl` | Defined entirely in `ControlTheme` | Custom chrome, named `PART_*` elements, pointer handling on parts |
| `ContentControl` | Theme template + one logical content child | Cards, panels, hosts with optional header/footer |
| `ItemsControl` | Theme template + item container generation | Lists, trees, accordions with repeated items |
| Primitive subclass | Base type's template + style classes | Search boxes, read-only labels with preset behavior |
| Primitive + attached property | No new type | Loading buttons, responsive grids on existing `Grid` |

If you reach for `TemplatedControl` but only need a colored border around arbitrary content, consider `ContentControl` with a theme template instead. Less C# surface area, same visual result.

## Step 2: Define the Public API

Declare bindable properties with `StyledProperty<T>`:

```csharp
public static readonly StyledProperty<string?> LabelProperty =
    AvaloniaProperty.Register<MyControl, string?>(nameof(Label));

public string? Label
{
    get => GetValue(LabelProperty);
    set => SetValue(LabelProperty, value);
}
```

Guidelines:

- Use `string?` for text that can be empty
- Set `defaultBindingMode: BindingMode.TwoWay` on input properties (`Text`, `SelectedIndex`, `IsExpanded`)
- Use `DirectProperty<T>` for computed read-only state (`PageCount`, `HasError`)
- Mark child content properties with `[Content]` so XAML nesting works: `<MyControl><TextBox /></MyControl>`

Register property changed handlers in the static constructor for reactions that should run on every instance:

```csharp
static MyControl()
{
    IsActiveProperty.Changed.AddClassHandler<MyControl>(
        (c, _) => c.UpdateVisualState());
}
```

### Property System Checklist

When defining each property, decide:

1. **Default value** — Register inline or accept `default(T)`
2. **Binding mode** — OneWay for display, TwoWay for inputs
3. **Changed handler** — Class handler for visual state; instance override only when you must call `base` first
4. **Coercion** — Rare in SkyUI; use when values must stay in range (e.g., clamped numeric step)

Avoid creating styled properties for values that belong in the view model unless the control genuinely owns that state (expanded/collapsed on an accordion item is control state; user email is not).

## Step 3: Declare the Template Contract

Name template parts as public constants:

```csharp
public const string RootPartName = "PART_Root";
public const string LabelPartName = "PART_Label";

[TemplatePart(RootPartName, typeof(Border))]
[TemplatePart(LabelPartName, typeof(TextBlock))]
[PseudoClasses("active", "disabled")]
public class MyControl : TemplatedControl
```

Declare pseudo-classes that the theme will style. Toggle them from property handlers:

```csharp
private void UpdateVisualState()
{
    PseudoClasses.Set(":active", IsActive);
    PseudoClasses.Set(":disabled", !IsEnabled);
}
```

Add style classes in the constructor for base styling:

```csharp
public MyControl()
{
    Classes.Add("sky");
    Classes.Add("sky-my-control");
}
```

### Style Classes vs Pseudo-Classes

SkyUI uses both deliberately:

- **Style classes** (`sky-card-hoverable`) — Configurable or persistent modes the app sets explicitly
- **Pseudo-classes** (`:horizontal`, `:disabled`, `:active`) — Derived from property values or interaction state

If the state is computed from a property, use pseudo-classes so theme selectors stay declarative. If the state is a feature flag (`IsHoverable`), use a style class.

## Step 4: Wire Template Parts in OnApplyTemplate

Templates are applied whenever the theme loads, the theme changes, or `Template` is replaced. `OnApplyTemplate` is your hook into the **visual tree**:

```csharp
private TextBlock? _label;
private EventHandler<RoutedEventArgs>? _clickHandler;

protected override void OnApplyTemplate(TemplateAppliedEventArgs e)
{
    base.OnApplyTemplate(e);

    // Detach from old visual tree parts
    if (_label is not null && _clickHandler is not null)
        _label.PointerPressed -= _clickHandler;

    _label = e.NameScope.Find(LabelPartName) as TextBlock;

    if (_label is not null)
    {
        _label.Text = Label;
        _clickHandler = OnLabelPressed;
        _label.PointerPressed += _clickHandler;
    }

    SyncAllState();
}
```

Rules:

- Always call `base.OnApplyTemplate` first
- Re-resolve parts every time the template is applied
- Unsubscribe event handlers from old parts before replacing references
- Call a `SyncAllState()` method to push current property values to parts and refresh pseudo-classes

### Visual Tree vs Logical Tree Here

Template parts live in the **visual tree** only. They are not logical children of your control. Use `e.NameScope.Find("PART_Label")` — not `this.FindControl<TextBlock>("PART_Label")` from logical APIs — unless the part is also declared in page XAML (it should not be).

If you need to find a logical child (content inside a `ContentControl`), use logical APIs in `OnApplyTemplate` or after `Content` changes, not `NameScope`.

## Step 5: Implement Interaction with Routed Events

For actions that parents should be able to handle:

```csharp
public static readonly RoutedEvent<RoutedEventArgs> ClickedEvent =
    RoutedEvent.Register<MyControl, RoutedEventArgs>(
        nameof(Clicked), RoutingStrategies.Bubble);

public event EventHandler<RoutedEventArgs>? Clicked
{
    add => AddHandler(ClickedEvent, value);
    remove => RemoveHandler(ClickedEvent, value);
}

private void OnClick()
{
    RaiseEvent(new RoutedEventArgs(ClickedEvent));
}
```

Bubble routing lets parent containers handle child actions — the same pattern `Button` uses for `Click`. Use `RoutingStrategies.Tunnel` only when you need preview semantics (rare in SkyUI layout controls).

For commands, expose `ICommand` properties and let the template bind a `Button` to them. Keep event raising for controls that mirror platform primitives.

## Step 6: Create the ControlTheme

In `SkyUI.Themes.Sky`, add a `ControlTheme`:

```xml
<ControlTheme x:Key="{x:Type controls:MyControl}"
              TargetType="controls:MyControl">
  <Setter Property="Background" Value="{DynamicResource SkySurfaceBrush}" />
  <Setter Property="Template">
    <ControlTemplate>
      <Border Name="PART_Root"
              Background="{TemplateBinding Background}"
              CornerRadius="{DynamicResource SkyRadiusMd}"
              Padding="{DynamicResource SkySpace16Px}">
        <TextBlock Name="PART_Label"
                   Text="{TemplateBinding Label}"
                   Foreground="{DynamicResource SkyTextPrimaryBrush}" />
      </Border>
    </ControlTemplate>
  </Setter>
</ControlTheme>
```

Use `{TemplateBinding PropertyName}` for properties defined on the control. Use `{DynamicResource TokenKey}` for design tokens.

### TemplateBinding vs Binding

| Markup | Source | Use for |
|--------|--------|---------|
| `{TemplateBinding Label}` | Control's styled property | Template chrome tied to control API |
| `{Binding Name}` | Data context | Content inside a `ContentPresenter` showing view model data |
| `{DynamicResource SkyAccentBrush}` | Resource dictionary | Theme tokens |

Mixing these incorrectly is a top source of "binding works in demo but not in app" bugs.

## Step 7: Add Style Variants

In the paired `.Styles.axaml` file:

```xml
<Style Selector="controls|MyControl.active">
  <Setter Property="BorderBrush" Value="{DynamicResource SkyAccentBrush}" />
</Style>

<Style Selector="controls|MyControl.disabled /template/ TextBlock#PART_Label">
  <Setter Property="Foreground" Value="{DynamicResource SkyTextDisabledBrush}" />
</Style>
```

Style selectors can target the control (`controls|MyControl.active`) or pierce the template (`/template/ TextBlock#PART_Label`). Prefer control-level setters when they map to styled properties; use template piercing for elements without a corresponding control property.

## Step 8: Register in the Theme Entry Point

Add an include in `ContentFirstDark.axaml`:

```xml
<ResourceDictionary.MergedDictionaries>
  <ResourceInclude Source="Controls/MyControl.axaml" />
</ResourceDictionary.MergedDictionaries>

<StyleInclude Source="Controls/MyControl.Styles.axaml" />
```

Order matters only when two resources share the same key — later merges win. SkyUI avoids key collisions by namespacing per control file.

## Step 9: Add a Demo Page and Tests

Create a demo page in `SkyUI.Demo` showing typical usage, edge cases, and binding scenarios:

```xml
<!-- Typical -->
<MyControl Label="Account name" />

<!-- Empty / null -->
<MyControl Label="{x:Null}" />

<!-- Two-way binding -->
<MyControl IsActive="{Binding IsEditing, Mode=TwoWay}" />

<!-- Disabled state -->
<MyControl IsEnabled="False" Label="Read only" />
```

Add headless tests that:

- Verify template parts resolve after `ApplyTemplate()`
- Verify property changes update pseudo-classes
- Verify routed events fire with correct routing
- Verify `Classes` contains expected `sky-*` entries

Headless tests run without a GPU; they are ideal for property and pseudo-class logic.

## Walkthrough: Build `SkyToggleChip` End to End

This condensed walkthrough ties all steps together. `SkyToggleChip` is a selectable pill with label text and `:selected` pseudo-class.

**Step 1 — Base class:** `TemplatedControl` (custom template, pointer input on root)

**Step 2 — Properties:**

```csharp
public static readonly StyledProperty<string?> ContentProperty = ...;
public static readonly StyledProperty<bool> IsSelectedProperty =
    AvaloniaProperty.Register<SkyToggleChip, bool>(
        nameof(IsSelected), defaultBindingMode: BindingMode.TwoWay);
```

**Step 3 — Contract:**

```csharp
[TemplatePart(RootPartName, typeof(Border))]
[PseudoClasses("selected")]
public class SkyToggleChip : TemplatedControl
{
    static SkyToggleChip()
    {
        IsSelectedProperty.Changed.AddClassHandler<SkyToggleChip>(
            (c, _) => c.UpdateVisualState());
    }

    private void UpdateVisualState() =>
        PseudoClasses.Set(":selected", IsSelected);
}
```

**Step 4 — Template wiring:** Find `PART_Root`, attach `PointerPressed` to toggle `IsSelected`

**Step 5 — Optional `SelectionChanged` routed event** if used inside a parent group

**Step 6 — ControlTheme** with `{TemplateBinding Content}`, token brushes, `SkyRadiusFull`

**Step 7 — Styles** for `:selected` background using `SkySelectedTintBrush`

**Step 8 — Register** in `ContentFirstDark.axaml`

**Step 9 — Demo + test** asserting `:selected` follows `IsSelected`

Real SkyUI controls like `SkyChip` follow this same arc with more variant classes.

## Optional: Adapter Interface for Data-Driven Controls

When your control displays hierarchical or virtualized data that can come from many model types, define an adapter interface:

```csharp
public interface IMyItemAdapter
{
    string GetLabel(object item);
    IEnumerable? GetChildren(object item);
    bool GetIsExpanded(object item);
    void SetIsExpanded(object item, bool expanded);
}
```

The control holds a `StyledProperty<IMyItemAdapter?>` with a default implementation. Consumers plug in their own adapter for domain models. `CheckedListBox` is the reference implementation of this pattern.

Adapters keep the control free of domain types. The visual tree still comes from templates; only the **data** feeding `ItemsControl` changes.

## Optional: Abstract Base for Shared Logic

When two controls share paging math, validation, or command wiring, extract an abstract base:

```csharp
public abstract class SkyPaginationBase : TemplatedControl
{
    // shared properties, computed PageCount, navigation commands
}

public class SkyPagination : SkyPaginationBase { /* UI-specific parts */ }
public class SkyDataPager : SkyPaginationBase { /* data-bound variant */ }
```

Keep the base class free of template parts that only one subclass needs. Shared bases own **algorithms**; subclasses own **template contracts**.

## ItemsControl-Specific Notes

If your control inherits `ItemsControl`:

1. Override `ContainerForItemOverride` to return your item container type (`SkyAccordionItem`, etc.)
2. Override `IsItemItsOwnContainerOverride` to reuse existing containers
3. Define a separate `ControlTheme` for the item container type
4. Use `ItemsPanel` in theme XAML for stack vs wrap layout

Item containers are separate controls with their own template parts. Do not cram item logic into the parent `ItemsControl` class.

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Skip `base.OnApplyTemplate` | Base template setup missing | Call base first |
| Cache template parts without re-fetch | Old template handlers fire after theme change | Re-resolve in every `OnApplyTemplate` |
| Use `FindControl` for `PART_*` | Part not found | Use `e.NameScope.Find` |
| `{Binding}` on `PART_Label` | Shows view model field, not `Label` property | Use `{TemplateBinding Label}` |
| Property logic in theme triggers | Untestable, inconsistent with SkyUI | Class handler in C# |
| No `sky` style class | Control unstyled if theme selector expects it | Add in constructor |
| `TemplatedControl` for pure styling | Unnecessary API surface | Style classes on primitive |

## Debugging Tips

**Template part is null after `OnApplyTemplate`.** The `Name` in XAML does not match the constant, or the `ControlTheme` is not registered (missing merge in `ContentFirstDark.axaml`).

**Pseudo-class style never applies.** Confirm C# calls `PseudoClasses.Set(":name", condition)` with the leading colon, and the style selector uses `controls|MyControl.name` without colon.

**Property changes but UI stale.** You updated a C# field but not the template part — push values in a shared `SyncAllState()` called from both property handlers and `OnApplyTemplate`.

**Memory leak warnings.** Event handlers on template parts were not removed before re-template. Unsubscribe in `OnApplyTemplate` before replacing part references.

**Double layout or flicker.** Property changed handler performs expensive work on every keystroke. Debounce or guard with equality checks.

## Checklist

| Step | Done? |
|------|-------|
| Base class chosen | |
| Styled properties defined with correct binding modes | |
| Template parts and pseudo-classes declared | |
| OnApplyTemplate wires parts and unsubscribes old handlers | |
| Style classes added in constructor | |
| ControlTheme uses token keys only | |
| Style variants for pseudo-classes and size variants | |
| Theme registered in ContentFirstDark | |
| Demo page covers typical, empty, bound, disabled cases | |
| Headless tests for parts, pseudo-classes, events | |

## Summary

Building a SkyUI control is a disciplined sequence: pick the right base class, define a bindable API on styled properties, separate visuals into theme XAML, and connect the two through template parts, pseudo-classes, and `OnApplyTemplate`. Understand where the visual tree ends and the logical tree begins, use template bindings for control chrome, and reserve data bindings for content presenters.

The remaining parts of this book apply this recipe to real controls across every major category — layout, forms, lists, navigation, and data.
