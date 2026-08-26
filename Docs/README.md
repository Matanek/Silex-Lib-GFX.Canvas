# GFX.Canvas

`GFX.Canvas` describes 2D drawing without imposing a renderer. A `Canvas`
records drawing intentions and exposes `paint`, `clear`, `rasterize`, and
inspection APIs directly. The same content can also be converted into geometry
by an alternative renderer.

```silex
use GFX.Color
use GFX.Canvas
use STD.Math

var canvas = Canvas()
canvas.paint(func(painter:&Canvas.Painter) {
    painter.fill(
        Canvas.Rect(Math.Vec2(), Math.Vec2(128.0, 48.0), 8.0),
        Canvas.Fill.solid(Color.indigo_600())
    )
    painter.text(
        "Silex",
        Math.Vec2(12.0, 10.0),
        Canvas.TextStyle(size:18.0, color:Color.white())
    )
})

let image = canvas.rasterize(128, 48)
```

Text uses `SDL3_ttf` behind the `Canvas.Font` API. GFX.Canvas embeds the
proportional `Canvas.Font.default()` and the terminal/editor-oriented
`Canvas.Font.monospace()`; an application may also supply a path or encoded
bytes. Text advances remain proportional across display densities so measured
layout positions stay aligned with high-density rasterization. No SDL handle
is public.

## Reuse drawing intentions

`Canvas.commands()` exposes a copy of the public commands, `Path.steps()`
exposes the geometric segments, and `Canvas.vector_geometry()` produces a
GPU-independent mesh. These three levels let an extension interpret drawing
intentions, tessellate paths itself, or reuse the supplied tessellation.

```silex
for command in canvas.commands() {
    match command {
        fill_rect(value) => { print("rectangle") }
        text(value) => { print(value.value) }
        else => {}
    }
}
```

`Canvas.snapshot(width, height)` preserves geometry and caches the text layer
at the requested density. It is an optional optimization, not a required
representation for third-party renderers.

A renderer presenting successive revisions can call
`Canvas.snapshot(width, height, reuse:previous)`. Vector geometry is still
compiled from the new commands, while identical text commands retain their
text revision and cached glyph coverage. `Snapshot.text_layers()` gives each
text command a stable identity and its own lazy coverage image. A renderer can
therefore preserve static labels while uploading only a changed value. Text
changes invalidate the corresponding layer automatically; callers never
compare commands themselves. Font coverage uses a bounded 256-entry cache so
repeated labels avoid another SDL_ttf render without unbounded growth.

## Surface

`Canvas.Surface` is a mutable RGBA pixel surface. It accepts the same painting
commands, can compose images, and returns a `GFX.Assets.Image`. The interactive
examples under `Examples/` display their results through Viewer. Their local
development dependency owns that presentation capability; GFX.Canvas remains
independent from Viewer and Scene2D in every consumer graph. Prepare it with
`silex install GFX.Canvas --dev`.

When a result needs to be displayed, `GFX.Viewer` accepts the Canvas itself and
its authored dimensions. Viewer retains it through Scene2D and follows window
resizing without creating an intermediate image:

```sx
Viewer.show(canvas, 960, 640, "Generated drawing")
```

`ShapeGallery2D.sx` exercises lines, caps, joins, paths, curves, analytic
shapes, gradients, transparency, layering and text in one retained gallery
presented by Viewer. The other examples display rasterized images and surfaces.
