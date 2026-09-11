---
title: Chapter 14 — Pagination and KPI Tiles
order: 14
---

# Chapter 14: SkyPaginationBase and SkyKpiTile

Data presentation controls surface information rather than collect it. This chapter examines two patterns: an abstract base class that shares paging logic between UI variants, and a self-contained templated control for metric display.

Pagination and KPI tiles sit at opposite ends of complexity. Pagination manages state, commands, and derived properties across multiple visual variants. KPI tiles are nearly stateless — bind values, apply theme, done. Studying both teaches when to invest in shared infrastructure versus keeping controls minimal.

## SkyPaginationBase: Abstract Base for Shared Logic

Pagination appears in two forms in SkyUI:

- `SkyPagination` — a standalone pager bar with page number buttons
- `SkyDataPager` — a compact pager integrated into data views

Both share the same math, state properties, and navigation commands. That shared behavior lives in `SkyPaginationBase`.

### Why Abstract Base Instead of Composition?

You could extract paging into a `SkyPaginationController` helper and inject it into both controls. SkyUI chose an abstract base because:

- Paging properties bind directly on the control in XAML (`CurrentPage`, `PageCount`)
- Template parts differ, but `OnApplyTemplate` logic is minimal
- Consumers expect a single control element, not control + controller wiring

If a third variant needed different commands or a different state machine, a helper class would become attractive. At two variants with identical behavior, the abstract base wins on simplicity.

### Shared State Properties

```csharp
public abstract class SkyPaginationBase : TemplatedControl
{
    public static readonly StyledProperty<int> CurrentPageProperty =
        AvaloniaProperty.Register<SkyPaginationBase, int>(nameof(CurrentPage), 1);

    public static readonly StyledProperty<int> PageSizeProperty =
        AvaloniaProperty.Register<SkyPaginationBase, int>(nameof(PageSize), 10);

    public static readonly StyledProperty<int> TotalCountProperty =
        AvaloniaProperty.Register<SkyPaginationBase, int>(nameof(TotalCount), 0);

    public static readonly StyledProperty<IEnumerable?> ItemsSourceProperty = ...;
    public static readonly StyledProperty<bool> IsTotalCountAutomaticProperty = ...;
    public static readonly StyledProperty<ItemsControl?> TargetProperty = ...;
}
```

`CurrentPage`, `PageSize`, and `TotalCount` are the three inputs to paging math. `ItemsSource` provides the full data set that gets sliced. `Target` optionally binds to an `ItemsControl` whose `ItemsSource` is updated automatically when the page changes.

`IsTotalCountAutomatic` when true sets `TotalCount` from `ItemsSource.Count()` whenever the source changes. Disable it when `TotalCount` comes from a server response that reports total records independently of the current page's fetched items.

### Computed State with Direct Properties

Read-only derived values use `DirectProperty<T>` with backing fields:

```csharp
public static readonly DirectProperty<SkyPaginationBase, int> PageCountProperty =
    AvaloniaProperty.RegisterDirect<SkyPaginationBase, int>(
        nameof(PageCount), control => control.PageCount);

private int pageCount;
public int PageCount => pageCount;
```

`DirectProperty` is appropriate when the value is computed in code and never set from XAML or styles. `StyledProperty` supports styling and inheritance; `DirectProperty` is lighter for read-only projections.

When `TotalCount` or `PageSize` changes, `OnPagingInputChanged` recalculates:

```csharp
private void OnPagingInputChanged()
{
    pageCount = SkyPaginationMath.ComputePageCount(TotalCount, PageSize);
    rangeStart = SkyPaginationMath.ComputeRangeStart(CurrentPage, PageSize, TotalCount);
    rangeEnd = SkyPaginationMath.ComputeRangeEnd(CurrentPage, PageSize, TotalCount);
    currentPageItems = SkyPaginationMath.SlicePage(ItemsSource, CurrentPage, PageSize);

    canGoToFirst = CurrentPage > 1;
    canGoToPrevious = CurrentPage > 1;
    canGoToNext = CurrentPage < pageCount;
    canGoToLast = CurrentPage < pageCount;

    RaisePropertyChanged(PageCountProperty);
    RaisePropertyChanged(RangeStartProperty);
    // ... other direct properties

    if (Target is not null)
        Target.ItemsSource = currentPageItems;
}
```

