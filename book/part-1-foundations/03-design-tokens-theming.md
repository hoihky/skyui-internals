---
title: Chapter 3 — Design Tokens and Theming
order: 3
---

# Chapter 3: Design Tokens and Theming

A control library that hard-codes colors, spacing, and font sizes in C# becomes impossible to theme. SkyUI avoids this by routing every visual decision through a **design token layer**. Controls reference token keys; the theme package resolves those keys to concrete values. This chapter explains how that layer works, how it connects to Avalonia's property and resource systems, and how you should use it when building custom controls.

## What Is a Design Token?

A design token is a named, stable identifier for a visual value. Instead of writing `Background = Brushes.DarkGray` in control code, you reference `SkySurfaceBrush` in XAML:

```xml
<Border Background="{DynamicResource SkySurfaceBrush}"
        CornerRadius="{DynamicResource SkyRadiusMd}"
        Padding="{DynamicResource SkySpace16Px}" />
```

`SkyTokenKeys` in `SkyUI.Core` defines these key strings as constants so C# code can reference them without magic strings:

```csharp
public static class SkyTokenKeys
{
    public static class Brush
    {
        public const string Surface = "SkySurfaceBrush";
        public const string Accent = "SkyAccentBrush";
        public const string Danger = "SkyDangerBrush";
        // ...
    }

    public static class Spacing
    {
        public const string Space8Px = "SkySpace8Px";
        public const string Space16Px = "SkySpace16Px";
        // ...
    }
}
```

Using `DynamicResource` rather than `StaticResource` is important. Dynamic resources re-resolve when the theme changes at runtime, which is how accent overrides and density switching work.

### Tokens and the Resource Dictionary Chain

Avalonia resolves `{DynamicResource SkySurfaceBrush}` by walking the **logical tree** upward from the control that requested the resource. Each ancestor may contribute a `ResourceDictionary`; merged dictionaries are searched in declaration order. At the top, your `Application` typically loads `ContentFirstDark.axaml`, which merges brush, spacing, radius, and typography dictionaries from `SkyUI.Themes.Sky`.

This lookup model has two practical consequences:

1. **Tokens are inherited like ambient context** — A child `Border` inside a `SkyCard` template finds the same `SkyCardBrush` as a sibling `TextBlock` in the page, because both walk up to the application resources.
2. **Runtime overrides inject at the application level** — When `SkyAccentOverrideApplicator` writes new brush entries into the application dictionary, every `{DynamicResource SkyAccentBrush}` binding anywhere in the visual tree picks up the new value on the next resource refresh.

You rarely set token values in control C#. The control class owns behavior; the theme owns appearance.

## DynamicResource vs StaticResource

Both markup extensions pull values from resource dictionaries, but they behave differently when the dictionary changes:

| Extension | When value is resolved | Updates on theme change? |
|-----------|------------------------|--------------------------|
| `StaticResource` | At load / compile time | No — value is fixed after first lookup |
| `DynamicResource` | On first use, then re-evaluated when resources change | Yes |

SkyUI theme XAML uses `DynamicResource` for every brush, spacing, radius, and density-dependent value. `StaticResource` appears only for references that are truly invariant — for example, pointing one style at another style key inside the same dictionary file.

If a control looks correct at startup but never updates when you change accent or density, search the template for `StaticResource` on token keys. That is the most common theming bug in custom controls.

### Binding to Token Values from C#

When C# must read a token (for example, to pass a brush to a custom draw operation), use the resource API rather than hard-coding:

```csharp
if (this.TryFindResource(SkyTokenKeys.Brush.Accent, out var resource)
    && resource is IBrush accentBrush)
{
    // use accentBrush
}
```

`TryFindResource` follows the same logical-tree lookup as `{DynamicResource}`. Prefer this over `Application.Current.FindResource`, which bypasses local overrides on nested resource hosts.

## The Spacing Scale

