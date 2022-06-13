---
layout: post
title: "Effective Mesh Size (CBC method) with Grass GIS"
date: 2021-09-07
---

Dans ce post, nous allons utiliser le logiciel Grass GIS pour calculer l'indicateur de fragmentation mEff CBC (Effective Meshsize, CBC method) sur un jeu de données raster : l'occupation du sol issue du produit Theia OSO [Carte d'Occupation des Sols de la France Métropolitaine](https://www.theia-land.fr/product/carte-doccupation-des-sols-de-la-france-metropolitaine/).

## Le contexte

La **taille effective de maille** est un indicateur de fragmentation des espaces naturels qui exprime la probabilité que 2 points choisis dans un territoire donné se trouve sur un espace naturel continu. Le territoire en question peut être la maille d'une grille régulière (hexagone, carré) ou une entité administrative (commune). Il est calculable selon 2 méthodes. 

- La méthode originale dite **CUT** (pour *Cutting Out*) proposée par Jaeger en 2000 consiste à calculer la somme de la surface au carré des patchs, divisée par la surface du territoire.
- La méthode dite **CBC** (pour *Cross-Boundary Connection*) proposée par le même auteur en 2007, consiste à multiplier la surface du patch à l'intérieur du territoire par la surface totale du patchs (intérieur et extérieur). On calcule ensuite la somme du produit, divisée par la surface du territoire. Cela permet de corriger un biais introduit dans la méthode **CUT** par la grille d'analyse qui coupe artificiellement les patchs.

## Vecteur ou raster ?

La **taille effective de maille** est généralement calculée à partir de données de type "Occupation du sol" desquelles on extrait les zones naturelles. Parfois on utilise en complément des surfaces issues de base de données routières et ferroviaires pour identifier les discontinuités. 

Il faut ensuite calculer la surface des patchs de zones naturelles et notamment la surface inclue dans les mailles.

Le calcul se fait différemment si on travaille en mode vecteur ou en mode raster.

- en mode **vecteur**, on calcule la surface totale des patchs, puis on intersecte les patchs avec la grille d'analyse, puis on calcule la surface des patchs découpés. C'est relativement facile à mettre en oeuvre même dans un logiciel comme QGIS.
- en mode **raster**, pour identifier les patchs, il faut disposer d'un outil qui identifie les plages continues de pixels. Un tel outil existe dans GRASS GIS : il s'agit de `r.clump`. Ensuite il faut combiner la couche raster des patchs (identifiés par un ID unique) et la couche raster de la grille d'analyse (avec un ID unique pour chaque maille également) et calculer le nombre total de pixels pour chaque combinaison. Dans GRASS GIS, l'outils `r.stats` permet cela. Il ne restera plus ensuite qu'à agréger les surfaces calculées pour obtenir la taille effective de maille.

Le logiciel Fragstats propose le calcul de la taille effective de maille avec la méthode **CUT**. Ce que je propose ici est plutôt le calcul de la méthode **CBC** en mode raster.

Pourquoi utiliser le logiciel Grass GIS dans ce cas ? Parce que Grass GIS reste très performant sur les gros rasters.

## Le workflow Calcul de l'indice "Effective Mesh Size CBC" sur Grass

### Fichiers en entrée :

    `OCS_2018_CESBIO.tif` = Couche occupation du sol CESBIO
    `grid-WGS84.shp` = grille WGS84 
    `emprise_fr.shp` = emprise zone analyse (pour masquer raster)

### Import données

- import raster `OCS_2018_CESBIO.tif` (SCR=EPSG:2154) => **OCS_2018_CESBIO**
- import vecteur `grid-WGS84.shp` (SCR=EPSG:4326) => **grid_L93** (SCR=EPSG:2154)
- import vecteur `emprise_fr.shp` => **emprise_5km**

### Traitement grille vecteur
- ajoute colonne **id_int** (INT)
- MAJ colonne id_int=id

### Définir zone analyse
- zone = emprise_5km alignée sur OCS_2018_CESBIO

### Convertir grille vecteur en raster
- conversion grid_L93 => **grid_rast**
- conversion emprise_5km => **rast_emprise_5km** (masque pour le calcul à venir) avec val=1

### Filtrer OCS zones naturelles (Calculatrice raster)  
- si OCS dans (13,18,19) et rast_emprise_5km = Vrai alors **OCS_cat131819_null** = Vrai sinon OCS_cat131819_null = null
- exporter OCS_cat131819_null => **result131819_clip.tif** (COMPRESS=DEFLATE,PREDICTOR=2)

### Identifier les patches
- identifier les patchs dans OCS_cat131819_null => **OCS_cat131819_clump**

### Stats patches X mailles
- superposer grid_rast X OCS_cat131819_clump, pour compter le nombre de pixels de chaque patch dans chaque maille de la grille => **stats_completes.txt**

### Calculer Effective Mesh Size
- lire stats_completes.txt dans un tableau
- calculer **a** = surface patch X maille dans le tableau **tbl_stats_grass**
- calculer **a_cmpl** = surface totale des patchs (grouper par id_patch puis somme) dans le tableau **tbl_patch_cmpl**
- calculer **aT** = surface totale des mailles (grouper par id puis somme) dans le tableau **tbl_msize**
- joindre tbl_stats_grass > tbl_patch_cmpl
- calculer **a2** = produit a * a_cmpl
- calculer (somme des a2) / aT
- joindre le résultat à la grille vectorielle pour visualiser l'indice de fragmentation

## Le Script shell pour GRASS GIS

```bash
    r.in.gdal -o input=D:\Travail\Support\Leandro\OCS_2018_CESBIO.tif output=OCS_2018_CESBIO
    v.in.ogr -o input=D:\Travail\Support\Leandro\FR\emprise_fr.shp output=emprise_5km

    # importer grille vecteur en lambert 93 => nelle couche vecteur grid_L93
    v.import input='C:\Support\Leandro\grid-WGS84.shp' layer=grid-WGS84 output=grid_L93
    # ajoute champ id_int INTEGER
    v.db.addcolumn map=grid_L93@PERMANENT columns='id_int INT'
    v.db.update map=grid_L93@PERMANENT column=id_int query_column=id

    g.region -s -p vector=emprise_5km@PERMANENT align=OCS_2018_CESBIO@PERMANENT
    v.to.rast input=grid_L93@PERMANENT type=area output=grid_rast use=attr attribute_column=id_int
    v.to.rast input=emprise_5km@PERMANENT type=area output=rast_emprise_5km use=val

    r.mapcalc --overwrite expression=OCS_cat131819 = (OCS_cat131819 = OCS_2018_CESBIO@PERMANENT == 13 || OCS_2018_CESBIO@PERMANENT == 18 || OCS_2018_CESBIO@PERMANENT == 19) && rast_emprise_5km@PERMANENT == 1
    r.out.gdal input=OCS_cat131819@PERMANENT output=D:\Travail\Support\Leandro\result131819_clip.tif format=GTiff type=Byte createopt=COMPRESS=DEFLATE,PREDICTOR=2 nodata=255
    r.mapcalc --overwrite expression=OCS_cat131819_null = OCS_cat131819@PERMANENT == 1 ? 1 : null()
    r.clump -d input=OCS_cat131819_null@PERMANENT output=OCS_cat131819_clump
    r.stats -a -c input=grid_rast@PERMANENT,OCS_cat131819_clump@PERMANENT output=C:\Support\Leandro\FR\stats_completes.txt separator=tab null_value=NA
```

## Script R pour agréger les données 

```R
    # Effective Mesh Size (CBC)

    library(vroom)
    library(dplyr)

    # pour surface en ha
    divisurf <- 10000

    nom_colonnes <- c("id", "id_patch", "surf_m2", "nb_pixels")
    tbl_stats_grass <- vroom("FR/stats_completes.txt", delim = "\t", col_names= nom_colonnes, col_types = "iidi")

    # surface en a, ou ha, ou km2 (selon divisurf)
    tbl_stats_grass <- mutate(tbl_stats_grass, a = surf_m2 / divisurf)
    tbl_stats_grass

    # calculer tableau avec surf complete des patchs
    # remarque : enlever id_patch=NA (pixels hors patches) avant de calculer sum 
    tbl_patch_cmpl <- tbl_stats_grass %>% 
    filter(!is.na(id_patch)) %>%
    group_by(id_patch) %>% 
    summarise(a_cmpl = sum(a), nb_cells = n())
    tbl_patch_cmpl

    # calculer tableau avec surf totale des mailles (patch et non patch)
    # remarque : enlever id=NA (pixels hors de la grille) 
    tbl_msize  <- tbl_stats_grass %>%
    filter(!is.na(id)) %>%
    group_by(id) %>% 
    summarise(aT = sum(a))
    tbl_msize

    # jointure aI-acpmlI + calcul produit
    tbl_stats_2 <- tbl_stats_grass %>% 
    filter(!is.na(id) & !is.na(id_patch)) %>%
    left_join(tbl_patch_cmpl, by="id_patch")
    tbl_stats_2 <-  mutate(tbl_stats_2, a2=a*a_cmpl)
    tbl_stats_2

    # calcule sommpe de produits (aires interieur / complet)
    tbl_stats_grid <- tbl_stats_2 %>% group_by(id) %>% 
    summarise(sum_a2 = sum(a2, na.rm = TRUE), 
                patches=n(), sum_aI = sum(a))
    tbl_stats_grid

    # calculer eff mesh size
    tbl_msize_2 <- left_join(tbl_msize, tbl_stats_grid, by="id")
    tbl_msize_2 <- mutate(
    tbl_msize_2, sum_a2=coalesce(sum_a2, 0), 
    patches=coalesce(patches, 0), sum_aI=coalesce(sum_aI, 0))
    tbl_msize_2 <- mutate(tbl_msize_2, cbc_msize=sum_a2/aT, divisurf)
    tbl_msize_2

    vroom_write(tbl_msize_2, "effmeshsize_cbc.txt")

    library(sf)
    sf_grid <- st_read("FR/grid-L93.shp")
    sf_grid_2 <- left_join(sf_grid, tbl_msize_2)
    st_write(sf_grid_2, "FR/grid_cbc_msize.shp", delete_dsn=TRUE)

```

## Un module Python pour GRASS GIS

Voir le GIST : <https://gist.github.com/cybernar/148022c2b7e9e6b29d7ff11a6d83a527>