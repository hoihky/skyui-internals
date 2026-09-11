---
title: Chapter 15 — SkyVirtualDataGrid
order: 15
---

# Chapter 15: SkyVirtualDataGrid

`SkyVirtualDataGrid` in the `SkyUI.Data` package is a high-performance tabular data control. Unlike `CheckedListBox`, which virtualizes a flat list of tree rows, the data grid must virtualize across two dimensions: rows and columns, while supporting sorting, column reorder, in-place editing, and CSV export. This chapter explains the virtual data source contract, row pooling, and update coordination.

Data grids are where UI performance meets data complexity. A naive implementation creates one row control per record — fine for fifty rows, catastrophic for fifty thousand. `SkyVirtualDataGrid` treats the viewport as a window into a much larger dataset, binding only the rows you can see.

## The Virtual Data Source Contract

The grid never holds the full dataset in memory. It reads rows through `IVirtualGridDataSource`:

```csharp
public interface IVirtualGridDataSource : INotifyPropertyChanged
{
    long RowCount { get; }
    object? GetCellValue(long rowIndex, SkyDataGridColumn column);
    void SetCellValue(long rowIndex, SkyDataGridColumn column, object? value);
    void Sort(SkyDataGridColumn column, ListSortDirection direction);
}
```

Key design points:

- **Row index is `long`** — supports datasets larger than `int.MaxValue` in theory
- **Column is passed to cell accessors** — the data source decides how to map columns to fields
- **Sort is delegated** — the grid raises sort UI; the source reorders its backing store
- **INotifyPropertyChanged** — when row count or data changes, the grid invalidates visible rows

A list-backed implementation wraps any `IList`:

```csharp
public class ListVirtualGridDataSource : IVirtualGridDataSource
{
    private readonly IList _items;

    public long RowCount => _items.Count;

    public object? GetCellValue(long rowIndex, SkyDataGridColumn column)
    {
        var item = _items[(int)rowIndex];
        return column.BindingPath is not null
            ? GetPropertyValue(item, column.BindingPath)
            : item;
    }
}
```

For server-side data, implement the interface against your repository or API, fetching pages on demand when the user scrolls.

### Server-Backed Data Source

A database-backed source fetches pages when the scroll window moves:

```csharp
public class PagedVirtualGridDataSource : IVirtualGridDataSource
{
    private readonly Dictionary<long, RowCache> _pageCache = new();
    private long _rowCount;

    public long RowCount => _rowCount;

    public object? GetCellValue(long rowIndex, SkyDataGridColumn column)
    {
        var page = rowIndex / PageSize;
        if (!_pageCache.TryGetValue(page, out var cache))
        {
            cache = FetchPage(page);
            _pageCache[page] = cache;
        }
        var item = cache.Items[(int)(rowIndex % PageSize)];
        return GetPropertyValue(item, column.BindingPath);
    }

    public event PropertyChangedEventHandler? PropertyChanged;
}
```

Invalidate the cache when data changes externally. Raise `PropertyChanged` for `RowCount` when records are added or deleted.

### Property Path Resolution

`GetPropertyValue` walks dotted paths (`"Address.City"`) via reflection or compiled expression trees:

```csharp
private static object? GetPropertyValue(object item, string? path)
{
    if (path is null) return item;

    object? current = item;
    foreach (var segment in path.Split('.'))
    {
        if (current is null) return null;
        var prop = current.GetType().GetProperty(segment);
        current = prop?.GetValue(current);
    }
    return current;
}
```

For hot paths, cache compiled accessors keyed by `(Type, path)`.

## Column Definition

Columns are defined in an `ObservableCollection<SkyDataGridColumn>` on the grid:

```csharp
public class SkyDataGridColumn : AvaloniaObject
{
    public static readonly StyledProperty<string?> HeaderProperty = ...;
    public static readonly StyledProperty<string?> BindingPathProperty = ...;
    public static readonly StyledProperty<double> WidthProperty = ...;
    public static readonly StyledProperty<bool> IsSortableProperty = ...;
    public static readonly StyledProperty<bool> IsReadOnlyProperty = ...;
    public static readonly StyledProperty<IDataTemplate?> CellTemplateProperty = ...;
}
```

`BindingPath` is a property name for reflection-based cell value extraction. `CellTemplate` overrides the default text cell for custom rendering.

### Custom Cell Templates

Use `CellTemplate` for non-text content:

```xml
<SkyDataGridColumn Header="Status" BindingPath="Status">
  <SkyDataGridColumn.CellTemplate>
    <DataTemplate x:DataType="models:Order">
      <Border Background="{Binding Status, Converter={StaticResource StatusBrushConverter}}"
              CornerRadius="4" Padding="4,2">
        <TextBlock Text="{Binding Status}" />
      </Border>
    </DataTemplate>
  </SkyDataGridColumn.CellTemplate>
</SkyDataGridColumn>
```

The data context for cell templates is the cell value or the row object depending on grid configuration. Verify binding context in DevTools if cells appear empty.

### Column Width and Resize

`Width` supports fixed pixels and star sizing through a grid layout in the header and row templates. When the user drags a column divider, the grid updates `Width` on the column object and schedules invalidation.

## Row Virtualization Architecture

When `EnableRowVirtualization` is true (the default), the grid maintains a **fixed pool** of row visuals rather than creating one row per data record:

```
Data: 100,000 rows
Visual pool: ~30 row controls (visible rows + buffer)

Scroll position → calculate first visible row index
                → bind pool rows to data rows [n, n+1, ..., n+29]
                → set virtual extent height = rowCount × rowHeight
```

The virtual extent is a spacer element whose height represents the total scrollable area. Only the visible slice gets real row controls positioned at the correct vertical offset.

### Calculating the Visible Window

```csharp
private void UpdateVisibleRows()
{
    var scrollOffset = _scrollViewer?.Offset.Y ?? 0;
    var firstRow = (int)(scrollOffset / RowHeight);
    var visibleCount = (int)Math.Ceiling(ViewportHeight / RowHeight) + 2; // buffer

    if (firstRow == _lastScrollFirst && !_forceRebind)
    {
        RepositionRows(firstRow, scrollOffset);
        return;
    }

    _lastScrollFirst = firstRow;
    RebindPoolRows(firstRow, visibleCount);
}
```

The +2 buffer rows above and below the viewport reduce flicker during fast scroll. Tune buffer size against memory — each pooled row holds one control per column.

### Key Template Parts

| Part | Role |
|------|------|
| `PART_Scroll` | Outer `ScrollViewer` |
| `PART_ScrollRoot` | Custom scroll root handling offset sync |
| `PART_HeaderGrid` | Column headers with sort and drag |
| `PART_VirtualExtent` | Height spacer for total row count |
| `PART_Rows` | `ItemsControl` hosting the row pool |
| `PART_ColumnDragIndicator` | Visual feedback during column drag |

`PART_VirtualExtent` is critical. Without it, the scroll bar thumb would reflect viewport height, not total data height. Set its height to `RowCount * RowHeight` whenever row count or row height changes.

### Update Coordinator

`SkyVirtualDataGridUpdateCoordinator` coalesces invalidation requests:

```csharp
internal class SkyVirtualDataGridUpdateCoordinator
{
    private readonly Action _invalidate;
    private bool _pending;

    public void RequestInvalidate()
    {
        if (_pending) return;
        _pending = true;
        Dispatcher.UIThread.Post(() =>
        {
            _pending = false;
            _invalidate();
        });
    }
}
```

Scroll events, data source changes, and column width adjustments all call `RequestInvalidate` rather than rebuilding immediately. This prevents layout thrashing during rapid scrolling.

### Scroll Synchronization

