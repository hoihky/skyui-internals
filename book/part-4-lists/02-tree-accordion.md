---
title: Chapter 13 — SkyVirtualTreeView and SkyAccordion
order: 13
---

# Chapter 13: SkyVirtualTreeView and SkyAccordion

This chapter covers two list controls that solve different problems. `SkyVirtualTreeView` demonstrates **inheritance for preset configuration** — reusing `CheckedListBox` machinery without checkboxes. `SkyAccordion` demonstrates the **ItemsControl container pattern** — auto-wrapping arbitrary data in typed item containers.

Choosing between inheritance and container overrides is one of the recurring design decisions in Avalonia control development. These two controls illustrate opposite ends of the spectrum: one adds almost no code; the other customizes how `ItemsControl` materializes children.

## SkyVirtualTreeView: Specialization by Inheritance

`SkyVirtualTreeView` is a tree view for navigation and browsing. It does not need checkboxes, cascade logic, or tri-state parents. Rather than duplicating the flattening and virtualization code, it subclasses `CheckedListBox` and sets sensible defaults:

```csharp
public class SkyVirtualTreeView : CheckedListBox
{
    public SkyVirtualTreeView()
    {
        ShowCheckBoxes = false;
        SelectionMode = CheckedListBoxSelectionMode.Single;
        Indent = 20;
        Classes.Add("sky-virtual-tree");
    }
}
```

That is the entire custom code. Everything else — adapter, row flattening, virtualization, expand/collapse — comes from the base class.

### When Inheritance Is the Right Tool

Inheritance for preset defaults is appropriate when:

- The subclass does not override template parts
- The base class API fully supports the narrowed use case
- The only differences are property defaults and style classes

It is **not** appropriate when the subclass needs a fundamentally different visual tree. In that case, compose or share logic through helper classes instead.

If you find yourself overriding `OnApplyTemplate` and replacing half the template parts, you have outgrown inheritance. Extract shared logic into `CheckedListRowBuilder` and `CheckedListCheckCoordinator` collaborators, then build a new `TemplatedControl` that uses them.

### Selection Mode

With `ShowCheckBoxes = false`, selection is driven by row click highlighting rather than checkbox state. `CheckedListBoxSelectionMode.Single` ensures only one row is selected at a time, matching conventional tree view behavior.

Selection is tracked on `CheckedListRowModel.IsSelected` and updated on pointer press:

```csharp
private void OnRowPointerPressed(CheckedListRowModel row, PointerPressedEventArgs e)
{
    if (SelectionMode == CheckedListBoxSelectionMode.Single)
        ClearSelection();

    row.IsSelected = true;
    SelectedItem = row.Item;
}
```

Bind `SelectedItem` two-way to your view model for navigation scenarios:

```xml
<SkyVirtualTreeView ItemsSource="{Binding RootFolders}"
                    SelectedItem="{Binding CurrentFolder, Mode=TwoWay}"
                    ItemAdapter="{x:Static local:FolderAdapter.Instance}" />
```

### Styling

The `sky-virtual-tree` style class adjusts row padding, expander icon size, and selected row highlight to match navigation tree conventions rather than checklist conventions:

```xml
<Style Selector="controls|SkyVirtualTreeView.sky-virtual-tree">
  <Setter Property="Background" Value="Transparent" />
</Style>

<Style Selector="controls|SkyVirtualTreeView.sky-virtual-tree ListBoxItem:selected">
  <Setter Property="Background" Value="{DynamicResource SkyAccentSubtleBrush}" />
</Style>
```

Style classes are preferable to separate control types when the behavior is identical. One class, multiple themes.

### Usage

```xml
<SkyVirtualTreeView ItemsSource="{Binding Folders}"
                    ItemAdapter="{x:Static local:FolderAdapter.Instance}" />
```

For async loading of large directory trees, provide an adapter backed by `IAsyncTreeDataSource`:

```csharp
public class AsyncFolderAdapter : IAsyncTreeDataSource
{
    public async Task<IReadOnlyList<object>> LoadChildrenAsync(
        object? parent, CancellationToken cancellationToken)
    {
        var path = parent is null ? "/" : ((Folder)parent).Path;
        var entries = await _fileSystem.ListAsync(path, cancellationToken);
        return entries.Cast<object>().ToList();
    }
}
```

Show a loading indicator by returning a placeholder child before the async call completes, then replace it with real children when the task finishes.

### Debugging Navigation Trees

| Issue | Check |
|-------|-------|
| Selection does not update VM | `SelectedItem` binding mode is TwoWay |
| Expand does nothing | Adapter `SetIsExpanded` mutates model; `GetIsExpanded` reads same property |
| Empty tree | `ItemsSource` binding path; root collection not empty |
| Slow expand | Async load blocking UI thread — await properly |

---

## SkyAccordion: The ItemsControl Container Pattern

`SkyAccordion` is a vertically stacked set of expandable sections. It inherits `ItemsControl` and customizes how items become visual containers.

`ItemsControl` is Avalonia's base for any control that displays a collection of items. ListBox, ComboBox, and Menu all extend it. The container factory overrides are the extension point that turns raw data into typed visual wrappers.

