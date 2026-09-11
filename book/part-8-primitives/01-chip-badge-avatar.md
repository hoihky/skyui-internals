---
title: Chapter 24 — Chip, Badge, and Avatar
order: 24
---

# Chapter 24: Chip, Badge, and Avatar

Primitive controls are small, focused widgets used everywhere: tags, notification counts, user photos. SkyUI's `Chip`, `Badge`, and `Avatar` demonstrate variant enums, routed events for interaction, size coercion, and overlay positioning. Notably, these three controls omit the `Sky` prefix — a legacy naming choice — but follow the same implementation conventions as prefixed controls. This chapter also covers Avalonia templating choices, style class mechanics, and practical debugging for high-frequency UI atoms.

## Why Primitives Matter

Buttons, text boxes, and lists dominate line counts, but chips, badges, and avatars appear dozens of times per screen. Inconsistent sizing or event behavior across these atoms makes an app feel unfinished. SkyUI treats them as first-class `TemplatedControl` / `ContentControl` implementations with explicit template contracts rather than ad-hoc `Border` + `TextBlock` stacks in every view.

## Chip: Interactive Tag

`Chip` renders a compact label with optional leading avatar and delete button. It supports filled and outlined variants, three sizes, and optional click handling.

### Properties and Events

```csharp
public class Chip : TemplatedControl
{
    public static readonly StyledProperty<string?> LabelProperty = ...;
    public static readonly StyledProperty<object?> AvatarContentProperty = ...;
    public static readonly StyledProperty<bool> IsDeletableProperty = ...;
    public static readonly StyledProperty<ChipVariant> VariantProperty = ...;
    public static readonly StyledProperty<ChipSize> SizeProperty = ...;
    public static readonly StyledProperty<bool> IsClickableProperty = ...;

    public static readonly RoutedEvent<RoutedEventArgs> DeleteRequestedEvent = ...;
    public static readonly RoutedEvent<RoutedEventArgs> ChipClickEvent = ...;
}
```

`Label` duplicates content you could put in `Content` — the split exists so simple tags need only a string while advanced chips still host composite content via `AvatarContent` without replacing the label text block in the template.

### Variant and Size as Style Classes

```csharp
static Chip()
{
    VariantProperty.Changed.AddClassHandler<Chip>((c, _) => c.ApplyVariant());
    SizeProperty.Changed.AddClassHandler<Chip>((c, _) => c.ApplySize());
    IsClickableProperty.Changed.AddClassHandler<Chip>((c, _) => c.ApplyClickableCursor());
}

private void ApplyVariant()
{
    Classes.Set("chip-filled", Variant == ChipVariant.Filled);
    Classes.Set("chip-outlined", Variant == ChipVariant.Outlined);
}

private void ApplySize()
{
    Classes.Set("chip-sm", Size == ChipSize.Small);
    Classes.Set("chip-md", Size == ChipSize.Medium);
    Classes.Set("chip-lg", Size == ChipSize.Large);
}
```

Enum properties map to mutually exclusive style classes. Theme selectors target `Chip.chip-filled.chip-md` for precise styling without inline setters.

### Avalonia Concept: Classes vs PseudoClasses on Primitives

`Chip` uses **style classes** (`Classes.Set`) for designer-facing variants because consumers pick filled vs outlined in XAML. Pressed/disabled states still come from Avalonia's built-in pseudo-classes (`:pointerover`, `:disabled`) combined in selectors:

```xml
<Style Selector="Chip.chip-outlined:pointerover">
  <Setter Property="BorderBrush" Value="{DynamicResource SkyAccentBrush}" />
</Style>
```

Do not manually toggle `:pointerover` — the input system manages it.

### Delete Button Handling

