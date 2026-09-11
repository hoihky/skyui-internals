---
title: Chapter 11 — Date and Time Pickers
order: 11
---

# Chapter 11: Date and Time Pickers

SkyUI's picker family (`SkyDatePicker`, `SkyTimePicker`, `SkyCalendar`, `SkyDateRangePicker`) shows how to extend Avalonia's calendar primitives with consistent styling and culture-aware formatting — without reimplementing calendar logic.

## The Picker Strategy: Subclass + Format Utilities

Avalonia provides `CalendarDatePicker`, `TimePicker`, and `Calendar`. These handle month grids, keyboard navigation, and popup lifecycle. SkyUI adds:

1. **Style classes** — `sky`, `sky-date-picker`, `sky-time-picker`
2. **Culture property** — explicit `CultureInfo` with change handler
3. **Shared formatting** — `SkyPickerFormat` and `SkyPickerFormatConverter`

```csharp
public class SkyDatePicker : CalendarDatePicker
{
    public static readonly StyledProperty<CultureInfo?> CultureProperty =
        AvaloniaProperty.Register<SkyDatePicker, CultureInfo?>(nameof(Culture));

    public SkyDatePicker()
    {
        Classes.Add("sky");
        Classes.Add("sky-date-picker");
        ApplyCulture(CultureInfo.CurrentCulture);
    }

    private void ApplyCulture(CultureInfo? culture) =>
        SkyPickerFormat.ApplyCulture(this, SkyPickerFormat.Resolve(culture));
}
```

### SkyPickerFormat

Centralizes date/time display strings per culture:

```csharp
public static class SkyPickerFormat
{
    public static CulturePickerFormat Resolve(CultureInfo? culture)
    {
        culture ??= CultureInfo.CurrentCulture;
        return new CulturePickerFormat(
            shortDate: culture.DateTimeFormat.ShortDatePattern,
            longDate: culture.DateTimeFormat.LongDatePattern,
            shortTime: culture.DateTimeFormat.ShortTimePattern);
    }

    public static void ApplyCulture(CalendarDatePicker picker, CulturePickerFormat format)
    {
        picker.SelectedDateFormat = CalendarDatePickerFormat.Short;
        // additional culture-specific watermark and format application
    }
}
```

When `Culture` changes on the picker, the static handler re-applies formats without recreating the control.

### Why Not TemplatedControl?

A calendar popup involves dozens of template parts (day cells, month navigation, year decade view). Reimplementing that in SkyUI would duplicate Avalonia maintenance burden. Subclassing keeps bug fixes upstream while SkyUI owns **look** (theme) and **format** (culture).

## SkyTimePicker

Mirrors the date picker pattern on `TimePicker`:

```csharp
public class SkyTimePicker : TimePicker
{
    public SkyTimePicker()
    {
        Classes.Add("sky");
        Classes.Add("sky-time-picker");
    }
}
```

Theme styles in `Pickers.axaml` set clock face colors, selected hour/minute highlight, and popup shadow from elevation tokens.

## SkyCalendar

Standalone month calendar for inline embedding (not popup):

```csharp
public class SkyCalendar : Calendar
{
    public SkyCalendar()
    {
        Classes.Add("sky");
        Classes.Add("sky-calendar");
    }
}
```

Use when the date selection UI should be always visible — booking flows, schedule builders.

## SkyDateRangePicker: Composing Pickers

`SkyDateRangePicker` is a `TemplatedControl` because it composes **two** `SkyDatePicker` instances plus preset buttons — behavior Avalonia does not provide as a single primitive.

### Template Parts

| Part | Type | Role |
|------|------|------|
| `PART_StartDate` | `SkyDatePicker` | Range start |
| `PART_EndDate` | `SkyDatePicker` | Range end |
| `PART_PresetToday` | `Button` | Quick preset |
| `PART_PresetWeek` | `Button` | Quick preset |

### Properties

```csharp
public static readonly StyledProperty<DateTime?> StartDateProperty = ...; // TwoWay
public static readonly StyledProperty<DateTime?> EndDateProperty = ...;   // TwoWay
public static readonly StyledProperty<SkyDateRangePreset> SelectedPresetProperty = ...;
public static readonly StyledProperty<CultureInfo?> CultureProperty = ...;
```

### Preset vs Manual Sync

Changing dates manually sets preset to `Custom`:

```csharp
private bool _isApplyingPreset;

private void OnManualDateChanged()
{
    if (_isApplyingPreset) return;
    SelectedPreset = SkyDateRangePreset.Custom;
}
```

Selecting a preset applies computed dates:

```csharp
private void ApplyPreset()
{
    _isApplyingPreset = true;
    try
    {
        switch (SelectedPreset)
        {
            case SkyDateRangePreset.Today:
                var today = DateTime.Today;
                StartDate = EndDate = today;
                break;
            case SkyDateRangePreset.ThisWeek:
                var culture = Culture ?? CultureInfo.CurrentCulture;
                var start = StartOfWeek(DateTime.Today, culture.DateTimeFormat.FirstDayOfWeek);
                StartDate = start;
                EndDate = start.AddDays(6);
                break;
        }
    }
    finally { _isApplyingPreset = false; }
}
```

The `_isApplyingPreset` guard prevents preset application from being interpreted as manual edit (which would immediately reset preset to `Custom`).

### Integration with SkyFormField

`GetInputValue` returns a value object:

```csharp
SkyDateRangePicker dateRange =>
    new SkyDateRangeValue(dateRange.StartDate, dateRange.EndDate),
```

Validators can check `StartDate <= EndDate` and non-null bounds.

### Usage

```xml
<SkyFormField Label="Reporting period">
  <SkyDateRangePicker StartDate="{Binding RangeStart, Mode=TwoWay}"
                      EndDate="{Binding RangeEnd, Mode=TwoWay}"
                      SelectedPreset="{Binding RangePreset, Mode=TwoWay}" />
</SkyFormField>
```

## SkyFilePicker: Static Service API

File dialogs are platform services, not visual controls:

```csharp
public static class SkyFilePicker
{
    public static Task<IReadOnlyList<string>> OpenFilesAsync(
        SkyFilePickerOptions options) => ...;

    public static Task<string?> SaveFileAsync(
        SkyFilePickerOptions options) => ...;
}
```

`SkyFilePickerOptions` carries filters, default directory, and multi-select flag. The implementation resolves `TopLevel` from the calling context and invokes Avalonia's storage provider.

## Theme Considerations

Picker popups render in a separate visual tree root. Theme must style:

- `CalendarDatePicker` / `TimePicker` popup content
- `CalendarDayButton` selected/hover states
- Popup `Border` shadow and corner radius

All colors reference `DynamicResource` tokens so dark theme and accent override apply inside popups.

## Building Your Own Picker

1. **Subclass** if Avalonia already has the primitive
2. **Extract format logic** to static helpers shared across pickers
3. **TemplatedControl** only when composing multiple pickers or adding presets
4. **Propagate Culture** to all child pickers in composed controls
5. **Guard flags** when bidirectional sync exists between presets and manual values

## Summary

SkyUI pickers lean on Avalonia for calendar mechanics and own culture formatting plus visual consistency. `SkyDateRangePicker` is the exception — a composed control with preset synchronization — and illustrates when subclassing is no longer sufficient.
