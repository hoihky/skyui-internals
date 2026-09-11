---
title: Chapter 12 — CheckedListBox
order: 12
---

# Chapter 12: CheckedListBox — Adapters, Virtualization, and Tree Flattening

`CheckedListBox` is one of the most sophisticated controls in SkyUI. It renders a hierarchical checklist with optional tri-state parent checkboxes, drag reorder, inline editing, and UI virtualization. This chapter explains the adapter pattern, row flattening, and checkbox cascade that make it work.

Tree controls with checkboxes appear simple in mockups but demand careful engineering. You must bridge hierarchical domain data with flat scrollable UI, keep parent and child check states consistent, and remain responsive with thousands of nodes. `CheckedListBox` solves these problems through adapters, flattening, and virtualization.

## What CheckedListBox Does

At first glance, `CheckedListBox` looks like a `TreeView` with checkboxes. But its design goals are broader:

- Display arbitrary domain models through a pluggable adapter
- Support async child loading for large trees
- Cascade check state from parents to children (with optional tri-state)
- Virtualize row rendering for performance
- Allow drag-and-drop reorder when enabled
- Support inline rename through an editable adapter

Unlike Avalonia's built-in `TreeView`, which expects `TreeDataTemplate` and `HierarchicalDataTemplate`, `CheckedListBox` delegates all tree semantics to an adapter. That indirection costs a few method calls per row but eliminates tight coupling to your domain model.

## The Adapter Pattern

The control never assumes a specific model type. All tree semantics flow through `ICheckedListItemAdapter`:

```csharp
public interface ICheckedListItemAdapter
{
    string GetLabel(object item);
    IEnumerable? GetChildren(object item);
    bool GetIsExpanded(object item);
    void SetIsExpanded(object item, bool expanded);
    bool? GetIsChecked(object item);
    void SetIsChecked(object item, bool? value);
    bool GetHasChildren(object item);
}
```

A default implementation handles simple in-memory trees:

```csharp
public class DefaultCheckedListItemAdapter : ICheckedListItemAdapter
{
    public string GetLabel(object item) =>
        item is ICheckedListBoxItem boxItem ? boxItem.Label : item.ToString() ?? "";

    public IEnumerable? GetChildren(object item) =>
        item is ICheckedListBoxItem boxItem ? boxItem.Children : null;
    // ...
}
```

Consumers with domain models implement the interface directly or wrap models in an adapter class. This is **dependency inversion**: the control depends on an abstraction, not on your `FolderNode` or `PermissionGroup` class.

### Implementing an Adapter: Step by Step

Suppose your domain model is a `Permission` record with nested children:

```csharp
public class Permission
{
    public string Name { get; set; } = "";
    public bool? IsChecked { get; set; }
    public bool IsExpanded { get; set; }
    public List<Permission> Children { get; set; } = new();
}
```

Your adapter maps each method to model properties:

```csharp
public class PermissionAdapter : ICheckedListItemAdapter
{
    public static readonly PermissionAdapter Instance = new();

    public string GetLabel(object item) => ((Permission)item).Name;

    public IEnumerable? GetChildren(object item) => ((Permission)item).Children;

    public bool GetIsExpanded(object item) => ((Permission)item).IsExpanded;

    public void SetIsExpanded(object item, bool expanded) =>
        ((Permission)item).IsExpanded = expanded;

    public bool? GetIsChecked(object item) => ((Permission)item).IsChecked;

    public void SetIsChecked(object item, bool? value) =>
        ((Permission)item).IsChecked = value;

    public bool GetHasChildren(object item) =>
        ((Permission)item).Children.Count > 0;
}
```

Keep adapters thin. Business rules belong in the model or view model, not in adapter methods.

### Optional Adapter Extensions

| Interface | Purpose |
|-----------|---------|
| `ICheckedListEditableAdapter` | Inline rename: `BeginEdit`, `CommitEdit`, `CancelEdit` |
| `ICheckedListRowActionProvider` | Per-row context menu actions |
| `IAsyncTreeDataSource` | Lazy child loading with `LoadChildrenAsync` |

