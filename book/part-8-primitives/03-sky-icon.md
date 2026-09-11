---
title: Chapter 26 — SkyIcon and SkyFontIcon
order: 26
---

# Chapter 26: SkyIcon and SkyFontIcon

Icons appear in navigation, command bars, chips, and alerts. SkyUI provides two mechanisms: **vector path icons** (`SkyIcon`) and **font glyph icons** (`SkyFontIcon`). This chapter focuses on `SkyIcon`, which demonstrates programmatic visual tree construction without a XAML template.

## SkyIcon: Viewbox + Path Geometry

```csharp
public class SkyIcon : Viewbox
{
    public static readonly StyledProperty<SkyIconKind> KindProperty = ...;
    public static readonly StyledProperty<SkyIconSize> SizeProperty = ...;
    public static readonly StyledProperty<IBrush?> ForegroundProperty =
        TextElement.ForegroundProperty.AddOwner<SkyIcon>();

    private readonly PathShape _path = new() { Stretch = Stretch.Fill };

    public SkyIcon()
    {
        Child = _path;
        ApplyIcon();
    }
}
```

### Why Viewbox Instead of TemplatedControl?

`SkyIcon` has a fixed structure: one `Path` inside a `Viewbox`. Creating a `ControlTheme` for this would add indirection without flexibility. Building the visual tree in the constructor is simpler and avoids `OnApplyTemplate` entirely.

`Viewbox` scales its child uniformly to fit the control's `Width`/`Height` — ideal for vector icons at multiple sizes.

### SkyIconKind Catalog

```csharp
public enum SkyIconKind
{
    None, Home, Search, Settings, Close, ChevronDown, Music, User, ...
}
```

`SkyIconGlyphs.TryGetGeometry(Kind)` returns `StreamGeometry` for each kind — path data stored in a static dictionary or generated source file in `SkyUI.Icons`.

### ApplyIcon Implementation

```csharp
protected override void OnPropertyChanged(AvaloniaPropertyChangedEventArgs change)
{
    base.OnPropertyChanged(change);

    if (change.Property == KindProperty
        || change.Property == SizeProperty
        || change.Property == ForegroundProperty)
        ApplyIcon();
}

private void ApplyIcon()
{
    _path.Data = SkyIconGlyphs.TryGetGeometry(Kind);
    _path.Fill = Foreground;
    _path.IsVisible = _path.Data is not null;

    var px = SkyIconGlyphs.ToPixels(Size);
    Width = Height = px;
}
```

| `SkyIconSize` | Pixels |
|---------------|--------|
| Small | 16 |
| Medium | 20 |
| Large | 24 |

Explicit pixel sizes align icons to the design grid — navigation uses 24px, inline buttons use 16px.

### Foreground via AddOwner

Re-using `TextElement.ForegroundProperty` means icons respond to inherited foreground like `TextBlock`:

```xml
<Button Classes="sky sky-primary">
  <StackPanel Orientation="Horizontal" Spacing="8">
    <SkyIcon Kind="Save" Foreground="{DynamicResource SkyOnAccentBrush}" />
    <TextBlock Text="Save" />
  </StackPanel>
</Button>
```

### Kind.None

When `Kind` is `None` or geometry is missing, `_path.IsVisible = false` — the `Viewbox` collapses to zero visual content but retains layout size from `Width`/`Height`. Set `IsVisible="False"` on the whole icon to remove layout space.

## SkyFontIcon: Glyph from Icon Font

```csharp
public class SkyFontIcon : TextBlock
{
    public static readonly StyledProperty<string?> GlyphProperty = ...;
    public static readonly StyledProperty<SkyIconSize> SizeProperty = ...;

    public SkyFontIcon()
    {
        Classes.Add("sky-font-icon");
        FontFamily = SkyIconFontFamily.Default;
    }
}
```

Font icons trade crisp scaling at any size for simpler authoring (single character per icon). SkyUI uses path icons (`SkyIcon`) as the primary system; font icons supplement when a glyph font is already loaded.

## Adding a New Icon Kind

1. Add geometry path data to `SkyIconGlyphs` (or SVG import pipeline)
2. Add enum value to `SkyIconKind`
3. No theme change required — geometry is resolved in code
4. Add demo entry in `SkyUI.Demo` icons page

## Performance Considerations

- `StreamGeometry` instances are cached statically — not recreated per `ApplyIcon` call
- `InvalidateVisual` is not needed; changing `Path.Data` and `Fill` triggers redraw automatically
- For lists with hundreds of icons, path icons outperform image-based icons (no bitmap decode)

## Usage Patterns

```xml
<!-- Navigation -->
<SkyNavigationViewItem>
  <StackPanel Orientation="Horizontal" Spacing="12">
    <SkyIcon Kind="Home" Size="Large" />
    <TextBlock Text="Home" />
  </StackPanel>
</SkyNavigationViewItem>

<!-- Command bar -->
<SkyCommandBarItem Icon="{StaticResource IconExport}" ... />
<!-- where Icon is a SkyIcon in XAML resources -->

<!-- Alert -->
<SkyAlert Variant="Info" Message="Update available">
  <!-- template may include SkyIcon Kind=Info -->
</SkyAlert>
```

