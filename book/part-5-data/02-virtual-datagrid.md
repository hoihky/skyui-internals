---
title: Chapter 15 — SkyVirtualDataGrid
order: 15
---

# Chapter 15: SkyVirtualDataGrid

`SkyVirtualDataGrid` in the `SkyUI.Data` package is a high-performance tabular data control. Unlike `CheckedListBox`, which virtualizes a flat list of tree rows, the data grid must virtualize across two dimensions: rows and columns, while supporting sorting, column reorder, in-place editing, and CSV export. This chapter explains the virtual data source contract, row pooling, and update coordination.

Data grids are where UI performance meets data complexity. A naive implementation creates one row control per record — fine for fifty rows, catastrophic for fifty thousand. `SkyVirtualDataGrid` treats the viewport as a window into a much larger dataset, binding only the rows you can see.

## The Virtual Data Source Contract

The grid does not require a materialized `ObservableCollection` of every row. It asks the data layer for **one row object at a time** through `IVirtualGridDataSource`:

```csharp
public interface IVirtualGridDataSource
{
    long RowCount { get; }
    object? GetRow(long index);
    event EventHandler? StructureChanged;

    void ApplySort(SkyDataGridColumn? column, SkyDataGridSortDirection direction) { }
}
```

Design points that matter when you implement or consume this API:

- **`GetRow` returns the row payload**, not a formatted cell string. Columns read properties from that object.
- **`StructureChanged`** is the invalidation signal. When row count or ordering changes, raise this event; the grid resets scroll bookkeeping and refreshes the pool.
- **`ApplySort` is optional** with a default no-op. The grid updates `SkyDataGridColumn.SortDirection` in the UI, then calls `ApplySort` so your source can reorder an in-memory list or append an `ORDER BY` clause for server data.
- **Reads should be fast** — `GetRow` runs on the UI thread during scroll. Heavy work belongs in background loading with `SkyVirtualDataGrid.Post()` to marshal refresh back to the UI thread (documented on the interface).

### ListVirtualGridDataSource

For moderate in-memory lists, SkyUI ships a bridge type:

```csharp
public sealed class ListVirtualGridDataSource : IVirtualGridDataSource
{
    public IList List { get; set; }  // subscribes to INotifyCollectionChanged

    public long RowCount => _list.Count;

    public object? GetRow(long index) =>
        index < 0 || index >= _list.Count ? null : _list[(int)index];

    public event EventHandler? StructureChanged;
}
```

Assign `List` to your view model collection. Collection changes automatically raise `StructureChanged`.

### Server-Backed Source (Custom)

A paged database source typically caches windows of rows:

```csharp
public sealed class PagedVirtualGridDataSource : IVirtualGridDataSource
{
    public long RowCount { get; private set; }

    public object? GetRow(long index)
    {
        var page = index / PageSize;
        var cache = EnsurePage(page);
        return cache[(int)(index % PageSize)];
    }

    public void ApplySort(SkyDataGridColumn? column, SkyDataGridSortDirection direction)
    {
        _sortColumn = column?.BindingPath;
        _sortDirection = direction;
        _pageCache.Clear();
        StructureChanged?.Invoke(this, EventArgs.Empty);
    }

    public event EventHandler? StructureChanged;
}
```

After `ApplySort`, clear caches so the next `GetRow` calls fetch with the new ordering.

### Where Cell Text Comes From

The data source does **not** expose per-column getters. Column display uses:

1. **`SkyDataGridColumn.BindingPath`** — dotted property path on the row object
2. **`SkyDataGridColumn.CellTemplate`** — full `IDataTemplate` when you need custom visuals
3. **`SkyDataGridCellFormatter`** — shared reflection helper for export and clipboard:

```csharp
public static class SkyDataGridCellFormatter
{
    public static object? ResolveValue(object? row, string? bindingPath);
    public static string FormatCell(object? row, SkyDataGridColumn column);
}
```

During virtualization, row templates bind to `SkyVirtualRowModel.Item` (the object from `GetRow`). The grid builds default text cells from `BindingPath` when no template is set.

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

