# Changelog

## [Unreleased]

- Les polices embarquées résolvent automatiquement les graphèmes absents avec les symboles Noto Sans Mono, Noto Emoji monochrome puis Noto Sans CJK SC. Faces de repli chargées paresseusement ; accents, séquences emoji et métriques restent cohérents entre mesure, rendu et grille fixe. Les faces explicites restent strictes.

- Adopte la déclaration `Result` unique : l’appel ordinaire rend la valeur
  ou arrête le programme sur erreur ; `try appel()` conserve le `Result`. Les
  anciens appels stockant un résultat et propagations deviennent explicites.
  Cette révision requiert la toolchain de développement avec cette sémantique
  de `try` ; sa publication doit coordonner les versions du compilateur et des
  packages.

- Ajouter `Painter.clip(rect, callback)` : découpage rectangulaire des groupes, clips imbriqués et révisions retenues. Les rasters et le contenu ordonné conservent la limite après les effets enfants ; les meshes seuls la refusent. Un changement de clip invalide les pixels de texte sans refaire leurs glyphes.

- La police Canvas par défaut emploie le monospace embarqué pour les graphèmes absents de sa face principale et présents dans cette face de secours. Mesure, raster, grille fixe et géométrie retenue partagent les mêmes runs ; les polices explicites restent inchangées. Les placements vectoriels identifient la face de chaque glyphe.
