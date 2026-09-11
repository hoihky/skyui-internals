---
title: Chapter 8 — SkyFormField
order: 8
---

# Chapter 8: SkyFormField — The Form Wrapper Pattern

Forms are the most common place where custom controls add value. Raw `TextBox` and `ComboBox` elements work, but product-quality forms need labels, hints, required indicators, and error messages. `SkyFormField` wraps any input control with this chrome and demonstrates composition, validation strategy, and logical-tree event wiring.

In Avalonia, a well-designed form wrapper sits at the intersection of three concerns: **layout** (how label and input align), **behavior** (when validation runs), and **styling** (how error states appear). `SkyFormField` keeps those concerns separate so each can evolve independently.

## The Problem SkyFormField Solves

Without a wrapper, every form field in XAML repeats the same structure:

```xml
<StackPanel Spacing="4">
  <TextBlock Text="Email" />
  <TextBox Watermark="you@example.com" />
  <TextBlock Text="Required" Foreground="Red" IsVisible="False" />
</StackPanel>
```

Validation error display, required state, and focus-triggered validation must be duplicated or managed in the view model for every field. `SkyFormField` centralizes this into one control.

The repetition problem gets worse as forms grow. A settings screen with twenty fields means twenty copies of label spacing, twenty error visibility bindings, and twenty places where accessibility attributes must be wired. When design changes the error color or label font, you touch every field. `SkyFormField` makes that a single theme update.

## Architecture Overview

```
┌─────────────────────────────────────┐
│ SkyFormField (TemplatedControl)     │
│  ┌───────────────────────────────┐  │
│  │ PART_Label                    │  │
│  │ PART_RequiredIndicator        │  │
│  │ PART_Hint                     │  │
│  ├───────────────────────────────┤  │
│  │ PART_InputHost                │  │
│  │   └── any child input control │  │
│  ├───────────────────────────────┤  │
│  │ PART_Error                    │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

The input is not hard-coded. `PART_InputHost` is a `ContentPresenter` that accepts any child control.

### Why TemplatedControl?

`SkyFormField` inherits `TemplatedControl` rather than `UserControl` because the visual tree must be swappable through themes. A `UserControl` bakes its XAML into the assembly; a `TemplatedControl` loads its visual tree from a `ControlTheme` at runtime. This means consumers can restyle every form field in an app by overriding one theme key, without subclassing.

The template also uses `TemplateBinding` to connect parts to properties. `PART_Label` binds to `Label`, `PART_Hint` binds to `Hint`, and `PART_Error` binds to `ErrorMessage`. The C# class never assigns `Text` on those elements directly.

## The Property API

```csharp
[PseudoClasses("error")]
[TemplatePart(InputHostPartName, typeof(ContentPresenter))]
public class SkyFormField : TemplatedControl
{
    public static readonly StyledProperty<string?> LabelProperty = ...;
    public static readonly StyledProperty<string?> HintProperty = ...;
    public static readonly StyledProperty<string?> ErrorMessageProperty = ...;
    public static readonly StyledProperty<bool> IsRequiredProperty = ...;
    public static readonly StyledProperty<bool> ValidateOnLostFocusProperty = ...;
    public static readonly StyledProperty<ISkyValidator?> ValidatorProperty = ...;
    public static readonly StyledProperty<object?> ContentProperty =
        ContentControl.ContentProperty.AddOwner<SkyFormField>();

    [Content]
    public object? Content { get; set; }
}
```

`ContentProperty` is **added** from `ContentControl` rather than re-declared. This is an Avalonia pattern for sharing the same property definition across types. The `[Content]` attribute lets consumers write:

```xml
<SkyFormField Label="Email" IsRequired="True">
  <TextBox Watermark="you@example.com" />
</SkyFormField>
```

Without `[Content]`, consumers would need explicit property syntax: `<SkyFormField Content="...">`, which reads awkwardly for a container control.

### StyledProperty vs DirectProperty

Most `SkyFormField` properties are `StyledProperty<T>` because they participate in styling, binding, and inheritance. Read-only computed values like `HasError` are plain CLR properties backed by `ErrorMessage`. If you later need to bind to `HasError` from XAML, promote it to a `DirectProperty<T>` with a backing field and call `RaisePropertyChanged` when the source changes.

## Error State via Pseudo-Classes

```csharp
static SkyFormField()
{
    ErrorMessageProperty.Changed.AddClassHandler<SkyFormField>(
        (f, _) => f.SyncErrorState());
}

