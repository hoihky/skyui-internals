---
title: Chapter 27 — DiagramSurface
order: 27
---

# Chapter 27: DiagramSurface — Custom Panel Rendering

Most SkyUI controls inherit `TemplatedControl` and delegate rendering to XAML templates. `DiagramSurface` in the `SkyUI.Diagram` package takes a different path: it subclasses `Panel` and manages its own visual children for nodes and edges. This chapter explains when to bypass templates, how to implement custom layout, pluggable strategies, pointer interaction state machines, and the Avalonia rendering APIs that make canvas-style editors possible.

## Why Panel Instead of TemplatedControl?

Diagram editors need:

- Arbitrary node positioning on a canvas
- Edge lines drawn between node ports
- Drag, connect, resize, and select interactions
- Zoom and pan across a large virtual canvas

A `ControlTemplate` with fixed `ContentPresenter` slots cannot express this. A custom `Panel` overrides `MeasureOverride` and `ArrangeOverride` to position child elements at absolute coordinates, and optionally renders edge geometry in `Render`.

### Avalonia Concept: Panel vs Canvas

Avalonia's built-in `Canvas` supports attached `Canvas.Left` / `Canvas.Top` positioning. `DiagramSurface` subclasses `Panel` directly for full control over measure semantics, child lifecycle tied to document nodes, and custom render passes for edges. `Canvas` would still require a separate edge layer — the diagram package unifies model sync, layout, hit testing, and drawing in one type.

## DiagramSurface Overview

```csharp
public sealed class DiagramSurface : Panel
{
    public static readonly StyledProperty<DiagramModel?> ModelProperty = ...;
    public static readonly StyledProperty<IEdgePathComputer?> PathComputerProperty = ...;
    public static readonly StyledProperty<IDiagramNodePresenterFactory?> NodeFactoryProperty = ...;
    public DiagramSelectionModel Selection { get; }
}
```

The surface binds to a **`DiagramModel`** (`IDiagramModel`) containing nodes and edges. `DiagramNodePresenter` controls sit on an internal nodes canvas; edges draw on a separate layer. `PathComputer` resolves port world positions; `NodeFactory` creates per-type presenters. Default straight routing ships as `StraightEdgePathComputer`.

## Pluggable Strategies

Strategy interfaces keep the surface stable while algorithms and node visuals vary — the same extensibility pattern as `IEdgePathComputer` in graph libraries and `IDocumentView` in IDE shells.

### IEdgePathComputer

Resolves **world-space endpoints** for each edge from port metadata on the model:

```csharp
public interface IEdgePathComputer
{
    (Point Start, Point End) GetEndpoints(IDiagramEdge edge, IDiagramModel model);
}
```

`StraightEdgePathComputer` looks up source and target nodes, finds ports by id, and calls `GetWorldPosition` on each port relative to node bounds. The surface draws a line segment (or polyline in custom routers) between those points whenever layout changes.

Orthogonal or spline routers still implement the same interface — only the segment construction differs. Cache per edge until `DiagramModelChangedEventArgs` reports `NodeLayout` or `Structure` changes.

### IDiagramNodePresenterFactory

Creates the visual control for each node:

```csharp
public interface IDiagramNodePresenterFactory
{
    Control CreatePresenter(DiagramNode node);
    void UpdatePresenter(Control presenter, DiagramNode node);
}
```

Different node types (task, decision, start/end) can have different visual representations without the surface knowing about specific node classes.

Register presenters by node type key:

```csharp
public sealed class TypedNodePresenterFactory : IDiagramNodePresenterFactory
{
    private readonly Dictionary<string, Func<DiagramNode, Control>> _creators = new();

    public Control CreatePresenter(DiagramNode node) =>
        _creators[node.Kind](node);

    public void UpdatePresenter(Control presenter, DiagramNode node)
    {
        if (presenter.DataContext != node)
            presenter.DataContext = node;
    }
}
```

### IDiagramSceneHitTester

Determines what is under the pointer:

```csharp
public interface IDiagramSceneHitTester
{
    DiagramHitTestResult HitTest(DiagramSurface surface, Point point);
}
```

Hit testing drives selection, connection dragging, and context menus.

Edges drawn in `Render` are not visual children — default hit testing misses them. The hit tester must:

1. Transform pointer point by inverse zoom/pan
2. Test node bounds in document space
3. Test edge geometry with `Geometry.FillContains` / distance-to-segment for strokes

