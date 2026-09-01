# Dessiner avec GFX.Canvas

`GFX.Canvas` décrit des intentions de dessin 2D sans imposer de renderer. Un
`Canvas` enregistre tracés, formes analytiques, remplissages, contours, texte et
groupes avec effets, puis les transforme en contenu retenu ou en image RGBA.

[Read this documentation in English.](../EN/README.md)

## Installer le package

```text
silex install GFX.Canvas
```

GFX.Canvas demande Silex 0.43.0 ou une version plus récente.

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

`Painter.text(..., clip:Canvas.Rect(...))` attache un découpage rectangulaire
aligné sur les axes à cette commande texte. Les renderers raster et retenus
peuvent ainsi conserver une ligne d'overscan jusqu'à son dernier pixel visible,
sans la laisser déborder sur les contrôles voisins. Le découpage appartient au
placement, pas à la font ni au cache de shaping.

## Remplir une forme avec une image

`ImagePaint` conserve une image autonome avec une identité et une révision.
Le mappage reste séparé du contour : un rectangle analytique garde donc ses
six sommets et ses coordonnées de distance, tandis que le renderer reçoit
également la source image et son repère logique.

```sx
use GFX.Assets.Image
use GFX.Canvas
use STD.Math

var portrait = Canvas.ImagePaint(Image.rgba8(2, 1, [
    255, 96, 64, 255,
    64, 128, 255, 255
]))
let frame = Canvas.Rect(Math.Vec2(8.0), Math.Vec2(96.0, 64.0))

var drawing = Canvas()
drawing.paint(func(painter:&Canvas.Painter) {
    painter.fill(
        frame,
        Canvas.Fill.image(
            portrait,
            Canvas.ImageMapping.fit(
                frame:frame,
                mode:Canvas.ImageFit.cover,
                alignment:Math.Vec2(0.5),
                sampling:Canvas.ImageSampling.smooth
            )
        )
    )
})
```

`stretch` associe les bords source et destination, `contain` conserve toute
l'image avec des zones transparentes et `cover` remplit le cadre en le
recadrant. `ImageMapping.tile(origin, texel_size, ...)` répète plutôt la région
source sur les deux axes, y compris avant une origine négative. Le repère est
celui de Canvas et des images : origine en haut à gauche, Y vers le bas. Une
`Image.Region` optionnelle désigne un rectangle de pixels demi-ouvert ; elle
est validée contre l'image courante avant préparation ou rastérisation.

`portrait.replace(next_image)` conserve l'identité, incrémente la révision et
invalide les préparations qui l'emploient. Une région devenue incompatible
échoue explicitement au prochain usage. `Canvas.rasterize`, `Surface.draw` et
Scene2D appliquent le même mappage. Pour un renderer alternatif,
`Prepared.content()` expose une valeur `image` contenant le mesh du masque,
l'`ImagePaint` et l'`ImageMapping`. Un tel remplissage rend
`vector_geometry_compatible()` faux, car le mesh seul ne peut pas transporter
la source image sans détourner ses coordonnées analytiques.

## Isoler un groupe et lui appliquer des effets

`Painter.layer` regroupe plusieurs intentions avant de leur appliquer une
ombre, un flou ou une opacité. L'ombre ci-dessous est produite depuis la forme
et le texte ensemble : le texte reste façonné par `GFX.Font` et conserve son
`GlyphRun`, ses contours vectoriels et sa couverture hintée.

```sx
use GFX.Canvas
use GFX.Color
use STD.Math

var card = Canvas()
card.paint(func(painter:&Canvas.Painter) {
    painter.layer(
        [Canvas.Effect.shadow(
            offset:Math.Vec2(0.0, 6.0),
            radius:12.0,
            color:Color.black().with_alpha(0.25)
        )],
        func(layer:&Canvas.Painter) {
            layer.fill(
                Canvas.Rect(Math.Vec2(8.0), Math.Vec2(160.0, 56.0), 10.0),
                Canvas.Fill.solid(Color.slate_800())
            )
            layer.text(
                "Projet",
                Math.Vec2(24.0, 18.0),
                Canvas.TextStyle(size:20.0, color:Color.white())
            )
        }
    )
})

let image = card.rasterize(184, 80)
```

Les effets d'une couche s'évaluent de gauche à droite. Par exemple,
`[blur, shadow, opacity]` floute d'abord le groupe, calcule l'ombre depuis son
alpha flouté, puis module le contenu et l'ombre. Les couches imbriquées
s'évaluent entièrement de l'intérieur vers l'extérieur. Le flou et la
composition utilisent une alpha prémultipliée afin de ne pas colorer les bords
transparents.

Les bornes publiques d'une couche incluent ses formes, l'encre de son texte et
l'expansion de ses effets. Une rasterisation cadrée découpe seulement le
résultat final à la racine. Une couche ne possède pas encore de clip local ;
les ombres internes, les modes de fusion artistiques et les filtres arbitraires
ne font pas partie de cette surface.

## Réutiliser les intentions

`Canvas.commands()` retourne une copie fidèle des commandes publiques, couches
imbriquées comprises, et `Path.steps()` expose les segments. Une extension peut
ainsi interpréter les intentions sans connaître les caches du renderer.

```sx
for command in drawing.commands() {
    match command {
        fill_rect(value) => { print("rectangle") }
        text(value) => { print(value.value) }
        layer(value) => { print("couche $(value.id()), révision $(value.revision())") }
        else => {}
    }
}
```

`Canvas.vector_geometry_compatible()` indique si le dessin peut être décrit
fidèlement par un mesh seul. `Canvas.vector_geometry()` produit ce mesh lorsque
la réponse est vraie et refuse explicitement un Canvas dont les effets exigent
une composition raster ; il ne supprime jamais silencieusement un effet.

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

`Snapshot.content()` et `Prepared.content()` conservent l'ordre général sous
forme de valeurs `geometry`, `image`, `text` et `layer`. Le contenu d'une couche expose
ses effets, ses bornes, son identité et sa révision. Un renderer alternatif
peut donc rejouer forme, texte et groupe dans l'ordre de l'auteur, sans tenter
de le reconstruire depuis un mesh agrégé et une liste de textes séparée.

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
