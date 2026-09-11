---
title: Chapter 17 — SkyNavigationView
order: 17
---

# Chapter 17: SkyNavigationView

`SkyNavigationView` is the application shell control. It provides adaptive navigation that switches between an expanded sidebar, a compact icon-only sidebar, and a bottom tab bar depending on available width. This chapter explains display mode resolution, dual list hosts, selection synchronization, content transitions, and the Avalonia mechanics that make adaptive shell chrome reliable across window sizes and platforms.

## Role in the Application Shell

Most desktop and tablet apps need a persistent navigation region and a content region that swaps as the user moves between sections. Avalonia gives you `ListBox`, `ContentControl`, and layout panels — but wiring them into a responsive shell with synchronized selection, animated content swaps, and theme-driven layout variants is repetitive. `SkyNavigationView` packages that wiring into one `TemplatedControl` with a predictable template contract.

Think of it as three layers:

1. **Layout resolution** — width drives display mode (expanded sidebar, compact icons, bottom tabs)
2. **Data host** — one `Items` collection feeds two visual list hosts
3. **Content host** — selected item drives the main `ContentPresenter`, optionally with motion

## Display Modes

```csharp
public enum SkyNavigationDisplayMode
{
    Auto,
    Expanded,
    Compact,
    Bottom
}
```

| Mode | Behavior |
|------|----------|
| `Auto` | Picks expanded, compact, or bottom based on control width |
| `Expanded` | Full sidebar with icons and labels |
| `Compact` | Narrow sidebar with icons only |
| `Bottom` | Horizontal bottom navigation bar |

`Auto` mode uses `SkyBreakpoint` thresholds similar to the grid breakpoint system but tuned for navigation chrome rather than content columns. Navigation breakpoints typically sit slightly below content grid breakpoints because sidebar chrome consumes horizontal space before content reflows.

### Avalonia Concept: StyledProperty vs DirectProperty

`DisplayMode` is a `StyledProperty<SkyNavigationDisplayMode>` — it participates in styling, binding, and inheritance like most control properties. The `Items` collection (covered later) is a `DirectProperty<IList>` backed by a field. Styled properties are ideal for values consumers set in XAML; direct properties suit mutable collections that the control owns internally and mutates without going through the styled property pipeline on every add/remove.

When debugging binding issues, check which kind of property you are targeting. A `{Binding Items}` on a view model will not populate a direct `[Content]` collection unless you explicitly bridge them.

### Width Thresholds in Auto Mode

A typical resolver looks like this:

```csharp
private ResolvedDisplayMode ResolveDisplayMode()
{
    if (DisplayMode != SkyNavigationDisplayMode.Auto)
        return MapFixedMode(DisplayMode);

    var width = Bounds.Width;
    if (width >= ExpandedMinWidth) return ResolvedDisplayMode.Expanded;
    if (width >= CompactMinWidth)  return ResolvedDisplayMode.Compact;
    return ResolvedDisplayMode.Bottom;
}
```

Threshold constants live near other shell breakpoints so designers and developers tune them in one place. If your sidebar labels truncate before the mode switches, lower `CompactMinWidth` rather than shrinking font sizes in the theme.

## Pseudo-Classes for Layout Variants

```csharp
[PseudoClasses("compact", "bottom")]
public class SkyNavigationView : TemplatedControl
```

When display mode resolves to compact or bottom, the corresponding pseudo-class activates:

```csharp
private void ApplyDisplayMode()
{
    var mode = ResolveDisplayMode();
    PseudoClasses.Set(":compact", mode == ResolvedDisplayMode.Compact);
    PseudoClasses.Set(":bottom", mode == ResolvedDisplayMode.Bottom);
    UpdateNavListVisibility(mode);
}
```

Theme styles use these pseudo-classes to show/hide the side nav host and bottom nav host, adjust widths, and change item templates.

### Pseudo-Classes vs Style Classes

Avalonia distinguishes two class systems:

| Mechanism | API | Typical use |
|-----------|-----|-------------|
| **Pseudo-classes** | `PseudoClasses.Set(":compact", true)` | Structural states tied to control logic (open, pressed, compact) |
| **Style classes** | `Classes.Set("chip-filled", true)` | Designer-facing variants consumers opt into (`sky-primary`, `chip-md`) |

`SkyNavigationView` uses pseudo-classes because compact and bottom modes are resolved by the control, not chosen by the page author. Theme selectors target `SkyNavigationView:compact` and `SkyNavigationView:bottom` without requiring XAML class attributes on every instance.

### Debugging Theme Selectors

If the bottom bar never appears when you shrink the window:

1. Confirm `DisplayMode="Auto"` (a fixed `Expanded` overrides width logic)
2. Inspect `Bounds.Width` on the control, not the window — nested margins and split panes reduce available width
3. Use Avalonia DevTools to verify `:bottom` is set on the control when width drops below threshold
4. Check theme selectors — a missing `:bottom` rule may leave `PART_BottomNavHost` at `IsVisible="False"` in the default template state

## Dual List Hosts

The template contains two `ListBox` elements for navigation items:

| Part | Role |
|------|------|
| `PART_SideNavList` | Vertical sidebar items |
| `PART_BottomNavList` | Horizontal bottom bar items |
| `PART_SideNavHost` | Container for side nav (hidden in bottom mode) |
| `PART_BottomNavHost` | Container for bottom nav (hidden in sidebar modes) |
| `PART_Content` | `ContentPresenter` for the main content area |

Both lists bind to the same `Items` collection. When display mode changes, visibility toggles between hosts but selection state must stay synchronized.

### Why Two ListBox Controls?

A single `ListBox` with a rotated `ItemsPanel` sounds simpler but breaks down quickly:

- Item templates differ (vertical label stack vs horizontal icon row)
- Focus visuals and keyboard navigation paths differ between sidebar and bottom bar
- Scroll behavior differs — sidebars scroll vertically; bottom bars often clip or scroll horizontally

Duplicating hosts with shared data is a common Avalonia pattern for adaptive chrome. The cost is selection sync, which the control handles internally.

### Template Binding Pattern

In the control theme, both lists typically bind like this:

```xml
<ListBox x:Name="PART_SideNavList"
         ItemsSource="{TemplateBinding Items}"
         SelectedIndex="{TemplateBinding SelectedIndex,
             Mode=TwoWay}" />
```

Using `TemplateBinding` keeps the lists in sync with the control's `SelectedIndex` property. The code-behind still mirrors indices during mode transitions because template reapplication or host visibility changes can reset internal list state.

## Selection Synchronization

```csharp
private bool syncingSelection;

private void OnSelectedIndexChanged(AvaloniaPropertyChangedEventArgs e)
{
    if (syncingSelection) return;
    syncingSelection = true;
    try
    {
        sideNavList?.SelectedIndex = SelectedIndex;
        bottomNavList?.SelectedIndex = SelectedIndex;
        UpdateContent();
    }
    finally
    {
        syncingSelection = false;
    }
}
```

The `syncingSelection` flag prevents infinite loops when updating one list triggers a change handler on the other. Both lists are set to the same index, and the content presenter updates to show the selected item's content.

### Re-entrancy Guards in Avalonia

Property changed handlers in Avalonia can chain: setting `SelectedIndex` on list A raises `SelectionChanged`, which sets the control's `SelectedIndex`, which sets list B, which raises again. A boolean guard is the standard fix. Alternatives like comparing old/new values before writing help but do not cover every path (template reapply, programmatic `-1` clears).

Hook both lists in `OnApplyTemplate`:

```csharp
protected override void OnApplyTemplate(TemplateAppliedEventArgs e)
{
    base.OnApplyTemplate(e);
    sideNavList = e.NameScope.Find(PART_SideNavList) as ListBox;
    bottomNavList = e.NameScope.Find(PART_BottomNavList) as ListBox;

    if (sideNavList is not null)
    {
        sideNavList.SelectionChanged -= OnSideSelectionChanged;
        sideNavList.SelectionChanged += OnSideSelectionChanged;
    }
    // same for bottomNavList
}
```