SkyUI uses an 8-pixel base unit. Spacing tokens map to multiples or fractions of that unit:

| Token | Typical use |
|-------|-------------|
| `SkySpace4Px` | Tight internal padding, icon gaps |
| `SkySpace8Px` | Default gap between related elements |
| `SkySpace12Px` | Intermediate inset when 8 feels tight and 16 feels loose |
| `SkySpace16Px` | Section padding, card insets |
| `SkySpace24Px` | Large section separation |
| `SkySpace32Px` | Page-level margins, hero spacing |

When you author a new control template, pick spacing from this scale rather than inventing arbitrary values. Consistency across controls is what makes an app feel cohesive.

Spacing tokens are `Thickness` or `double` resources depending on context. In a `ControlTemplate`, write:

```xml
<StackPanel Spacing="{DynamicResource SkySpace8Px}"
            Margin="{DynamicResource SkySpace16Px}" />
```

Do not mix raw `Margin="12"` with token-based spacing on adjacent controls — the eye notices one-off values even when users cannot name them.

## Brush Tokens and Semantic Naming

Brush tokens use semantic names, not color names. `SkyTextSecondaryBrush` describes the role (secondary text) rather than the hue (gray). This matters because:

- Dark and light palettes map the same semantic key to different colors
- High-contrast themes can boost contrast without renaming keys
- Accent overrides replace `SkyAccentBrush` globally without touching individual controls

Common brush categories:

- **Surfaces** — `SkyBackgroundBrush`, `SkySurfaceBrush`, `SkyCardBrush`
- **Text** — `SkyTextPrimaryBrush`, `SkyTextSecondaryBrush`, `SkyTextDisabledBrush`
- **Accent** — `SkyAccentBrush`, `SkyOnAccentBrush`, `SkyAccentMutedBrush`
- **Feedback** — `SkyDangerBrush`, `SkyWarningBrush`, `SkyInfoBrush`, `SkySuccessBrush`
- **Interaction** — `SkyHoverTintBrush`, `SkySelectedTintBrush`, `SkyFocusRingBrush`
- **Borders** — `SkyBorderSubtleBrush`, `SkyBorderStrongBrush`

Semantic naming also keeps templates readable. `Foreground="{DynamicResource SkyTextSecondaryBrush}"` documents intent; `Foreground="#888888"` documents nothing about accessibility or theme switching.

## Radius, Typography, and Elevation Tokens

Tokens are not limited to color and spacing. SkyUI defines:

- **Corner radius** — `SkyRadiusSm`, `SkyRadiusMd`, `SkyRadiusLg`, `SkyRadiusFull`
- **Typography** — `SkyFontSizeBody`, `SkyFontSizeCaption`, `SkyFontWeightSemiBold` (often paired with `SkyUI.Fonts` family resources)
- **Elevation** — `SkyElevationSm`, `SkyElevationMd` as `BoxShadow` resources used on cards and menus

`SkyCard` uses `SkyRadiusLg` and `SkyElevationSm` together so elevated surfaces share the same corner language as dialogs and popovers. When you add a new floating surface, copy that pairing rather than inventing a new shadow.

## ControlTheme vs Style

Avalonia 11+ uses `ControlTheme` as the primary mechanism for control templates. SkyUI defines one `ControlTheme` per custom control type, keyed by `{x:Type controls:SkyCard}`.

Separate `Style` elements handle variants:

```xml
<!-- ControlTheme: default template -->
<ControlTheme x:Key="{x:Type controls:SkyDivider}" TargetType="controls:SkyDivider">
  <Setter Property="Template">
    <ControlTemplate>
      <!-- visual tree -->
    </ControlTemplate>
  </Setter>
</ControlTheme>

<!-- Style: orientation variant -->
<Style Selector="controls|SkyDivider.horizontal">
  <Setter Property="Height" Value="1" />
</Style>
<Style Selector="controls|SkyDivider.vertical">
  <Setter Property="Width" Value="1" />
</Style>
```