Consumers bind UI to `PageCount`, `CanGoToNext`, and `RangeStart` without calculating these values themselves.

### Clamping CurrentPage

When `TotalCount` shrinks (filter applied, items deleted), `CurrentPage` may exceed `PageCount`. Clamp in the changed handler:

```csharp
private static void OnCurrentPageChanged(SkyPaginationBase control, AvaloniaPropertyChangedEventArgs e)
{
    var page = (int)e.NewValue!;
    var maxPage = SkyPaginationMath.ComputePageCount(control.TotalCount, control.PageSize);
    if (maxPage > 0 && page > maxPage)
        control.CurrentPage = maxPage;
    else
        control.OnPagingInputChanged();
}
```

Without clamping, `SlicePage` returns empty results and confuses users who were on page 5 of a list that now has 2 pages.

### Pagination Math

`SkyPaginationMath` is a static helper keeping formulas in one place:

```csharp
public static class SkyPaginationMath
{
    public static int ComputePageCount(int totalCount, int pageSize) =>
        pageSize <= 0 ? 0 : Math.Max(1, (int)Math.Ceiling(totalCount / (double)pageSize));

    public static int ComputeRangeStart(int currentPage, int pageSize, int totalCount) =>
        totalCount == 0 ? 0 : (currentPage - 1) * pageSize + 1;

    public static int ComputeRangeEnd(int currentPage, int pageSize, int totalCount) =>
        Math.Min(currentPage * pageSize, totalCount);

    public static IEnumerable? SlicePage(IEnumerable? source, int page, int pageSize)
    {
        if (source is null || pageSize <= 0) return null;
        var skip = (page - 1) * pageSize;
        return source.Cast<object?>().Skip(skip).Take(pageSize);
    }
}
```

Extracting math into a static class makes it unit-testable without creating control instances.

Note that `ComputePageCount` returns 1 even when `totalCount` is 0. The UI shows "page 1 of 1" with zero items, which is less confusing than "page 1 of 0." `RangeStart` returns 0 when empty to produce copy like "Showing 0–0 of 0."

### Server-Side Pagination

Client-side slicing works for in-memory collections. For server-side data, override the pattern:

```csharp
public class ServerPagedViewModel : INotifyPropertyChanged
{
    public int CurrentPage { get; set; } = 1;
    public int PageSize { get; set; } = 25;
    public int TotalCount { get; set; }

    public ObservableCollection<Item> CurrentPageItems { get; } = new();

    public async Task LoadPageAsync()
    {
        var result = await _api.GetItemsAsync(CurrentPage, PageSize);
        CurrentPageItems.Clear();
        foreach (var item in result.Items)
            CurrentPageItems.Add(item);
        TotalCount = result.TotalCount;
    }
}
```

Bind `SkyPagination` to `TotalCount` and `CurrentPage`, but set `ItemsSource` to null and bind the list directly to `CurrentPageItems`. Handle `PageChanged` to call `LoadPageAsync`.

### Navigation Commands

The base class exposes `ICommand` properties for first, previous, next, and last page:

```csharp
public ICommand FirstPageCommand => firstPageCommand;
public ICommand PreviousPageCommand => previousPageCommand;
public ICommand NextPageCommand => nextPageCommand;
public ICommand LastPageCommand => lastPageCommand;
```

Commands are implemented as lightweight relay commands that check `CanExecute` against the `canGoTo*` flags:

```csharp
private sealed class SkyPaginationRelayCommand : ICommand
{
    private readonly Action _execute;
    private readonly Func<bool> _canExecute;

    public bool CanExecute(object? parameter) => _canExecute();
    public void Execute(object? parameter) => _execute();

    public event EventHandler? CanExecuteChanged;

    public void RaiseCanExecuteChanged() =>
        CanExecuteChanged?.Invoke(this, EventArgs.Empty);
}
```

