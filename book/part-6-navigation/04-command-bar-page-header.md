---
title: Chapter 20 — SkyCommandBar, SkyPageHeader, and SkyDrawer
order: 20
---

# Chapter 20: SkyCommandBar, SkyPageHeader, and SkyDrawer

Application chrome — toolbars, page titles, slide-in panels — gives users context and actions. This chapter covers three shell controls that appear in page scaffolds and standalone layouts.

## SkyPageHeader: Title Row with Back Navigation

```csharp
public class SkyPageHeader : TemplatedControl
{
    public const string BackButtonPartName = "PART_BackButton";

    public static readonly StyledProperty<string?> TitleProperty = ...;
    public static readonly StyledProperty<string?> SubtitleProperty = ...;
    public static readonly StyledProperty<bool> IsBackButtonVisibleProperty = ...;
    public static readonly StyledProperty<ICommand?> BackCommandProperty = ...;
    public static readonly StyledProperty<object?> ActionContentProperty = ...;

    public static readonly RoutedEvent<RoutedEventArgs> BackRequestedEvent = ...;
}
```

### Command + Routed Event Dual Pattern

```csharp
private void OnBackClick(object? sender, RoutedEventArgs e)
{
    if (BackCommand?.CanExecute(null) == true)
        BackCommand.Execute(null);

    RaiseEvent(new RoutedEventArgs(BackRequestedEvent));
}
```

MVVM apps bind `BackCommand` to navigation logic. Code-behind apps handle `BackRequested`. Both fire on the same click — neither is exclusive.

### ActionContent Slot

The right side of the header accepts arbitrary content — icon buttons, `SkyCommandBar` fragments, or status text:

```xml
<SkyPageHeader Title="Edit playlist"
               Subtitle="12 tracks"
               IsBackButtonVisible="True"
               BackCommand="{Binding GoBack}">
  <SkyPageHeader.ActionContent>
    <Button Classes="sky sky-primary" Content="Save"
            Command="{Binding SaveCommand}" />
  </SkyPageHeader.ActionContent>
</SkyPageHeader>
```

### Template Binding

Title and subtitle bind through `{TemplateBinding}` in the theme. `ActionContent` uses a `ContentPresenter` aligned to the trailing edge.

---

## SkyCommandBar: Grouped Commands with Overflow

```csharp
public class SkyCommandBar : TemplatedControl
{
    public const string PrimaryItemsPartName = "PART_PrimaryItems";
    public const string SecondaryItemsPartName = "PART_SecondaryItems";
    public const string OverflowItemsPartName = "PART_OverflowItems";

    [Content]
    public IList PrimaryCommands => primaryCommands;

    public IList SecondaryCommands => secondaryCommands;
    public IList OverflowCommands => overflowCommands;
}
```

### Three Command Tiers

| Collection | Typical content | Visibility |
|------------|-----------------|------------|
| `PrimaryCommands` | Save, New, Delete | Always visible on wide layouts |
| `SecondaryCommands` | Export, Share | Visible when space allows |
| `OverflowCommands` | Rare actions | Collapsed into ⋮ menu |

The theme template uses responsive visual states or measures available width to move items to overflow — similar to Microsoft UI command bar behavior.

### SkyCommandBarItem Model

```csharp
public class SkyCommandBarItem
{
    public string? Label { get; set; }
    public ICommand? Command { get; set; }
    public object? Icon { get; set; }
    public string? KeyboardAccelerator { get; set; }
}
```

```xml
<SkyCommandBar Title="Documents">
  <SkyCommandBarItem Label="New" Icon="{StaticResource AddIcon}"
                     Command="{Binding NewCommand}" />
  <SkyCommandBarItem Label="Upload" Command="{Binding UploadCommand}" />
</SkyCommandBar>
```

`[Content]` on `PrimaryCommands` allows XAML children without wrapper element.

### DirectProperty Collections

