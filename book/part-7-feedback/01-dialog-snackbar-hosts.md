---
title: Chapter 21 — Dialog and Snackbar Hosts
order: 21
---

# Chapter 21: Dialog and Snackbar Hosts

Transient UI — modals, toasts, and bottom sheets — requires a different architecture than inline controls. SkyUI uses **host controls** embedded at the application root, paired with **static facades** for imperative show/close APIs. This chapter explains the host pattern, animation integration, concurrency control, hit testing, and the Avalonia overlay mechanics that keep feedback UI predictable across platforms.

## Why Overlays Need Hosts

Inline controls participate in normal measure/arrange passes and share the page's z-order. Modals must sit above navigation chrome, block input to content beneath, and animate without disturbing layout below. Putting a dialog inside a page `Grid` row causes clipping, incorrect centering when keyboards open, and z-order fights with sibling panels.

The host pattern lifts overlays to a dedicated layer at the window root.

## The Host Pattern

```
Window
└── Main content
└── SkyDialogHost (overlay layer, z-index top)
└── SkySnackbarHost (toast layer)
└── SkySheetHost (bottom sheet layer)
```

Each host is a `TemplatedControl` placed once at the root of the visual tree. Application code calls static methods or sets properties on the host to show content.

### Avalonia Concept: Z-Order and Panel Children

In a root `Panel`, later children paint above earlier ones. Declare hosts **after** main content in XAML so they render on top without explicit `ZIndex`. If you use a `Grid` with shared cells, set `Panel.ZIndex` on hosts explicitly — default z-order follows declaration order only within the same parent.

Hosts should stretch to fill the window:

```xml
<Panel>
  <local:MainView />
  <SkyDialogHost HorizontalAlignment="Stretch"
                 VerticalAlignment="Stretch" />
  <SkySnackbarHost VerticalAlignment="Bottom"
                   HorizontalAlignment="Center" />
</Panel>
```

## SkyDialogHost

### Properties and Events

```csharp
public class SkyDialogHost : TemplatedControl
{
    public static readonly StyledProperty<bool> IsOpenProperty = ...;
    public static readonly StyledProperty<string?> TitleProperty = ...;
    public static readonly StyledProperty<object?> DialogContentProperty = ...;
    public static readonly StyledProperty<string?> PrimaryButtonTextProperty = ...;
    public static readonly StyledProperty<string?> SecondaryButtonTextProperty = ...;
    public static readonly StyledProperty<bool> IsDismissibleProperty = ...;

    public static readonly RoutedEvent<RoutedEventArgs> ClosedEvent = ...;
    public static readonly RoutedEvent<RoutedEventArgs> PrimaryActionEvent = ...;
    public static readonly RoutedEvent<RoutedEventArgs> SecondaryActionEvent = ...;
}
```

`IsDismissible` gates overlay-click and escape-key dismissal — use `false` for destructive confirmations that require an explicit button choice.

### Open/Close Lifecycle

```csharp
static SkyDialogHost()
{
    IsOpenProperty.Changed.AddClassHandler<SkyDialogHost>(
        (host, e) => _ = host.OnIsOpenChangedAsync(e));
}

public SkyDialogHost()
{
    IsHitTestVisible = false; // closed host does not block clicks
}

public void Show() => IsOpen = true;

public void Close()
{
    if (!IsOpen) return;
    IsOpen = false;
    RaiseEvent(new RoutedEventArgs(ClosedEvent));
}
```

When closed, `IsHitTestVisible` is false so the host is transparent to pointer events. When open, it becomes true and the overlay captures all input.

### Avalonia Concept: IsHitTestVisible

`IsHitTestVisible="False"` removes the element from hit testing while leaving it in the visual tree. Closed hosts must not intercept clicks meant for content below. Some developers hide hosts with `Opacity="0"` instead — that still blocks hits unless `IsHitTestVisible` is false. Always pair visibility state with hit test visibility for overlay roots.

### Animation on Open/Close