The grid tracks `_lastScrollFirst` (the first visible row index) and compares it on every scroll offset change. When the first visible row changes, pool rows are rebound to new data indices. When only the sub-row pixel offset changes within the same row window, existing bindings are repositioned without rebinding.

Repositioning is cheaper than rebinding. During smooth scroll within the same row window, update `Canvas.Top` or `Margin` on pooled rows rather than calling `GetCellValue` for every cell.

## Column Reorder

When `AllowColumnReorder` is true, header pointer events track drag state:

```csharp
private int? _headerPointerColumn;
private Point _headerPointerOrigin;
private bool _headerDragged;
```

On pointer release after a drag exceeding a threshold, the column collection is reordered and header cells are rebuilt. A `PART_ColumnDragIndicator` border follows the pointer during drag.

Persist column order by serializing column `Header` or a stable `ColumnId` property after reorder. Restore on next app launch.

## Sorting

Clicking a sortable column header cycles through ascending, descending, and unsorted:

```csharp
private void OnHeaderClick(SkyDataGridColumn column)
{
    var newDirection = column.SortDirection switch
    {
        ListSortDirection.Ascending => ListSortDirection.Descending,
        _ => ListSortDirection.Ascending
    };

    DataSource?.Sort(column, newDirection);
    column.SortDirection = newDirection;
    InvalidateVisibleRows();
}
```

The grid updates header sort indicators; the data source performs the actual reorder.

### Multi-Column Sort

The base interface sorts one column at a time. For multi-column sort, extend the data source:

```csharp
void Sort(IReadOnlyList<(SkyDataGridColumn Column, ListSortDirection Direction)> sortKeys);
```

The grid UI may only expose single-column sort while the source supports compound keys internally.

## In-Place Editing

Double-clicking an editable cell enters edit mode. The grid swaps the read-only `TextBlock` for an input control defined by the column's `CellEditingTemplate`. On commit, `SetCellValue` is called on the data source.

```csharp
private void BeginEdit(long rowIndex, SkyDataGridColumn column)
{
    if (column.IsReadOnly) return;

    _editingRow = rowIndex;
    _editingColumn = column;
    ShowEditingTemplate(rowIndex, column);
}

private void CommitEdit(object? newValue)
{
    DataSource?.SetCellValue(_editingRow, _editingColumn!, newValue);
    EndEdit();
    InvalidateVisibleRows();
}
```

Handle Escape to cancel without calling `SetCellValue`. Handle Enter and lost-focus to commit.

Validation before commit can mirror `SkyFormField` — call a validator and show an error adorner on the cell without closing edit mode.

## Selection

`SelectedRowIndex` is a two-way styled property. When the user clicks a row, the grid updates the index and applies a selection pseudo-class to the pooled row visual:

```csharp
[PseudoClasses("selected")]
public class SkyDataGridRow : TemplatedControl { }
```

Because rows are recycled, clear selection styling when rebinding a pooled row to a different data index.

## CSV Export

`SkyDataGridCsvExporter` walks the data source sequentially:

```csharp
public static async Task ExportAsync(
    IVirtualGridDataSource source,
    IReadOnlyList<SkyDataGridColumn> columns,
    Stream output,
    CancellationToken cancellationToken)
{
    await using var writer = new StreamWriter(output);

    // Header row
    await writer.WriteLineAsync(string.Join(",", columns.Select(c => Escape(c.Header))));

    for (long i = 0; i < source.RowCount; i++)
    {
        cancellationToken.ThrowIfCancellationRequested();

        var values = columns.Select(c => source.GetCellValue(i, c));
        await writer.WriteLineAsync(string.Join(",", values.Select(Escape)));
    }
}

private static string Escape(object? value)
{
    var text = value?.ToString() ?? "";
    if (text.Contains(',') || text.Contains('"'))
        return $"\"{text.Replace("\"", "\"\"")}\"";
    return text;
}
```

Export uses the virtual interface, so it works with both in-memory lists and database-backed sources. Run export on a background thread for large datasets; marshal progress updates to the UI thread.