The control toggles pseudo-classes in C#; the theme responds with selectors. This keeps orientation logic out of the template itself.

### How ControlTheme Connects to the Property System

A `ControlTheme` is a bundle of setters keyed to a control type. When Avalonia creates a `SkyDivider`, the theme engine applies the matching `ControlTheme` setters before the control appears in the visual tree. The `Template` setter replaces the control's default visual subtree with your `ControlTemplate`.

Inside the template, `{TemplateBinding Text}` creates a one-way binding from the templated control's styled property to the template element. Template bindings participate in the property system at **template application time** — they are not ordinary `{Binding}` paths to the data context. Use `TemplateBinding` for properties defined on the control (`Text`, `Background`, `IsEnabled`). Use `{Binding}` only when the template element should read from the control's `DataContext` (uncommon in SkyUI chrome).

Styles layered on top can override individual properties without replacing the whole template. That separation — **ControlTheme for structure, Style for variation** — is the main theming pattern in SkyUI.

## Runtime Accent Override

Applications can change the accent color without recompiling:

```xml
<Application sky:SkyThemeProperties.AccentOverride="#1ED760">
```

`SkyAccentOverrideApplicator` watches this attached property and injects overridden brush values into the application resource dictionary. Controls that reference `SkyAccentBrush` automatically pick up the new color because they use dynamic resources.

### Walkthrough: What Happens When Accent Changes

Follow this sequence when debugging accent override:

1. **Application loads** — `ContentFirstDark.axaml` merges token dictionaries; `SkyAccentBrush` resolves to the default green.
2. **User sets `AccentOverride`** — The attached property changed handler on `SkyThemeProperties` runs.
3. **Applicator runs** — `SkyAccentOverrideApplicator` derives related brushes (`SkyOnAccentBrush`, focus rings, selected tints) from the override color and writes them into the application `Resources` collection.
4. **Dynamic resources refresh** — Every `{DynamicResource SkyAccentBrush}` in open templates re-resolves. Buttons, links, focus rings, and KPI highlights update together.

Controls that bypass tokens — hard-coded accent hex in a template or a `StaticResource` — stay on the old color. That inconsistency is visible immediately in the demo app accent picker.

## Density

Density is a second runtime axis. `SkyThemeProperties.Density` accepts values like `Comfortable` and `Compact`. `SkyDensityApplicator` swaps density-specific resource dictionaries that adjust padding, min-heights, and font sizes.

Button and navigation pill templates read density tokens rather than fixed pixel values:

```xml
<Setter Property="MinHeight" Value="{DynamicResource SkyButtonMinHeight}" />
<Setter Property="Padding" Value="{DynamicResource SkyButtonPadding}" />
```

When density changes, `SkyButtonMinHeight` resolves to a different value and all buttons resize together. Navigation items, form fields, and list row heights follow the same mechanism through their own density token keys.

Density and accent are independent. An app can run compact layout with a custom brand accent without forking the theme.

## Motion Tokens

Enter and exit animations for overlays use `SkyMotionAnimator` from `SkyUI.Core`. Dialog and snackbar hosts call it rather than hard-coding animation durations:

```csharp
await SkyMotionAnimator.Default.AnimateFadeScaleInAsync(
    dialogPanel, overlayPanel, cancellationToken);
```

Centralizing motion in one animator ensures dialogs, sheets, and snackbars feel consistent. Duration and easing curves live in motion token constants inside `SkyUI.Core`, not scattered across individual controls.

If you add a new overlay host, call the same animator methods. Users perceive inconsistent motion as a product quality issue even when individual animations look fine in isolation.

## Styling Avalonia Primitives

Not every SkyUI visual requires a custom control class. `SkyPreset.Primitives.axaml` styles native Avalonia types:

```xml
<Style Selector="Button.sky">
  <Setter Property="Background" Value="{DynamicResource SkyAccentBrush}" />
  <Setter Property="Foreground" Value="{DynamicResource SkyOnAccentBrush}" />
  <Setter Property="CornerRadius" Value="{DynamicResource SkyRadiusMd}" />
  <Setter Property="Padding" Value="{DynamicResource SkyButtonPadding}" />
</Style>

<Style Selector="Button.sky-primary">
  <Setter Property="Classes" Value="sky sky-primary" />
</Style>
```

Variant classes stack on the base `sky` class:

```xml
<Button Classes="sky sky-outlined" Content="Cancel" />
<Button Classes="sky sky-primary" Content="Save" />
<Button Classes="sky sky-ghost" Content="Learn more" />
```

The `sky` class activates base typography, focus visuals, and pointer feedback. Modifier classes (`sky-outlined`, `sky-ghost`, `sky-danger`) adjust brush assignments through additional style selectors without duplicating full templates.

### Attached Properties on Primitives

`SkyButtonProperties.IsLoading` is an attached property on `Button`:

```csharp
public static readonly AttachedProperty<bool> IsLoadingProperty =
    AvaloniaProperty.RegisterAttached<SkyButtonProperties, Button, bool>("IsLoading");

public static void SetIsLoading(Button button, bool value) =>
    button.SetValue(IsLoadingProperty, value);
```

When `IsLoading` becomes true, the property changed handler swaps button content for a `SkyProgressRing` and adds the `sky-loading` class. This demonstrates how attached properties extend primitives without subclassing.

The handler also toggles `IsEnabled` to prevent double-submit. Theme styles target `Button.sky-loading` to preserve button width while the spinner displays.

## Visual Tree Impact of Theming

When a `ControlTemplate` loads, Avalonia builds a **visual subtree** under the control. Theme setters affect that subtree through:

- **Direct setters** on the control (`Background`, `Padding`) forwarded via `{TemplateBinding}`
- **Resources** resolved per element (`{DynamicResource}` on template children)
- **Styles** matching selectors on the control or its pseudo-classes

The **logical tree** still shows only what you declared in page XAML. Template internals (`PART_Root`, label text blocks, dividers) exist only in the visual tree. That is why theme authors edit `ControlTheme` XAML rather than expecting consumers to nest styling elements in page markup.

If you use `{Binding}` inside a template where you meant `{TemplateBinding}`, the bound element may read the page data context instead of the control property — a frequent source of "the theme ignores my property" reports.

## Authoring Theme for a New Control

When you add a custom control, follow this checklist:

1. Create `ControlTheme` in the appropriate `Controls/*.axaml` file under `SkyUI.Themes.Sky`
2. Reference only `DynamicResource` token keys for colors, spacing, radii, typography, and shadows
3. Add `Style` selectors in the paired `*.Styles.axaml` for pseudo-classes and size variants
4. Register the new files in `ContentFirstDark.axaml` (merged dictionary + style include)
5. Never set colors in the C# control class — not even as fallbacks
6. Add a demo page scenario that toggles accent and density to verify dynamic resolution

### Step-by-Step: Theme a New `SkyStatusPill` Control

Assume a templated control with `Status` (`Success`, `Warning`, `Error`) and optional `Label` text.

**Step 1 — Default ControlTheme**

```xml
<ControlTheme x:Key="{x:Type controls:SkyStatusPill}"
              TargetType="controls:SkyStatusPill">
  <Setter Property="Template">
    <ControlTemplate>
      <Border Name="PART_Root"
              Background="{DynamicResource SkySurfaceBrush}"
              BorderBrush="{DynamicResource SkyBorderSubtleBrush}"
              BorderThickness="1"
              CornerRadius="{DynamicResource SkyRadiusFull}"
              Padding="{DynamicResource SkySpace8Px}">
        <StackPanel Orientation="Horizontal" Spacing="{DynamicResource SkySpace4Px}">
          <Ellipse Name="PART_Indicator" Width="8" Height="8" />
          <TextBlock Name="PART_Label"
                     Text="{TemplateBinding Label}"
                     Foreground="{DynamicResource SkyTextPrimaryBrush}" />
        </StackPanel>
      </Border>
    </ControlTemplate>
  </Setter>
</ControlTheme>
```

