---
title: Chapter 16 — FilterEditor
order: 16
---

# Chapter 16: FilterEditor

`FilterEditor` in the `SkyUI.Data` package is a hierarchical filter builder — groups with AND/OR logic containing individual field conditions. It demonstrates **ViewModel inside a control**, **document model separation**, and **pluggable SQL export**.

## Problem and Shape

Data grids often need ad-hoc filtering: "Status is Active AND (Region is EU OR Region is UK)". `FilterEditor` provides a visual tree for building that expression, with optional SQL preview for power users.

```
FilterDocument
└── FilterGroupNode (AND)
    ├── FilterConditionNode (Status = Active)
    └── FilterGroupNode (OR)
        ├── FilterConditionNode (Region = EU)
        └── FilterConditionNode (Region = UK)
```

## Control API

```csharp
public class FilterEditor : TemplatedControl
{
    public static readonly StyledProperty<FilterDocument?> DocumentProperty = ...;
    public static readonly StyledProperty<IFilterSqlExporter?> SqlExporterProperty = ...;
    public static readonly StyledProperty<bool> ShowSqlPreviewProperty = ...; // default true
}
```

| Property | Role |
|----------|------|
| `Document` | Root filter tree model |
| `SqlExporter` | Strategy for SQL string generation |
| `ShowSqlPreview` | Toggles preview panel in template |

## ViewModel Bridge Pattern

The control creates an internal `FilterEditorViewModel` when `Document` is assigned:

```csharp
private void OnDocumentOrExporterChanged()
{
    if (Document is null)
    {
        TearDownViewModel();
        DataContext = null;
        return;
    }

    if (_viewModel?.Document != Document)
    {
        TearDownViewModel();
        _viewModel = new FilterEditorViewModel(Document, SqlExporter ?? new BasicFilterSqlExporter());
        DataContext = _viewModel;
    }
    else
    {
        _viewModel!.SqlExporter = SqlExporter ?? new BasicFilterSqlExporter();
    }
}
```

### Why Internal ViewModel?

Filter UI involves dozens of bindable commands: add group, add condition, remove node, toggle AND/OR, change operator. Putting all of that on the control class would bloat the public API. The internal ViewModel:

- Keeps `FilterEditor` properties minimal (`Document`, `SqlExporter`, `ShowSqlPreview`)
- Enables template bindings to `{Binding AddConditionCommand}` etc.
- Allows unit testing ViewModel logic without visual tree

### Lifecycle Cleanup

```csharp
protected override void OnDetachedFromLogicalTree(LogicalTreeAttachmentEventArgs e)
{
    TearDownViewModel();
    base.OnDetachedFromLogicalTree(e);
}

private void TearDownViewModel()
{
    _viewModel?.Dispose();
    _viewModel = null;
}
```

Disposing on detach prevents subscriptions to `Document` change events from leaking.

## Document Model

```csharp
public class FilterDocument
{
    public FilterGroupNode Root { get; }
    public IReadOnlyList<FilterFieldDescriptor> Fields { get; }
}

public abstract class FilterNode { }

public class FilterGroupNode : FilterNode
{
    public FilterGroupOperator Operator { get; set; } // And, Or
    public ObservableCollection<FilterNode> Children { get; }
}

public class FilterConditionNode : FilterNode
{
    public string FieldName { get; set; }
    public FilterOperator Operator { get; set; } // Equals, Contains, GreaterThan, ...
    public object? Value { get; set; }
}
```

`FilterFieldDescriptor` describes available fields (name, type, allowed operators) so the UI can populate dropdowns without hard-coding schema.

## IFilterSqlExporter Strategy

```csharp
public interface IFilterSqlExporter
{
    string Export(FilterGroupNode root);
}
```

`BasicFilterSqlExporter` generates parameterized SQL WHERE clauses. Consumers with custom dialects implement the interface:

```csharp
public class PostgreSqlFilterExporter : IFilterSqlExporter
{
    public string Export(FilterGroupNode root) =>
        BuildWhereClause(root, dialect: SqlDialect.PostgreSql);
}
```

```xml
<FilterEditor Document="{Binding FilterDoc}"
              SqlExporter="{x:Static local:PostgreSqlExporter.Instance}"
              ShowSqlPreview="True" />
```

## Template Structure

The theme template typically contains:

- Tree view or nested panels for group/condition hierarchy
- Operator dropdowns on group headers
- Field/operator/value editors on condition rows
- SQL preview `TextBox` bound to `ViewModel.SqlPreview` (read-only)

## Integration with Virtual Grid