## Disabling Virtualization

Set `EnableRowVirtualization="False"` for small datasets where you want every row materialized (simpler debugging, supports row reorder animations). The grid creates one visual row per data row. Use this only when row count is modest (under a few hundred).

With virtualization off, DevTools shows one row control per data row, making it easier to inspect bindings. Switch back to virtualization before profiling performance.

## Debugging the Data Grid

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Blank cells | Wrong `BindingPath` or null data | Verify `GetCellValue` return value |
| Scroll jumps | `RowHeight` mismatch with actual row height | Set explicit `RowHeight`; check theme padding |
| Stale data after edit | `SetCellValue` does not raise change notification | Raise `PropertyChanged` on source |
| Only first page visible | `PART_VirtualExtent` height wrong | Set height to `RowCount * RowHeight` |
| Sort does nothing | `Sort` not implemented in source | Implement reorder in data source |
| Pool rows overlap | Reposition logic skipped | Check `_lastScrollFirst` comparison |

Enable logging in `UpdateVisibleRows` to print `firstRow`, `visibleCount`, and rebind vs reposition decisions.

## Usage Example

```xml
<SkyVirtualDataGrid DataSource="{Binding GridSource}"
                    SelectedRowIndex="{Binding SelectedIndex, Mode=TwoWay}"
                    RowHeight="32"
                    AllowColumnReorder="True">
  <SkyVirtualDataGrid.Columns>
    <SkyDataGridColumn Header="Name" BindingPath="Name" Width="200" />
    <SkyDataGridColumn Header="Email" BindingPath="Email" Width="240" />
    <SkyDataGridColumn Header="Role" BindingPath="Role" Width="120" />
  </SkyVirtualDataGrid.Columns>
</SkyVirtualDataGrid>
```

View model setup:

```csharp
public class CrudViewModel
{
    public ListVirtualGridDataSource GridSource { get; }
        = new ListVirtualGridDataSource(Records);

    public void RefreshGrid()
    {
        GridSource.NotifyDataChanged();
    }
}
```

Wire double-click to open a detail panel:

```xml
<SkyVirtualDataGrid DoubleTapped="OnRowDoubleTapped" ... />
```

```csharp
private void OnRowDoubleTapped(object? sender, TappedEventArgs e)
{
    if (DataContext is CrudViewModel vm && vm.SelectedIndex >= 0)
        vm.OpenDetail(vm.GridSource.GetItem(vm.SelectedIndex));
}
```

## Building Your Own Virtualized Control

Lessons from the data grid:

1. **Define a narrow data interface** — the control asks for rows by index, not for collections
2. **Pool visuals** — fixed pool size based on viewport, not data count
3. **Virtual extent** — a spacer element sets scrollable height without creating children
4. **Coalesce updates** — one invalidation per frame maximum
5. **Track scroll window** — rebind only when the first visible index changes
6. **Separate coordinator** — scroll math and invalidation scheduling belong in a helper class
7. **Distinguish rebind from reposition** — sub-row scroll should not re-query the data source
8. **Test with 100k rows** — profile memory and scroll frame time early

### Walkthrough: Scroll Event to Visible Rows

1. User scrolls down 64 pixels with `RowHeight = 32`.
2. `ScrollChanged` fires on `PART_Scroll`.
3. Coordinator calls `RequestInvalidate()`.
4. Next UI frame: `UpdateVisibleRows()` computes `firstRow = 2`.
5. `_lastScrollFirst` was 0 → full rebind.
6. Pool row 0 binds to data row 2; pool row 1 to data row 3; etc.
7. `PART_VirtualExtent` height unchanged; row positions offset by `scrollOffset % RowHeight`.

## Deep Dive: The Row Pool and `InvalidateVisibleRowsCore`

The heart of virtualization is `InvalidateVisibleRowsCore`. Understanding this method line-by-line explains why the grid stays fast at 100,000 rows.

### Computing the Visible Window

