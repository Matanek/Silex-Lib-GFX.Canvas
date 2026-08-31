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

`GFX.Font` possède les faces proportionnelle et monospace distribuées ;
`Canvas.Font.default()` et `Canvas.Font.monospace()` restent les façades
compatibles. Une application peut aussi fournir un chemin, des octets encodés
ou envelopper une `GFX.Font.Face` déjà validée avec `Canvas.Font(face)`. Canvas
utilise les runs et couvertures hintées de GFX.Font sans exposer de handle
natif.

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

`Canvas.snapshot()` conserve la géométrie dans ses coordonnées locales, sans
inventer de rectangle de dessin. Chaque couche de texte calcule sa propre
couverture, y compris lorsque son origine est négative. La forme
`Canvas.snapshot(width, height)` reste disponible lorsqu’un viewport, un layout
ou un export doit découper le texte selon un cadre précis.

Pour des révisions successives, passez l’ancien snapshot par `reuse:previous`.
Les commandes texte identiques gardent leur révision et leur couverture de
glyphes, tandis que la géométrie vectorielle peut changer indépendamment.

Un renderer qui anime continuellement le même dessin emploie plutôt une
préparation retenue. `drawing.prepared()` fournit la préparation non bornée
partagée par tous les placements du dessin ;
`drawing.prepared(width, height)` en conserve une par taille de cadre. Une
révision suivante du même Canvas met ces objets à jour en place. Le renderer
peut donc partager la compilation sans reconstruire ni remplacer chaque
composant.

```sx
var prepared = drawing.prepared()

// Après clear(), paint(...) ou toute autre révision du même Canvas :
prepared = drawing.prepared()
let geometry = prepared.geometry()
```

Pour posséder explicitement une préparation indépendante, créez
`Canvas.Prepared` et passez-la à `prepare(...)`. Le Canvas réécrit alors sa
géométrie dans deux stockages retenus alternés : leur capacité est recyclée, la
topologie précédente reste disponible pour détecter une géométrie inchangée et
les couches de texte sont mises à jour indépendamment.

```sx
var prepared = Canvas.Prepared()
drawing.prepare(prepared)

// Après clear(), paint(...) ou toute autre révision du même Canvas :
drawing.prepare(prepared)
let geometry = prepared.geometry()
```

Les surcharges `prepared(width, height)` et
`prepare(prepared, width, height)` cadrent les couches de texte.
Les cercles, rectangles et lignes droites sont compilés en quads analytiques
de six sommets ; rayon, largeur, orientation et terminaison sont évalués par le
shader au lieu de tesseller à nouveau les contours ronds sur le CPU.

`Snapshot.text_layers()` donne à chaque commande de texte une identité stable,
son `GlyphRun` canonique et sa propre image de couverture paresseuse.
`glyph_run()` retourne la ligne homogène ; `glyph_runs()` couvre le multiligne
et `vector_outlines()` conserve les mêmes glyph IDs, clusters et positions pour
un renderer vectoriel. `coverage_runs(density)` expose les mêmes runs avec leur
origine physique arrondie, sans rastériser : un renderer peut consulter son
atlas avant de demander la couverture d'un glyphe absent. Un renderer peut donc
transférer seulement le libellé modifié. Le cache Canvas direct de bitmaps
hintés est borné à 2 048 emplacements et 16 Mio,
en plus des budgets natifs de GFX.Font.

`TextLayer.shaping_revision()` ne change que lorsque le texte, la face, la
taille, les variations ou le placement typographique changent. Un renderer
retenu peut donc lire `color()` et mettre à jour une instance de couleur sans
redécomposer ni retesseller les contours.

`TextStyle.fixed_advance` façonne la ligne entière, désactive les ligatures,
alternatives contextuelles et kernings qui traverseraient une cellule, puis
place chaque cluster Unicode sur une origine de grille exacte. Les marks d’un
cluster restent associés au même run ; aucun octet UTF-8 n’est traité
isolément. Cette option convient aux lignes de terminal ou d’éditeur ; le
texte ordinaire conserve les avances naturelles de la police.

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

GFX.Canvas ne possède plus de moteur ni d’archive de fonte. Il dépend de
GFX.Font pour les faces, le shaping, les contours et les couvertures hintées.
Son provider SDL3 restant sert uniquement les surfaces Canvas existantes et
compose `GFX.SDL3` sans redistribuer SDL3. GFX.Font porte sa propre matrice
d’artefacts macOS ARM64, Linux x64, Windows x64 et Windows ARM64.