### Selection Modes

```csharp
public enum SkyAccordionSelectionMode
{
    Multiple,
    Single
}
```

In `Multiple` mode, any number of sections can be expanded simultaneously. In `Single` mode, expanding one section collapses all others.

Single mode suits wizards and settings where only one section should be visible. Multiple mode suits dashboards and FAQs where users compare content across sections.

### Single-Selection Enforcement

Two mechanisms cooperate:

**On selection mode change** — collapse all but the first expanded item:

```csharp
private void OnSelectionModeChanged()
{
    if (SelectionMode != SkyAccordionSelectionMode.Single)
        return;

    SkyAccordionItem? first = null;
    foreach (var ac in this.GetVisualDescendants().OfType<SkyAccordionItem>())
    {
        if (!ac.IsExpanded) continue;
        if (first is null) { first = ac; continue; }
        ac.IsExpanded = false;
    }
}
```

**On child expanded event** — collapse siblings when a section opens:

```csharp
public SkyAccordion()
{
    AddHandler(SkyAccordionItem.ExpandedEvent, OnChildExpanded, RoutingStrategies.Bubble);
}

private void OnChildExpanded(object? sender, RoutedEventArgs e)
{
    if (SelectionMode != SkyAccordionSelectionMode.Single) return;
    if (e.Source is not SkyAccordionItem opened) return;

    foreach (var ac in this.GetVisualDescendants().OfType<SkyAccordionItem>())
    {
        if (!ReferenceEquals(ac, opened))
            ac.IsExpanded = false;
    }
}
```

Using a bubbling routed event means the accordion does not need direct references to its children. Any `SkyAccordionItem` anywhere in the subtree that raises `ExpandedEvent` triggers the handler.

### Routed Events in Avalonia

`ExpandedEvent` is registered with `RoutingStrategies.Bubble`, so it travels from the source item up through ancestors. The accordion attaches a class handler with `AddHandler` rather than wiring each child in `PrepareContainerForItemOverride`. New items added later automatically participate.

To raise the event:

```csharp
item.RaiseEvent(new RoutedEventArgs(ExpandedEvent));
```

The source (`e.Source`) identifies which item expanded; the sender parameter on the handler may be the accordion itself.

### Container Factory Overrides

This is the core of the ItemsControl container pattern:

```csharp
protected override bool NeedsContainerOverride(
    object? item, int index, out object? recycleKey)
{
    recycleKey = null;
    return item is not SkyAccordionItem;
}

protected override Control CreateContainerForItemOverride(
    object? item, int index, object? recycleKey) =>
    new SkyAccordionItem();

protected override void PrepareContainerForItemOverride(
    Control container, object? item, int index)
{
    base.PrepareContainerForItemOverride(container, item, index);
    if (container is not SkyAccordionItem ac || item is SkyAccordionItem)
        return;
    if (ac.Header is null && item is not null)
        ac.Header = item.ToString();
}
```

Walkthrough:

1. `NeedsContainerOverride` returns `true` when the item is not already a `SkyAccordionItem`. Plain strings and view model objects need wrapping.

2. `CreateContainerForItemOverride` returns a new `SkyAccordionItem` container.

3. `PrepareContainerForItemOverride` sets the header from `item.ToString()` when consumers bind a collection of strings or objects without explicit headers.

The `recycleKey` parameter supports container recycling for heterogeneous collections. `SkyAccordion` returns `null` because all containers are the same type.

### ItemTemplate and DataContext

When you set `ItemTemplate`, the template applies to the **content** inside each `SkyAccordionItem`, not the item itself. The data context for the template is the bound collection item:

```xml
<DataTemplate x:DataType="vm:SettingGroupViewModel">
  <StackPanel Spacing="8">
    <TextBox Text="{Binding SettingValue}" />
    <CheckBox Content="Enabled" IsChecked="{Binding IsEnabled}" />
  </StackPanel>
</DataTemplate>
```

For richer headers, use `ItemTemplate` on the accordion for content and a separate `HeaderTemplate` or map header in `PrepareContainerForItemOverride`:

```csharp
if (item is SettingGroupViewModel vm)
    ac.Header = vm.Title;
```

### SkyAccordionItem

Each item is a `ContentControl` with expand/collapse behavior:

```csharp
[PseudoClasses("expanded")]
public class SkyAccordionItem : ContentControl
{
    public static readonly StyledProperty<object?> HeaderProperty = ...;
    public static readonly StyledProperty<bool> IsExpandedProperty = ...;

    public static readonly RoutedEvent<RoutedEventArgs> ExpandedEvent =
        RoutedEvent.Register<SkyAccordionItem, RoutedEventArgs>(
            nameof(Expanded), RoutingStrategies.Bubble);

    static SkyAccordionItem()
    {
        IsExpandedProperty.Changed.AddClassHandler<SkyAccordionItem>(
            (item, e) =>
            {
                item.PseudoClasses.Set(":expanded", (bool)e.NewValue!);
                if ((bool)e.NewValue!)
                    item.RaiseEvent(new RoutedEventArgs(ExpandedEvent));
            });
    }
}
```