```csharp
var bodyOffsetY = Math.Max(0, GetScrollOffsetY());
var first = (long)(bodyOffsetY / rh);  // rh = RowHeight
var maxFirst = Math.Max(0, count - _rowModels.Count);
if (first > maxFirst)
    first = maxFirst;
```

`first` is the data row index bound to pool slot 0. If the viewport fits 20 rows and the pool holds 22 (viewport + buffer), scrolling row 50,000 means pool slot 0 shows row 50,000 — not row 0.

### Logical Scroll vs Physical Spacer

SkyUI supports two scroll strategies via `SkyVirtualDataGridScrollRoot`:

| Mode | How rows move | When used |
|------|---------------|-----------|
| **Logical scroll** | `ItemsControl` margin uses negative sub-row offset (`-partial`) | Default with scroll root |
| **Physical spacer** | `ItemsControl` margin = `first * RowHeight` | Fallback or non-virtualized mode |

Logical scroll avoids repositioning the entire pool when the user scrolls less than one row height — only the margin nudges by the fractional pixel remainder:

```csharp
var partial = bodyOffsetY - first * rh;
_rowsItems.Margin = new Thickness(0, -partial, 0, 0);
```

Physical spacer mode sets margin to push the pool down to the correct absolute offset. Both approaches keep the number of live `Control` instances constant.

### `SkyVirtualRowModel`: The Binding Bridge

Each pool slot wraps a `SkyVirtualRowModel`:

```csharp
_rowModels[i].Update(idx, ds.GetRow(idx));
_rowModels[i].SetSelected(sel.HasValue && sel.Value == idx);
```

`Update` stores the logical row index and the opaque row object from `IVirtualGridDataSource.GetRow(long)`. The row template binds cells through `GetCellValue(rowIndex, column)` — the model object is available for `RowFormatting` events but cell text comes from the data source, not from reflection on every scroll frame unless your source does that internally.

`Clear()` marks a pool slot unused when `first + i >= count` (scrolled past the last row).

### Pool Size Calculation

```csharp
private int ComputeVirtualRowPoolSize(double viewportHeight, double rowHeight, long rowCount)
{
    var visibleRows = (int)Math.Ceiling(viewportHeight / rowHeight) + 2; // +2 buffer
    var maxPool = 512; // safety cap
    // ...
}
```

The **+2 buffer** rows above/below the viewport reduce blank flashes during fast scroll before `InvalidateVisibleRowsCore` runs on the next frame. The **512 cap** prevents runaway pool growth if someone sets an enormous row height or viewport.

### Update Coordinator: Burst Protection

When the data source fires `StructureChanged` fifty times in one database refresh, the coordinator coalesces:

```csharp
public void RequestRefresh()
{
    lock (_gate)
    {
        if (_pending) return;
        _pending = true;
    }
    Dispatcher.UIThread.Post(() => { _pending = false; _flush(); }, DispatcherPriority.Background);
}
```

Only one `InvalidateVisibleRowsCore` runs per UI frame burst. Without this, sorting a column while scrolling could queue hundreds of layout passes.

### Custom Scroll Host: `SkyVirtualDataGridScrollRoot`

The header grid lives **outside** `PART_Scroll`. Body rows scroll independently. `SetLogicalExtent` tells Avalonia's scroll infrastructure the total scrollable height without creating that many children:

```csharp
_scrollRoot.SetLogicalExtent(new Size(Math.Max(1, w), bodyH));
// bodyH = RowCount * RowHeight
```

`GetScrollViewportSize()` carefully merges `ScrollViewer.Bounds`, `ScrollViewer.Viewport`, and grid `Bounds` because during template apply, not all values are valid yet — guards against `NaN` and zero prevent divide-by-zero in pool math.

### Column Hooks and Header Rebuild

Each `SkyDataGridColumn` registers hooks when added to `Columns`:

- Width changes → `UpdateVirtualExtent()` + invalidate rows
- Sort direction changes → header glyph update
- Visibility changes → header grid column rebuild

`RebuildHeader()` synchronizes `PART_HeaderGrid` column definitions with the `Columns` collection. Column drag reorder raises `ColumnReorderRequested` so the view model can persist order; the grid then mirrors the new column sequence.

### Cell Editing Pipeline

1. User double-clicks an editable cell (`IsReadOnly = false` on column).
2. Grid swaps read template for `CellEditingTemplate`.
3. On commit (Enter or focus leave), `SetCellValue(rowIndex, column, newValue)` on data source.
4. `CellEditCommitted` event fires for validation or audit logging.
5. `InvalidateVisibleRowsCore` refreshes the cell display.

Keep editing logic in the data source when rows are server-backed — the grid only knows indices.

### Row Formatting and Selection Classes

`RowFormatting` is deferred to `DispatcherPriority.Render` so rapid scroll does not invoke user code on every intermediate frame:

```csharp
var args = new SkyDataGridRowFormattingEventArgs(model.RowIndex, model.Item, rowRoot, cells);
RowFormatting.Invoke(this, args);
foreach (var cls in args.RowClasses)
    rowRoot.Classes.Add(cls);
```

Use this to highlight overdue invoices or error rows without custom row templates.

Selection sync adds `sky-grid-row-selected` to the row root after containers are realized — pool reuse means selection classes must be re-applied whenever row index at slot `i` changes.

### CSV Export and Clipboard

`ExportToCsvAsync` walks the data source sequentially by index — it never materializes the full collection:

```csharp
public Task ExportToCsvAsync(Stream destination, long startIndex, long maxRows, CancellationToken ct)
```

`CopySelectionToClipboardAsync` uses `SkyVirtualDataGridClipboard` to format TSV for Excel paste. Both paths call `GetCellValue` per cell, so they work with server-backed sources if `GetCellValue` is implemented.

### Non-Virtualized Mode

`EnableRowVirtualization = false` materializes one pool row per data row (capped by int row count). Use only for &lt; 500 rows when you need row reorder animations or simpler debugging. The code path in `InvalidateVisibleRowsCore` skips scroll window math and binds all rows directly.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│ SkyVirtualDataGrid                                       │
│  PART_HeaderGrid (fixed, sync column widths)            │
│  PART_Scroll                                             │
│    └── SkyVirtualDataGridScrollRoot (logical extent)    │
│          └── PART_VirtualExtent (body height spacer)      │
│          └── PART_Rows (ItemsControl)                    │
│                └── Pool[N] × SkyVirtualRowModel          │
│                      └── Row template (cells)            │
├─────────────────────────────────────────────────────────┤
│ IVirtualGridDataSource ←→ GetRow / GetCellValue         │
│ SkyVirtualDataGridUpdateCoordinator (coalesce)          │
└─────────────────────────────────────────────────────────┘
```

### Extended Debugging Checklist

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Blank rows while scrolling fast | Pool too small or coordinator lag | Increase buffer; check `RowHeight` matches template |
| Wrong data in rows after sort | `GetRow` not updated after sort | Data source must reorder backing store and raise `StructureChanged` |
| Scrollbar wrong size | `UpdateVirtualExtent` not called | Ensure `RowCount` notifies property change |
| Header/body column misalignment | Column width changed without header rebuild | Call `InvalidateStructure()` after bulk column changes |
| Edit commits wrong row | Stale row index during async edit | Disable edit until `GetRow` returns for current index |
| Memory grows unbounded | `EnableRowVirtualization = false` on large set | Enable virtualization |

## Summary

`SkyVirtualDataGrid` is the most performance-sensitive control in SkyUI. Its architecture — virtual data source, row pool, logical scroll with sub-row offset, update coordinator, and custom scroll root — is the reference implementation for any control that must display large datasets without choking on memory or layout cost. Master `InvalidateVisibleRowsCore` and you understand the entire control.