```csharp
private async Task OnIsOpenChangedAsync(AvaloniaPropertyChangedEventArgs e)
{
    animationCancellation?.Cancel();
    animationCancellation = new CancellationTokenSource();
    var token = animationCancellation.Token;

    if (e.NewValue is true)
    {
        IsHitTestVisible = true;
        await SkyMotionAnimator.Default.AnimateFadeScaleInAsync(
            dialogPanel!, overlayPanel!, token);
    }
    else
    {
        await SkyMotionAnimator.Default.AnimateFadeScaleOutAsync(
            dialogPanel!, overlayPanel!, token);
        IsHitTestVisible = false;
    }
}
```

`SkyMotionAnimator` provides consistent fade + scale transitions shared with snackbar and sheet hosts. Cancellation prevents animation glitches when open/close is toggled rapidly.

Animate **opacity and scale** on `PART_DialogPanel`, not the entire host — scaling the full window-sized overlay looks wrong. The scrim fades independently.

### Overlay Click to Dismiss

```csharp
private void OnOverlayPointerPressed(object sender, PointerPressedEventArgs e)
{
    if (!IsDismissible) return;
    if (e.Source == overlayPanel)
        Close();
}
```

Only clicks directly on the scrim dismiss the dialog. Clicks on the dialog panel itself do not propagate to the overlay handler.

Compare `e.Source == overlayPanel`, not `e.Source == sender`, when nested transparent panels exist inside the overlay. Source identifies the original element hit; sender is the element that registered the handler.

### Keyboard Dismissal

Register for escape key on open:

```csharp
private void OnKeyDown(object? sender, KeyEventArgs e)
{
    if (IsOpen && IsDismissible && e.Key == Key.Escape)
    {
        Close();
        e.Handled = true;
    }
}
```

Attach in `OnApplyTemplate` to the host or overlay panel. Mark handled to prevent escape from also closing a parent flyout.

### Template Parts

| Part | Role |
|------|------|
| `PART_Overlay` | Dimmed background scrim |
| `PART_DialogPanel` | Centered dialog card |
| `PART_CloseButton` | X button in header |
| `PART_PrimaryButton` | Confirm action |
| `PART_SecondaryButton` | Cancel action |

Wire button clicks in `OnApplyTemplate`:

```csharp
primaryButton!.Click += (_, _) =>
{
    RaiseEvent(new RoutedEventArgs(PrimaryActionEvent));
    Close();
};
```

Raise action events **before** close if handlers need to read dialog content still mounted in the tree.

## SkyMessageBox: Static Facade

`SkyMessageBox` provides a familiar API without requiring consumers to manage dialog state:

```csharp
public static class SkyMessageBox
{
    private static SkyDialogHost? attachedHost;
    private static readonly SemaphoreSlim dialogGate = new(1, 1);

    public static void Attach(SkyDialogHost host) => attachedHost = host;

    public static async Task<SkyMessageBoxResult> ConfirmAsync(
        string message,
        string? title = "Confirm",
        string primaryText = "OK",
        string secondaryText = "Cancel",
        SkyDialogHost? dialogHost = null,
        CancellationToken cancellationToken = default)
    {
        var host = dialogHost ?? attachedHost
            ?? throw new InvalidOperationException(
                "Call SkyMessageBox.Attach(SkyDialogHost) or pass dialogHost.");

        await dialogGate.WaitAsync(cancellationToken);
        try
        {
            return await ShowOnHostAsync(host, message, title, primaryText, secondaryText);
        }
        finally
        {
            dialogGate.Release();
        }
    }
}
```

### TaskCompletionSource Pattern

`ShowOnHostAsync` typically completes when the dialog closes:

```csharp
private static async Task<SkyMessageBoxResult> ShowOnHostAsync(
    SkyDialogHost host, string message, ...)
{
    var tcs = new TaskCompletionSource<SkyMessageBoxResult>();

    void OnPrimary(object? s, RoutedEventArgs e)
    {
        host.PrimaryAction -= OnPrimary;
        host.SecondaryAction -= OnSecondary;
        host.Closed -= OnClosed;
        tcs.TrySetResult(SkyMessageBoxResult.Primary);
    }

    void OnSecondary(object? s, RoutedEventArgs e) { /* map to Secondary */ }
    void OnClosed(object? s, RoutedEventArgs e) { tcs.TrySetResult(SkyMessageBoxResult.Dismissed); }

    host.PrimaryAction += OnPrimary;
    host.SecondaryAction += OnSecondary;
    host.Closed += OnClosed;

    host.Title = title;
    host.DialogContent = new SkyAlert { Message = message };
    host.PrimaryButtonText = primaryText;
    host.SecondaryButtonText = secondaryText;
    host.Show();

    return await tcs.Task;
}
```