When a row enters edit mode, the control swaps the label `TextBlock` for a `TextBox`. The editable adapter receives commit and cancel callbacks:

```csharp
public interface ICheckedListEditableAdapter
{
    bool CanEdit(object item);
    string GetEditText(object item);
    bool TryCommitEdit(object item, string newText, out string? error);
}
```

`TryCommitEdit` returns false with an error message when the new name violates validation rules (duplicate name, empty string).

## Row Flattening

Tree controls face a fundamental challenge: tree data is hierarchical, but scrolling lists are flat. `CheckedListBox` maintains a flattened `ObservableCollection<CheckedListRowModel>` called `Rows`:

```
Tree:                    Flat Rows:
├─ A (expanded)          [0] A, depth=0
│  ├─ B                  [1] B, depth=1
│  └─ C                  [2] C, depth=1
└─ D (collapsed)         [3] D, depth=0
```

Only expanded nodes contribute children to the flat list. Collapsing a node removes its descendants from `Rows` without destroying the underlying tree data.

`CheckedListRowBuilder` performs the flattening walk:

```csharp
internal static void Rebuild(
    IEnumerable? items,
    ICheckedListItemAdapter adapter,
    ObservableCollection<CheckedListRowModel> rows,
    double indent,
    IComparer<object?>? comparer)
{
    rows.Clear();
    if (items is null) return;

    foreach (var item in SortItems(items, comparer))
        AppendSubtree(item, adapter, rows, indent, depth: 0);
}
```

Each `CheckedListRowModel` carries:

- Reference to the source item
- Depth level (for left indent)
- Cached label and check state
- Expand/collapse affordance visibility

### Expand and Collapse

When the user clicks an expander, the adapter updates `IsExpanded` on the source item, then the control schedules a rebuild:

```csharp
private void OnExpanderClick(CheckedListRowModel row)
{
    var expanded = !ItemAdapter!.GetIsExpanded(row.Item);
    ItemAdapter.SetIsExpanded(row.Item, expanded);
    ScheduleRebuild();
}
```

The rebuild walks the tree again from the root. This is O(n) in total nodes but coalesced to once per frame (see Rebuild Scheduling below). For very large trees, incremental flattening (insert/remove descendant rows only) is an optimization, but full rebuild keeps the code correct and simple.

### Indent Calculation

Row indent is `depth * Indent` where `Indent` is a styled property on the control (default 16–20 pixels). The row template applies a left margin or padding based on `CheckedListRowModel.Depth`:

```xml
<Border Margin="{Binding Indent, RelativeSource={RelativeSource AncestorType=ContentPresenter}}" />
```

Bind indent through the row model rather than computing in the template converter when possible — it simplifies debugging.

## UI Virtualization

The template hosts a `VirtualizingStackPanel` inside a `ScrollViewer`:

```xml
<ScrollViewer Name="PART_ScrollViewer">
  <ItemsControl Name="PART_ItemsHost">
    <ItemsControl.ItemsPanel>
      <ItemsPanelTemplate>
        <VirtualizingStackPanel />
      </ItemsPanelTemplate>
    </ItemsControl.ItemsPanel>
  </ItemsControl>
</ScrollViewer>
```

Virtualization recycles row visuals. Only visible rows get live `Control` instances. The flattened `Rows` collection can hold thousands of entries, but the visual tree stays small.

**Important limitation:** flattening still allocates one `CheckedListRowModel` per visible expanded node. Extremely deep, fully expanded trees consume memory proportional to node count even though rendering is virtualized.

### How Avalonia Virtualization Works

`VirtualizingStackPanel` measures the total extent (sum of item heights) but only creates containers for items in the viewport plus a small buffer. When you scroll, containers are recycled — the same `ContentPresenter` may bind to a different `CheckedListRowModel` after scroll.

Requirements for virtualization to work:

1. Items must be in a `VirtualizingStackPanel` (not a regular `StackPanel`)
2. The `ScrollViewer` must have a bounded height (or fill available space)
3. Item containers should have consistent height for best results