```csharp
public DiagramHitTestResult HitTest(DiagramSurface surface, Point point)
{
    var docPoint = surface.ScreenToDocument(point);

    foreach (var edge in surface.Model!.Edges.Reverse())
    {
        if (HitTestEdge(edge, docPoint, out var hit))
            return DiagramHitTestResult.Edge(edge, hit);
    }

    foreach (var node in surface.Model.Nodes.Reverse())
    {
        var rect = new Rect(node.X, node.Y, node.Width, node.Height);
        if (rect.Contains(docPoint))
            return DiagramHitTestResult.Node(node);
    }

    return DiagramHitTestResult.Empty;
}
```

Reverse iteration hits top-most items first — match paint order.

## Layout: Measure and Arrange

As a `Panel`, `DiagramSurface` positions node presenters at their model coordinates:

```csharp
protected override Size MeasureOverride(Size availableSize)
{
    foreach (var child in Children)
    {
        child.Measure(new Size(double.PositiveInfinity, double.PositiveInfinity));
    }
    return ComputeContentSize(); // aggregates node bounds from Model
}

protected override Size ArrangeOverride(Size finalSize)
{
    foreach (var child in Children)
    {
        if (child.Tag is DiagramNode node)
        {
            child.Arrange(new Rect(node.X, node.Y, node.Width, node.Height));
        }
    }
    return finalSize;
}
```

Node position and size come from the model, not from layout constraints. The panel's available size is effectively infinite (canvas semantics).

### Avalonia Concept: Desired Size vs Model Bounds

`ComputeContentSize()` unions node `Bounds` from `Model` so `MeasureOverride` reports the intrinsic canvas size. Scrollable hosts wrap the surface in a `ScrollViewer` — report document size, not viewport size.

### Syncing Children to Model

Subscribe to `Model.Nodes` / `Model.Edges` collection changes and `DiagramModel.Changed`:

```csharp
private void OnNodesChanged(object? sender, NotifyCollectionChangedEventArgs e)
{
    switch (e.Action)
    {
        case NotifyCollectionChangedAction.Add:
            foreach (DiagramNode node in e.NewItems!)
                AddNodePresenter(node);
            break;
        case NotifyCollectionChangedAction.Remove:
            foreach (DiagramNode node in e.OldItems!)
                RemoveNodePresenter(node);
            break;
    }
    InvalidateMeasure();
    InvalidateVisual();
}

private void AddNodePresenter(DiagramNode node)
{
    var presenter = NodeFactory!.CreatePresenter(node);
    presenter.Tag = node;
    Children.Add(presenter);
}
```

`DiagramNodePresenter` holds a reference to the `DiagramNode` model. Unsubscribe in `AttachModel()` when `Model` is replaced to avoid leaks.

## Edge Rendering

Edges are typically not `Control` children. They are drawn in `Render`:

```csharp
public override void Render(DrawingContext context)
{
    base.Render(context);

    if (Model is null || PathComputer is null)
        return;

    foreach (var edge in Model.Edges)
    {
        var (start, end) = PathComputer.GetEndpoints(edge, Model);
        var pen = Selection.IsEdgeSelected(edge.Id) ? SelectedEdgePen : EdgePen;
        context.DrawLine(pen, start, end);
    }
}
```

Keeping edges in the render pass rather than as child controls avoids the overhead of thousands of `Control` instances in large diagrams.

### Avalonia Concept: DrawingContext and Pens

`DrawGeometry` takes fill brush and pen. For stroke-only lines, pass `null` fill and a `Pen` with desired thickness. Hairline edges at odd zoom levels alias — snap coordinates or use `Pen` with `LineCap.Round` for cleaner joins.

Call `InvalidateVisual()` when edge routes change — layout invalidation alone does not rerun `Render`.

### Port Position

```csharp
private Point GetPortPosition(DiagramNode node, DiagramPort port)
{
    var rect = new Rect(node.X, node.Y, node.Width, node.Height);
    return port.Side switch
    {
        PortSide.Right  => rect.TopRight + new Vector(0, rect.Height * port.NormalizedOffset),
        PortSide.Left   => rect.TopLeft  + new Vector(0, rect.Height * port.NormalizedOffset),
        PortSide.Top    => rect.TopLeft  + new Vector(rect.Width * port.NormalizedOffset, 0),
        PortSide.Bottom => rect.BottomLeft + new Vector(rect.Width * port.NormalizedOffset, 0),
        _ => rect.Center
    };
}
```

## Zoom and Pan

Apply transform in `Render` and inverse transform in hit testing:

```csharp
private Matrix BuildZoomPanTransform() =>
    Matrix.CreateTranslation(PanOffset.X, PanOffset.Y) *
    Matrix.CreateScale(Zoom, Zoom);

public Point ScreenToDocument(Point screen) =>
    BuildZoomPanTransform().Invert().Transform(screen);
```