Repositioning is cheaper than rebinding. During smooth scroll within the same row window, update `Canvas.Top` or `Margin` on pooled rows rather than re-running `GetRow` and formatter logic for every cell.

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

    column.SortDirection = MapToGridSort(newDirection);
    DataSource?.ApplySort(column, column.SortDirection);
    InvalidateVisibleRows();
}
```

The grid updates header sort indicators; **`ApplySort`** is where your source reorders an in-memory list or records sort metadata for a remote query. The default interface implementation is a no-op.

### Multi-Column Sort

The stock grid toggles one column at a time. Compound sort belongs in your `ApplySort` implementation: read `column.BindingPath` and `SkyDataGridSortDirection`, then sort by multiple keys inside the backing store before raising `StructureChanged`.

## In-Place Editing

Double-clicking an editable cell (`IsReadOnly = false` on the column) enters inline edit mode. The grid swaps the read-only `TextBlock` for a `TextBox` inside the cell host. On commit, it raises **`CellEditCommitted`** — the data source is not updated automatically.

```csharp
private void BeginEdit(long rowIndex, SkyDataGridColumn column)
{
    if (column.IsReadOnly) return;

    _editingRow = rowIndex;
    _editingColumn = column;
    ShowEditingTemplate(rowIndex, column);
}

private void CommitEdit(string newText)
{
    var column = _editingColumn!;
    var rowIndex = _editingRow;
    EndEdit();
    CellEditCommitted?.Invoke(this,
        new SkyDataGridCellEditEventArgs(rowIndex, column, newText));
    InvalidateVisibleRows();
}
```

Handle Escape to cancel without raising `CellEditCommitted`. Handle Enter and lost-focus to commit. Persist changes in a `CellEditCommitted` handler by updating the object returned from `GetRow(rowIndex)`.

Validation before commit can mirror `SkyFormField` — call a validator and show an error adorner on the cell without closing edit mode.

## Selection

`SelectedRowIndex` is a two-way styled property. When the user clicks a row, the grid updates the index and applies a selection pseudo-class to the pooled row visual:

```csharp
[PseudoClasses("selected")]
public class SkyDataGridRow : TemplatedControl { }
```

Because rows are recycled, clear selection styling when rebinding a pooled row to a different data index.

## CSV Export

`SkyVirtualDataGrid.ExportToCsvAsync` delegates to `SkyDataGridCsvExporter`, which walks row indices and formats each column through `SkyDataGridCellFormatter`:

```csharp
public Task ExportToCsvAsync(Stream destination, long startIndex, long maxRows, CancellationToken ct = default)
{
    var exporter = new SkyDataGridCsvExporter();
    return exporter.ExportAsync(DataSource!, Columns, destination, startIndex, maxRows, ct);
}
```

Inside the exporter, each row is `dataSource.GetRow(i)` and each cell is `SkyDataGridCellFormatter.FormatCell(row, columns[c])`. Custom `CellTemplate` content is not rasterized — export uses binding paths and formatter logic, matching clipboard behavior.

Export uses the virtual interface, so it works with both in-memory lists and database-backed sources. Run export on a background thread for large datasets; marshal progress updates to the UI thread.

## Disabling Virtualization

Set `EnableRowVirtualization="False"` for small datasets where you want every row materialized (simpler debugging, supports row reorder animations). The grid creates one visual row per data row. Use this only when row count is modest (under a few hundred).

With virtualization off, DevTools shows one row control per data row, making it easier to inspect bindings. Switch back to virtualization before profiling performance.

## Debugging the Data Grid

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Blank cells | Wrong `BindingPath` or null `GetRow` | Verify `ResolveValue` on row object |
| Scroll jumps | `RowHeight` mismatch with actual row height | Set explicit `RowHeight`; check theme padding |
| Stale data after edit | Handler did not mutate row object | Update model in `CellEditCommitted`; call `StructureChanged` if needed |
| Only first page visible | `PART_VirtualExtent` height wrong | Set height to `RowCount * RowHeight` |
| Sort does nothing | `ApplySort` left as default no-op | Reorder backing store and raise `StructureChanged` |
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

`Update` stores the logical row index and the opaque row object from `IVirtualGridDataSource.GetRow(long)`. Default cells bind to properties on `Item` via each column's `BindingPath`; `SkyDataGridCellFormatter` centralizes the same resolution for export and clipboard. `RowFormatting` receives both index and `Item` so you can add row classes without custom templates.

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
2. Grid hosts an inline `TextBox` in the cell (editing template support is column-driven).
3. On commit (Enter or focus leave), the grid raises **`CellEditCommitted`** with row index, column, and new text.
4. Your handler updates the row model (or calls an API) and optionally raises `StructureChanged`.
5. `InvalidateVisibleRowsCore` refreshes the cell display.

The grid never mutates your domain objects directly — indices and text are all it knows. Server-backed rows should persist in the commit handler, then refresh or patch the cache before the next `GetRow`.

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

`CopySelectionToClipboardAsync` uses `SkyVirtualDataGridClipboard` to format TSV for Excel paste. Both clipboard and CSV paths call `GetRow` once per row and `SkyDataGridCellFormatter` per column, so server-backed sources work as long as `GetRow` returns current data.

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
│ IVirtualGridDataSource ←→ GetRow + CellFormatter        │
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