```csharp
public class CrudViewModel
{
    public FilterDocument FilterDocument { get; } = CreateDefaultFilter();
    public IVirtualGridDataSource GridSource { get; private set; }

    public void ApplyFilter()
    {
        var sql = _exporter.Export(FilterDocument.Root);
        GridSource = new SqlFilteredDataSource(_connection, sql);
    }
}
```

```xml
<SkyDrawer IsOpen="{Binding IsFilterOpen}" DrawerContent="{Binding FilterPanel}" />
<!-- FilterPanel contains FilterEditor -->
```

## Patterns Summary

| Pattern | Application |
|---------|-------------|
| Document model | Serializable filter tree independent of UI |
| Internal ViewModel | Rich command surface without public control API bloat |
| Strategy | `IFilterSqlExporter` for SQL dialects |
| IDisposable teardown | Unsubscribe on logical detach |

## Building a Similar Editor Control

1. Define a **document** model the app can save/load
2. Create an internal **ViewModel** with commands for tree mutations
3. Set `DataContext = viewModel` when document changes
4. Keep **export/evaluation** pluggable via interfaces
5. **Dispose** subscriptions when document clears or control detaches

## Deep Dive: Document Model and Visitor Pattern

### FilterDocument Root

```csharp
public sealed class FilterDocument
{
    public FilterGroupNode Root { get; } = new();
    public ObservableCollection<FilterFieldDescriptor> Fields { get; } = new();
}
```

`Fields` describes what the user can filter on — name, CLR type, allowed operators. The UI populates field dropdowns from this collection; SQL export uses `FieldPath` for column identifiers.

### FilterNodeBase Hierarchy

```
FilterNodeBase (abstract)
├── FilterGroupNode     → LogicalKind: And | Or, Children collection
└── FilterConditionNode → FieldPath, Operator, ValueText
```

Every node implements the **visitor pattern**:

```csharp
public abstract class FilterNodeBase : INotifyPropertyChanged
{
    public abstract T Accept<T>(IFilterNodeVisitor<T> visitor);
    public FilterGroupNode? Parent { get; internal set; }
}
```

Visitors traverse the tree without `switch` on node types scattered through the UI — SQL export, JSON serialization, and runtime evaluation each get a dedicated visitor implementation.

### FilterConditionNode

```csharp
public sealed class FilterConditionNode : FilterNodeBase
{
    public string FieldPath { get; set; }
    public FilterCompareOperator Operator { get; set; }
    public string? ValueText { get; set; }
}
```

Operators include `Equal`, `NotEqual`, `Contains`, `StartsWith`, `IsNull`, `GreaterThan`, and more. The theme renders operator dropdowns from `FilterEditorThemeLists.CompareOperators`.

`ValueText` is deliberately a string — the editor normalizes display; exporters parse to SQL literals. Typed evaluation visitors can parse to `int`, `DateTime`, etc.

### Group Mutation API

```csharp
public FilterConditionNode AddCondition(string fieldPath, FilterCompareOperator op, string? valueText = null)
{
    var c = new FilterConditionNode(fieldPath, op, valueText);
    _children.Add(c);
    return c;
}

public FilterGroupNode AddGroup(FilterLogicalKind kind = FilterLogicalKind.And)
{
    var g = new FilterGroupNode { LogicalKind = kind };
    _children.Add(g);
    return g;
}
```

`FilterGroupNode` maintains parent pointers on `CollectionChanged` so `RemoveSelected` can walk up the tree and delete nodes without orphaned references.

## Deep Dive: FilterEditorViewModel Commands

The ViewModel is created once per `Document` instance and becomes the control's `DataContext`:

```csharp
AddAndGroupCommand = new FilterEditorRelayCommand(() => AddGroup(FilterLogicalKind.And));
AddOrGroupCommand = new FilterEditorRelayCommand(() => AddGroup(FilterLogicalKind.Or));
AddConditionCommand = new FilterEditorRelayCommand(AddCondition);
RemoveSelectedCommand = new FilterEditorRelayCommand(RemoveSelected, CanRemoveSelected);
RemoveNodeCommand = new FilterEditorNodeCommand(RemoveNode);
```

### Adding a Condition to the Selected Group

```csharp
private void AddCondition()
{
    var target = SelectedNode switch
    {
        FilterGroupNode g => g,
        FilterConditionNode c => c.Parent,
        _ => _document.Root
    };
    if (target is null) return;

    var field = _document.Fields.FirstOrDefault()?.FieldPath ?? "Field";
    target.AddCondition(field, FilterCompareOperator.Equal, "");
    RefreshSql();
}
```

Selection context matters: if the user selects a condition row, new conditions append to its parent group — matching DevExpress-style filter UIs.