Variable-height rows work but cause more frequent re-measurement. `CheckedListBox` assumes uniform row height for predictable scroll behavior.

### Debugging Virtualization

If rows do not appear or scroll behaves erratically:

- Confirm `Rows` collection has items (bind count to a debug `TextBlock`)
- Check that the control has a defined height — virtualization needs a viewport
- Verify `ItemsSource` on the inner `ItemsControl` binds to `Rows`
- In DevTools, count visual children of the panel — should be ~viewport size, not total row count

## Checkbox Cascade

`CheckedListCheckCoordinator` manages parent-child check relationships:

```csharp
internal class CheckedListCheckCoordinator
{
    public void OnItemCheckedChanged(object item, bool? newValue)
    {
        if (_cascadeToChildren && newValue.HasValue)
            CascadeToChildren(item, newValue.Value);

        UpdateParentState(item);
    }
}
```

When `CascadeToChildren` is true and a parent is checked, all descendants receive the same check state. When `UseThreeStateForParents` is true, a parent with some but not all children checked shows an indeterminate (null) state.

Parent state recalculation walks up the tree:

1. If all children are checked → parent is checked
2. If no children are checked → parent is unchecked
3. Otherwise → parent is indeterminate (null)

### Suppressing Recursive Updates

Cascade operations can trigger a chain of `SetIsChecked` calls. The coordinator uses a reentrancy guard:

```csharp
private bool _isUpdating;

public void OnItemCheckedChanged(object item, bool? newValue)
{
    if (_isUpdating) return;

    _isUpdating = true;
    try
    {
        // cascade and parent update logic
    }
    finally
    {
        _isUpdating = false;
    }
}
```

Without the guard, updating a parent triggers child updates, which trigger parent recalculation, which can loop or fire redundant collection change notifications.

### Tri-State CheckBox Binding

Avalonia's `CheckBox.IsChecked` is `bool?`. Bind row check state directly:

```xml
<CheckBox IsChecked="{Binding CheckState, Mode=TwoWay}"
          IsThreeState="True" />
```

The row model exposes `CheckState` as a cached copy refreshed during rebuild or updated inline when the user toggles.

## Collection Change Tracking

When `ItemsSource` changes, the control subscribes to `INotifyCollectionChanged` on the root collection and recursively on child collections exposed by the adapter. Property changes on individual items (`INotifyPropertyChanged`) also trigger row updates.

This subscription graph is the most complex lifecycle concern in the control. On `ItemsSource` replacement, all old subscriptions are torn down before new ones are established to prevent leaks.

```csharp
private void UnsubscribeAll()
{
    foreach (var sub in _collectionSubscriptions)
        sub.Dispose();
    _collectionSubscriptions.Clear();
}

private void SubscribeToCollection(INotifyCollectionChanged collection)
{
    collection.CollectionChanged += OnCollectionChanged;
    _collectionSubscriptions.Add(new Subscription(() =>
        collection.CollectionChanged -= OnCollectionChanged));
}
```

Walk child collections recursively when items are expanded so that adding a child to a nested node triggers a rebuild.

### INotifyPropertyChanged on Items

If `GetLabel` reads a property that can change without collection changes, items should implement `INotifyPropertyChanged`:

```csharp
private void SubscribeToItem(object item)
{
    if (item is INotifyPropertyChanged npc)
    {
        npc.PropertyChanged += OnItemPropertyChanged;
        _itemSubscriptions[item] = npc;
    }
}

private void OnItemPropertyChanged(object? sender, PropertyChangedEventArgs e)
{
    if (e.PropertyName is nameof(Permission.Name) or nameof(Permission.IsChecked))
        ScheduleRebuild();
}
```

## Drag Reorder

When `AllowReorder` is true, `CheckedListDragReorderHandler` manages pointer capture, ghost overlay rendering, and drop index calculation:

```csharp
internal class CheckedListDragReorderHandler
{
    private readonly CheckedListBox owner;
    private Panel? reorderOverlay;

    public void OnPointerPressed(CheckedListRowModel row, PointerPressedEventArgs e) { ... }
    public void OnPointerMoved(PointerEventArgs e) { ... }
    public void OnPointerReleased(PointerReleasedEventArgs e) { ... }
}
```

