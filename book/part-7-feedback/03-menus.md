---
title: Chapter 23 — SkyMenuBar and SkyContextMenu
order: 23
---

# Chapter 23: SkyMenuBar and SkyContextMenu

Menus expose commands hierarchically — top-level menu bar for discoverability, context menus for situational actions. SkyUI styles Avalonia's built-in menu types rather than reimplementing menu logic.

## Avalonia Menu Types

| Type | Base | Role |
|------|------|------|
| `Menu` | `MenuBase` | Horizontal menu bar |
| `MenuItem` | `HeaderedItemsControl` | Entry with optional submenu |
| `ContextMenu` | `MenuBase` | Right-click popup |
| `MenuFlyout` | `PopupFlyoutBase` | Flyout attached to buttons |

SkyUI subclasses `Menu` and `ContextMenu` only to add style classes.

## SkyMenuBar

```csharp
public class SkyMenuBar : Menu
{
    public SkyMenuBar()
    {
        Classes.Add("sky");
        Classes.Add("sky-menu-bar");
    }
}
```

### Inherited Behavior from Menu

- Keyboard accelerators via `HotKey` on `MenuItem`
- Submenu open on hover or click (platform style)
- `MenuItem` nesting for arbitrary depth
- Access key mnemonics with `_` prefix in header text (`_File`)

### Theme Styling

`Menus.axaml` styles:

```xml
<Style Selector="Menu.sky-menu-bar">
  <Setter Property="Background" Value="{DynamicResource SkySurfaceBrush}" />
  <Setter Property="Height" Value="{DynamicResource SkyMenuBarHeight}" />
</Style>
<Style Selector="MenuItem.sky-menu-item">
  <Setter Property="Padding" Value="{DynamicResource SkySpace8Px}" />
</Style>
```

Popup submenus inherit `sky-context-menu` styles for consistent dropdown appearance.

### Usage

```xml
<SkyMenuBar>
  <MenuItem Header="_File">
    <MenuItem Header="_New" Command="{Binding NewCommand}" />
    <MenuItem Header="_Open" Command="{Binding OpenCommand}" />
    <Separator />
    <MenuItem Header="E_xit" Command="{Binding ExitCommand}" />
  </MenuItem>
  <MenuItem Header="_Edit">
    <MenuItem Header="_Undo" InputGesture="Ctrl+Z" />
    <MenuItem Header="_Redo" InputGesture="Ctrl+Y" />
  </MenuItem>
</SkyMenuBar>
```

### SkyAccelerator Utilities

`SkyAccelerator` and `SkyAcceleratorFormatConverter` register and display keyboard shortcuts consistently:

```csharp
SkyAccelerator.Register(mainWindow, Key.S, KeyModifiers.Control, SaveCommand);
```

The converter formats `KeyModifiers` + `Key` for display in menu item headers.

---

## SkyContextMenu

```csharp
public class SkyContextMenu : ContextMenu
{
    public SkyContextMenu()
    {
        Classes.Add("sky");
        Classes.Add("sky-context-menu");
    }
}
```

### Attaching to Controls

```xml
<ListBox ItemsSource="{Binding Items}">
  <ListBox.ContextMenu>
    <SkyContextMenu>
      <MenuItem Header="Copy" Command="{Binding CopyCommand}" />
      <MenuItem Header="Delete" Command="{Binding DeleteCommand}" />
    </SkyContextMenu>
  </ListBox.ContextMenu>
</ListBox>
```

Or in code:

```csharp
listBox.ContextMenu = new SkyContextMenu
{
    Items = { new MenuItem { Header = "Refresh" } }
};
```

### Context Menu and DataContext

`ContextMenu` is a separate visual tree root. By default it inherits `DataContext` from its placement target, so `Command="{Binding DeleteCommand}"` on a `MenuItem` resolves against the `ListBox`'s data context — usually the view model you want.

If binding fails, set `ContextMenu.DataContext` explicitly or use `Tag` on the placement target.

## SkyMenuFlyout

For command bars and buttons that need dropdown menus without a full menu bar:

```csharp
public class SkyMenuFlyout : MenuFlyout
{
    public SkyMenuFlyout()
    {
        // theme applied via flyout presenter classes
    }
}
```

```xml
<Button Content="Actions">
  <Button.Flyout>
    <SkyMenuFlyout>
      <MenuItem Header="Duplicate" />
      <MenuItem Header="Archive" />
    </SkyMenuFlyout>
  </Button.Flyout>
</Button>
```

## SkyMenuGestures

Static helpers wire platform-appropriate context menu invocation (right-click on desktop, long-press on touch). Mobile demos use these alongside `SkyTouchTarget` for accessible hit areas.

