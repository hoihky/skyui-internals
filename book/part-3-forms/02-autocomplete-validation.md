---
title: Chapter 9 — SkyAutocomplete and Validation
order: 9
---

# Chapter 9: SkyAutocomplete and Validation Infrastructure

`SkyAutocomplete` is a typeahead input that filters local items or queries an async provider. It demonstrates popup lifecycle management, search debouncing, and cancellation. This chapter also deepens the validation infrastructure introduced with `SkyFormField`.

Autocomplete controls are deceptively complex. The surface is a text box; underneath you coordinate popups, keyboard routing, async data, and selection state. Avalonia's `Popup` control and `DispatcherTimer` provide the primitives; `SkyAutocomplete` wires them into a cohesive input.

## SkyAutocomplete Architecture

The control composes three template parts:

| Part | Type | Role |
|------|------|------|
| `PART_TextBox` | `TextBox` | User text input |
| `PART_Popup` | `Popup` | Floating suggestion panel |
| `PART_Suggestions` | `ListBox` | Suggestion list inside popup |

```csharp
public class SkyAutocomplete : TemplatedControl
{
    public const string TextBoxPartName = "PART_TextBox";
    public const string PopupPartName = "PART_Popup";
    public const string SuggestionsPartName = "PART_Suggestions";

    public static readonly StyledProperty<string?> TextProperty =
        AvaloniaProperty.Register<SkyAutocomplete, string?>(
            nameof(Text), defaultBindingMode: BindingMode.TwoWay);

    public static readonly StyledProperty<object?> SelectedItemProperty =
        AvaloniaProperty.Register<SkyAutocomplete, object?>(
            nameof(SelectedItem), defaultBindingMode: BindingMode.TwoWay);

    public static readonly StyledProperty<ISkyAutocompleteProvider?> ProviderProperty = ...;
    public static readonly StyledProperty<IEnumerable?> ItemsSourceProperty = ...;
    public static readonly StyledProperty<bool> IsFreeTextAllowedProperty = ...;
    public static readonly StyledProperty<int> MinimumQueryLengthProperty = ...;
}
```

Two binding modes deserve attention:

- `Text` is two-way because the user types directly
- `SelectedItem` is two-way because picking a suggestion updates the selection

### Template Wiring in OnApplyTemplate

After the template loads, the control connects events:

```csharp
protected override void OnApplyTemplate(TemplateAppliedEventArgs e)
{
    base.OnApplyTemplate(e);

    if (_textBox is not null)
        _textBox.TextChanged -= OnTextBoxTextChanged;

    _textBox = e.NameScope.Find<TextBox>(TextBoxPartName);
    _popup = e.NameScope.Find<Popup>(PopupPartName);
    _suggestions = e.NameScope.Find<ListBox>(SuggestionsPartName);

    if (_textBox is not null)
        _textBox.TextChanged += OnTextBoxTextChanged;

    if (_suggestions is not null)
    {
        _suggestions.SelectionChanged += OnSuggestionSelectionChanged;
        _suggestions.PointerPressed += OnSuggestionPointerPressed;
    }
}
```

Always unsubscribe old handlers before subscribing new ones. Templates can be reapplied when themes change, and duplicate handlers cause double-firing.

## Local vs Remote Data Sources

`SkyAutocomplete` supports two data paths:

**Local filtering** — Set `ItemsSource` to an in-memory collection. As the user types, the control filters items whose display text contains the query.

**Async provider** — Set `Provider` to an `ISkyAutocompleteProvider`:

```csharp
public interface ISkyAutocompleteProvider
{
    Task<IReadOnlyList<object>> SearchAsync(
        string query, CancellationToken cancellationToken);
}
```

A typical implementation calls a REST API:

```csharp
public class CityAutocompleteProvider : ISkyAutocompleteProvider
{
    public async Task<IReadOnlyList<object>> SearchAsync(
        string query, CancellationToken cancellationToken)
    {
        var results = await _api.SearchCitiesAsync(query, cancellationToken);
        return results.Cast<object>().ToList();
    }
}
```

The control does not know about HTTP, databases, or view models. It only knows the interface.

### Local Filtering Implementation