Pointer wheel with Ctrl modifies `Zoom`; middle-button drag or space+drag modifies `PanOffset`. Clamp zoom between sensible min/max to prevent numerical instability in invert.

Optional: render grid background in `Render` before edges using document-space line loop — cheap visual anchor for users.

## Interaction: Connect and Drag

Pointer handlers on the surface manage interaction state machines:

1. **Select** — click node, update selection in document
2. **Drag node** — pointer capture, update node X/Y on move
3. **Connect** — drag from output port, show rubber-band line, snap to input port on release
4. **Resize** — drag resize grip, update node width/height

Each interaction mode is a separate handler class or method group, similar to how `CheckedListDragReorderHandler` is separated from `CheckedListBox`.

### Mode State Machine

```csharp
private enum InteractionMode { None, DragNode, Connect, Resize, Pan }
private InteractionMode _mode;

private void OnPointerPressed(object? sender, PointerPressedEventArgs e)
{
    var hit = HitTester!.HitTest(this, e.GetPosition(this));

    if (hit.IsPort && hit.Port!.IsOutput)
    {
        _mode = InteractionMode.Connect;
        _connectSource = hit;
        e.Pointer.Capture(this);
        return;
    }

    if (hit.IsNode)
    {
        _mode = InteractionMode.DragNode;
        _dragNode = hit.Node;
        _dragStart = e.GetPosition(this);
        e.Pointer.Capture(this);
        SelectNode(hit.Node);
    }
}
```

Only one mode active at a time. `PointerReleased` clears mode and capture.

### Rubber-Band Connection Line

Draw provisional edge in `Render` when `_mode == Connect`:

```csharp
if (_mode == InteractionMode.Connect && _connectSource is not null)
{
    var start = GetPortPosition(_connectSource.Node!, _connectSource.Port!);
    var end = ScreenToDocument(_currentPointer);
    context.DrawLine(RubberBandPen, start, end);
}
```

On release, if hit target is compatible input port, add `DiagramEdge` to document.

## DiagramModel

```csharp
public sealed class DiagramModel : IDiagramModel
{
    public ObservableCollection<DiagramNode> Nodes { get; }
    public ObservableCollection<DiagramEdge> Edges { get; }
    public event EventHandler<DiagramModelChangedEventArgs>? Changed;

    public DiagramNode AddNode(string nodeTypeKey, string label, Rect initialBounds);
    public DiagramEdge? TryAddEdge(string sourceNodeId, string sourcePortId,
        string targetNodeId, string targetPortId);
    public IDiagramNode? FindNode(string nodeId);
}
```

Nodes carry `Bounds`, ports, and labels. Edges reference node and port ids. Mutations raise `Changed` with `DiagramModelChangeKind` (`Structure`, `NodeLayout`, `Edge`) so the surface can invalidate measure, sync presenters, and redraw edges without polling every property.

Keep the model serializable in your app layer — the diagram package focuses on editing semantics, not persistence format.

## Clipboard Support

Diagram operations integrate with `SkyClipboard` for copy/paste of selected nodes and edges. Serialized node data is placed on the clipboard as JSON or a custom format, and paste creates new nodes offset from the originals.

Paste offset (20, 20) prevents exact overlap. Remap edge endpoints to new node IDs during paste — storing edges without ID translation produces dangling connections.

## Performance at Scale

| Technique | Benefit |
|-----------|---------|
| Edge in `Render` | Avoid N `Control` instances for edges |
| Invalidate visual on change only | Skip full tree layout per frame |
| Hit test broad phase | Spatial hash or grid before geometry tests |
| Freeze pens/brushes | Reduce allocations in `Render` |
| Virtualize off-screen nodes | Optional for very large graphs — advanced |

Profile with hundreds of nodes before optimizing — Avalonia handles moderate diagrams well if you avoid redundant `InvalidateMeasure` on every pointer move during drag (invalidate arrange or update model + single invalidate).

## Debugging DiagramSurface

| Symptom | Likely cause |
|---------|--------------|
| Nodes clip at origin | Arrange uses wrong coordinates; zoom transform missing |
| Edges disconnected from ports | Port offset math or stale layout before render |
| Clicks miss edges | Hit tester ignores render-only geometry |
| Drag jumps | Screen vs document space mix-up |
| Children duplicate | CollectionChanged add without remove on replace |
| Blank surface | `Model` null or `NodeFactory` null — guard in `AttachModel` |

Enable diagnostic overlay drawing node bounds and port dots in debug builds.

## When to Build a Custom Panel

Use a custom `Panel` subclass when:

- Children need absolute positioning, not flow layout
- You need to render geometry that is not a `Control` (lines, curves, grids)
- Performance requires avoiding a `Control` per data item (thousands of edges)
- Interaction is canvas-like (drag, connect, zoom, pan)

Stay with `TemplatedControl` when:

- The visual structure is a fixed set of named parts
- Items map 1:1 to `Control` instances with templates
- Layout is flow-based (stack, grid, wrap)

## Building Your Own Canvas Control

1. **Separate model from view** — `DiagramModel` is independent of `DiagramSurface`
2. **Factory for visuals** — node types vary; the surface should not hard-code presenters
3. **Strategy for geometry** — edge routing algorithms are pluggable
4. **Render non-control graphics** — use `DrawingContext` for lines and shapes
5. **Hit test separately** — do not rely on default visual hit testing for render-only content
6. **Interaction state machine** — one active mode at a time (select, drag, connect, resize)
7. **Transform consistency** — same matrix in render, hit test, and drag delta math
8. **Invalidate correctly** — `InvalidateVisual` for paint; `InvalidateMeasure` for document bounds

## Deep Dive: DiagramModel Sync and Presenter Lifecycle

`DiagramSurface` does not bind XAML to nodes. When `Model` changes, `AttachModel()` wires collection events and syncs presenters:

```csharp
private void AttachModel()
{
    DetachModel();
    _wiredModel = Model;
    if (_wiredModel is null) return;

    _wiredModel.Nodes.CollectionChanged += OnNodesChanged;
    _wiredModel.Edges.CollectionChanged += OnEdgesChanged;
    foreach (var node in _wiredModel.Nodes)
        EnsureNodePresenter(node);
    foreach (var edge in _wiredModel.Edges)
        EnsureEdgePresenter(edge);
}
```

### Node Presenter Factory

```csharp
public interface IDiagramNodePresenterFactory
{
    DiagramNodePresenter Create(DiagramNode node);
    void Update(DiagramNodePresenter presenter, DiagramNode node);
}
```

`DefaultDiagramNodePresenterFactory` returns a bordered rectangle with label `TextBlock`. Custom factories render database tables, decision diamonds, or image thumbnails — the surface only positions `DiagramNodePresenter` controls on `_nodesCanvas`.

```csharp
internal void SyncNodeCanvasPosition(DiagramNodePresenter p)
{
    Canvas.SetLeft(p, n.Bounds.X);
    Canvas.SetTop(p, n.Bounds.Y);
    p.Width = n.Bounds.Width;
    p.Height = n.Bounds.Height;
    UpdateAllEdges();
}
```

Every node move triggers edge reroute — edges are dependent on node port positions.

### Edge Presenters and Z-Order

```csharp
private readonly Canvas _nodesCanvas = new();
private readonly Canvas _overlayCanvas = new();  // preview line during connect

private const int EdgePresenterZIndex = 10;
private const int SelectedNodePresenterZIndex = 30;
```

Edges are `Control` children with higher `ZIndex` than unselected nodes so lines paint on top. Selected nodes bump to 30 so resize handles receive pointer events above edges.

Custom routers implement `GetEndpoints` or post-process those points into polylines before `DrawLine`. The stock `StraightEdgePathComputer` only resolves port world coordinates; the surface owns the actual stroke.

---

## Deep Dive: Pointer Drag State Machine

`DiagramSurface` tracks one active drag mode:

```csharp
private enum DragKind { None, MoveNode, ResizeNode, Connect, Reconnect }
private DragKind _drag;
```

### Move Node

```csharp
public void BeginMoveNode(DiagramNode node, PointerPressedEventArgs e)
{
    _drag = DragKind.MoveNode;
    _moveNodeId = node.Id;
    _moveStartBounds = node.Bounds;
    _pressDiagramPoint = e.GetPosition(this);
    e.Pointer.Capture(this);
    HookDrag();
}
```

On `PointerMoved`, delta translates `node.Bounds` and calls `SyncNodeCanvasPosition`. `Pointer.Capture` ensures moves continue when the pointer leaves the node bounds.

### Connect (Rubber Band)

Starting drag from an output port:

1. `_drag = DragKind.Connect`
2. `_previewLine` on `_overlayCanvas` follows pointer
3. On release over input port, `Model.Edges.Add(new DiagramEdge(...))`
4. On release elsewhere, cancel — no edge created

`HitTester` identifies port vs node body vs empty canvas.

### Resize

`ResizeHandle` enum (TopLeft, BottomRight, etc.) determines which bounds edges move. `MinWidth`/`MinHeight` on node model prevent collapse to zero.

### Reconnect