Always unsubscribe before resubscribe in `OnApplyTemplate` — themes can reapply and duplicate handlers leak memory and double-fire events.

### Debugging Selection Drift

Symptoms and fixes:

| Symptom | Likely cause |
|---------|--------------|
| Content updates but highlight wrong | Lists out of sync; guard flag stuck `true` after exception |
| Selection resets on resize | `SelectedIndex` not persisted across `ApplyDisplayMode` |
| `-1` selection blank content | No default item; set `SelectedIndex="0"` on load |
| Click does nothing | `IsEnabled="False"` on inactive host still capturing hit tests |

## Items Collection

Navigation items use a `DirectProperty<IList>` backed by an `AvaloniaList`:

```csharp
private readonly AvaloniaList<SkyNavigationViewItem> items = new();

[Content]
public IList Items => items;
```

The `[Content]` attribute allows implicit collection syntax in XAML — child elements become items without a wrapper property.

`SkyNavigationViewItem` is a `ContentControl` with icon, label, and optional badge properties:

```xml
<SkyNavigationView>
  <SkyNavigationViewItem Content="Home" Tag="{x:Static local:Pages.Home}" />
  <SkyNavigationViewItem Content="Library" Tag="{x:Static local:Pages.Library}" />
  <SkyNavigationViewItem Content="Settings" Tag="{x:Static local:Pages.Settings}" />
</SkyNavigationView>
```

### Mapping Selection to Page Content

Three common strategies:

**Tag + factory** — store a page key or view model type on `Tag`, resolve in `UpdateContent()`:

```csharp
private void UpdateContent()
{
    if (SelectedIndex < 0 || SelectedIndex >= items.Count) return;
    var item = items[SelectedIndex];
    contentPresenter!.Content = item.Tag switch
    {
        Pages.Home     => homeView,
        Pages.Library  => libraryView,
        Pages.Settings => settingsView,
        _              => null
    };
}
```

**Content on item** — each `SkyNavigationViewItem` holds its view as `Content`; the nav view presents the selected item's content in `PART_Content`.

**View model index** — bind `SelectedIndex` two-way to the shell view model; pages swap via a `DataTemplate` in the content area.

Pick one strategy per app — mixing Tag resolution and inline content confuses debugging.

## Content Transitions

When the selected item changes, the content area animates using `SkyMotionAnimator`:

```csharp
private async void UpdateContentWithTransition(object? newContent)
{
    contentAnimationCancellation?.Cancel();
    contentAnimationCancellation = new CancellationTokenSource();
    var token = contentAnimationCancellation.Token;

    if (contentPresenter?.Content is not null)
        await SkyMotionAnimator.Default.AnimateFadeOutAsync(contentPresenter, token);

    if (token.IsCancellationRequested) return;

    contentPresenter!.Content = newContent;
    await SkyMotionAnimator.Default.AnimateFadeInAsync(contentPresenter, token);
}
```

Cancellation ensures rapid navigation clicks do not stack animations. Only the latest selection's animation completes.

### Avalonia Concept: async void in Controls

`async void` is acceptable in UI event handlers when exceptions are contained and cancellation is handled. Here, rapid clicks cancel prior tokens so stale animations never overwrite newer content. If you refactor to `async Task`, callers rarely await — but unit tests can.

Avoid starting transitions before `OnApplyTemplate` assigns `contentPresenter`. Guard with a null check and fall back to synchronous `Content = newContent`.

### Reduced Motion

Respect system accessibility settings by skipping animation when reduced motion is preferred:

```csharp
if (SkyMotionPreferences.ShouldReduceMotion)
{
    contentPresenter!.Content = newContent;
    return;
}
```

Shell controls should not be the only place this check lives, but navigation transitions are a high-impact spot.

## Width-Based Mode Resolution

```csharp
protected override void OnPropertyChanged(AvaloniaPropertyChangedEventArgs change)
{
    base.OnPropertyChanged(change);
    if (change.Property == BoundsProperty)
        ApplyDisplayMode();
}
```

