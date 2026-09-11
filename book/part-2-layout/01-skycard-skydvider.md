---
title: Chapter 5 — SkyCard and SkyDivider
order: 5
---

# Chapter 5: SkyCard and SkyDivider

Part II examines layout controls — the building blocks that structure content on screen. We begin with two controls at opposite ends of the complexity spectrum: `SkyCard`, a composed container built on `ContentControl`, and `SkyDivider`, a minimal `TemplatedControl` that demonstrates pseudo-class-driven orientation.

Both controls illustrate core Avalonia ideas: the property system for bindable API surface, `ControlTheme` for visual structure, and the split between logical content (what you declare) and visual tree nodes (what the template renders).

## SkyCard: Composition Without Templates

`SkyCard` is an elevated surface that groups related content with optional header and footer regions. It inherits `ContentControl` rather than `TemplatedControl` because its visual structure is simple enough to live entirely in theme XAML while the C# class focuses on API surface.

Choosing `ContentControl` signals intent: the control **hosts** content; it does not own complex interaction logic inside named template parts. Avalonia already provides `Content`, `ContentTemplate`, and content presenter infrastructure. SkyCard adds optional header/footer slots on top of that model.

### The Property Surface

```csharp
public class SkyCard : ContentControl
{
    public static readonly StyledProperty<object?> HeaderProperty =
        AvaloniaProperty.Register<SkyCard, object?>(nameof(Header));

    public static readonly StyledProperty<IDataTemplate?> HeaderTemplateProperty =
        AvaloniaProperty.Register<SkyCard, IDataTemplate?>(nameof(HeaderTemplate));

    public static readonly StyledProperty<object?> FooterProperty =
        AvaloniaProperty.Register<SkyCard, object?>(nameof(Footer));

    public static readonly StyledProperty<IDataTemplate?> FooterTemplateProperty =
        AvaloniaProperty.Register<SkyCard, IDataTemplate?>(nameof(FooterTemplate));

    public static readonly StyledProperty<bool> IsHoverableProperty =
        AvaloniaProperty.Register<SkyCard, bool>(nameof(IsHoverable));
}
```

Four design decisions are visible here:

1. **Header and Footer as `object?`** — They accept a string, a `Control`, or any data object paired with a template
2. **Separate template properties** — `HeaderTemplate` and `FooterTemplate` follow the Avalonia item-template pattern used by `ContentControl` itself
3. **Main content via inherited `Content`** — `ContentControl` already provides `Content` and `ContentTemplate`; SkyCard does not duplicate them
4. **`IsHoverable` as behavioral flag** — Drives a style class rather than a pseudo-class because hover is a configurational preference, not a transient interaction state like `:pointerover`

### ContentControl and the Logical Tree

When you write:

```xml
<SkyCard>
  <ListBox ItemsSource="{Binding Items}" />
</SkyCard>
```

The `ListBox` is a **logical child** of the `SkyCard`. Data context inheritance flows from parent to child unless explicitly overridden. Bindings inside the list resolve against the page view model, not against `SkyCard` properties — unless you set `SkyCard.DataContext` separately.

Header and footer set through attached property syntax remain logical content owned by the card:

```xml
<SkyCard Header="Settings">
  ...
</SkyCard>
```

The string `"Settings"` is stored on `HeaderProperty`; the theme's `ContentPresenter` displays it via `{TemplateBinding Header}`.

### Style Classes in the Constructor

```csharp
public SkyCard()
{
    Classes.Add("sky");
    Classes.Add("sky-card");
}

static SkyCard()
{
    IsHoverableProperty.Changed.AddClassHandler<SkyCard>(
        (card, _) => card.SyncHoverClass());
}

private void SyncHoverClass() =>
    Classes.Set("sky-card-hoverable", IsHoverable);
```

Adding classes in the constructor guarantees every `SkyCard` instance is styled even if the consumer forgets to set `Classes` in XAML. The hover class toggles reactively: when `IsHoverable` becomes true, `sky-card-hoverable` is added; when false, it is removed.

Theme styles target `SkyCard.sky-card-hoverable` to apply elevation or border changes on pointer over. Because this is a **style class** toggled from C#, the card does not need `:pointerover` pseudo-class logic in the control class — Avalonia's built-in pointer pseudo-classes still apply on the control; the extra class gates whether hover **visuals** activate.

### Theme Template Structure

The `ControlTheme` for `SkyCard` typically looks like:

```xml
<ControlTheme x:Key="{x:Type controls:SkyCard}" TargetType="controls:SkyCard">
  <Setter Property="Template">
    <ControlTemplate>
      <Border Name="PART_Root"
              Background="{DynamicResource SkyCardBrush}"
              CornerRadius="{DynamicResource SkyRadiusLg}"
              BoxShadow="{DynamicResource SkyElevationSm}">
        <StackPanel>
          <ContentPresenter Name="PART_HeaderPresenter"
                            Content="{TemplateBinding Header}"
                            ContentTemplate="{TemplateBinding HeaderTemplate}"
                            IsVisible="{TemplateBinding Header, Converter={x:Static ObjectConverters.IsNotNull}}" />
          <ContentPresenter Content="{TemplateBinding Content}"
                            ContentTemplate="{TemplateBinding ContentTemplate}"
                            Margin="{DynamicResource SkySpace16Px}" />
          <ContentPresenter Name="PART_FooterPresenter"
                            Content="{TemplateBinding Footer}"
                            ContentTemplate="{TemplateBinding FooterTemplate}"
                            IsVisible="{TemplateBinding Footer, Converter={x:Static ObjectConverters.IsNotNull}}" />
        </StackPanel>
      </Border>
    </ControlTemplate>
  </Setter>
</ControlTheme>
```

`PART_Root` is declared with `[TemplatePart]` metadata even though `SkyCard` does not override `OnApplyTemplate`. The part name documents the template contract for theme authors and enables future C# logic (such as elevation animation on press) without breaking the template.

### Visual Tree Layout

After template application, the visual tree resembles:

```
SkyCard (logical + visual)
└── Border [PART_Root]          ← visual only
    └── StackPanel               ← visual only
        ├── ContentPresenter [PART_HeaderPresenter]
        ├── ContentPresenter     ← main content host
        │   └── ListBox          ← logical child appears here visually
        └── ContentPresenter [PART_FooterPresenter]
```

Developers navigate the logical tree for data context and lifecycle. Tools and hit-testing walk the visual tree. When debugging "my header does not show," verify both that `Header` is set and that `PART_HeaderPresenter` is visible in the visual tree.

### Step-by-Step: Card with Bound Header and Custom Content

**Goal:** Dashboard tile with dynamic title and a chart control inside.

1. **View model** exposes `TileTitle` and chart data series
2. **XAML:**

```xml
<SkyCard Header="{Binding TileTitle}"
         IsHoverable="True">
  <SkySparkline Values="{Binding Series}" />
</SkyCard>
```

3. **Binding path** resolves `TileTitle` from the card's inherited data context
4. **`Header` binding** is one-way by default — sufficient for display text
5. **Theme** applies hover elevation when pointer enters and `sky-card-hoverable` is present

To replace the default header text block with an icon + title layout, set `HeaderTemplate` on the card or define a implicit data template in resources — same pattern as `ItemsControl` items.

### Usage Patterns

**Simple card with text content:**

```xml
<SkyCard>
  <TextBlock Text="No new notifications." />
</SkyCard>
```

**Card with header and hover effect:**

```xml
<SkyCard Header="Top Artists"
         IsHoverable="True">
  <ListBox ItemsSource="{Binding Artists}" />
</SkyCard>
```

**Card with custom header template:**

```xml
<SkyCard>
  <SkyCard.Header>
    <StackPanel Orientation="Horizontal" Spacing="8">
      <SkyIcon Kind="Music" />
      <TextBlock Text="Playlist" FontWeight="SemiBold" />
    </StackPanel>
  </SkyCard.Header>
  <ItemsControl ItemsSource="{Binding Tracks}" />
</SkyCard>
```

**Nested cards (use sparingly):**

```xml
<SkyCard Header="Account">
  <StackPanel Spacing="16">
    <SkyCard Header="Profile">...</SkyCard>
    <SkyCard Header="Security">...</SkyCard>
  </StackPanel>
</SkyCard>
```

Nested elevated surfaces increase visual noise. Prefer a single card with section headers (`SkyDivider Text="Profile"`) for settings-style pages.

### Lessons from SkyCard

- Use `ContentControl` when the control is fundamentally a container
- Expose header/footer as `object?` plus template properties for flexibility
- Drive visual variants through style classes when the state is configurational
- Keep C# minimal; let the theme own structure
- Use `{TemplateBinding}` in the theme for header, footer, and main content presenters

---

## SkyDivider: Pseudo-Classes for Orientation

`SkyDivider` is a thin rule that separates content, optionally with centered label text. It is a teaching example for pseudo-class synchronization and lightweight templated controls.

Unlike `SkyCard`, `SkyDivider` inherits `TemplatedControl` because its visual is entirely custom chrome (lines, optional text) with no hosted content child.