public bool HasError => !string.IsNullOrWhiteSpace(ErrorMessage);

private void SyncErrorState() => PseudoClasses.Set(":error", HasError);
```

When `ErrorMessage` is set, the `:error` pseudo-class activates. Theme styles change border color, error text visibility, and focus ring color. The C# class never touches brushes directly.

### Theme Selectors for Error State

In your `ControlTheme`, target the error pseudo-class on both the field and the hosted input:

```xml
<Style Selector="controls|SkyFormField:error">
  <Setter Property="BorderBrush" Value="{DynamicResource SkyDangerBrush}" />
</Style>

<Style Selector="controls|SkyFormField:error /template/ ContentPresenter#PART_Error">
  <Setter Property="IsVisible" Value="True" />
</Style>

<Style Selector="controls|SkyFormField:error /template/ TextBox">
  <Setter Property="BorderBrush" Value="{DynamicResource SkyDangerBrush}" />
</Style>
```

The descendant selector (`/template/ TextBox`) reaches into the template and styles the hosted input when the parent field is in error. This is how a wrapper control can theme its children without the children knowing about the wrapper.

## Validation Strategy

`ISkyValidator` is a simple strategy interface:

```csharp
public interface ISkyValidator
{
    SkyValidationResult Validate(object? value);
}

public readonly struct SkyValidationResult
{
    public bool IsValid { get; }
    public string? ErrorMessage { get; }
}
```

Built-in validators live in `SkyValidators`:

```csharp
public static class SkyValidators
{
    public static ISkyValidator Required(string message = "This field is required.") =>
        new RequiredValidator(message);

    public static ISkyValidator Email(string message = "Enter a valid email address.") =>
        new EmailValidator(message);

    public static ISkyValidator MinLength(int min, string? message = null) =>
        new MinLengthValidator(min, message);
}
```

Composite validation chains multiple validators:

```csharp
var validator = new CompositeSkyValidator(
    SkyValidators.Required(),
    SkyValidators.Email());

// Assign to field
field.Validator = validator;
```

The `Validate` method on the field:

```csharp
public bool Validate()
{
    if (Validator is null)
        return !HasError;

    var result = Validator.Validate(GetInputValue());
    ErrorMessage = result.IsValid ? null : result.ErrorMessage;
    return result.IsValid;
}
```

Validation is opt-in. Without a `Validator`, the field only shows errors set manually through `ErrorMessage` binding.

### When to Validate: Three Triggers

`SkyFormField` supports three validation moments, and you can combine them:

| Trigger | Mechanism | Best for |
|---------|-----------|----------|
| Lost focus | `ValidateOnLostFocus="True"` | Inline feedback after the user leaves a field |
| Manual | Call `field.Validate()` from code | Submit button, tab change |
| Binding | Set `ErrorMessage` from view model | Server-side validation results |

Lost-focus validation feels responsive without being aggressive. Submit-time validation catches cross-field rules (password confirmation). View-model-driven errors handle async server responses.

### Writing a Custom Validator

Validators are plain classes. A phone number validator might look like:

```csharp
public sealed class PhoneValidator : ISkyValidator
{
    private static readonly Regex Pattern = new(@"^\+?[\d\s\-()]{7,20}$");

    public SkyValidationResult Validate(object? value)
    {
        var text = value as string;
        if (string.IsNullOrWhiteSpace(text))
            return SkyValidationResult.Valid; // let Required handle empties

        return Pattern.IsMatch(text)
            ? SkyValidationResult.Valid
            : SkyValidationResult.Invalid("Enter a valid phone number.");
    }
}
```

Register it in XAML through a static resource or assign in code:

```csharp
phoneField.Validator = new CompositeSkyValidator(
    SkyValidators.Required(),
    new PhoneValidator());
```

Keep validators stateless when possible. If a validator needs configuration (min length, regex pattern), pass those values through the constructor and store them in readonly fields.

## Reading Values from Heterogeneous Inputs

`GetInputValue` is the most instructive method in the class. It pattern-matches on the hosted control type:

```csharp
public object? GetInputValue()
{
    if (Content is not Control input)
        return Content;

