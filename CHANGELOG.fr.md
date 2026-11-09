# Changelog

## [0.5.0] - 2026-11-09

- Précise que TextInputTarget porte une zone en coordonnées logiques de dessin : le présentateur assure sa conversion vers la fenêtre, comme pour la source accessible. Viewer tient compte de position et anchor automatiquement.

- Canvas.Session propose accessibility_source() optionnel, avec défaut null : les conformances existantes restent compatibles. Requiert Silex 0.48.0 ou supérieur, qui prend en charge les corps de méthodes de protocole.

- `Canvas.Session` demande désormais `pointer_cursor() GFX.Window.Cursor` ; retourner `standard` pour conserver le pointeur flèche. Les consommateurs et le guide de migration sont adaptés.

- Canvas.Session demande text_input_target() : cible opaque et zone de saisie native, ou null lorsque le producteur ne fournit pas de zone. Les sessions UI la relaient automatiquement ; les producteurs sans texte retournent null.

- Les polices embarquées résolvent automatiquement les graphèmes absents avec les symboles Noto Sans Mono, Noto Emoji monochrome puis Noto Sans CJK SC. Faces de repli chargées paresseusement ; accents, séquences emoji et métriques restent cohérents entre mesure, rendu et grille fixe. Les faces explicites restent strictes.

- Adopte la déclaration `Result` unique : l’appel ordinaire rend la valeur
  ou arrête le programme sur erreur ; `try appel()` conserve le `Result`. Les
  anciens appels stockant un résultat et propagations deviennent explicites.
  Cette version requiert Silex 0.48.0 ou supérieur et les versions coordonnées
  de ses dépendances.

- Ajouter `Painter.clip(rect, callback)` : découpage rectangulaire des groupes, clips imbriqués et révisions retenues. Les rasters et le contenu ordonné conservent la limite après les effets enfants ; les meshes seuls la refusent. Un changement de clip invalide les pixels de texte sans refaire leurs glyphes.

- La police Canvas par défaut emploie le monospace embarqué pour les graphèmes absents de sa face principale et présents dans cette face de secours. Mesure, raster, grille fixe et géométrie retenue partagent les mêmes runs ; les polices explicites restent inchangées. Les placements vectoriels identifient la face de chaque glyphe.