### Full Implementation

```csharp
[PseudoClasses("horizontal", "vertical")]
public class SkyDivider : TemplatedControl
{
    public static readonly StyledProperty<string?> TextProperty =
        AvaloniaProperty.Register<SkyDivider, string?>(nameof(Text));

    public static readonly StyledProperty<Orientation> OrientationProperty =
        AvaloniaProperty.Register<SkyDivider, Orientation>(
            nameof(Orientation), Orientation.Horizontal);

    static SkyDivider()
    {
        OrientationProperty.Changed.AddClassHandler<SkyDivider>((divider, e) =>
            divider.SyncOrientation((Orientation)e.NewValue!));
    }

    public SkyDivider()
    {
        Classes.Add("sky");
        Classes.Add("sky-divider");
        SyncOrientation(Orientation.Horizontal);
    }

    private void SyncOrientation(Orientation orientation)
    {
        PseudoClasses.Set(":horizontal", orientation == Orientation.Horizontal);
        PseudoClasses.Set(":vertical", orientation == Orientation.Vertical);
    }
}
```

The entire behavioral logic fits in under fifty lines. The `Orientation` property is the single source of truth; `SyncOrientation` translates it into mutually exclusive pseudo-classes.

Calling `SyncOrientation` from the constructor ensures the initial visual state matches the default property value **before** the first layout pass. Without that call, pseudo-classes might be unset until the property change handler runs.

### Why Pseudo-Classes Instead of Triggers?

You could handle orientation with a `DataTrigger` in XAML, but pseudo-classes are preferable because:

1. **Testability** — You can assert pseudo-class state in a headless test without loading full theme triggers
2. **Consistency** — Every SkyUI control uses the same pseudo-class pattern for enum-like state
3. **Selector power** — Theme styles can combine pseudo-classes with style classes: `SkyDivider.horizontal.sky-divider`
4. **Property system integration** — Orientation remains a normal styled property bindable from view models

### Theme Template and Styles

**ControlTheme (simplified horizontal layout with optional text):**

```xml
<ControlTheme x:Key="{x:Type controls:SkyDivider}" TargetType="controls:SkyDivider">
  <Setter Property="Template">
    <ControlTemplate>
      <Grid Name="PART_Root" ColumnDefinitions="*,Auto,*">
        <Border Grid.Column="0"
                Height="1"
                VerticalAlignment="Center"
                Background="{DynamicResource SkyBorderSubtleBrush}" />
        <TextBlock Grid.Column="1"
                   Text="{TemplateBinding Text}"
                   Margin="{DynamicResource SkySpace8Px}"
                   Foreground="{DynamicResource SkyTextSecondaryBrush}"
                   IsVisible="{TemplateBinding Text, Converter={x:Static StringConverters.IsNotNullOrEmpty}}" />
        <Border Grid.Column="2"
                Height="1"
                VerticalAlignment="Center"
                Background="{DynamicResource SkyBorderSubtleBrush}" />
      </Grid>
    </ControlTemplate>
  </Setter>
</ControlTheme>
```

**Orientation styles (dimension overrides):**

```xml
<Style Selector="controls|SkyDivider.horizontal">
  <Setter Property="Height" Value="1" />
  <Setter Property="HorizontalAlignment" Value="Stretch" />
</Style>

<Style Selector="controls|SkyDivider.vertical">
  <Setter Property="Width" Value="1" />
  <Setter Property="VerticalAlignment" Value="Stretch" />
</Style>
```

When `Text` is empty, the center `TextBlock` collapses via visibility converter and the two borders still form a continuous line. When `Text` is non-empty, the template renders a horizontal line–text–line layout.

Vertical orientation typically swaps the template to a row-based grid or applies styles that rotate alignment — SkyUI keeps dimension logic in styles and structural layout in the template.

### Step-by-Step: Section Layout with Dividers

**Goal:** Settings page with labeled sections.

```xml
<StackPanel Spacing="{DynamicResource SkySpace16Px}">
  <SkyCard Header="Notifications">
    <StackPanel Spacing="8">
      <CheckBox Content="Email alerts" />
      <CheckBox Content="Push alerts" />
    </StackPanel>
  </SkyCard>

  <SkyDivider Text="Advanced" />

  <SkyCard Header="Developer">
    <TextBlock Text="API keys and webhooks." />
  </SkyCard>
</StackPanel>
```

1. First card groups related toggles
2. `SkyDivider Text="Advanced"` creates a labeled section break without a second card header
3. Spacing tokens on the outer stack keep vertical rhythm aligned with the design system