    return input switch
    {
        SkySearchBox searchBox => searchBox.Text,
        SkyPasswordBox passwordBox => passwordBox.Text,
        SkyMaskedTextBox maskedTextBox => maskedTextBox.RawText,
        SkyAutocomplete autocomplete => autocomplete.SelectedItem ?? autocomplete.Text,
        SkyNumericUpDown numericUpDown => numericUpDown.Value,
        TextBox textBox => textBox.Text,
        CheckBox checkBox => checkBox.IsChecked,
        ComboBox comboBox => comboBox.SelectedItem,
        _ => input.GetValue(TextBox.TextProperty) is string text ? text : input
    };
}
```

This is the **adapter pattern applied locally**. Instead of requiring consumers to implement an interface, the form field knows how to extract values from the input types SkyUI supports. When you add a new input control, extend this switch expression.

The fallback case attempts to read `TextBox.TextProperty` from any control that exposes it, which covers thin TextBox subclasses.

### Alternative: Value Extractor Registry

For larger control libraries, a switch expression becomes unwieldy. An alternative is a registry of `Func<Control, object?>` delegates keyed by type:

```csharp
private static readonly Dictionary<Type, Func<Control, object?>> Extractors = new()
{
    [typeof(TextBox)] = c => ((TextBox)c).Text,
    [typeof(CheckBox)] = c => ((CheckBox)c).IsChecked,
};

public static void RegisterExtractor<T>(Func<T, object?> extractor)
    where T : Control =>
    Extractors[typeof(T)] = c => extractor((T)c);
```

SkyUI uses the switch for clarity at its current scale. If you expect dozens of input types, consider the registry pattern.

## Lost-Focus Validation via Logical Tree

```csharp
protected override void OnAttachedToLogicalTree(LogicalTreeAttachmentEventArgs e)
{
    base.OnAttachedToLogicalTree(e);
    if (e.Source is InputElement input && IsDescendantInput(input))
        input.LostFocus += OnInputLostFocus;
}

protected override void OnDetachedFromLogicalTree(LogicalTreeAttachmentEventArgs e)
{
    if (e.Source is InputElement input && IsDescendantInput(input))
        input.LostFocus -= OnInputLostFocus;
    base.OnDetachedFromLogicalTree(e);
}

private void OnInputLostFocus(object? sender, RoutedEventArgs e)
{
    if (ValidateOnLostFocus)
        Validate();
}
```

Rather than subscribing to a specific `TextBox` in `OnApplyTemplate`, the field listens to the logical tree. Any `InputElement` added as a descendant automatically gets a `LostFocus` handler. This works regardless of how deeply nested the input is inside the content.

`IsDescendantInput` excludes the field itself:

```csharp
private bool IsDescendantInput(InputElement input) =>
    input != this && this.IsLogicalAncestorOf(input);
```

### Logical Tree vs Visual Tree

Avalonia maintains two parallel trees. The **logical tree** reflects the XAML structure and data flow — parent-child relationships that matter for inheritance and routing. The **visual tree** includes template-generated elements like `PART_Label` and `PART_InputHost`.

`OnAttachedToLogicalTree` fires when any element joins the logical tree beneath this control. That includes the `TextBox` you place as content, but not the template parts created inside `OnApplyTemplate`. This is exactly what you want: you care about the consumer's input, not internal chrome.

### Debugging Focus and Validation

If lost-focus validation never fires, check these common causes:

1. **`ValidateOnLostFocus` is false** — the default may be false; set it explicitly in XAML.
2. **The hosted control is not an `InputElement`** — custom controls must inherit `InputElement` (or `TemplatedControl` which does) to receive focus events.
3. **Focus moves within the field** — clicking from a `TextBox` to a button inside the same `SkyFormField` may not trigger validation if focus stays within the field's visual subtree. For most forms this is acceptable; use submit-time validation for stricter checks.
4. **Handler leak** — if you override tree attachment methods, always unsubscribe in `OnDetachedFromLogicalTree`. Missing unsubscription causes validation on disposed controls.

Use Avalonia DevTools to inspect the logical tree and confirm your input appears as a descendant of `SkyFormField`.

## SkyComboBoxField: Inheritance for Specialization

`SkyComboBoxField` extends `SkyFormField` and embeds a styled `ComboBox`:

```csharp
public class SkyComboBoxField : SkyFormField
{
    public static readonly StyledProperty<IEnumerable?> ItemsSourceProperty = ...;
    public static readonly StyledProperty<object?> SelectedItemProperty = ...;

