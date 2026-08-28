# Dessiner avec GFX.Canvas

`GFX.Canvas` décrit des intentions de dessin 2D sans imposer de renderer. Un
`Canvas` enregistre tracés, formes analytiques, remplissages, contours et texte,
puis les transforme en géométrie vectorielle ou en image RGBA.

[Read this documentation in English.](../EN/README.md)

## Installer le package

```text
silex install GFX.Canvas
```

GFX.Canvas demande Silex 0.39.0 ou une version plus récente.

## Enregistrer et rastériser un dessin

```sx
use GFX.Canvas
use GFX.Color
use STD.Math

var drawing = Canvas()
drawing.paint(func(painter:&Canvas.Painter) {
    painter.fill(
        Canvas.Rect(Math.Vec2(), Math.Vec2(128.0, 48.0), 8.0),
        Canvas.Fill.solid(Color.indigo_600()),
    )
    painter.text(
        "Silex",
        Math.Vec2(12.0, 10.0),
        Canvas.TextStyle(size:18.0, color:Color.white()),
    )
})

let image = drawing.rasterize(128, 48)
```

Le package incorpore une police proportionnelle par défaut et une police
monospace accessible par `Canvas.Font.monospace()`. Une application peut aussi
fournir un chemin ou des octets encodés. SDL3_ttf reste derrière l’API Font ;
aucun handle SDL n’est public.

## Réutiliser les intentions

`Canvas.commands()` retourne une copie des commandes publiques,
`Path.steps()` expose les segments et `Canvas.vector_geometry()` produit un
mesh indépendant du GPU. Une extension peut ainsi interpréter les intentions,
tesseller elle-même ou réutiliser la tessellation fournie.

```sx
for command in drawing.commands() {
    match command {
        fill_rect(value) => { print("rectangle") }
        text(value) => { print(value.value) }
        else => {}
    }
}
```

`Canvas.snapshot(width, height)` conserve la géométrie et met les couches de
texte en cache à la densité demandée. Pour des révisions successives, passez
l’ancien snapshot par `reuse:previous`. Les commandes texte identiques gardent
leur révision et leur couverture de glyphes, tandis que la géométrie vectorielle
peut changer indépendamment.

`Snapshot.text_layers()` donne à chaque commande de texte une identité stable
et sa propre image de couverture paresseuse. Un renderer peut donc transférer
seulement le libellé modifié. Le cache de couverture est borné à 256 entrées.

`TextStyle.fixed_advance` place les graphèmes Unicode sur une grille
horizontale explicite tout en gardant une seule commande texte. Cette option
convient aux lignes de terminal ou d’éditeur ; le texte ordinaire conserve les
avances naturelles de la police.

## Peindre une surface mutable

`Canvas.Surface` est une surface de pixels RGBA mutable. Elle accepte les mêmes
commandes, compose des images et retourne un `GFX.Assets.Image`.

Canvas ne dépend ni de Viewer ni de Scene2D. Quand un dessin doit être affiché,
`GFX.Viewer` accepte le Canvas et ses dimensions, tandis que `GFX.Scene2D`
fournit un rendu GPU retenu. La [galerie 2D](https://github.com/Matanek/Silex-Examples/tree/main/Sources/ShapeGallery2D)
et le benchmark [RetainedCanvasGeometry](https://github.com/Matanek/Silex-Benchmarks/blob/main/Sources/RetainedCanvasGeometry.sx)
appartiennent à leurs dépôts centraux.

## Produire un dessin interactif

`Canvas.Session` décrit un producteur avec état sans choisir son architecture
de présentation :

```sx
class Preview:Canvas.Session {
    func drawing(width:int, height:int) Canvas { return build(width, height) }
    func update(delta:float) bool { return advance(delta) }
    func handle(event:@GFX.Input.Event) bool { return interact(event) }
    func accepts_text_input() bool { return editing() }
}
```

Les fonctions du fragment appartiennent au producteur. Les booléens invalident
le dessin courant. La session conserve son état et reçoit l’input de plateforme,
mais ne possède ni fenêtre, ni Application, ni renderer, ni monde ECS.

## Comprendre la frontière native

GFX.Canvas possède SDL3_ttf et compose son provider avec `GFX.SDL3` sans
redistribuer SDL3. Des artefacts épinglés existent pour macOS ARM64, Linux x64,
Windows x64 et Windows ARM64 ; cette dernière cible reste reconnue et
expérimentale dans la matrice Silex.