For local data, filtering runs synchronously on the UI thread:

```csharp
private IReadOnlyList<object> FilterLocalItems(string query)
{
    if (ItemsSource is null)
        return Array.Empty<object>();

    return ItemsSource
        .Cast<object>()
        .Where(item => FormatItem(item)
            .Contains(query, StringComparison.OrdinalIgnoreCase))
        .Take(MaxSuggestionCount)
        .ToList();
}
```

Local filtering is appropriate for collections under a few thousand items. Beyond that, pre-index or switch to an async provider that queries a database.

### Display Text Formatting

The control needs a consistent way to turn items into display strings:

```csharp
protected virtual string FormatItem(object? item) =>
    item switch
    {
        null => string.Empty,
        string s => s,
        _ => item.ToString() ?? string.Empty
    };
```

Override `FormatItem` in a subclass or set `ItemTemplate` on the suggestions list for rich display (icons, secondary text). The suggestions `ListBox` uses `ItemTemplate` for rendering; `FormatItem` is for setting `Text` after selection.

## Debouncing and Cancellation

Rapid keystrokes would trigger a search on every character without debouncing. `SkyAutocomplete` uses a version counter and `CancellationTokenSource`:

```csharp
private CancellationTokenSource? searchCancellation;
private int searchVersion;

private void QueueTextChanged()
{
    var version = ++searchVersion;

    DispatcherTimer.RunOnce(async () =>
    {
        if (version != searchVersion)
            return; // superseded by newer keystroke

        searchCancellation?.Cancel();
        searchCancellation = new CancellationTokenSource();

        await RunSearchAsync(searchCancellation.Token);
    }, TimeSpan.FromMilliseconds(200));
}
```

When a new character arrives before the timer fires, `searchVersion` increments and the pending search is discarded. When the timer fires, any in-flight async search is cancelled before starting a new one.

This is a standard pattern for search-as-you-type controls. The 200ms delay balances responsiveness against request volume.

### Why Not Task.Delay?

`DispatcherTimer.RunOnce` schedules on the UI thread's dispatcher. If you use `Task.Delay` inside an async handler attached to `TextChanged`, you must marshal back to the UI thread before updating `ItemsSource` on the suggestions list. `DispatcherTimer` keeps everything on the correct thread by default.

### Cancellation in the Provider

Providers should respect `CancellationToken`:

```csharp
public async Task<IReadOnlyList<object>> SearchAsync(
    string query, CancellationToken cancellationToken)
{
    var response = await _httpClient.GetAsync(
        $"/api/search?q={Uri.EscapeDataString(query)}",
        cancellationToken);

    cancellationToken.ThrowIfCancellationRequested();

    var items = await response.Content.ReadFromJsonAsync<List<SearchResult>>(
        cancellationToken: cancellationToken);

    return items?.Cast<object>().ToList() ?? new List<object>();
}
```

Without cancellation, abandoned searches still complete and may overwrite newer results if you forget the version check.

## Popup Lifecycle

The popup opens when:

1. The query length meets `MinimumQueryLength`
2. There are suggestions to show
3. The text box has focus

It closes when:

1. The user selects an item
2. The text box loses focus
3. Escape is pressed
4. The query is cleared

`IsSuggestionOpen` is a styled property bound to the popup's `IsOpen` in the template, but the C# class also sets it directly to keep logic centralized.

### Popup Placement

Avalonia's `Popup` supports `PlacementMode` and `PlacementTarget`. For autocomplete, anchor below the text box:

```xml
<Popup Name="PART_Popup"
       PlacementTarget="{Binding #PART_TextBox}"
       Placement="Bottom"
       IsLightDismissEnabled="False">
```

Set `IsLightDismissEnabled="False"` so clicking a suggestion does not dismiss the popup before the click reaches the list. The control closes the popup programmatically after selection.

### Focus Traps

When the popup is open, keyboard events on the text box must route Up/Down to the suggestions list. Handle `KeyDown` on the text box:

```csharp
private void OnTextBoxKeyDown(object? sender, KeyEventArgs e)
{
    if (!IsSuggestionOpen || _suggestions is null)
        return;

    switch (e.Key)
    {
        case Key.Down:
            MoveSuggestionSelection(1);
            e.Handled = true;
            break;
        case Key.Up:
            MoveSuggestionSelection(-1);
            e.Handled = true;
            break;
        case Key.Enter:
            CommitHighlightedSuggestion();
            e.Handled = true;
            break;
        case Key.Escape:
            IsSuggestionOpen = false;
            e.Handled = true;
            break;
    }
}
```

Mark handled events with `e.Handled = true` to prevent parent controls from also processing them.

## Selection Behavior

When the user picks a suggestion:

```csharp
private void OnSuggestionSelected(object? item)
{
    SelectedItem = item;
    Text = FormatItem(item);
    IsSuggestionOpen = false;
    textBox?.Focus();
}
```

`IsFreeTextAllowed` controls whether the user can commit text that does not match any suggestion. When false, leaving the field with unmatched text clears the selection or reverts to the last valid item:

```csharp
private void OnTextBoxLostFocus(object? sender, RoutedEventArgs e)
{
    if (IsFreeTextAllowed)
        return;

    if (SelectedItem is null || FormatItem(SelectedItem) != Text)
    {
        Text = SelectedItem is not null ? FormatItem(SelectedItem) : string.Empty;
    }

    IsSuggestionOpen = false;
}
```

This prevents the view model from receiving free-text values when the field requires a valid selection from the provider.

## Integration with SkyFormField

`SkyFormField.GetInputValue` handles autocomplete specifically:

```csharp
SkyAutocomplete autocomplete => autocomplete.SelectedItem ?? autocomplete.Text,
```

When a suggestion is selected, validation runs against the selected object. When the user types free text, validation runs against the raw string. Your validator should handle both cases or you should set `IsFreeTextAllowed="False"` for fields that require a valid selection.

### Validator for Selected Objects

When validating a selected item, check the object type:

```csharp
public class CitySelectedValidator : ISkyValidator
{
    public SkyValidationResult Validate(object? value)
    {
        if (value is City city)
            return SkyValidationResult.Valid;

        if (value is string text && !string.IsNullOrWhiteSpace(text))
            return SkyValidationResult.Invalid("Select a city from the list.");

        return SkyValidationResult.Invalid("City is required.");
    }
}
```

## Validation Infrastructure Deep Dive

### SkyValidationResult

A lightweight struct avoids allocating objects for successful validation:

```csharp
public readonly struct SkyValidationResult
{
    public static SkyValidationResult Valid => new(true, null);
    public static SkyValidationResult Invalid(string message) => new(false, message);

    public bool IsValid { get; }
    public string? ErrorMessage { get; }
}
```

Structs work well here because validation runs frequently (every keystroke if you validate on text change, every lost focus otherwise). The invalid path carries a string; the valid path allocates nothing.

### CompositeSkyValidator

Chains validators in order, stopping at the first failure:

```csharp
public class CompositeSkyValidator : ISkyValidator
{
    private readonly ISkyValidator[] _validators;

    public SkyValidationResult Validate(object? value)
    {
        foreach (var validator in _validators)
        {
            var result = validator.Validate(value);
            if (!result.IsValid)
                return result;
        }
        return SkyValidationResult.Valid;
    }
}
```

Order matters. Put `Required` before `Email` so empty fields get "required" rather than "invalid email."

### Conditional Validators

Sometimes validation depends on another field's value. Wrap a validator with a condition:

```csharp
public class ConditionalValidator : ISkyValidator
{
    private readonly Func<bool> _condition;
    private readonly ISkyValidator _inner;

    public SkyValidationResult Validate(object? value) =>
        _condition() ? _inner.Validate(value) : SkyValidationResult.Valid;
}
```

Use this for "company name required only when account type is Business" without putting that logic inside `SkyFormField`.

### SkyValidationSummary

For form-level error display, `SkyValidationSummary` aggregates errors from multiple fields:

```xml
<SkyValidationSummary Items="{Binding ValidationErrors}" />
```

Each item is a `SkyValidationSummaryItem` with `PropertyName` and `ErrorMessage`. This is useful for accessibility: screen readers can announce all errors at once on submit.