Like `SkyResponsiveGrid`, the navigation view watches `BoundsProperty`. When the window is resized across a breakpoint threshold, display mode recalculates and pseudo-classes update.

`BoundsProperty` fires after layout passes complete, so width reflects actual arranged size. If mode resolution runs too early (constructor, before measure), `Bounds.Width` may be zero and incorrectly resolve to bottom mode. Defer initial `ApplyDisplayMode()` to `OnAttachedToVisualTree` or guard zero-width:

```csharp
if (Bounds.Width <= 0) return;
```

## Keyboard and Focus Navigation

Sidebar mode typically maps Up/Down to item navigation; bottom mode maps Left/Right. Avalonia's built-in `ListBox` keyboard behavior handles this per orientation if the `ItemsPanel` is a vertical or horizontal stack.

Ensure only the **visible** host has `IsVisible="True"` and receives focus — hidden lists still in the visual tree can steal focus if left enabled. `UpdateNavListVisibility` should set `IsEnabled` or `Focusable="False"` on the inactive host.

For page content, call `Focus()` on the first focusable element after navigation if your app targets keyboard-heavy workflows (kiosk, accessibility audits).

## Usage in an Application Shell

```xml
<SkyNavigationView DisplayMode="Auto"
                   SelectedIndex="{Binding SelectedNavIndex}">
  <SkyNavigationViewItem>
    <StackPanel Orientation="Horizontal" Spacing="12">
      <SkyIcon Kind="Home" />
      <TextBlock Text="Dashboard" />
    </StackPanel>
  </SkyNavigationViewItem>
  <!-- more items -->
  <local:DashboardView />
</SkyNavigationView>
```

The last child without being a `SkyNavigationViewItem` can serve as default content, or content can be set programmatically based on selection.

### Shell View Model Pattern

```csharp
public partial class ShellViewModel : ObservableObject
{
    [ObservableProperty] private int selectedNavIndex;

    partial void OnSelectedNavIndexChanged(int value)
    {
        CurrentPage = value switch
        {
            0 => new DashboardViewModel(),
            1 => new LibraryViewModel(),
            _ => new SettingsViewModel()
        };
    }

    [ObservableProperty] private object? currentPage;
}
```

Bind `SelectedIndex` two-way and place `CurrentPage` inside `PART_Content` via a `ContentControl` if you prefer MVVM over Tag-based resolution.

## Design Patterns

| Pattern | Application |
|---------|-------------|
| Adaptive layout | Width-driven display mode with pseudo-classes |
| Dual hosts | Same data, different visual hosts for different modes |
| Selection sync | Guard flag prevents re-entrant updates |
| Motion | Content transitions via shared animator |
| Direct property collection | `Items` as `DirectProperty<IList>` with `[Content]` |

## Building Your Own Navigation Shell

1. Define display modes as an enum with an `Auto` option
2. Map width to mode in a dedicated resolver
3. Use pseudo-classes for mode-specific theme selectors
4. Host the same items collection in multiple list controls if layouts differ
5. Guard selection sync with a boolean flag
6. Animate content changes with cancellable async transitions
7. Subscribe and unsubscribe template events in `OnApplyTemplate`
8. Disable or hide inactive list hosts so focus and hit testing stay correct

## Debugging Checklist

Before filing a shell bug, walk through this list:

- [ ] Is `DisplayMode` set to `Auto` for adaptive behavior?
- [ ] Does `Bounds.Width` on the nav control (not the window) cross the threshold?
- [ ] Are `:compact` / `:bottom` pseudo-classes active in DevTools?
- [ ] Is `syncingSelection` false after navigation clicks?
- [ ] Are both list handlers wired in `OnApplyTemplate` without duplicates?
- [ ] Does inactive host have visibility/focus disabled?
- [ ] Is content transition cancelled on rapid selection changes?

The next chapter covers `SkySplitView` and the page scaffolds that compose shell primitives into complete page layouts.