Like `SkyNavigationView.Items`, command lists use `DirectProperty<IList>` backed by `AvaloniaList<object>`. The list is mutable in code; XAML children populate `PrimaryCommands` at parse time.

---

## SkyDrawer: Slide-In Panel

`SkyDrawer` provides a filter/settings panel that slides from the left or right edge, with scrim overlay.

### Properties and Template Parts

```csharp
[TemplatePart(OverlayPartName, typeof(Panel))]
[TemplatePart(DrawerPanelPartName, typeof(Border))]
[TemplatePart(CloseButtonPartName, typeof(Button))]
public class SkyDrawer : TemplatedControl
{
    public static readonly StyledProperty<bool> IsOpenProperty = ...; // TwoWay
    public static readonly StyledProperty<string?> TitleProperty = ...;
    public static readonly StyledProperty<object?> DrawerContentProperty = ...;
    public static readonly StyledProperty<SkyDrawerPlacement> PlacementProperty = ...;
    public static readonly StyledProperty<double> DrawerWidthProperty = ...; // default 320
}
```

### Hit-Testing Lifecycle

```csharp
public SkyDrawer()
{
    IsHitTestVisible = false; // closed drawer passes clicks through
}

private void ApplyOpenState()
{
    var open = IsOpen;
    IsHitTestVisible = open;
    if (overlay is not null)
    {
        overlay.IsVisible = open;
        overlay.IsHitTestVisible = open;
    }
    if (drawerPanel is not null)
        drawerPanel.IsVisible = open;
}
```

Same pattern as `SkyDialogHost`: invisible host does not block interaction with page content.

### Placement Style Classes

```csharp
private void SyncPlacementClass()
{
    Classes.Set("sky-drawer-left", Placement == SkyDrawerPlacement.Left);
    Classes.Set("sky-drawer-right", Placement == SkyDrawerPlacement.Right);
}
```

Theme animations slide from the correct edge based on placement class.

### Overlay Dismiss

```csharp
private void OnOverlayPointerPressed(object? sender, PointerPressedEventArgs e)
{
    if (e.Source == overlay)
        Close();
}
```

Checking `e.Source == overlay` ensures clicks on drawer content do not close the panel.

### Closed Routed Event

```csharp
public void Close()
{
    if (!IsOpen) return;
    IsOpen = false;
    RaiseClosed();
}
```

Parents can reset filter state or persist draft edits on `Closed`.

### Usage in SkyListDetailPage

Page scaffolds expose `IsFilterDrawerOpen` and `FilterContent` mapped to an embedded `SkyDrawer`:

```xml
<SkyListDetailPage IsFilterDrawerOpen="{Binding IsFilterOpen, Mode=TwoWay}"
                   FilterContent="{Binding FilterView}" />
```

## Composing Shell Controls

A typical detail page stacks:

```
SkyPageHeader (back + title + save)
SkyCommandBar (bulk actions)
[main content]
SkyDrawer (filters, optional)
```

`SkyListDetailPage` encodes this composition once; individual pages supply content properties.

## Deep Dive: SkyCommandBar Layout Strategy

The command bar template divides horizontal space into zones:

```
[ Title ]  [ Primary commands ··· ]  [ Secondary ]  [ Overflow ⋮ ]
```

### Overflow Mechanics

When available width shrinks, secondary commands move into `OverflowCommands` and appear in a `MenuFlyout` triggered by the ⋮ button. The theme measures `PART_PrimaryItems` and `PART_SecondaryItems` `ItemsControl` bounds on `SizeChanged` — commands that clip below `MinWidth` relocate.

This is **responsive chrome** without separate mobile/desktop XAML: one command collection, runtime repacking.

### Command Item Templates

`SkyCommandBarItem` supports:

```csharp
public class SkyCommandBarItem
{
    public string? Label { get; set; }
    public ICommand? Command { get; set; }
    public object? Icon { get; set; }       // often SkyIcon
    public string? ToolTip { get; set; }
    public bool IsEnabled { get; set; } = true;
}
```

