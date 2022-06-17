---
layout: post
title: "Analyse de la structure du paysage avec GRASS GIS et Windows"
date: 2022-01-01
---

Lorsqu'on a besoin d'analyser des données d'Occupation du Sol du point de vue de la structure paysagère (% de zone naturelle, taille et diversité des patches), **GRASS GIS** offre des possibilités intéressantes, notamment pour les données au format raster.

En effet, GRASS GIS :

- permet de traiter de très gros raster (exemple NLCD, Theia OSO), même avec des machines qui disposent de peu de mémoire
- propose des outils d'analyse du paysage inspirés de Fragstats

Ces derniers outils, qui porte le préfixe "r.li", sont documentés ici :

<https://grass.osgeo.org/grass78/manuals/topic_landscape_structure_analysis.html>

Ce post décrit le cas de figure qui suit :

- nous avons un raster - [OSO](https://www.theia-land.fr/product/carte-doccupation-des-sols-de-la-france-metropolitaine/) - avec l'occupation du sol en métropole a une résolution de 10 m et souhaitons calculer l'indice de diversité Shannon
- l'indice sera calculé par chaque maille d'une grille d'analyse - les mailles sont carrées, avec une largeur et une hauteur de 5000 m.

Chaque maille couvre donc 500 x 500 soit 250000 pixels sur le raster à analyser. Nous souhaitons calculer l'indice de Shannon sur chaque maille, pour obtenir en sortie soit une grille vecteur, soit un raster avec une résolution de 5000 m.

## Le problème sur Windows : configuration de la grille d'analyse

Le préalable pour utiliser ces outils est de créer un fichier de configuration qui décrit les surfaces à analyser. Pour cela, GRASS GIS un outil graphique, `g.gui.rlisetup`, qui hélas semble poser des problèmes sur Windows. Le bouton **Suivant** reste déactivé sur le 1er écran, ce qui empêche de créer le fichier de configuration.

Ce bug devrait être corrigé à l'avenir, en tout cas sur les versions de GRASS GIS antérieures à juin 2022 j'ai malgré tout pu faire tourner les outils d'analyse du paysage, de la manière qui suit.

Le fichier de configuration est un fichier texte de 3 lignes qui doit être enregistré à un emplacement bien précis de votre poste, dans votre dossier utilisateur : le répertoire `C:\Users\BernardC\AppData\Roaming\GRASS7\r.li`.

Les différentes configurations possibles pour les zones d'échantillonnage sont décrites dans l'aide en ligne de `g.gui.rlisetup`. 
Dans notre cas, utiliser une grille avec des mailles carrés ou rectangulaires correspond à la configuration **SYSTEMATIC CONTIGUOUS**.

Le nombre de lignes et de colonnes de la grille est important, car il faut reporter dans le fichier la dimension d'une maille par rapport à la dimension de la zone analysée.

Exemple : 

dans le dossier caché `AppData` de votre dossier utilisateur.
L'outil `g.gui.rlisetup` génère 

g.region -p                                                                     
projection: 99 (RGF93 v1 / Lambert-93)
zone:       0
datum:      towgs84=0,0,0,0,0,0,0
ellipsoid:  grs80
north:      6518826.6973
south:      6133826.6973
west:       311759.6886
east:       931759.6886
nsres:      10
ewres:      10
rows:       38500
cols:       62000
cells:      2387000000

rows: 38500
cols: 62000

1/77
1/124

    SAMPLINGFRAME 0|0|1|1
    SAMPLEAREA -1|-1|0.012987012987013|0.0080645161290323
    SYSTEMATICCONTIGUOUS

C:\Users\BernardC\AppData\Roaming\GRASS7\r.li


## Fichier en sortie

Dans le cas Le fichier généré 