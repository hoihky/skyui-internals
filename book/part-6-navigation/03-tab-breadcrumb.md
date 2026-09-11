---
title: Chapter 19 — SkyTabView and SkyBreadcrumb
order: 19
---

# Chapter 19: SkyTabView and SkyBreadcrumb

Navigation is not only sidebars. Tabs organize peer views at the same hierarchy level; breadcrumbs show where the user is in a deep hierarchy. This chapter covers `SkyTabView` (minimal `TabControl` specialization) and `SkyBreadcrumb` (styled `ItemsControl`).

## SkyTabView: Overriding Defaults

```csharp
public class SkyTabView : TabControl
{
    static SkyTabView()
    {
        TabStripPlacementProperty.OverrideDefaultValue<SkyTabView>(Dock.Top);
    }
}
```

That is the entire C# implementation. Visual design lives in the theme.

### Avalonia Concept: OverrideDefaultValue

`OverrideDefaultValue<T>()` in a static constructor changes the default for a styled property **for this type only**. `TabControl` defaults to top placement already, but explicit override documents intent and protects against upstream Avalonia default changes.

### What TabControl Provides

`TabControl` is an `ItemsControl` with:

- `Items` / `ItemsSource` — tab entries
- `SelectedIndex` / `SelectedItem` — active tab
- `TabStripPlacement` — top, bottom, left, right
- Automatic content switching — only selected tab's content is visible

SkyUI's theme applies **pill-style** tab strips:

```xml
<Style Selector="TabControl.sky-tab-view">
  <Setter Property="Padding" Value="0" />
</Style>
<Style Selector="TabControl.sky-tab-view > TabItem">
  <Setter Property="Template" Value="{StaticResource SkyTabItemPillTemplate}" />
</Style>
```

### Usage with View Models

```xml
<SkyTabView SelectedIndex="{Binding SelectedTabIndex}">
  <TabItem Header="Overview">
    <local:OverviewView />
  </TabItem>
  <TabItem Header="Details">
    <local:DetailsView />
  </TabItem>
  <TabItem Header="History">
    <local:HistoryView />
  </TabItem>
</SkyTabView>
```

Or data-bound:

```xml
<SkyTabView ItemsSource="{Binding Sections}"
            SelectedItem="{Binding CurrentSection, Mode=TwoWay}">
  <TabControl.ItemTemplate>
    <DataTemplate>
      <TextBlock Text="{Binding Title}" />
    </DataTemplate>
  </TabControl.ItemTemplate>
  <TabControl.ContentTemplate>
    <DataTemplate>
      <ContentControl Content="{Binding View}" />
    </DataTemplate>
  </TabControl.ContentTemplate>
</SkyTabView>
```

### When to Use Tabs vs NavigationView

| Scenario | Control |
|----------|---------|
| App-level sections (Home, Library, Settings) | `SkyNavigationView` |
| Page-level sub-views (Details, Permissions, Audit) | `SkyTabView` |
| Mobile bottom navigation | `SkyNavigationView` bottom mode |

---

## SkyBreadcrumb: Minimal ItemsControl

```csharp
public class SkyBreadcrumb : ItemsControl
{
    public static readonly StyledProperty<string> SeparatorProperty =
        AvaloniaProperty.Register<SkyBreadcrumb, string>(nameof(Separator), "/");
}
```

### Design Philosophy

`SkyBreadcrumb` exposes only `Separator`. Item rendering, click handling, and truncation are template concerns:

```xml
<ControlTheme TargetType="controls:SkyBreadcrumb">
  <Setter Property="ItemsPanel">
    <ItemsPanelTemplate>
      <StackPanel Orientation="Horizontal" Spacing="4" />
    </ItemsPanelTemplate>
  </Setter>
  <Setter Property="ItemTemplate">
    <DataTemplate>
      <Button Classes="sky sky-subtle"
              Content="{Binding Label}"
              Command="{Binding NavigateCommand}" />
    </DataTemplate>
  </Setter>
</ControlTheme>
```

Separators between items are typically implemented with an `ItemsControl` alternate template or a custom panel in advanced themes. The `Separator` property supplies the string inserted between items (default `/`).

### View Model Pattern

```csharp
public class BreadcrumbItem
{
    public string Label { get; init; } = "";
    public ICommand? NavigateCommand { get; init; }
    public bool IsCurrent { get; init; }
}
```

```csharp
Breadcrumbs = new[]
{
    new BreadcrumbItem { Label = "Projects", NavigateCommand = GoProjects },
    new BreadcrumbItem { Label = "SkyUI", NavigateCommand = GoProject },
    new BreadcrumbItem { Label = "Settings", IsCurrent = true },
};
```

```xml
<SkyBreadcrumb ItemsSource="{Binding Breadcrumbs}"
               Separator="›" />
```

### Accessibility

Mark the current (last) item with `IsCurrent` in your template — disable its button or render as `TextBlock` so screen readers announce it as the current page, not a link.

## ItemsControl vs TemplatedControl for Lists

Both `SkyBreadcrumb` and `SkyTabView` build on `ItemsControl` family types rather than `TemplatedControl` because:

