---
layout: post
title: "Vector spatial data in R, the basics with sp"
date: 2016-06-14
---

## Some GPS coordinates

We start from a `data.frame` object with 5 rows and 4 columns. The numeric vector lon and lat give the GPS coordinates of 5 points, in decimal degrees (WGS84) .

```
library(sp)
lon <- c(3.86379, 3.86291, 3.86243, 3.86220, 3.86314)
lat <- c(43.63838, 43.63878, 43.63863, 43.63821, 43.63810)
name <- c("AA", "BB", "CC", "DD", "EE")
color <- c("green", "green", "green", "blue", "blue")
df <- data.frame(name, lon, lat, color)
```

## The SpatialPoints class

SpatialPoints class is a data structure to store points : only the "spatial" part, not the "attributes" part.
To build a SpatialPoints object, we just need :

- a 2-columns matrix (with XY coordinates, or "longitude latitude" if you prefer)
- if possible, a CRS object made of the **proj4 definition** of the coordinate reference system. We found the WGS84 on epsg.io website, from this URL : <http://epsg.io/4326>.

```
matcoords <- as.matrix(df[,c("lon","lat")])
spts <- SpatialPoints(matcoords, proj4string = CRS("+proj=longlat +datum=WGS84 +no_defs"))
# the following proj4string definition with EPSG ID is equivalent to the explicit definition ...
spts <- SpatialPoints(matcoords, proj4string = CRS("+init=EPSG:4326"))
slotNames(spts)
```
