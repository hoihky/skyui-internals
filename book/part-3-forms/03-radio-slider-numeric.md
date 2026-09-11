---
title: Chapter 10 — SkyRadioGroup, SkySlider, and SkyNumericUpDown
order: 10
---

# Chapter 10: SkyRadioGroup, SkySlider, and SkyNumericUpDown

Part III continues with three form controls that illustrate different Avalonia techniques: **programmatic child creation** (`SkyRadioGroup`), **property re-ownership** (`SkySlider`), and **parse/format with culture** (`SkyNumericUpDown`).

## SkyRadioGroup: MVVM-Friendly Single Selection

Native `RadioButton` groups require a shared `GroupName` so only one button is checked. `SkyRadioGroup` wraps this into a single control with `SelectedValue` bound to your model's enum or string.

### API Surface

```csharp
[TemplatePart(ItemsHostPartName, typeof(Panel))]
public class SkyRadioGroup : TemplatedControl
{
    public static readonly StyledProperty<object?> SelectedValueProperty =
        AvaloniaProperty.Register<SkyRadioGroup, object?>(
            nameof(SelectedValue), defaultBindingMode: BindingMode.TwoWay);

    public static readonly StyledProperty<Orientation> OrientationProperty = ...;

    [Content]
    public IList Items => _items;  // SkyRadioGroupItem collection
}
```

`SelectedValue` compares against each item's `Value` property — not the display label — so you can bind directly to view model enums:

```xml
<SkyRadioGroup SelectedValue="{Binding SortOrder}"
               Orientation="Horizontal">
  <SkyRadioGroupItem Label="Name" Value="Name" />
  <SkyRadioGroupItem Label="Date" Value="Date" />
  <SkyRadioGroupItem Label="Size" Value="Size" />
</SkyRadioGroup>
```

### Dynamic RadioButton Creation

Unlike `SkyAccordion` which uses `ItemsControl` container overrides, `SkyRadioGroup` **rebuilds** `RadioButton` children in code:

```csharp
protected override void OnApplyTemplate(TemplateAppliedEventArgs e)
{
    base.OnApplyTemplate(e);
    _itemsHost = e.NameScope.Find(ItemsHostPartName) as Panel;
    RebuildRadioButtons();
}

private void RebuildRadioButtons()
{
    if (_itemsHost is null) return;

    foreach (var rb in _itemsHost.Children.OfType<RadioButton>())
        rb.IsCheckedChanged -= OnRadioButtonChecked;

    _itemsHost.Children.Clear();

    foreach (var item in _items)
    {
        var rb = new RadioButton
        {
            Content = item.Label,
            GroupName = _groupName,
            Tag = item.Value,
            IsChecked = Equals(item.Value, SelectedValue)
        };
        rb.IsCheckedChanged += OnRadioButtonChecked;
        _itemsHost.Children.Add(rb);
    }
}
```

**Why rebuild instead of ItemsControl?** Radio buttons need a unique `GroupName` per group instance. Using `Guid.NewGuid()` in `_groupName` prevents collisions when multiple `SkyRadioGroup` controls exist on one page.

### Selection Sync with Guard Flag

```csharp
private bool _syncingSelection;

private void OnRadioButtonChecked(object? sender, RoutedEventArgs e)
{
    if (_syncingSelection || sender is not RadioButton rb) return;
    SelectedValue = rb.Tag;
}

private void SyncRadioButtonChecks()
{
    _syncingSelection = true;
    try
    {
        foreach (var rb in _itemsHost!.Children.OfType<RadioButton>())
            rb.IsChecked = Equals(rb.Tag, SelectedValue);
    }
    finally { _syncingSelection = false; }
}
```

Without `_syncingSelection`, setting `SelectedValue` from the view model would check a radio button, which fires `IsCheckedChanged`, which sets `SelectedValue` again — potentially raising duplicate `SelectionChanged` events.

### Avalonia Concept: GroupName

`RadioButton.GroupName` is a string identifier. All radio buttons with the same group name are mutually exclusive. This is an Avalonia platform feature `SkyRadioGroup` orchestrates rather than reimplements.

---

## SkySlider: Property Re-Ownership

`SkySlider` wraps Avalonia's `Slider` with a label and formatted value readout.

### Re-Owning RangeBase Properties

```csharp
public static readonly StyledProperty<double> ValueProperty =
    RangeBase.ValueProperty.AddOwner<SkySlider>();

public static readonly StyledProperty<double> MinimumProperty =
    RangeBase.MinimumProperty.AddOwner<SkySlider>();

public static readonly StyledProperty<double> TickFrequencyProperty =
    Slider.TickFrequencyProperty.AddOwner<SkySlider>();
```

`AddOwner<T>()` registers the **same** underlying property on `SkySlider`. Bindings on the outer control propagate to the inner `PART_Slider` via template:

```xml
<Slider Name="PART_Slider"
        Value="{TemplateBinding Value}"
        Minimum="{TemplateBinding Minimum}"
        Maximum="{TemplateBinding Maximum}" />
```

### Value Label Formatting

```csharp
static SkySlider()
{
    ValueProperty.Changed.AddClassHandler<SkySlider>((s, _) => s.UpdateValueLabel());
}

private void UpdateValueLabel()
{
    if (_valueLabel is null) return;
    _valueLabel.Text = string.Format(ValueFormat, Value);
    _valueLabel.IsVisible = ShowValueLabel;
}
```

