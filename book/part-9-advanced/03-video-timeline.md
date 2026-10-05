---
title: Chapter 29 — VideoTimeline
order: 29
---

# Chapter 29: VideoTimeline

`VideoTimeline` is a sealed `TemplatedControl` for multi-track video editing UIs — clips on lanes, playhead scrubbing, range selection, markers, and clipboard operations. At over 1,600 lines, it is SkyUI's largest single control and demonstrates **imperative canvas interaction** within a templated shell.

## Domain Model

Bindable collections represent timeline data:

```csharp
public sealed class VideoTimeline : TemplatedControl
{
    public static readonly DirectProperty<VideoTimeline, ObservableCollection<TimelineTrackItem>> TracksProperty = ...;
    public static readonly DirectProperty<VideoTimeline, ObservableCollection<TimelineClipItem>> ClipsProperty = ...;
    public static readonly DirectProperty<VideoTimeline, ObservableCollection<TimelineMarkerItem>> MarkersProperty = ...;

    public static readonly StyledProperty<double> DurationProperty = ...;        // default 120 seconds
    public static readonly StyledProperty<double> PixelsPerSecondProperty = ...; // default 40
    public static readonly StyledProperty<double> PlayheadTimeProperty = ...;
    public static readonly StyledProperty<bool> IsPlayingProperty = ...;
}
```

| Type | Represents |
|------|------------|
| `TimelineTrackItem` | Named lane (video, audio, subtitle) |
| `TimelineClipItem` | Segment on a track with start time and duration |
| `TimelineMarkerItem` | Point annotation on the ruler |

## TimelineHost: Where the Logic Lives

`VideoTimeline` is intentionally thin. The constructor creates **`TimelineHost`**, which owns:

| Collaborator | Responsibility |
|--------------|----------------|
| `TimelineProject` | Canonical `Tracks`, `Clips`, `Markers` collections |
| `TimelineLayoutEngine` | Lane heights, clip rectangles, snap (`TimelineSnapSettings`) |
| `TimelineSelectionModel` | Selected clips, tracks, time range |
| `TimelineUndoStack` | `CanUndo` / `CanRedo`; mutating edits push commands |
| `TimelineRenderer` | Paints clips, playhead, selection on `PART_MainCanvas` |
| `TimelineGestureCoordinator` | Registers zoom, keyboard, lane, ruler, clip, and header interactors |

Public API on `VideoTimeline` forwards to the host: `Project`, `Undo()` / `Redo()`, clipboard helpers, and collection properties (`Tracks`, `Clips`, `Markers`) expose the same instances as `_host.Project`. Property changed handlers on duration, pixels-per-second, and playhead call into the host so layout and playback stay coherent.

When you debug timeline behavior, start in `TimelineHost.ApplyTemplate` (template part wiring and scroll sync) and `TimelineClipGestureInteractor` (drag-move and trim), not only in the control's styled properties.

## Template Parts for Scroll Sync

Multiple synchronized scroll viewers:

| Part | Role |
|------|------|
| `PART_RulerScroll` | Horizontal time ruler |
| `PART_RulerCanvas` | Tick marks and time labels |
| `PART_VerticalTrackScroll` | Track headers |
| `PART_MainScroll` | Clip canvas horizontal scroll |
| `PART_MainCanvas` | Clip rectangles, selection, playhead |

Ruler and main canvas scroll horizontally together — pointer handlers on one update offset on the other.

## Coordinate System

Horizontal position maps time to pixels:

```csharp
private double TimeToX(double time) => time * PixelsPerSecond;
private double XToTime(double x) => x / PixelsPerSecond;
```

`Duration` and `PixelsPerSecond` define virtual canvas width = `Duration * PixelsPerSecond`. Vertical position maps to track index from `Tracks` collection order.

## Playback Timer

```csharp
public void Play()
{
    IsPlaying = true;
    _playTimer.Start();
}

private void OnPlayTick()
{
    PlayheadTime = Math.Min(PlayheadTime + tickDelta, Duration);
    if (PlayheadTime >= Duration)
        Stop();
    InvalidateVisual();
}
```

`PlayheadTime` is bindable — external preview players can sync to the same property.

## Interaction State Machine

Pointer handlers manage modes:

1. **Scrub** — drag on ruler moves playhead
2. **Select clip** — click clip updates selection, raises `ClipSelectionChanged`
3. **Move clip** — drag clip changes start time on track
4. **Range select** — drag on empty canvas selects time range (`TimeRangeSelectionChanged`)
5. **Zoom** — mouse wheel adjusts `PixelsPerSecond` within clamped bounds

Each mode captures pointer on press and releases on pointer up. `Focusable = true` enables keyboard shortcuts (delete selected clips, copy/paste).