Call `RaiseCanExecuteChanged` on all commands whenever `OnPagingInputChanged` updates the `canGoTo*` flags. Without this, buttons stay enabled after navigation because WPF/Avalonia does not automatically requery `CanExecute`.

Wire commands in XAML:

```xml
<Button Content="Previous"
        Command="{Binding PreviousPageCommand, ElementName=Pager}" />
<Button Content="Next"
        Command="{Binding NextPageCommand, ElementName=Pager}" />
```

### PageChanged Routed Event

```csharp
public static readonly RoutedEvent<RoutedEventArgs> PageChangedEvent =
    RoutedEvent.Register<SkyPaginationBase, RoutedEventArgs>(
        nameof(PageChanged), RoutingStrategies.Bubble);
```

Raised only when `CurrentPage` actually changes, allowing view models to reload data on page navigation:

```csharp
pager.AddHandler(SkyPaginationBase.PageChangedEvent, (_, _) =>
{
    _ = viewModel.LoadPageAsync();
});
```

### Usage

```xml
<StackPanel>
  <ListBox ItemsSource="{Binding CurrentPageItems}"
           x:Name="ItemList" />
  <SkyPagination ItemsSource="{Binding AllItems}"
                 PageSize="25"
                 TotalCount="{Binding TotalCount}"
                 Target="{Binding #ItemList}" />
</StackPanel>
```

The `Target` binding automatically updates the list box when the user clicks next page.

Display range text in the pager template:

```xml
<TextBlock>
  Showing <Run Text="{Binding RangeStart}" />
  –<Run Text="{Binding RangeEnd}" />
  of <Run Text="{Binding TotalCount}" />
</TextBlock>
```

### Debugging Pagination

| Symptom | Cause | Fix |
|---------|-------|-----|
| Empty list on page 2 | `CurrentPage` not clamped after filter | Clamp page in changed handler |
| Next button always enabled | `CanExecuteChanged` not raised | Raise after flag updates |
| Wrong item count | `IsTotalCountAutomatic` false but `TotalCount` stale | Refresh `TotalCount` on source change |
| Target not updating | `Target` binding broken | Use `ElementName` or `x:Reference` |
| Skip/Take slow | Large `IEnumerable` re-enumerated | Use `IList` with index access |

### When to Use an Abstract Base

Extract an abstract base when:

- Two or more controls share identical state machine logic
- Subclasses differ only in visual template, not behavior
- Computed properties and commands are identical

Do not use an abstract base when subclasses need different template parts with conflicting `OnApplyTemplate` logic. In that case, extract a helper class instead.

---

## SkyKpiTile: Focused Templated Control

`SkyKpiTile` displays a single key performance indicator: a title, a large value, and an optional delta trend.

```csharp
public class SkyKpiTile : TemplatedControl
{
    public static readonly StyledProperty<string?> TitleProperty = ...;
    public static readonly StyledProperty<string?> ValueProperty = ...;
    public static readonly StyledProperty<string?> DeltaProperty = ...;
    public static readonly StyledProperty<SkyKpiDeltaTrend> DeltaTrendProperty = ...;
}

public enum SkyKpiDeltaTrend
{
    Neutral,
    Up,
    Down
}
```

`DeltaTrend` drives color selection in the theme: green for up, red for down, muted for neutral. The C# class sets a style class or pseudo-class based on the enum; the theme maps each variant to `SkyAccentBrush`, `SkyDangerBrush`, or `SkyTextSecondaryBrush`.

### Syncing Trend to Style Classes

In the static constructor:

```csharp
static SkyKpiTile()
{
    DeltaTrendProperty.Changed.AddClassHandler<SkyKpiTile>((tile, e) =>
    {
        var trend = (SkyKpiDeltaTrend)e.NewValue!;
        tile.Classes.Set("kpi-up", trend == SkyKpiDeltaTrend.Up);
        tile.Classes.Set("kpi-down", trend == SkyKpiDeltaTrend.Down);
        tile.Classes.Set("kpi-neutral", trend == SkyKpiDeltaTrend.Neutral);
    });
}
```

Style classes on the control root let the template use simple selectors without converters in XAML.

### Template Structure

```xml
<ControlTheme x:Key="{x:Type controls:SkyKpiTile}" TargetType="controls:SkyKpiTile">
  <Setter Property="Template">
    <ControlTemplate>
      <Border Background="{DynamicResource SkyCardBrush}"
              CornerRadius="{DynamicResource SkyRadiusMd}"
              Padding="{DynamicResource SkySpace16Px}">
        <StackPanel Spacing="4">
          <TextBlock Text="{TemplateBinding Title}"
                     Foreground="{DynamicResource SkyTextSecondaryBrush}"
                     FontSize="{DynamicResource SkyFontSizeSm}" />
          <TextBlock Text="{TemplateBinding Value}"
                     Foreground="{DynamicResource SkyTextPrimaryBrush}"
                     FontSize="{DynamicResource SkyFontSize2Xl}"
                     FontWeight="Bold" />
          <TextBlock Text="{TemplateBinding Delta}"
                     Classes.kpi-up="{TemplateBinding DeltaTrend, Converter=...}" />
        </StackPanel>
      </Border>
    </ControlTemplate>
  </Setter>
</ControlTheme>
```

KPI tiles are intentionally simple. They have no interaction, no template parts wired in C#, and no collection management. They are pure presentation.

### Formatting Values in the View Model

Keep formatting logic out of the control:

```csharp
public string FormattedMau => $"{MonthlyActiveUsers:N0}";
public string FormattedDelta => $"{DeltaPercent:+0.0;-0.0;0}%";
public SkyKpiDeltaTrend Trend =>
    DeltaPercent > 0 ? SkyKpiDeltaTrend.Up :
    DeltaPercent < 0 ? SkyKpiDeltaTrend.Down :
    SkyKpiDeltaTrend.Neutral;
```

```xml
<SkyKpiTile Title="Monthly Active Users"
            Value="{Binding FormattedMau}"
            Delta="{Binding FormattedDelta}"
            DeltaTrend="{Binding Trend}" />
```

This keeps `SkyKpiTile` culture-agnostic and reusable.

### Dashboard Composition

KPI tiles compose naturally with `SkyResponsiveGrid`:

```xml
<SkyResponsiveGrid>
  <SkyKpiTile Title="Monthly Active Users"
              Value="48,291"
              Delta="+12.4%"
              DeltaTrend="Up" />
  <SkyKpiTile Title="Churn Rate"
              Value="2.1%"
              Delta="-0.3%"
              DeltaTrend="Down" />
  <SkyKpiTile Title="Support Tickets"
              Value="1,204"
              Delta="0%"
              DeltaTrend="Neutral" />
</SkyResponsiveGrid>
```

For loading states, wrap tiles in a container that shows skeleton placeholders until data arrives. `SkyKpiTile` does not need built-in loading UI — a parent `ContentControl` with a `DataTrigger` or view model `IsLoading` flag suffices.

### Accessibility

KPI tiles should expose a single accessible name combining title and value:

```xml
<Setter Property="AutomationProperties.Name"
        Value="{Binding Title, RelativeSource={RelativeSource Self}}" />
```

Or set in the view model: `AutomationProperties.SetName(tile, $"Monthly Active Users: 48,291")`.

## Summary

`SkyPaginationBase` demonstrates the **abstract base class** pattern for shared behavioral logic with variant visuals. `SkyKpiTile` demonstrates the **simple templated control** pattern for read-only data display. Data presentation controls tend toward one of these two poles: stateful logic with commands, or stateless templates with enum-driven styling.

The next chapter covers `SkyVirtualDataGrid`, the most complex data presentation control in SkyUI.