### Subscription Graph

On construction, the ViewModel wires every node in the tree:

```csharp
WireFilterGroup(_document.Root);

private void WireFilterGroup(FilterGroupNode g)
{
    g.Children.CollectionChanged += OnGroupChildrenChanged;
    foreach (var child in g.Children)
        WireNode(child);
}

private void WireNode(FilterNodeBase node)
{
    node.PropertyChanged += OnNodePropertyChanged;
    if (node is FilterGroupNode childGroup)
        WireFilterGroup(childGroup);
}
```

Any change — new child, operator edit, value text change — calls `RefreshSql()`. On dispose, `UnwireFilterGroup` tears down the graph to prevent leaks when `Document` is replaced.

### Live SQL Preview

```csharp
private void RefreshSql()
{
    SqlPreview = _exporter.ToSql(_document);
}
```

The preview panel binds `{Binding SqlPreview}` read-only. Power users verify the filter before applying; developers debug exporter implementations.

## Deep Dive: BasicFilterSqlExporter

SQL generation is a visitor over the same tree the UI edits:

```csharp
public string VisitGroup(FilterGroupNode node)
{
    if (node.Children.Count == 0)
        return "(1 = 1)";

    var sep = node.LogicalKind == FilterLogicalKind.And ? " AND " : " OR ";
    var sb = new StringBuilder();
    sb.Append('(');
    foreach (var child in node.Children)
    {
        sb.Append(sep);
        sb.Append(child.Accept(this));
    }
    sb.Append(')');
    return sb.ToString();
}
```

Condition visit maps operators to SQL:

| Operator | SQL shape |
|----------|-----------|
| `Contains` | `"Column" LIKE '%' \|\| 'value' \|\| '%'` |
| `IsNull` | `"Column" IS NULL` |
| `GreaterThan` | `"Column" > 'value'` |

Identifiers are double-quoted; string literals single-quoted with escape doubling. Replace `BasicFilterSqlExporter` with `PostgreSqlFilterExporter` or `SqlServerFilterExporter` for dialect-specific syntax — the UI unchanged.

## End-to-End Walkthrough: Filter + Grid

**Step 1 — Define fields when the page loads:**

```csharp
var doc = new FilterDocument();
doc.Fields.Add(new FilterFieldDescriptor("Status", typeof(string)));
doc.Fields.Add(new FilterFieldDescriptor("Amount", typeof(decimal)));
doc.Root.AddCondition("Status", FilterCompareOperator.Equal, "Active");
```

**Step 2 — Bind the editor:**

```xml
<FilterEditor Document="{Binding FilterDoc}"
              SqlExporter="{x:Static local:AppSqlExporter.Instance}"
              ShowSqlPreview="True" />
```

**Step 3 — User builds `(Status = Active AND Amount > 1000)` in the UI.**

**Step 4 — Apply button runs:**

```csharp
var whereClause = exporter.ToSql(FilterDoc);
GridSource = new SqlVirtualGridDataSource(_db, baseQuery, whereClause);
Grid.InvalidateStructure();
```

**Step 5 — Grid's `IVirtualGridDataSource` executes paged queries with the WHERE clause appended.

### Serializing Filters

Because the model is a plain object tree, serialize to JSON for saved views:

```csharp
// Pseudocode: visitor that emits JSON nodes
var json = FilterJsonVisitor.Export(document.Root);
await File.WriteAllTextAsync("saved-filter.json", json);
```

Restore on load and assign back to `FilterEditor.Document`.

## Why ViewModel Inside the Control?

Alternatives considered in SkyUI:

| Approach | Drawback |
|----------|----------|
| Put commands on `FilterEditor` | 15+ commands pollute control API |
| Require external ViewModel | Every consumer duplicates wiring |
| Internal ViewModel | ✅ Single assignment: set `Document` |

Consumers with advanced needs can still bind their own ViewModel alongside — set `Document` from outside and listen to `PropertyChanged` on condition nodes.

## Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Reuse `FilterDocument` without unwiring | Duplicate SQL refresh handlers |
| Empty `Fields` collection | Add Condition creates invalid field paths |
| Mutate tree without `ObservableCollection` | UI does not update |
| SQL exporter not parameterized | Injection risk if `ValueText` comes from untrusted input — sanitize in exporter |

## Summary

`FilterEditor` is SkyUI's example of a **domain-specific editor control** — document tree + visitor pattern for export, internal ViewModel for commands, and subscription wiring for live SQL preview. The complexity lives in the model and ViewModel; the `FilterEditor` class itself is under 90 lines because it delegates aggressively.
