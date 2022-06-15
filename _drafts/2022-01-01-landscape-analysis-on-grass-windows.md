---
layout: post
title: "Analyse de la structure du paysage avec GRASS GIS : le problème avec Windows"
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

## Configuration de la grille d'analyse

Le préalable pour utiliser ces outils est de créer un fichier de configuration qui décrit les surfaces à analyser. Pour cela, GRASS GIS un outil graphique, g.gui.rlisetup, qui hélas semble poser des problèmes sur Windows.

Ce bug devrait être corrigé à l'avenir, en tout cas sur les versions de GRASS GIS antérieures à juin 2022 j'ai malgré tout pu faire tourner les outils d'analyse du paysage, de la manière qui suit.