```csharp
protected override void OnApplyTemplate(TemplateAppliedEventArgs e)
{
    base.OnApplyTemplate(e);
    _deleteButton = e.NameScope.Find(DeletePartName) as Button;

    if (_deleteButton is not null)
    {
        _deleteButton.Click -= OnDeleteClick;
        _deleteButton.Click += OnDeleteClick;
    }
    SyncDeleteVisibility();
}

private void OnDeleteClick(object? sender, RoutedEventArgs e)
{
    e.Handled = true; // prevent ChipClick from also firing
    RaiseEvent(new RoutedEventArgs(DeleteRequestedEvent));
}
```

`e.Handled = true` prevents the delete click from bubbling up to the chip root, which would also raise `ChipClickEvent`.

Hide the delete part when `IsDeletable` is false — do not leave a collapsed button that still captures tab focus:

```csharp
private void SyncDeleteVisibility()
{
    if (_deleteButton is not null)
        _deleteButton.IsVisible = IsDeletable;
}
```

### Clickable State

When `IsClickable` is true, the root border gets a hand cursor and a pointer pressed handler raises `ChipClickEvent`. When false, the chip is purely presentational.

Use `PointerPressed` with `e.GetCurrentPoint(this).Properties.IsLeftButtonPressed` rather than `Tapped` if you need parity with mouse and pen. Set `Focusable="True"` only when clickable — screen readers should not land on decorative tags.

### Keyboard Accessibility for Clickable Chips

Treat clickable chips like compact buttons:

```csharp
protected override void OnKeyDown(KeyEventArgs e)
{
    if (IsClickable && (e.Key == Key.Enter || e.Key == Key.Space))
    {
        RaiseEvent(new RoutedEventArgs(ChipClickEvent));
        e.Handled = true;
    }
    base.OnKeyDown(e);
}
```

Set `AutomationProperties.Name` from `Label` in the theme or code-behind.

### Usage

```xml
<StackPanel Orientation="Horizontal" Spacing="8">
  <Chip Label="Rock" Variant="Filled" />
  <Chip Label="Jazz" Variant="Outlined" IsDeletable="True"
        DeleteRequested="OnTagRemoved" />
  <Chip Label="Anna" IsClickable="True" ChipClick="OnChipClicked">
    <Chip.AvatarContent>
      <Avatar DisplayName="Anna K" Size="24" />
    </Chip.AvatarContent>
  </Chip>
</StackPanel>
```

### Chip List in a View Model

```csharp
public partial class TagChipViewModel : ObservableObject
{
    public string Label { get; init; } = "";
    public ICommand? DeleteCommand { get; init; }
}

// XAML with ItemsControl
<ItemsControl ItemsSource="{Binding Tags}">
  <ItemsControl.ItemTemplate>
    <DataTemplate x:DataType="local:TagChipViewModel">
      <Chip Label="{Binding Label}"
            IsDeletable="{Binding DeleteCommand, Converter={x:Static ObjectConverters.IsNotNull}}"
            DeleteRequested="OnDeleteTag" />
    </DataTemplate>
  </ItemsControl.ItemTemplate>
</ItemsControl>
```

Wire `DeleteRequested` to a handler that resolves the data context item — routed events do not automatically invoke `ICommand` on the view model.

### Debugging Chip

| Issue | Likely cause |
|-------|--------------|
| Delete fires click too | Missing `e.Handled = true` |
| Variant style wrong | Class not applied; check enum default |
| Avatar overlaps label | `chip-sm` template padding not adjusted for avatar slot |
| Chip not keyboard activatable | `Focusable` false or no key handler |

---

## Badge: Overlay Indicator

`Badge` is a `ContentControl` that overlays a count or dot on its content — typically an icon or avatar.

```csharp
public class Badge : ContentControl
{
    public static readonly StyledProperty<int> CountProperty = ...;
    public static readonly StyledProperty<bool> IsDotProperty = ...;
    public static readonly StyledProperty<BadgePlacement> PlacementProperty = ...;
    public static readonly StyledProperty<int> MaxCountProperty = ...;
}
```

### Why ContentControl?

