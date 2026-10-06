# Changelog

## [Unreleased]

- Ajouter `Painter.clip(rect, callback)` : découpage rectangulaire des groupes, clips imbriqués et révisions retenues. Les rasters et le contenu ordonné conservent la limite après les effets enfants ; les meshes seuls la refusent. Un changement de clip invalide les pixels de texte sans refaire leurs glyphes.

- La police Canvas par défaut emploie le monospace embarqué pour les graphèmes absents de sa face principale et présents dans cette face de secours. Mesure, raster, grille fixe et géométrie retenue partagent les mêmes runs ; les polices explicites restent inchangées. Les placements vectoriels identifient la face de chaque glyphe.