Unsubscribe all handlers in every exit path — leaked handlers cause wrong results on the next dialog.

### Key Design Decisions

1. **Attach pattern** — Register the host once at app startup; all static calls use it
2. **SemaphoreSlim gate** — Only one dialog at a time; concurrent calls queue
3. **Optional host override** — Pass a specific host for testing or multi-window scenarios
4. **Task-based API** — `ConfirmAsync` returns a result enum; callers await the user's choice

### Application Setup

```csharp
// In MainWindow constructor or OnOpened
SkyMessageBox.Attach(this.FindControl<SkyDialogHost>("DialogHost"));
```

```xml
<Panel>
  <local:MainView />
  <SkyDialogHost x:Name="DialogHost" />
</Panel>
```

Call `Attach` after `InitializeComponent` when named hosts exist. For `App.axaml` lifetime, attach in `MainWindow.Opened` so the visual tree is fully materialized.

### Usage

```csharp
var result = await SkyMessageBox.ConfirmAsync(
    "Delete this item permanently?",
    title: "Confirm Delete",
    primaryText: "Delete",
    secondaryText: "Cancel");

if (result == SkyMessageBoxResult.Primary)
    await DeleteItemAsync();
```

The static method builds a `SkyAlert` as dialog content, sets title and button text on the host, awaits the close event, and maps the button clicked to the result enum.

### Debugging SkyMessageBox

| Exception / behavior | Fix |
|---------------------|-----|
| `InvalidOperationException` on Attach | Call `Attach` before first `ConfirmAsync` |
| Second dialog never shows | Prior await stuck; gate not released in `finally` |
| Wrong button result | Handler leak from previous show |
| Dialog behind content | Host declared before main content in panel |
| Clicks pass through open dialog | `IsHitTestVisible` still false mid-animation |

## SkySnackbarHost

Snackbars are non-blocking notifications that auto-dismiss. The host manages a queue:

```csharp
public class SkySnackbarHost : TemplatedControl
{
    private readonly Queue<SkySnackbarItem> _queue = new();

    public void Enqueue(
        string message,
        SkyFeedbackVariant variant = SkyFeedbackVariant.Neutral,
        TimeSpan? duration = null)
    {
        var item = new SkySnackbarItem(message, variant, duration ?? TimeSpan.FromSeconds(4));
        _queue.Enqueue(item);
        ProcessQueueAsync();
    }
}
```

### Queue Processing

Only one snackbar is visible at a time. When the current snackbar's timer expires or the user dismisses it, the next item in the queue animates in. This prevents notification stacking from covering critical UI.

```csharp
private async Task ProcessQueueAsync()
{
    if (_isShowing || _queue.Count == 0) return;
    _isShowing = true;

    while (_queue.Count > 0)
    {
        var item = _queue.Dequeue();
        await ShowItemAsync(item);
        await Task.Delay(item.Duration);
        await HideItemAsync();
    }

    _isShowing = false;
}
```

Use a cancellation token per item so `Enqueue` with urgent messages can skip the delay when you add priority support later.

Unlike dialogs, snackbars should **not** set `IsHitTestVisible` on the full host — only the bar itself needs hits for the dismiss button. The rest of the window stays interactive.

### Static Facade for Snackbars

Mirror the dialog attach pattern:

```csharp
public static class SkySnackbar
{
    private static SkySnackbarHost? attachedHost;

    public static void Attach(SkySnackbarHost host) => attachedHost = host;

    public static void Show(string message, SkyFeedbackVariant variant = SkyFeedbackVariant.Neutral)
    {
        (attachedHost ?? throw new InvalidOperationException("Attach snackbar host first"))
            .Enqueue(message, variant);
    }
}
```

View models call `SkySnackbar.Show("Saved!")` without injecting the host.