## When to Subclass vs Style

Menus demonstrate the **style-only** extreme:

- Avalonia's menu implementation handles popups, scrolling, separators, icons, and accelerators
- SkyUI needs consistent dark-theme colors and padding
- Subclassing adds `Classes` in constructor — zero behavior change

Before writing a custom `TemplatedControl` menu, verify Avalonia's `MenuItem` template cannot be retargeted with `ControlTheme`.

## Deep Dive: Menu Item Hierarchy and Popups

### MenuItem Tree Structure

```xml
<MenuItem Header="_File">
  <MenuItem Header="_New" InputGesture="Ctrl+N" Command="{Binding NewCommand}" />
  <MenuItem Header="_Open" />
  <Separator />
  <MenuItem Header="Recent">
    <MenuItem Header="doc1.txt" />
    <MenuItem Header="doc2.txt" />
  </MenuItem>
</MenuItem>
```

Each submenu is a nested `MenuItem` collection. Avalonia opens submenus on hover (desktop) or click (touch). `Separator` inserts a horizontal rule — use sparingly to group related actions.

### Icons in Menu Items

```xml
<MenuItem Header="Save" Command="{Binding SaveCommand}">
  <MenuItem.Icon>
    <SkyIcon Kind="Save" Size="Small" />
  </MenuItem.Icon>
</MenuItem>
```

Icon column width is uniform in the theme template so text aligns across items.

### InputGesture Display

`InputGesture="Ctrl+S"` renders accelerator text right-aligned in the menu row. `SkyAcceleratorFormatConverter` formats platform-specific modifiers (⌘ on macOS when configured).

---

## Deep Dive: SkyAccelerator Registration

Global shortcuts work even when menus are closed:

```csharp
public static class SkyAccelerator
{
    public static void Register(
        InputElement root,
        Key key,
        KeyModifiers modifiers,
        ICommand command)
    {
        root.KeyBindings.Add(new KeyBinding
        {
            Gesture = new KeyGesture(key, modifiers),
            Command = command
        });
    }
}
```

Register on `Window` or `MainView` root. Menu `InputGesture` and `KeyBinding` can share the same `ICommand` — one implementation, two discovery paths.

---

## Deep Dive: Context Menu Placement

`ContextMenu` opens as a popup positioned at pointer coordinates. Avalonia flips placement when near screen edges. `SkyContextMenu` theme adds:

- Drop shadow from `SkyElevationMd`
- Rounded corners from `SkyRadiusMd`
- Max height with scroll for long menus

### Opening from Code

```csharp
void OnListBoxRightClick(object? sender, PointerPressedEventArgs e)
{
    if (e.GetCurrentPoint(listBox).Properties.IsRightButtonPressed)
    {
        var menu = new SkyContextMenu
        {
            Items =
            {
                new MenuItem { Header = "Copy", Command = CopyCommand },
                new MenuItem { Header = "Delete", Command = DeleteCommand }
            }
        };
        menu.Open(listBox);
    }
}
```

Prefer declarative `ContextMenu` in XAML when the menu structure is static.

### DataContext Inheritance

`ContextMenu` is a separate visual tree root. Avalonia assigns `PlacementTarget.DataContext` to the menu by default. If bindings fail:

```xml
<ListBox.ContextMenu>
  <SkyContextMenu DataContext="{Binding PlacementTarget.DataContext, RelativeSource={RelativeSource Self}}">
```

---

## SkyMenuFlyout vs ContextMenu

| Type | Attached to | Typical use |
|------|-------------|-------------|
| `SkyContextMenu` | `Control.ContextMenu` | Right-click on list rows |
| `SkyMenuFlyout` | `Button.Flyout` | Explicit ⋮ or Actions button |

Flyouts support the same `MenuItem` children but open below the button — better for touch than right-click.

---

## Accessibility

- Mnemonics: `_File` → Alt+F on Windows/Linux
- Set `AutomationProperties.Name` on icon-only menu items
- Disable items with `IsEnabled=false` rather than hiding — users learn unavailable actions exist

## Debugging Tips

| Issue | Fix |
|-------|-----|
| Command not firing | Check `DataContext` on context menu |
| Submenu does not open | Parent `MenuItem` must be enabled |
| Accelerator shows wrong | Verify `KeyGesture` matches OS |
| Menu styled wrong | Missing `sky-context-menu` class on template root |

## Summary

`SkyMenuBar` and `SkyContextMenu` inherit platform menu infrastructure and own visual design in theme XAML. Accelerator registration, context menu data context, and flyout vs context menu choice complete the desktop command surface story.