## Comparison: SkyIcon vs Image Control

| Approach | Pros | Cons |
|----------|------|------|
| SkyIcon (path) | Crisp at any DPI, themes via Foreground brush | Fixed catalog |
| Bitmap Image | Any raster artwork | Needs multiple resolutions |
| SkyFontIcon | Easy bulk import from font | Less precise than paths |

## Deep Dive: SkyIconGlyphs Geometry Catalog

`SkyIconGlyphs` maps each `SkyIconKind` to `StreamGeometry` path data:

```csharp
public static class SkyIconGlyphs
{
    private static readonly Dictionary<SkyIconKind, StreamGeometry> Cache = new();

    public static Geometry? TryGetGeometry(SkyIconKind kind)
    {
        if (kind == SkyIconKind.None)
            return null;
        if (Cache.TryGetValue(kind, out var g))
            return g;
        var geometry = LoadGeometry(kind);
        if (geometry is not null)
            Cache[kind] = geometry;
        return geometry;
    }

    public static double ToPixels(SkyIconSize size) => size switch
    {
        SkyIconSize.Small => 16,
        SkyIconSize.Medium => 20,
        SkyIconSize.Large => 24,
        _ => 20
    };
}
```

Geometries are parsed once and cached statically — scrolling a list of 500 `SkyIcon` rows does not re-parse SVG path strings per row.

### Design Grid

Icons are authored on a 24×24 viewbox. `Viewbox` scales uniformly to `Width`/`Height` from `ToPixels(Size)`. Stroke-based icons are converted to filled paths for consistent `Fill` binding.

### Adding a Custom Kind

1. Add path data to the glyphs source (often generated from design tool export)
2. Add enum value to `SkyIconKind`
3. Register in `TryGetGeometry` switch or dictionary loader
4. No theme XAML change — color comes from `Foreground`

---

## Deep Dive: Foreground and Theme Integration

`SkyIcon` re-owns `TextElement.ForegroundProperty`:

```csharp
public static readonly StyledProperty<IBrush?> ForegroundProperty =
    TextElement.ForegroundProperty.AddOwner<SkyIcon>();
```

In templates, bind to token brushes:

```xml
<SkyIcon Kind="Warning"
         Foreground="{DynamicResource SkyWarningBrush}" />
```

When parent `TextBlock` sets foreground, icons without explicit `Foreground` can inherit via visual tree if the theme sets `{Binding Foreground, RelativeSource=Ancestor}` — SkyUI icons typically set brush explicitly for semantic colors.

### High Contrast

Because icons are vectors, high-contrast themes swap `Foreground` via resource overrides — no `@2x` bitmap assets required.

---

## SkyFontIcon: When Paths Are Not Enough

```csharp
public class SkyFontIcon : TextBlock
{
    public static readonly StyledProperty<string?> GlyphProperty =
        AvaloniaProperty.Register<SkyFontIcon, string?>(nameof(Glyph));

    public SkyFontIcon()
    {
        Classes.Add("sky-font-icon");
        FontFamily = SkyIconFontResources.IconFont;
    }
}
```

```xml
<SkyFontIcon Glyph="&#xE700;" FontSize="16" />
```

Font icons suit rapid prototyping when designers deliver a glyph font. Tradeoffs:

| SkyIcon (path) | SkyFontIcon |
|----------------|-------------|
| Per-icon color easy | Single foreground per glyph |
| Crisp at any size | Hinting varies by OS |
| Fixed catalog | Any Unicode code point in font |

---

## Performance Notes

- **Avoid** binding `Kind` in high-frequency lists to rapidly changing values — each change reparses layout
- **Prefer** `IsVisible="False"` over `Kind="None"` when toggling — `None` still allocates viewbox space unless collapsed
- **Batch** icon updates outside scroll handlers — icon property changes during scroll cause unnecessary `ApplyIcon` calls

### List Virtualization Interaction

In virtualized lists, recycled row containers may show stale icons until `Kind` binding updates. Use `x:DataType` compiled bindings for zero-lag updates when rows reuse.

---

## Resource Dictionary Pattern

Define reusable icons in XAML resources:

```xml
<SkyIcon x:Key="IconSave" Kind="Save" Size="Medium" />
<SkyIcon x:Key="IconDelete" Kind="Delete" Size="Medium"
         Foreground="{DynamicResource SkyDangerBrush}" />
```

Reference in command bar items:

```xml
<SkyCommandBarItem Label="Save" Icon="{StaticResource IconSave}" />
```

Centralizing icons ensures command bar, menus, and dialogs use identical geometry.

## Summary

`SkyIcon` constructs a cached `Path` inside a `Viewbox` — no `TemplatedControl` required. `SkyIconGlyphs` provides performance and consistency; `SkyFontIcon` supplements with font glyphs. Master `Foreground` via `AddOwner` and static geometry cache when building your own icon system.