The `:expanded` pseudo-class drives the chevron rotation and content visibility in the theme template.

### Accordion Template Anatomy

A typical `SkyAccordionItem` template:

```xml
<ControlTemplate>
  <Border>
    <DockPanel>
      <ToggleButton DockPanel.Dock="Top"
                    IsChecked="{TemplateBinding IsExpanded, Mode=TwoWay}"
                    Content="{TemplateBinding Header}" />
      <ContentPresenter IsVisible="{TemplateBinding IsExpanded}"
                        Content="{TemplateBinding Content}" />
    </DockPanel>
  </Border>
</ControlTemplate>
```

Alternatively, drive visibility through the `:expanded` pseudo-class on the content presenter:

```xml
<Style Selector="controls|SkyAccordionItem:expanded ContentPresenter">
  <Setter Property="IsVisible" Value="True" />
</Style>
```

Animating height with `Transitions` on `MaxHeight` or `Opacity` produces smoother expand/collapse than toggling `IsVisible` abruptly.

### Usage Patterns

**XAML children (explicit items):**

```xml
<SkyAccordion SelectionMode="Single">
  <SkyAccordionItem Header="General" IsExpanded="True">
    <StackPanel>
      <CheckBox Content="Enable notifications" />
    </StackPanel>
  </SkyAccordionItem>
  <SkyAccordionItem Header="Privacy">
    <TextBlock Text="Privacy settings here." />
  </SkyAccordionItem>
</SkyAccordion>
```

**Data-bound with automatic containers:**

```xml
<SkyAccordion ItemsSource="{Binding SettingGroups}"
              ItemTemplate="{StaticResource SettingGroupTemplate}" />
```

When `ItemsSource` contains view model objects, the container factory wraps each in a `SkyAccordionItem` and applies `ItemTemplate` to the content.

**Mixed: explicit headers from view model:**

```csharp
public class SettingGroup
{
    public string Title { get; set; } = "";
    public ObservableCollection<Setting> Settings { get; set; } = new();
}
```

```xml
<SkyAccordion ItemsSource="{Binding Groups}">
  <ItemsControl.ItemTemplate>
    <DataTemplate x:DataType="vm:SettingGroup">
      <ItemsControl ItemsSource="{Binding Settings}" />
    </DataTemplate>
  </ItemsControl.ItemTemplate>
</SkyAccordion>
```

Override `PrepareContainerForItemOverride` to set `Header` from `Title`.

## Comparing the Two Approaches

| Aspect | SkyVirtualTreeView | SkyAccordion |
|--------|-------------------|--------------|
| Base class | CheckedListBox (inheritance) | ItemsControl (container override) |
| Data shape | Hierarchical tree | Flat list of sections |
| Virtualization | Yes (from base) | No (typically few sections) |
| Primary pattern | Preset defaults on existing control | Auto-container wrapping |
| Best for | File trees, nav hierarchies | Settings panels, FAQs |
| Customization | Style class + adapter | Container factory + item template |

## Building Your Own ItemsControl

When you need an items control with custom item chrome:

1. Subclass `ItemsControl`
2. Override `NeedsContainerOverride` to detect when wrapping is needed
3. Override `CreateContainerForItemOverride` to return your container type
4. Override `PrepareContainerForItemOverride` to map data properties to container properties
5. Use routed events on the container for interaction that the parent must handle
6. Use pseudo-classes on the container for expand/collapse, selection, or error states
7. Set `ItemsPanel` to control layout (vertical stack, wrap, grid)

### Walkthrough: Tag List Control

Imagine a read-only tag list that wraps each string in a pill-shaped border:

```csharp
public class SkyTagList : ItemsControl
{
    protected override bool NeedsContainerOverride(
        object? item, int index, out object? recycleKey)
    {
        recycleKey = null;
        return item is not SkyTagItem;
    }

    protected override Control CreateContainerForItemOverride(
        object? item, int index, object? recycleKey) =>
        new SkyTagItem();

    protected override void PrepareContainerForItemOverride(
        Control container, object? item, int index)
    {
        base.PrepareContainerForItemOverride(container, item, index);
        if (container is SkyTagItem tag && item is string text)
            tag.Label = text;
    }
}
```

This follows the same three-override pattern as `SkyAccordion`.

### Debugging Container Issues

- **Items appear empty** — `ItemTemplate` may be null; check `PrepareContainerForItemOverride` maps data to container properties
- **Duplicate wrappers** — `NeedsContainerOverride` returns true for items already wrapped
- **Wrong data context** — Template binds to container instead of item; ensure `Content` or `DataContext` is set in prepare
- **Single mode not collapsing** — Routed event not bubbling; verify `RoutingStrategies.Bubble` and `AddHandler` on parent

## Summary

`SkyVirtualTreeView` proves that not every control needs new logic — sometimes the right move is configuring an existing control for a narrower scenario. `SkyAccordion` shows the Avalonia container pattern that every list-based control builds upon. Together they cover the two most common ways to extend list behavior in SkyUI: inheritance and container factory overrides.