`Badge` wraps arbitrary content (icon, avatar, button). Subclassing `ContentControl` gives a single `Content` property and a template with two layers: the child content and the badge overlay `PART_Badge`. `TemplatedControl` would work but `ContentControl` matches the mental model — badge is adornment **on** something else.

### Auto-Hide Logic

```csharp
static Badge()
{
    CountProperty.Changed.AddClassHandler<Badge>((b, _) => b.UpdateVisibility());
    IsDotProperty.Changed.AddClassHandler<Badge>((b, _) => b.UpdateVisibility());
}

private void UpdateVisibility()
{
    var show = IsDot || Count > 0;
    if (_badgeElement is not null)
        _badgeElement.IsVisible = show;
}
```

When count is zero and dot mode is off, the badge element hides entirely. This lets consumers bind `Count` directly without wrapping visibility logic in the view model.

Negative counts should coerce to zero in a property coerce callback — binding glitches can produce `-1` transiently during refresh.

### Count Overflow

Display text caps at `MaxCount` (default 99):

```csharp
private string FormatCount() =>
    Count > MaxCount ? $"{MaxCount}+" : Count.ToString();
```

Bind the badge text block to a formatted property or update text in `UpdateVisibility` when count changes.

### Placement

`BadgePlacement` enum (TopRight, TopLeft, BottomRight, BottomLeft) drives margin and alignment on the badge element through style classes or template bindings.

```csharp
private void ApplyPlacement()
{
    Classes.Set("badge-top-right", Placement == BadgePlacement.TopRight);
    Classes.Set("badge-top-left", Placement == BadgePlacement.TopLeft);
    // ...
}
```

Template uses a `Grid` with the content centered and the badge in a secondary cell aligned to the chosen corner. Negative margins pull the badge over the content edge — tune per size token.

### Avalonia Concept: Layout Rounding

Badge positioning uses small negative margins. At non-integer DPI scales, sub-pixel layout can blur the badge edge. Snap badge width/height to even integers in the theme and use `UseLayoutRounding="True"` on the badge root.

### Usage

```xml
<Badge Count="{Binding UnreadCount}" Placement="TopRight">
  <SkyIcon Kind="Notification" />
</Badge>

<Badge IsDot="True" Placement="TopRight">
  <Avatar DisplayName="User" />
</Badge>
```

### Debugging Badge

| Issue | Check |
|-------|-------|
| Badge always visible | `IsDot` true or count binding not clearing |
| Badge clipped | Parent `ClipToBounds="True"` — move badge outside clipped container or disable clip |
| Wrong corner | Placement class not toggled exclusively |
| "99+" never shows | `MaxCount` overridden too low |

---

## Avatar: Image or Initials

`Avatar` displays a profile image or falls back to initials derived from `DisplayName`.

```csharp
public class Avatar : TemplatedControl
{
    public const string ImagePartName = "PART_Image";
    public const string InitialsPartName = "PART_Initials";

    public static readonly StyledProperty<IImage?> SourceProperty = ...;
    public static readonly StyledProperty<string?> DisplayNameProperty = ...;
    public static readonly StyledProperty<double> SizeProperty = ...;
}
```

### Initials Extraction

```csharp
private static string ExtractInitials(string? displayName)
{
    if (string.IsNullOrWhiteSpace(displayName))
        return "?";

    var parts = displayName.Split(' ', StringSplitOptions.RemoveEmptyEntries);
    if (parts.Length == 1)
        return parts[0][..Math.Min(2, parts[0].Length)].ToUpperInvariant();

    return $"{parts[0][0]}{parts[^1][0]}".ToUpperInvariant();
}
```

"Anna Kwan" becomes "AK". "Cher" becomes "CH" (first two characters of a single name).

For CJK names without spaces, consider grabbing one character — localization may require a different extractor injected via attached property in localized apps.

### Image vs Initials Visibility