For toolbar-style vertical separation:

```xml
<StackPanel Orientation="Horizontal" Spacing="8">
  <Button Classes="sky sky-ghost" Content="Edit" />
  <SkyDivider Orientation="Vertical" />
  <Button Classes="sky sky-ghost" Content="Delete" />
</StackPanel>
```

Ensure the hosting stack has sufficient height; vertical dividers stretch only when the parent grants vertical space.

### Usage

```xml
<StackPanel Spacing="16">
  <TextBlock Text="Section A" />
  <SkyDivider />
  <TextBlock Text="Section B" />
  <SkyDivider Text="OR" />
</StackPanel>

<!-- Vertical divider in a horizontal stack -->
<StackPanel Orientation="Horizontal" Spacing="8">
  <Button Content="Edit" />
  <SkyDivider Orientation="Vertical" />
  <Button Content="Delete" />
</StackPanel>
```

Bind orientation dynamically when layout changes at runtime:

```xml
<SkyDivider Orientation="{Binding SplitOrientation}" />
```

The enum binds through the styled property system; `SyncOrientation` updates pseudo-classes when the bound value changes.

### Lessons from SkyDivider

- Map enum properties to pseudo-classes in a dedicated sync method
- Call the sync method from both the constructor and the property changed handler
- Keep orientation logic in C#; keep dimension overrides in theme styles
- Use `{TemplateBinding Text}` for optional label text inside the template

---

## Comparing the Two Controls

| Aspect | SkyCard | SkyDivider |
|--------|---------|------------|
| Base class | ContentControl | TemplatedControl |
| C# complexity | Low (properties + one class toggle) | Very low (orientation sync) |
| Template complexity | Medium (header, content, footer) | Low (line + optional text) |
| State mechanism | Style class (`sky-card-hoverable`) | Pseudo-classes (`:horizontal`, `:vertical`) |
| Primary pattern | Container composition | Property-to-pseudo-class mapping |
| Logical children | Yes (Content, Header, Footer) | No hosted content |
| Typical binding | Header, Content, IsHoverable | Text, Orientation |

Both controls follow the SkyUI recipe from Chapter 4 but sit at different points on the simplicity spectrum. When you build your own layout primitive, ask whether it is closer to a container (like SkyCard) or a stateful visual element (like SkyDivider), and choose the base class accordingly.

## Common Mistakes

| Mistake | Control | Fix |
|---------|---------|-----|
| Subclass `TemplatedControl` for a simple panel | SkyCard-like | Use `ContentControl` + theme template |
| Duplicate `ContentProperty` on card | SkyCard | Use inherited `Content` from `ContentControl` |
| Use `:hoverable` pseudo-class for `IsHoverable` | SkyCard | Use style class `sky-card-hoverable` |
| Forget constructor sync | SkyDivider | Call `SyncOrientation` in constructor |
| Both `:horizontal` and `:vertical` true | SkyDivider | Make pseudo-classes mutually exclusive |
| Vertical divider in short stack | SkyDivider | Give parent min height or stretch alignment |
| Hard-coded divider color | SkyDivider | Use `SkyBorderSubtleBrush` token |

## Debugging Tips

**SkyCard content shows but header does not.** Confirm `Header` is set (not `Title` or wrong property name). Check `PART_HeaderPresenter` visibility binding — null header hides the presenter.

**Hover effect never triggers.** Verify `IsHoverable="True"` and that theme defines styles for `sky-card-hoverable` combined with `:pointerover`.

**SkyDivider always horizontal despite `Orientation=Vertical`.** Inspect pseudo-classes in dev tools — if `:vertical` is false, the property changed handler may not run (check binding path). If pseudo-class is true but layout wrong, theme vertical styles may be missing.

**Divider text overlaps lines.** Spacing token on center `TextBlock` may be too small; increase horizontal margin or shorten label text.

**Bindings inside card content fail.** Data context may be broken by setting `SkyCard.DataContext` to the card's own view model without propagating parent context — use `{Binding #Root.DataContext}` patterns or nest view models deliberately.

## Summary

`SkyCard` demonstrates how far you can get with `ContentControl`, a rich theme template, and minimal C# — header/footer templates, token-driven elevation, and style-class hover gating. `SkyDivider` demonstrates the other pole: a tiny `TemplatedControl` that delegates all visuals to XAML and uses pseudo-classes to bridge one enum property to theme selectors.

Together they form the layout vocabulary SkyUI builds on: grouped content in elevated surfaces, and lightweight separation between sections. The next chapter extends layout to width-driven responsive grids that reflow dashboard tiles as windows resize.