**Step 2 — Pseudo-class styles for status**

```xml
<Style Selector="controls|SkyStatusPill.success /template/ Ellipse#PART_Indicator">
  <Setter Property="Fill" Value="{DynamicResource SkySuccessBrush}" />
</Style>
<Style Selector="controls|SkyStatusPill.warning /template/ Ellipse#PART_Indicator">
  <Setter Property="Fill" Value="{DynamicResource SkyWarningBrush}" />
</Style>
<Style Selector="controls|SkyStatusPill.error /template/ Ellipse#PART_Indicator">
  <Setter Property="Fill" Value="{DynamicResource SkyDangerBrush}" />
</Style>
```

Note the `/template/` selector syntax: it pierces into the visual tree to style template parts without exposing them as public properties.

**Step 3 — C# syncs status enum to pseudo-classes** (in the control class, not the theme)

**Step 4 — Register in `ContentFirstDark.axaml` and verify in demo with accent override**

This walkthrough mirrors real SkyUI controls like `SkyBadge` and `SkyAlert`: token-based chrome, pseudo-class semantics, template-part styling.

## Common Mistakes

| Mistake | Why it fails | Fix |
|---------|--------------|-----|
| Hard-coded `#FF1E1E1E` in template | Breaks light theme and accent override | Use `SkySurfaceBrush` |
| `StaticResource` for theme brushes | Does not update on runtime theme change | Use `DynamicResource` |
| Pixel padding not from scale | Visual rhythm breaks across controls | Use `SkySpace*` tokens |
| Logic in XAML triggers | Hard to test, hard to reuse | Move to C# property handlers |
| `{Binding Label}` on template part | Binds to page data context, not control | Use `{TemplateBinding Label}` |
| Fallback color in C# `OnApplyTemplate` | Bypasses theme; hides missing resource bugs | Fix theme registration instead |
| Different spacing on every control | App feels "designed by committee" | Stick to the 8px scale |

## Debugging Tips

**Resource not found at runtime.** Avalonia logs missing `{DynamicResource}` keys in debug output. Check that the token key string matches `SkyTokenKeys` exactly — typos like `SkySurfaceBgBrush` fail silently in release builds.

**Theme changes but one control stays stale.** Inspect that control's template for `StaticResource`, local `Background` setters in C#, or a cached `IBrush` field assigned once in `OnApplyTemplate`.

**Accent override works on buttons but not custom control.** Your custom `ControlTheme` likely hard-codes accent hex or uses `StaticResource`. Grep the theme file for `#` and `StaticResource`.

**Density toggle does not affect your control.** You used fixed `MinHeight` or `Padding` instead of density tokens. Compare your template to `SkyButton` theme setters.

**Template part styles never apply.** Selector may be wrong. Use Avalonia DevTools (when available) to inspect the visual tree and confirm pseudo-classes and `Name` attributes match your style selectors.

## Summary

Design tokens decouple control logic from visual values. SkyUI controls are token consumers; the theme package is the token provider. Dynamic resources, semantic brush names, and shared spacing scales let accent, density, and future palette changes propagate without recompiling controls.

When you build a custom control, treat the token layer as your color palette and spacing ruler, keep all visual definitions in theme XAML, and use `ControlTheme` plus layered `Style` selectors for variation. Understand how resource lookup walks the logical tree and how template bindings differ from data bindings — those two ideas explain most theming bugs before you reach for the debugger.

The next chapter brings everything together into a step-by-step recipe for creating a new SkyUI-style control from scratch.
