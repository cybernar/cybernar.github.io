# Comment télécharger et insérer des données OpenStreetMap dans une base PostGIS

## Téléchargement des données OpenStreetMap

_Exemple avec les données de Guadeloupe :_

- L'autre manière est de télécharger les données de Guadeloupe au format compressé `.osm.bz2` sur le site de **geofabrik** : <http://download.geofabrik.de/> puis Europe > France > Guadeloupe
- Une autre possibilité serait de télécharger le secteur recherché au format `.osm` avec le logiciel **JOSM**, téléchargeable ici : <https://josm.openstreetmap.de/wiki/Fr:WikiStart>

## Création d'une base de données PostGIS

Appelons-la *bd_spatial*.

``` shell
createdb gis
psql -d gis -c 'CREATE EXTENSION postgis; CREATE EXTENSION hstore;'
```

## Chargement des données dans PostGIS

Nous chargeons *toutes* les données OpenStreetMap (noeuds, lignes, relations) dans la base *bd_spatial*

``` shell
osm2psql --slim --username postgres --password --database bd_spatial D:\\Travail\\Support\\guyane-latest.osm.pbf
```

## Création d'une nouvelle table à partir du tag place=island

D'abord on commence par créer une nouvelle table plus légère : elle ne contiendra que le contour de l'île (filtrage avec le tag `place=island`). On transforme la géométrie dans le système de coordonnées en vigueur <https://epsg.io/5490>.

``` sql
CREATE TABLE public.data_island AS
SELECT osm_id, name AS name_, ST_Transform(way, 5490)::geometry(Polygon,5490) as geom
FROM planet_osm_polygon
WHERE place='island';
-- ajoute colonne fid (clé primaire)
ALTER TABLE public.data_island
ADD COLUMN fid serial,
ADD CONSTRAINT data_island_pkey PRIMARY KEY (fid);
-- crée index spatial
CREATE INDEX data_island_osm_geom_idx
ON public.data_island USING gist (geom);
```

## Calcul de la distance entre 2 entités avec ST_Distance

La fonction `ST_Distance` retourne la distance minimale entre 2 entités. Dans le cas de 2 polygones il s'agit donc de la distance "bord à bord". Voir <https://postgis.net/docs/manual-2.5/ST_Distance.html>

Calculons la distance entre l'Île de Marie-Galante et ses voisines :

``` sql
WITH mg AS (
	SELECT fid, name_ AS src, geom FROM data_island 
	WHERE name_='Marie-Galante'
), other AS (
	SELECT fid AS fid_dest, name_ AS dest, geom FROM data_island 
	WHERE name_!='Marie-Galante'
) 
SELECT src, dest, fid_dest, round(ST_Distance(mg.geom, other.geom)) AS dist_m
FROM mg, other;
```

src|dest|fid_dest|dist_m
-|-|-|-
Marie-Galante|Grande-Terre|1|25977
Marie-Galante|Terre-de-Haut|2|24655
Marie-Galante|La Désirade|3|36562
Marie-Galante|Terre-de-Bas|4|30680
Marie-Galante|Grand Îlet|5|27975
Marie-Galante|Terre-de-Haut|6|25345
Marie-Galante|Basse-Terre|7|26379
Marie-Galante|Terre-de-Bas|9|23106
Marie-Galante|Le Rocher de l'Éperon|10|29197

Comment déterminer, pour chaque île, sa plus proche voisine ? Il faut calculer la distance entre chaque île et ses voisines, puis recherche la distance minimale. Dans notre cas, la requête calculera la distance entre 90 paires de polygones. **Le temps de calcul de ce type de requête peut rapidement augmenter** en fonction du nombre et de la complexité des polygones !

``` sql
WITH all_distances AS (
	SELECT t1.fid AS fid_src, t2.fid AS fid_dst, 
		t1.name_ AS name_src, t2.name_ AS name_dst,
		ST_Distance(t1.geom, t2.geom) AS dist_m
	FROM data_island t1, data_island t2
	WHERE t1.fid!=t2.fid
), sorted_distances AS (
	SELECT fid_src, fid_dst, name_src, name_dst, dist_m,
		rank() OVER (PARTITION BY fid_src ORDER BY dist_m) 
	FROM all_distances
)
SELECT fid_src, fid_dst, name_src, name_dst, round(dist_m) as dist_m
FROM sorted_distances
WHERE rank=1;
```
fid_src|fid_dst|name_src|name_dst|dist_m
-|-|-|-|-
1|7|Grande-Terre|Basse-Terre|58
2|9|Terre-de-Haut|Terre-de-Bas|161
3|1|La Désirade|Grande-Terre|9371
4|6|Terre-de-Bas|Terre-de-Haut|850
5|6|Grand Îlet|Terre-de-Haut|1071
6|4|Terre-de-Haut|Terre-de-Bas|850
7|1|Basse-Terre|Grande-Terre|58
8|9|Marie-Galante|Terre-de-Bas|23106
9|2|Terre-de-Bas|Terre-de-Haut|161
10|1|Le Rocher de l'Éperon|Grande-Terre|570

## Matérialiser la ligne la plus courte entre 2 entités avec ST_ShortestLine

La fonction `ST_ShortestLine` renvoie la ligne de plus courte distance entre 2 polygones ou autres types d'entité. Voir <https://postgis.net/docs/manual-2.5/ST_ShortestLine.html>.

Créons une nouvelle table dans laquelle nous matérialisons la ligne la plus directe entre chaque île et ses voisines.

```sql
CREATE TABLE lines_island AS
SELECT t1.fid AS fid_src, t2.fid AS fid_dst, 
    t1.name_ AS name_src, t2.name_ AS name_dst,
    ST_ShortestLine(t1.geom, t2.geom)::geometry(LineString,5490) AS geom
FROM data_island t1, data_island t2
WHERE t1.fid!=t2.fid;
-- ajoute colonne longueur
ALTER TABLE lines_island
    ADD COLUMN long_m longint,
    ADD COLUMN fid serial;
UPDATE lines_island SET long_m = ST_Length(geom);
```

Le résultat peut maintenant être visualisé dans QGIS :

![Guadeloupe ShortLines](/images/postgis_distances.png)