### SkySnackbarBar

Each notification is rendered by `SkySnackbarBar`, a `TemplatedControl` with variant-driven styling:

```csharp
public enum SkyFeedbackVariant
{
    Neutral, Info, Warning, Danger, Success
}
```

The variant maps to brush tokens: `SkyInfoBrush`, `SkyWarningBrush`, `SkyDangerBrush`, and accent colors for success.

Apply variant via style classes in a changed handler — same pattern as `Chip`:

```csharp
Classes.Set("feedback-info", variant == SkyFeedbackVariant.Info);
Classes.Set("feedback-danger", variant == SkyFeedbackVariant.Danger);
```

## SkySheetHost and SkyActionSheet

Mobile action sheets use the same host pattern. `SkyActionSheet` is a static facade over `SkySheetHost`:

```csharp
public static class SkyActionSheet
{
    public static void Attach(SkySheetHost host) => attachedHost = host;

    public static async Task<SkyActionSheetResult> ShowAsync(
        IReadOnlyList<SkyActionSheetItem> items,
        string? title = null,
        SkySheetHost? host = null)
    {
        // Build button list, show sheet, await selection
    }
}
```

`SkyActionSheetItem` carries title, destructive flag, and cancel flag. The host animates the sheet up from the bottom with the same motion system as dialogs.

Destructive items use `SkyDangerBrush` in the template. Cancel items are visually separated — often in their own row with top margin — following platform conventions even on desktop when simulating mobile flows.

Sheets share the semaphore pattern with dialogs if only one sheet should be visible. Alternatively, sheets queue like snackbars depending on product rules.

## Layer Coordination

When dialog, sheet, and snackbar hosts coexist:

| Layer | Input block | Typical z-order |
|-------|-------------|-----------------|
| Dialog | Full window | Highest |
| Sheet | Full window or bottom region | High |
| Snackbar | None (bar only) | Above content, below modal |

Opening a dialog while a snackbar shows: either dismiss the snackbar immediately or leave it dimmed beneath the scrim — pick one behavior app-wide.

## SkyFeedbackVariant Across Controls

The `SkyFeedbackVariant` enum is shared across `SkyAlert`, `SkyBanner`, `SkySnackbarBar`, and message box content. One enum drives consistent semantic coloring everywhere feedback is shown.

Centralizing variants means token renames propagate through one mapping table in the theme layer.

## Avalonia Concept: Routed Events for Host Actions

Host controls expose `RoutedEvent` instances for primary/secondary actions rather than `event EventHandler` CLR events. Routed events bubble, support class handlers, and integrate with Avalonia's event routing diagnostics. Facades subscribe temporarily per show cycle; do not use routed events for long-lived view model bindings.

## Building Your Own Overlay System

1. **One host per overlay type** — dialog, toast, sheet
2. **Embed at root** — above all page content
3. **Toggle `IsHitTestVisible`** — only capture input when open (dialogs/sheets)
4. **Animate with shared motion** — consistent enter/exit across overlay types
5. **Static facade for common cases** — message box, action sheet
6. **Semaphore for modal concurrency** — one dialog at a time
7. **Queue for toasts** — one visible, rest wait
8. **TaskCompletionSource for awaitable APIs** — map user action to task result
9. **Unsubscribe handlers** — every show cycle cleans up prior subscriptions

## Testing Overlay Hosts

Headless tests can attach a host to a `Window`, call `Show()`, simulate button clicks via `RaiseEvent`, and assert task results. Test the semaphore by starting two `ConfirmAsync` calls without awaiting the first — second should wait, not throw.

For visual regression, snapshot the host in open state with fake content — overlay animations are often disabled in test builds via `SkyMotionPreferences`.

## Summary

The host + facade pattern separates **where** overlays render (root-level host) from **how** they are triggered (static API or direct property setting). Animation, hit testing, and concurrency are handled inside the host so application code stays simple: `await SkyMessageBox.ConfirmAsync(...)` or `snackbarHost.Enqueue("Saved!")`. Understanding `IsHitTestVisible`, z-order, and TaskCompletionSource wiring resolves most overlay bugs without rewriting page layout.