- Item count is dynamic
- Each item may use `DataTemplate`
- Platform (`TabControl`, `ItemsControl`) handles selection and containers

Reach for `TemplatedControl` when the control has a **fixed** structure with named parts; reach for `ItemsControl` when the control is primarily a **collection of variable items**.

## Deep Dive: TabControl Selection and Content Lifecycle

### How TabControl Switches Content

`TabControl` keeps all tab items in the visual tree but typically shows only the selected tab's content. Avalonia's `TabItem` template includes a `ContentPresenter` whose `IsVisible` or content host toggles with selection. Understanding this matters when tabs host heavy views:

```xml
<SkyTabView SelectedIndex="{Binding TabIndex, Mode=TwoWay}">
  <TabItem Header="Dashboard">
    <local:DashboardView />  <!-- constructed when tab strip loads -->
  </TabItem>
</SkyTabView>
```

For **lazy-loaded** tabs, use `ContentTemplate` with a ViewModel that creates views on first selection:

```csharp
public class TabSection
{
    public string Title { get; init; } = "";
    public object? View { get; set; }  // null until first activate
    public Func<object> ViewFactory { get; init; } = () => new TextBlock();
}

public void OnTabSelected(TabSection section)
{
    section.View ??= section.ViewFactory();
}
```

### Tab Strip Placement and Keyboard

`TabStripPlacement` supports `Top`, `Bottom`, `Left`, `Right`. SkyUI's pill theme targets top placement. For vertical tabs (settings side nav), set:

```xml
<SkyTabView TabStripPlacement="Left" />
```

Arrow keys navigate between tabs when the tab strip has focus — inherited from Avalonia `TabControl` keyboard navigation. Ensure `TabIndex` on surrounding controls does not trap focus away from the strip.

### SkyTabView Theme Anatomy

The pill template typically structures:

```
Border (strip background)
└── ItemsPresenter (horizontal TabItems)
      └── TabItem template
            ├── Border (pill background, :selected pseudo)
            └── ContentPresenter (header)
Content area below strip
└── ContentPresenter (selected tab content only)
```

`:selected` on `TabItem` drives accent fill. Inactive tabs use `SkyTextSecondaryBrush`; active uses `SkyOnAccentBrush` on accent background.

---

## Deep Dive: Breadcrumb Navigation Stack

Real apps derive breadcrumbs from a **navigation stack**, not a static list:

```csharp
public class NavigationStack
{
    private readonly List<NavigationFrame> _frames = new();

    public void Push(string label, object parameter)
    {
        _frames.Add(new NavigationFrame(label, parameter));
        Breadcrumbs = BuildBreadcrumbs();
    }

    public void PopTo(int index)
    {
        _frames.RemoveRange(index + 1, _frames.Count - index - 1);
        Breadcrumbs = BuildBreadcrumbs();
    }

    private IReadOnlyList<BreadcrumbItem> BuildBreadcrumbs()
    {
        var items = new List<BreadcrumbItem>();
        for (var i = 0; i < _frames.Count; i++)
        {
            var idx = i;
            items.Add(new BreadcrumbItem
            {
                Label = _frames[i].Label,
                IsCurrent = i == _frames.Count - 1,
                NavigateCommand = i < _frames.Count - 1
                    ? new RelayCommand(() => PopTo(idx))
                    : null
            });
        }
        return items;
    }
}
```

Clicking "Projects" in `Home › Projects › Settings` calls `PopTo(1)` — the stack truncates and the detail view reloads for that frame.

### Truncation for Deep Paths

When depth exceeds five levels, collapse middle segments:

```
Home › … › Folder › File › Properties
```

Implement in `ItemTemplate` with a `BreadcrumbItem.IsEllipsis` flag that renders `…` non-clickable.

### Separator Property

`SkyBreadcrumb.Separator` defaults to `"/"`. Unicode arrows (`›`, `→`) improve scanability. The separator is a styled property — change at runtime for locale:

```csharp
breadcrumb.Separator = CultureInfo.CurrentCulture.TextInfo.ListSeparator;
```

## Tab vs Breadcrumb: When to Use Both

| Pattern | Answers |
|---------|---------|
| Tabs | "Which peer view at this level?" |
| Breadcrumb | "Where am I in the hierarchy?" |

A file manager might use breadcrumbs for folder path and tabs for "Details | Permissions | History" on the selected file — orthogonal concerns.

## Debugging Tips

| Issue | Check |
|-------|-------|
| Tab content empty | `SelectedIndex` -1 or no TabItem children |
| Wrong tab selected on load | Two-way binding overwriting before items added |
| Breadcrumb click does nothing | `NavigateCommand` null on non-current items |
| Separator not showing | Custom `ItemsPanel` may omit separator logic in template |

## Summary

`SkyTabView` and `SkyBreadcrumb` sit at the lightweight end of SkyUI's navigation spectrum — thin subclasses whose value is mostly in themed templates and consistent class names. Master `TabControl` selection lifecycle and breadcrumb stack derivation to use them effectively in real applications.