```csharp
private void SyncImageState()
{
    var hasImage = Source is not null;
    if (_image is not null) _image.IsVisible = hasImage;
    if (_initials is not null)
    {
        _initials.IsVisible = !hasImage;
        _initials.Text = ExtractInitials(DisplayName);
    }
}
```

Subscribe to `SourceProperty` and `DisplayNameProperty` changed handlers that call `SyncImageState`. Async image loading may set `Source` after first layout — handle `Bitmap` decode completion if using lazy loaders.

### Avalonia Concept: IImage and Async Loading

`Source` is `IImage?`, not `string` URI — view models often expose `Bitmap` or `AsyncImageLoader` callbacks. Binding a URL string requires a converter or loading in the view model:

```csharp
[ObservableProperty] private IImage? profileImage;

async Task LoadAvatarAsync(Uri uri)
{
    ProfileImage = await Task.Run(() => Bitmap.DecodeToWidth(uri.AbsoluteUri, (int)Size));
}
```

When load fails, set `Source` null so initials appear — do not leave a broken image placeholder unless the template provides one.

### Size Coercion

Avatar size is coerced to snap to standard sizes (24, 32, 40, 48, 64):

```csharp
private static double CoerceSize(AvaloniaObject _, double size) =>
    size switch
    {
        <= 28 => 24,
        <= 36 => 32,
        <= 44 => 40,
        <= 56 => 48,
        _ => 64
    };

// Register with coerce in property metadata
public static readonly StyledProperty<double> SizeProperty =
    AvaloniaProperty.Register<Avatar, double>(
        nameof(Size), 40, coerce: CoerceSize);
```

Coercion ensures avatars align with the design system's size scale even when consumers pass arbitrary values.

Template binds width and height to `Size` on a circular clip border. Font size for initials scales via theme setters per coerced size class if you add `avatar-sm` / `avatar-lg` classes when `Size` changes.

### Usage

```xml
<Avatar DisplayName="Anna Kwan" Size="40" />
<Avatar Source="{Binding ProfileImage}" DisplayName="Anna Kwan" Size="48" />
<Badge Count="3" Placement="BottomRight">
  <Avatar DisplayName="Team Lead" Size="32" />
</Badge>
```

## Composing Chip, Badge, and Avatar

Common combinations:

| UI pattern | Composition |
|------------|-------------|
| Tag with person | `Chip` + `AvatarContent` |
| Notify icon | `Badge` wrapping `SkyIcon` |
| User menu trigger | `Badge` wrapping `Avatar` |
| Filter tags row | `ItemsControl` of deletable `Chip` |

Keep avatar size inside chips one step smaller than standalone avatars (24 inside medium chip) — define token pairs in the theme so designers do not guess.

## Common Patterns Across Primitives

| Pattern | Chip | Badge | Avatar |
|---------|------|-------|--------|
| Base class | TemplatedControl | ContentControl | TemplatedControl |
| Enum → style class | Variant, Size | Placement | — |
| Routed events | Delete, Click | — | — |
| Auto visibility | Delete button | Badge on zero count | Image vs initials |
| Template parts | Root, Label, Delete | Badge overlay | Image, Initials |
| Coercion | — | MaxCount default | Size snap |

## Performance Notes

These controls are instantiated frequently. Avoid heavy bindings inside chip templates. Prefer `DisplayName` string over full view model in avatar lists. Virtualize tag rows with `ItemsRepeater` or virtualizing panels when tag count exceeds a few dozen — non-virtualized wrap panels stutter during window resize.

## Building Your Own Primitive

Primitives should be:

1. **Small API surface** — few properties, clear defaults
2. **Self-contained** — no dependency on app-level services
3. **Enum-driven variants** — map enums to style classes in property handlers
4. **Defensive visibility** — hide elements that have nothing to show
5. **Standard sizes** — coerce to design scale values
6. **Explicit event boundaries** — handle or bubble deliberately (`Delete` vs `Click`)
7. **Template part null checks** — degrade when theme incomplete

The next chapter covers attached properties, the other major primitive extension mechanism in SkyUI.