Icon-only primary commands save width; labels appear in tooltips. Bind `Command` to `RelayCommand` with `CanExecute` tied to selection state:

```csharp
DeleteCommand = new RelayCommand(DeleteSelected, () => SelectedItems.Any());
```

---

## Deep Dive: SkyPageHeader in MVVM Navigation

### Back Navigation Patterns

**Frame-based** (mobile): `BackCommand` pops the navigation stack.

**Modal overlay**: `BackCommand` sets `IsOpen = false` on a dialog.

**Master-detail**: `BackCommand` clears `SelectedItem` on mobile when detail is full-screen.

Always raise `BackRequested` even when `BackCommand` executes — analytics hooks often listen to the routed event.

### Subtitle for Context

`Subtitle` carries secondary context without cluttering the title:

```xml
<SkyPageHeader Title="Invoice #1042"
               Subtitle="Due March 15 · Acme Corp" />
```

Bind subtitle from computed view model property that updates when selection changes.

---

## Deep Dive: SkyDrawer Open State Machine

```
Closed → IsOpen=true → ApplyOpenState → overlay visible, hit-test on
       → user dismisses → IsOpen=false → Closed event
```

`templateApplied` flag prevents `ApplyOpenState` from running before parts exist — early `IsOpen=true` in XAML would otherwise no-op.

### Placement and Width

```csharp
public enum SkyDrawerPlacement { Left, Right }
```

`DrawerWidth` default 320px matches filter panel conventions. Right placement suits property inspectors; left suits navigation drawers on narrow windows.

### Animation Hook Points

Theme styles animate `TranslateTransform` on `PART_DrawerPanel`:

- Open: `X` from `-DrawerWidth` to `0` (left placement)
- Close: reverse with `SkyMotionAnimator` or CSS-like transitions in `Style.Animations`

`IsHitTestVisible` on the host flips only after open animation completes if you want clicks to pass through during slide — SkyUI sets it immediately for simpler semantics.

---

## SkyListDetailPage: How Scaffolds Wire Shell Controls

`SkyListDetailPage` template binds scaffold properties to child controls:

```xml
<!-- Conceptual template structure -->
<Grid RowDefinitions="Auto,Auto,*">
  <SkyPageHeader Grid.Row="0" Title="{TemplateBinding Title}" ... />
  <SkyCommandBar Grid.Row="1" Title="{TemplateBinding CommandBarTitle}" />
  <SkySplitView Grid.Row="2"
                PaneContent="{TemplateBinding ListContent}"
                DetailContent="{TemplateBinding DetailContent}" />
  <SkyDrawer IsOpen="{TemplateBinding IsFilterDrawerOpen}"
             DrawerContent="{TemplateBinding FilterContent}" />
</Grid>
```

Consumers set one property surface:

```xml
<SkyListDetailPage Title="Customers"
                   ListContent="{Binding CustomerList}"
                   DetailContent="{Binding CustomerDetail}"
                   IsFilterDrawerOpen="{Binding ShowFilters}"
                   FilterContent="{Binding FilterPanel}" />
```

The scaffold **hides composition** — page authors think in content slots, not grid row indices.

## Debugging Tips

| Symptom | Likely cause |
|---------|--------------|
| Drawer does not block clicks when open | `IsHitTestVisible` false on overlay |
| Back button no-op | `BackCommand` null and no `BackRequested` handler |
| Commands clipped, no overflow | `OverflowCommands` empty — move items there in VM |
| Header action not showing | `ActionContent` null or template `ContentPresenter` missing |

## Summary

| Control | Pattern | Key Avalonia concept |
|---------|---------|---------------------|
| SkyPageHeader | TemplatedControl + ICommand | Routed event + command |
| SkyCommandBar | DirectProperty IList collections | [Content] on primary list |
| SkyDrawer | Open state + hit-test gating | Overlay pointer dismiss |

These primitives are the building blocks scaffolds assemble. Command bar overflow, drawer state machine, and page header navigation patterns translate directly to production shell design.