Drag an existing edge endpoint to a different port — `_reconnectEdgeId` and `_reconnectSourceEnd` track which end moves.

---

## Deep Dive: Selection and Clipboard

```csharp
public DiagramSelectionModel Selection { get; } = new();

public void SelectNode(string nodeId, bool additive)
{
    if (!additive) Selection.Clear();
    Selection.AddNode(nodeId);
}
```

Shift-click adds to selection; plain click replaces. `SelectionChanged` updates visual affordances (border, resize grips).

### Copy / Paste

```csharp
private List<NodeClipboardEntry>? _nodeClipboard;
private List<EdgeClipboardEntry>? _edgeClipboard;
```

Copy serializes selected nodes and **internal** edges (both endpoints selected). Paste clones with offset IDs so pasted nodes do not collide. Edge clipboard entries remap old IDs to new IDs through a dictionary built during node paste.

---

## Deep Dive: Hit Testing Rendered Edges

Edges drawn as `Path` controls participate in visual hit testing. For geometry-only render passes, implement `IDiagramSceneHitTester`:

```csharp
public interface IDiagramSceneHitTester
{
    DiagramHitResult HitTest(DiagramSurface surface, Point diagramPoint);
}
```

`DiagramSceneHitTester` walks nodes top-down, then edges by distance-to-path threshold. Without custom hit testing, thin lines are hard to click — widen hit tolerance in the tester, not visual stroke.

---

## MeasureOverride and Infinite Canvas

```csharp
protected override Size MeasureOverride(Size availableSize)
{
    var needed = ComputeContentSize();  // max node right/bottom + padding
    return new Size(
        Math.Max(availableSize.Width, needed.Width),
        Math.Max(availableSize.Height, needed.Height));
}
```

The panel grows to fit all nodes — parent `ScrollViewer` provides pan when content exceeds viewport. `InvalidateMeasure()` when nodes move outside current bounds.

---

## Usage Example

```xml
<diagram:DiagramSurface Model="{Binding FlowModel}"
                       PathComputer="{x:Static local:OrthogonalRouter.Instance}"
                       NodeFactory="{x:Static local:FlowchartNodeFactory.Instance}" />
```

```csharp
public class FlowViewModel
{
    public DiagramModel FlowModel { get; } = new();

    public void AddStep(string label, double x, double y)
    {
        FlowModel.Nodes.Add(new DiagramNode
        {
            Id = Guid.NewGuid().ToString(),
            Label = label,
            Bounds = new Rect(x, y, 120, 60)
        });
    }
}
```

---

## Debugging Checklist

| Symptom | Cause |
|---------|-------|
| Edges not updating | `UpdateAllEdges` not called after node move |
| Cannot click edge | Hit tolerance too small |
| Nodes jump on drag | Mixing screen coords with diagram coords — always `GetPosition(this)` |
| Paste duplicates IDs | Clipboard remap dictionary incomplete |
| Memory leak on model swap | `DetachModel` not unsubscribing collection events |

## Summary

`DiagramSurface` is a custom layout engine: model-driven presenter sync, drag state machine, pluggable routing, and explicit hit testing. Where `TemplatedControl` ends, `Panel` + canvas children + strategy injection begin — the pattern for diagrams, flow editors, and whiteboards.

---

## Epilogue: Patterns Across SkyUI

This book has covered SkyUI from foundations to advanced rendering. The patterns recur:

| Pattern | Examples |
|---------|----------|
| TemplatedControl + ControlTheme | SkyFormField, Chip, SkyDialogHost |
| ContentControl composition | SkyCard, Badge |
| ItemsControl containers | SkyAccordion |
| Primitive subclass + style class | SkySearchBox, SkyDatePicker |
| Attached properties | IsLoading, GridLayout, TouchTarget |
| Adapter interface | CheckedListBox, VirtualDataGrid, Diagram |
| Abstract base class | SkyPaginationBase |
| Inheritance for presets | SkyVirtualTreeView, SkyComboBoxField |
| Host + static facade | SkyDialogHost + SkyMessageBox |
| Pseudo-classes | SkyDivider, SkySplitView, SkyNavigationView |
| Design tokens | All theme templates |
| Custom Panel | DiagramSurface |

Every custom control you build can be classified into one or more of these patterns. Start with the simplest approach — style class on a primitive — and add complexity only when the behavior demands it. That is the SkyUI way.

When you reach for `Panel` + `Render`, you trade template convenience for explicit geometry and interaction ownership. The diagram package shows that tradeoff executed with clear model boundaries, strategy injection, and disciplined invalidation — a template you can reuse for charts, whiteboards, timelines, and any scene where data is spatial rather than hierarchical.