Populate the summary from your view model after form-wide validation:

```csharp
public void ValidateForm()
{
    ValidationErrors.Clear();

    if (!ValidateEmail())
        ValidationErrors.Add(new SkyValidationSummaryItem("Email", "Invalid email address."));

    if (!ValidatePassword())
        ValidationErrors.Add(new SkyValidationSummaryItem("Password", "Password too short."));
}
```

### SkyEmptyState

List and form pages need a deliberate zero-data state. `SkyEmptyState` is a templated placeholder with `Title`, `Description`, `IconKind` (`SkyIconKind`), and optional primary action (`ActionText`, `ActionCommand`, or `ActionContent` for custom button chrome). Template parts `PART_Icon` and `PART_Action` let the theme align icon size and button placement.

Use it inside `SkyListDetailPage` detail panes, search results, or empty grids instead of leaving a blank `ScrollViewer`:

```xml
<SkyEmptyState Title="No projects yet"
               Description="Create a project to see it here."
               IconKind="Folder"
               ActionText="New project"
               ActionCommand="{Binding CreateProjectCommand}" />
```

The control adds `sky` and `sky-empty-state` classes on construction so variant styles apply consistently with other form chrome.

## Debugging Autocomplete Issues

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Popup never opens | `MinimumQueryLength` too high | Lower threshold or check query length |
| Stale results appear | Missing version check | Verify `searchVersion` comparison before applying results |
| Popup flashes and closes | `IsLightDismissEnabled` true | Set false on popup |
| Selection does not update VM | One-way binding on `SelectedItem` | Use `Mode=TwoWay` |
| Async errors swallowed | Missing try/catch in `RunSearchAsync` | Log exceptions; show empty suggestions on failure |
| Keyboard nav broken | `KeyDown` not handled | Wire handler on text box, set `e.Handled` |

Use Avalonia DevTools to inspect popup visibility and the suggestions `ItemsSource` while typing.

## Usage Examples

**Local static list:**

```xml
<SkyAutocomplete ItemsSource="{Binding Countries}"
                 Text="{Binding CountryName, Mode=TwoWay}"
                 SelectedItem="{Binding SelectedCountry, Mode=TwoWay}"
                 MinimumQueryLength="1" />
```

**Async provider:**

```xml
<SkyAutocomplete Provider="{x:Static local:CityProvider.Instance}"
                 IsFreeTextAllowed="False"
                 MinimumQueryLength="2" />
```

**Inside SkyFormField:**

```xml
<SkyFormField Label="City" IsRequired="True" ValidateOnLostFocus="True">
  <SkyAutocomplete Provider="{x:Static local:CityProvider.Instance}"
                   IsFreeTextAllowed="False" />
</SkyFormField>
```

## Building Your Own Autocomplete

Key implementation checklist:

1. **Three template parts** — text input, popup, suggestion list
2. **Two-way binding** on text and selected item
3. **Provider interface** for async data; local `ItemsSource` for static lists
4. **Debounce** with version counter to avoid stale results
5. **Cancel** in-flight async operations on new input
6. **Keyboard navigation** — up/down to move selection, enter to commit, escape to close
7. **Free text policy** — explicit property controlling whether unmatched text is valid
8. **Focus management** — return focus to text box after selection
9. **Error handling** — degrade gracefully when provider fails

### Walkthrough: Async Search Flow

1. User types "lon" in the text box.
2. `TextChanged` fires → `QueueTextChanged()` increments version to 3.
3. After 200ms, timer callback checks version still equals 3.
4. Previous `CancellationTokenSource` is cancelled; new one created.
5. `Provider.SearchAsync("lon", token)` called.
6. Results return → suggestions `ItemsSource` updated → popup opens.
7. User presses Down, Enter → `SelectedItem` set, `Text` formatted, popup closes.

## Summary

`SkyAutocomplete` combines template composition, async data access through a provider interface, and debounced search with cancellation. The validation layer (`ISkyValidator`, composites, validation summary) provides a parallel strategy-based system for form correctness. Together they show how SkyUI separates **data acquisition** (provider), **presentation** (template parts), and **correctness** (validators) into independent concerns.
