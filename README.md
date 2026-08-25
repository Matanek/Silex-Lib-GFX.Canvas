# GFX.Canvas

`GFX.Canvas` records portable 2D drawing intentions and turns them into vector
geometry or RGBA images. It provides paths, analytic shapes, fills, strokes,
text, snapshots and mutable pixel surfaces without imposing a window or a
renderer.

```sh
silex install GFX.Canvas
```

To run the interactive examples as well:

```sh
silex install GFX.Canvas --dev
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

The interactive programs under `Examples/` use the package's
`devDependencies` to open a window through Viewer. Viewer therefore remains
absent from the public dependency graph consumed by applications.

See [Docs/README.md](Docs/README.md) for the full drawing, geometry, text and
surface contracts.