    public SkyComboBoxField()
    {
        var combo = new ComboBox { Classes = { "sky" } };
        combo.SelectionChanged += (_, _) =>
            SetValue(SelectedItemProperty, combo.SelectedItem);
        Content = combo;
    }
}
```

This shows when inheritance is appropriate: the combo box field always hosts the same input type, so embedding it in the constructor is cleaner than requiring XAML children.

The subclass forwards `ItemsSource` and `SelectedItem` to the inner combo through property changed handlers in `OnApplyTemplate` or through styled property changed callbacks. Consumers bind to the field's properties, not the inner combo:

```xml
<SkyComboBoxField Label="Country"
                  ItemsSource="{Binding Countries}"
                  SelectedItem="{Binding SelectedCountry, Mode=TwoWay}" />
```

## Accessibility Considerations

Form fields are high-traffic controls for assistive technology. `SkyFormField` should wire:

- `AutomationProperties.Name` on the input from `Label`
- `AutomationProperties.HelpText` from `Hint`
- Error text exposed through `AutomationProperties.LabeledBy` or live region updates

In the template, associate the label with the input:

```xml
<TextBlock x:Name="PART_Label"
           Target="{Binding #PART_InputHost}" />
```

Avalonia's `AccessKey` and `KeyBindings` on the label can focus the input when the user presses Alt+letter. These details belong in the theme template, not scattered across every form in the app.

## Usage Examples

**Basic text field with validation:**

```xml
<SkyFormField Label="Username"
              Hint="3–20 characters"
              IsRequired="True"
              ValidateOnLostFocus="True"
              Validator="{x:Static local:MyValidators.Username}">
  <TextBox MaxLength="20" />
</SkyFormField>
```

**View-model-driven error:**

```xml
<SkyFormField Label="Email"
              ErrorMessage="{Binding EmailError}">
  <TextBox Text="{Binding Email, Mode=TwoWay}" />
</SkyFormField>
```

**Password field with custom input:**

```xml
<SkyFormField Label="Password" IsRequired="True">
  <SkyPasswordBox Text="{Binding Password, Mode=TwoWay}" />
</SkyFormField>
```

**Form-wide validation on submit:**

```csharp
bool allValid = formFields.All(f => f.Validate());
if (!allValid) return;
await SaveAsync();
```

Collect field references by walking the visual tree or by registering them in a list during view initialization:

```csharp
private readonly List<SkyFormField> _fields = new();

private void RegisterFields(Control root)
{
    foreach (var field in root.GetVisualDescendants().OfType<SkyFormField>())
        _fields.Add(field);
}

private bool ValidateAll() => _fields.All(f => f.Validate());
```

The explicit list approach is faster for large forms than tree walks on every submit.

## Building Your Own Form Wrapper

If you create a form wrapper for your own toolkit:

1. Accept content through `[Content]` on a `ContentPresenter` part
2. Keep validation logic in strategy objects, not in the wrapper
3. Use pseudo-classes for error/active/disabled visual states
4. Extract values with a type switch or a registry of value extractors
5. Subscribe to logical tree events for focus-based validation
6. Wire accessibility properties in the template, not in consumer XAML
7. Test with keyboard-only navigation and screen reader announcements

### Walkthrough: Minimal Form Field from Scratch

Step 1 — Create the control class with properties:

```csharp
public class MyFormField : TemplatedControl
{
    public static readonly StyledProperty<string?> LabelProperty =
        AvaloniaProperty.Register<MyFormField, string?>(nameof(Label));

    public static readonly StyledProperty<string?> ErrorMessageProperty =
        AvaloniaProperty.Register<MyFormField, string?>(nameof(ErrorMessage));

    public static readonly StyledProperty<object?> ContentProperty =
        ContentControl.ContentProperty.AddOwner<MyFormField>();

    [Content]
    public object? Content
    {
        get => GetValue(ContentProperty);
        set => SetValue(ContentProperty, value);
    }
}
```

Step 2 — Define the `ControlTheme` with template parts.

Step 3 — Add pseudo-class sync in the static constructor.

Step 4 — Add `Validate()` and `GetInputValue()` methods.

Step 5 — Test with a `TextBox`, a `ComboBox`, and a custom control to verify value extraction.

## Design Patterns Summary

| Pattern | Where |
|---------|-------|
| Composition | Input hosted in `ContentPresenter` |
| Strategy | `ISkyValidator` pluggable validation |
| Pseudo-class | `:error` drives theme |
| Logical tree hook | Lost-focus on any descendant input |
| Template part | Named presenters for label, hint, error |
| Inheritance | `SkyComboBoxField` for fixed input type |
| Adapter | `GetInputValue` type switch |

The next chapter covers `SkyAutocomplete`, a more complex input that combines popup management, async search, and debouncing.