The handler is an internal collaborator class rather than inline logic in `CheckedListBox`, keeping the main class readable.

Drag reorder only applies within siblings at the same tree depth. Moving a node across parent boundaries requires additional logic in your view model to restructure the tree data.

## Async Child Loading

For large trees, load children on expand:

```csharp
public interface IAsyncTreeDataSource
{
    Task<IReadOnlyList<object>> LoadChildrenAsync(
        object? parent, CancellationToken cancellationToken);
}
```

When the user expands a node with unloaded children, the control shows a placeholder row, awaits `LoadChildrenAsync`, then appends children to the model and rebuilds. Cancel in-flight loads if the user collapses before completion.

## Rebuild Scheduling

Tree mutations can trigger cascading updates. To avoid rebuilding the flat list multiple times per frame, changes schedule a single rebuild:

```csharp
private void ScheduleRebuild()
{
    if (_rebuildScheduled) return;
    _rebuildScheduled = true;
    Dispatcher.UIThread.Post(() =>
    {
        _rebuildScheduled = false;
        RebuildAll();
    });
}
```

This coalesces rapid collection changes into one layout pass.

## Row Template and ItemContainer

Each flat row renders through an `ItemTemplate` or `ItemTemplateSelector`. A typical row includes:

- Indent spacer
- Expander toggle (visible when `HasChildren`)
- CheckBox (visible when `ShowCheckBoxes`)
- Label or edit `TextBox`
- Optional action buttons

```xml
<DataTemplate x:DataType="models:CheckedListRowModel">
  <Grid ColumnDefinitions="Auto,Auto,*">
    <ToggleButton Grid.Column="0" IsVisible="{Binding ShowExpander}" />
    <CheckBox Grid.Column="1" IsVisible="{Binding ShowCheckBox}" />
    <TextBlock Grid.Column="2" Text="{Binding Label}" />
  </Grid>
</DataTemplate>
```

## Usage Example

```xml
<CheckedListBox ItemsSource="{Binding Permissions}"
                ItemAdapter="{x:Static local:PermissionAdapter.Instance}"
                CascadeToChildren="True"
                UseThreeStateForParents="True"
                ShowCheckBoxes="True"
                AllowReorder="False" />
```

With a custom adapter:

```csharp
public class PermissionAdapter : ICheckedListItemAdapter
{
    public string GetLabel(object item) => ((Permission)item).Name;
    public IEnumerable? GetChildren(object item) => ((Permission)item).Children;
    public bool GetIsExpanded(object item) => ((Permission)item).IsExpanded;
    public void SetIsExpanded(object item, bool expanded) =>
        ((Permission)item).IsExpanded = expanded;
    // check state methods...
}
```

Reading selected items from the view model:

```csharp
public IEnumerable<Permission> GetCheckedPermissions(Permission root) =>
    root.Children.SelectMany(Flatten)
        .Where(p => p.IsChecked == true);

private IEnumerable<Permission> Flatten(Permission node)
{
    yield return node;
    foreach (var child in node.Children)
        foreach (var desc in Flatten(child))
            yield return desc;
}
```

## Lessons for Custom Tree/List Controls

1. **Flatten early** — Keep a flat observable collection for the view; keep tree structure in the adapter
2. **Adapter interface** — Never bind directly to domain model types inside the control
3. **Virtualize rendering** — Use `VirtualizingStackPanel` for the items host
4. **Coalesce rebuilds** — Schedule layout-affecting work to avoid O(n²) updates
5. **Separate collaborators** — Drag, cascade, and build logic belong in dedicated classes
6. **Subscribe carefully** — Track collection and property change subscriptions to prevent leaks
7. **Guard reentrancy** — Cascade updates need suppression to avoid infinite loops
8. **Test with large trees** — Expand all nodes and verify memory and scroll performance

The next chapter shows how `SkyVirtualTreeView` specializes `CheckedListBox` for the common tree-without-checkboxes case, and how `SkyAccordion` uses the `ItemsControl` container pattern.