## Clipboard API

```csharp
public void CopySelectionToClipboard();
public void CutSelectionToClipboard();
public void PasteClipboardAtPlayhead();
```

Uses `SkyClipboard` helpers to serialize selected `TimelineClipItem` data. Paste creates new clips at `PlayheadTime` offset.

## Collection Change Handling

```csharp
public ObservableCollection<TimelineClipItem> Clips
{
    set
    {
        if (_clips == value) return;
        UnhookClips(_clips);
        _clips = value;
        HookClips(_clips);
        FullRebuild();
    }
}
```

`INotifyCollectionChanged` on clips and tracks triggers `FullRebuild()` to recalculate layout metrics. Individual property changes on clip items (start time, duration) trigger partial invalidation.

## MVVM Integration

You can bind directly to `Tracks` / `Clips` / `Markers` on the control (they are the host's `TimelineProject` collections), or mutate `timeline.Project` after construction. Either way, use the **same collection instances** the timeline already owns — replacing collections goes through the direct properties on `VideoTimeline`.

```csharp
public class EditorViewModel
{
    public VideoTimeline Timeline { get; }

    public EditorViewModel(VideoTimeline timeline) => Timeline = timeline;

    public void LoadDemoClips()
    {
        Timeline.Clips.Add(new TimelineClipItem { /* ... */ });
    }
}
```

Listen to `ClipChanged`, `ClipsRemoved`, and `UndoRedoStateChanged` to persist edits. The host applies undoable commands during drag; your view model should not fight those mutations by resetting collections on every pointer move.

## Custom Rendering vs Child Controls

Clips are drawn on `PART_MainCanvas` via `DrawingContext` or positioned `Border` children — the implementation chooses based on interaction needs. Hit testing maps pointer position to clip bounds stored in layout cache.

This hybrid — templated scroll chrome + imperative canvas content — is common in professional editing controls.

## When to Study VideoTimeline

Use this control as reference when building:

- Gantt charts with drag-resize bars
- Audio waveforms with scrubbing
- Scheduling grids with time axes
- Any zoomable, scrollable custom canvas

Do **not** use it as reference for simple form controls — the complexity is domain-driven.

## Deep Dive: Scroll Synchronization

Three `ScrollViewer` instances must stay aligned:

| ScrollViewer | Axis | Synced with |
|--------------|------|-------------|
| `PART_RulerScroll` | Horizontal | `PART_MainScroll` |
| `PART_MainScroll` | Horizontal + vertical | Ruler horizontal |
| `PART_VerticalTrackScroll` | Vertical | Main vertical |

When the user scrolls the clip canvas horizontally, handlers copy `Offset.X` to the ruler scroll viewer so tick marks align with clips:

```csharp
private void OnMainScrollChanged(object? sender, ScrollChangedEventArgs e)
{
    if (_rulerScroll is not null && Math.Abs(_rulerScroll.Offset.X - e.NewOffset.X) > 0.5)
        _rulerScroll.Offset = new Vector(e.NewOffset.X, _rulerScroll.Offset.Y);
}
```

Guard with epsilon to prevent feedback loops when both handlers react to each other.

### Layout Scroll Restore

After `FullRebuild()`, scroll positions restore from `_pendingLayoutScrollRestore` — rebuilding clip visuals resets scroll without this would jump the playhead off-screen.

---

## Deep Dive: Clip Visual Lifecycle

Clips are not one-control-per-model in the logical tree indefinitely. `FullRebuild()` clears `_clipBorders` and recreates `Border` elements positioned on `PART_MainCanvas`:

```csharp
private void FullRebuild()
{
    _mainCanvas?.Children.Clear();
    _clipBorders.Clear();
    foreach (var clip in Clips)
        AddClipVisual(clip);
    DrawPlayhead();
    DrawMarkers();
}
```

`AddClipVisual` computes:

```csharp
var left = TimeToX(clip.StartTime);
var width = TimeToX(clip.Duration);
var top = TrackIndexToY(clip.TrackId);
```

Hook `PropertyChanged` on each `TimelineClipItem` for live updates when view model mutates start/duration during drag.

### Trim Handles

`TrimHandleWidth = 8` pixels at clip left/right edges. Pointer near edge starts trim drag instead of move drag — changes `StartTime` or `Duration` without moving the whole clip.

`MinClipDurationSeconds = 0.08` prevents collapse to zero length.

---

## Deep Dive: Selection and Clipboard

```csharp
private readonly HashSet<string> _selectedClipIds = new(StringComparer.Ordinal);

public IReadOnlyList<TimelineClipItem> GetSelectedClips() =>
    Clips.Where(c => _selectedClipIds.Contains(c.Id)).ToList();
```

Selection is internal — the control does not expose `SelectedClips` as a bindable property to avoid fighting MVVM selection state. Listen to `ClipSelectionChanged` and call `GetSelectedClips()` to sync outward.

### Copy / Cut / Paste

```csharp
public void CopySelectionToClipboard();
public void CutSelectionToClipboard();
public void PasteClipboardAtPlayhead();
```

Internal `_clipboard` list clones `TimelineClipItem` data. Paste offsets clips to `PlayheadTime` so inserted segments align with the current scrub position. `ClipsPasted` event carries new IDs for undo stack integration.

---

## Deep Dive: Playback Loop

```csharp
private readonly DispatcherTimer _playTimer = new() { Interval = TimeSpan.FromMilliseconds(33) };

// ~30 fps
private void OnPlayTick(object? sender, EventArgs e)
{
    var next = PlayheadTime + 0.033;
    if (next >= Duration)
    {
        PlayheadTime = Duration;
        Stop();
    }
    else
        PlayheadTime = next;
    InvalidatePlayhead();
}
```

`PlayheadTime` is a styled property with coercion to `[0, Duration]`. External preview engines can bind two-way and drive the same property while muting the internal timer.

`Seek(double time)` jumps playhead and raises `PlayheadChanged` without starting playback.

---

## Deep Dive: Time Range Selection

Drag on empty canvas (not on a clip) creates a shaded range between two times:

```csharp
public bool HasTimeRangeSelection { get; }
public (double Start, double End) TimeRangeSelection { get; }
```

`TimeRangeSelectionChanged` fires when the user completes drag. Export, delete range, or loop playback can consume the range — the control only surfaces geometry.

---

## Deep Dive: Track Reorder

Drag track header in `PART_HeaderStack` past `TrackReorderDragThreshold` (8px):

1. Highlight drop index
2. On release, reorder `Tracks` collection
3. Raise `TrackOrderChanged` with old/new indices

Clips reference `TrackId` string — reordering tracks does not mutate clips, only Y layout mapping.

---

## Deep Dive: Wheel Zoom

```csharp
private void OnMainCanvasPointerWheelChanged(object? sender, PointerWheelEventArgs e)
{
    var factor = e.Delta.Y > 0 ? 1.1 : 0.9;
    PixelsPerSecond = Math.Clamp(PixelsPerSecond * factor, MinPps, MaxPps);
    FullRebuild();
}
```

`PixelsPerSecond` coercion keeps zoom within usable range. Zoom anchors at playhead or pointer X depending on implementation — check source for `ZoomAt` behavior when extending.

---

## Property Coercion

```csharp
public static readonly StyledProperty<double> DurationProperty =
    AvaloniaProperty.Register<VideoTimeline, double>(nameof(Duration), 120, coerce: CoerceDuration);
```

Coercion clamps `PlayheadTime` when `Duration` shrinks, preventing playhead stranded past end. Always register coerce callbacks when properties have cross-dependencies.

---

## MVVM Contract Summary

| Bind to VM | Control owns |
|------------|--------------|
| `Tracks`, `Clips`, `Markers` collections | Clip selection set |
| `Duration`, `PixelsPerSecond` | Scroll sync internals |
| `PlayheadTime`, `IsPlaying` | Play timer |
| `SelectedTrackId` | Track highlight |

Mutate collections on the VM; listen to events for edits initiated inside the control (drag, delete, paste).

---

## Architecture Diagram

```
┌──────────────── VideoTimeline ─────────────────┐
│ PART_RulerScroll → PART_RulerCanvas (ticks)   │
│ PART_VerticalTrackScroll → PART_HeaderStack   │
│ PART_MainScroll → PART_MainCanvas             │
│     ├── Clip borders (positioned by time)     │
│     ├── Playhead line                         │
│     ├── Range selection shade                 │
│     └── Markers                               │
│ DispatcherTimer (playback)                    │
│ Pointer state: move | trim | select | scrub   │
└───────────────────────────────────────────────┘
```

## Debugging Checklist

| Symptom | Check |
|---------|-------|
| Ruler out of sync | Main scroll handler not copying X offset |
| Clips wrong track Y | `TrackId` mismatch after reorder |
| Playhead jumps on rebuild | Scroll restore not applied |
| Selection lost after zoom | `FullRebuild` clearing without restoring `_selectedClipIds` |
| Paste at wrong time | `PlayheadTime` not updated before paste |

## Summary

`VideoTimeline` pushes SkyUI's templated control model to its limit: triple scroll sync, imperative clip canvas, trim/move pointer modes, internal selection with clipboard, and coerced time properties. It complements `DiagramSurface` (Chapter 27) — spatial data on a time axis instead of free-form graph coordinates.