`ValueFormat` defaults to `"{0:0}"` but accepts any `string.Format` pattern — `"{0:P0}"` for percentages, `"{0:F1}"` for one decimal place.

### Usage

```xml
<SkyFormField Label="Volume">
  <SkySlider Minimum="0" Maximum="100"
             Value="{Binding Volume, Mode=TwoWay}"
             ValueFormat="{}{0}%"
             ShowValueLabel="True" />
</SkyFormField>
```

### Lesson

When your composite control's inner primitive already exposes the right properties, **re-own** them rather than duplicating backing fields. The outer control becomes a thin coordinator for label formatting and layout.

---

## SkyNumericUpDown: Culture-Aware Parsing

`SkyNumericUpDown` combines a text box with increment/decrement buttons and handles the messy reality of locale-specific decimal separators.

### Properties

```csharp
public static readonly StyledProperty<decimal> ValueProperty =
    AvaloniaProperty.Register<SkyNumericUpDown, decimal>(
        nameof(Value), defaultBindingMode: BindingMode.TwoWay);

public static readonly StyledProperty<decimal> MinimumProperty = ...; // default 0
public static readonly StyledProperty<decimal> MaximumProperty = ...; // default 100
public static readonly StyledProperty<decimal> StepProperty = ...;  // default 1
public static readonly StyledProperty<bool> IsIntegerProperty = ...;
public static readonly StyledProperty<CultureInfo?> CultureProperty = ...;
public static readonly StyledProperty<string?> FormatStringProperty = ...;
```

### Clamp Pipeline

Every value change passes through clamping:

```csharp
private void SetClampedValue(decimal raw)
{
    var clamped = Clamp(raw);
    if (IsInteger)
        clamped = decimal.Truncate(clamped);
    if (clamped != Value)
        Value = clamped;
    else
        UpdateDisplay();
}

private decimal Clamp(decimal v) =>
    Math.Clamp(v, Minimum, Maximum);
```

`IsInteger` truncates toward zero after clamp — useful for quantity fields.

### Text Parse on Lost Focus

The text box shows formatted values; users type raw text:

```csharp
private void OnTextBoxLostFocus(object? sender, RoutedEventArgs e)
{
    if (TryParse(_textBox!.Text, out var parsed))
        SetClampedValue(parsed);
    else
        UpdateDisplay(); // revert invalid input
}
```

Invalid input does not corrupt `Value` — the display reverts to the last valid formatted string. This is essential UX for numeric fields.

### Keyboard Stepping

```csharp
private void OnTextBoxKeyDown(object? sender, KeyEventArgs e)
{
    if (e.Key == Key.Up) { SetClampedValue(Value + Step); e.Handled = true; }
    if (e.Key == Key.Down) { SetClampedValue(Value - Step); e.Handled = true; }
}
```

Arrow keys step by `Step` without leaving the field — standard spreadsheet behavior.

### Formatting

```csharp
private string FormatValue(decimal v)
{
    if (!string.IsNullOrEmpty(FormatString))
        return string.Format(Culture, FormatString, v);
    return IsInteger
        ? v.ToString("N0", Culture)
        : v.ToString("N", Culture);
}
```

`Culture` defaults to `CultureInfo.CurrentCulture` so `1.234,56` displays correctly for German users when appropriate.

### Template Part Wiring

```csharp
protected override void OnApplyTemplate(TemplateAppliedEventArgs e)
{
    base.OnApplyTemplate(e);
    DetachHandlers();

    _textBox = e.NameScope.Find(TextBoxPartName) as TextBox;
    _increment = e.NameScope.Find(IncrementButtonPartName) as Button;
    _decrement = e.NameScope.Find(DecrementButtonPartName) as Button;

    if (_textBox is not null)
    {
        _textBox.LostFocus += OnTextBoxLostFocus;
        _textBox.KeyDown += OnTextBoxKeyDown;
    }
    if (_increment is not null) _increment.Click += (_, _) => SetClampedValue(Value + Step);
    if (_decrement is not null) _decrement.Click += (_, _) => SetClampedValue(Value - Step);

    UpdateDisplay();
}
```

`DetachHandlers()` is called before re-subscribing — mandatory when templates reapply.

### Usage

```xml
<SkyFormField Label="Quantity" Validator="{x:Static local:Validators.Positive}">
  <SkyNumericUpDown Value="{Binding Quantity, Mode=TwoWay}"
                    Minimum="1" Maximum="999"
                    IsInteger="True"
                    Step="1" />
</SkyFormField>
```

## Pattern Comparison

| Control | Primary pattern | Avalonia concept highlighted |
|---------|----------------|------------------------------|
| SkyRadioGroup | Programmatic children + GroupName | RadioButton mutual exclusion |
| SkySlider | Property AddOwner | TemplateBinding to inner Slider |
| SkyNumericUpDown | Parse/format pipeline | CultureInfo, LostFocus validation |

## Summary

These three controls show that "form control" does not always mean the same implementation strategy. Choose programmatic child management when platform primitives need orchestration (radio groups), re-ownership when wrapping a single primitive (slider), and explicit parse/format when text display diverges from typed values (numeric up-down).
