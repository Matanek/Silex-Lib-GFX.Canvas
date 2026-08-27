# GFX.Canvas

`GFX.Canvas` records portable 2D drawing intentions and turns them into vector
geometry or RGBA images. It provides paths, analytic shapes, fills, strokes,
text, snapshots and mutable pixel surfaces without imposing a window or a
renderer.

```sh
silex install GFX.Canvas
```

```silex
use GFX.Canvas
use GFX.Color
use STD.Math

var drawing = Canvas()
drawing.paint(func(painter:&Canvas.Painter) {
    painter.fill(
        Canvas.Rect(Math.Vec2(), Math.Vec2(128.0, 48.0), 8.0),
        Canvas.Fill.solid(Color.indigo_600())
    )
    painter.text("Silex", Math.Vec2(12.0, 10.0), Canvas.TextStyle(size:18.0))
})

let image = drawing.rasterize(128, 48)
```

The package embeds both its proportional default font and a reusable monospace
font exposed by `Canvas.Font.monospace()`. It also owns the complete SDL3_ttf
boundary. SDL handles remain private, and the provider composes with `GFX.SDL3`
instead of redistributing SDL3. Canvas has no dependency on Viewer or Scene2D;
applications can add `GFX.Viewer` when they want one-call presentation or
`GFX.Scene2D` when they want retained GPU rendering.

`Canvas.Session` is the platform-neutral contract for an interactive Canvas
producer. It owns drawing, update, input and text-input intentions without
owning a window, an Application loop or an ECS world. Presenters such as
`GFX.Viewer` drive that contract; UI packages can adapt their own retained
state to it.

Successive retained renderings can pass their previous `Canvas.Snapshot` to
`snapshot(width, height, reuse:previous)`. Unchanged text preserves its cached
glyph coverage and `text_revision()`, while vector commands remain free to
change independently. `Snapshot.text_layers()` additionally exposes stable
per-command identities to retained renderers, so changing one label does not
invalidate every other text texture. SDL_ttf coverage is cached by bounded
font, size, weight and content keys.

`Canvas.TextStyle(fixed_advance:...)` places Unicode grapheme clusters on an
explicit horizontal grid while retaining one text command. This is intended
for terminal and editor rows whose cell positions must stay exact; ordinary
proportional and monospace text keeps its natural advances by default.

The complete retained [2D shape gallery](https://github.com/Matanek/Silex-Examples/tree/main/Sources/ShapeGallery2D)
lives in `Silex-Examples`; focused image, text, surface, and presentation usage
is illustrated in the package documentation and verified by its tests. Viewer
therefore remains absent from both the public and development dependency graphs
of Canvas.

See [Docs/README.md](Docs/README.md) for the full drawing, geometry, text and
surface contracts.
