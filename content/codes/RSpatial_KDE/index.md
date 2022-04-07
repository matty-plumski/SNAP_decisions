---
title: Kernel density estimation in R spatial
description:
  Here's one way to do kernel density estimation in R spatial
# View.
#   1 = List
#   2 = Compact
#   3 = Card
view: 2

# Optional header image (relative to `static/media/` folder).
header:
  caption: ""
  image: ""

type: book

tags:
- R
- Geography
- Geospatial Analysis

authors:
- DavidOSullivan
---
---
title: "Kernel density estimation in R spatial"
description: |
  Here's one way to do kernel density estimation in R spatial
date: 10-21-2021
output:
  distill::distill_article
---

## Packages
This requires a surprising number of moving parts (at least the way I did it):

```{r message=FALSE}
library(sf)
library(tmap)
library(spatstat)
library(maptools)
library(raster)
library(dplyr)
```

## Data
The data are some [point data](abb.gpkg?raw=true) (Airbnb listings from [here](http://insideairbnb.com/new-zealand/)) and some [polygon data](sa2.gpkg?raw=true) (NZ census Statistical Area 2 data).

### Load the data

```{r message = FALSE}
polys <- st_read("sa2.gpkg")
pts <- st_read("abb.gpkg")
```

And have a look

```{r}
tm_shape(polys) +
  tm_polygons() + 
  tm_shape(pts) + 
  tm_dots()
```
![](https://i.imgur.com/oatM2bY.png)

## `spatstat` for density estimation
The best way I know to do density estimation in the _R_ ecosystem is using the [`spatstat`](https://spatstat.org/) library's specialisation of base _R_'s `density` function. That means converting the point data to a `spatstat` planar point pattern (`ppp`) object, which involves a couple of steps.

```{r}
pts.ppp <- pts$geom %>% 
  as("Spatial") %>% # we need to convert to sp as a bridge to spatstat
  as.ppp()          # and this is what we need maptools for...
```

A point pattern also needs a 'window', which we'll make from the polygons.

```{r}
pts.ppp$window <- polys %>%
  st_union() %>%       # combine all the polygons into a single shape
  as("Spatial") %>%    # convert to sp
  as.owin()            # convert to spatstat owin - again maptools...
```

### Now the kernel density
We need some bounding box info to manage the density estimation resolution

```{r}
bb <- st_bbox(polys)
cellsize <- 100
height <- (bb$ymax - bb$ymin) / cellsize
width <- (bb$xmax - bb$xmin) / cellsize
```

Now we specify the size of the raster we want with `dimyx` (note the order, y then x) using `height` and `width`. 

We can convert this directly to a raster, but have to supply a CRS which we pull from the original points input dataset. At the time of writing (August 2021) you'll get a complaint about the New Zealand Geodetic Datum 2000 because recent changes in how projections and datums are handled are still working themselves out.

```{r}
kde <- density(pts.ppp, sigma = 500, dimyx = c(height, width)) %>%
  raster(crs = st_crs(pts)$wkt) # a ppp has no CRS information
```

### Let's see what we got
We can map this using `tmap`.

```{r}
tm_shape(kde) +
  tm_raster(palette =  "Reds")
```

![](https://i.imgur.com/PiaGn3N.png)


### A fallback sanity check
To give us an alternative view of the data, let's just count points in polygons

```{r}
polys$n <- polys %>%
  st_contains(pts) %>%
  lengths()
```

And map the result

```{r}
tm_shape(polys) +
  tm_polygons(col = "n", palette = "Reds", title = "Points in polygons")
```
![](https://i.imgur.com/KNG7OW5.png)

### Aggregate the density surface pixels to polygons
This isn't at all necessary, but is also useful to know. This is also a relatively slow operation. Note that we add together the density estimates in the pixels contained by each polygon.

```{r}
summed_densities <- raster::extract(kde, polys, fun = sum)
```

Append this to the polygons and rescale so the result is an estimate of the original count. We multiply by `cellsize^2` because each cell contains an estimate of the per sq metre (in this case, but per sq distance unit in general) density, so multiplying by the area of the cells gives an estimated count.

```{r}
polys$estimated_count = summed_densities[, 1] * cellsize ^ 2
```

And now we can make another map

```{r}
tm_shape(polys) + 
  tm_polygons(col = "estimated_count", palette = "Reds", title = "500m KDE estimate")
```

![](https://i.imgur.com/sGYTwOw.png)


Clearly, this approach doesn't work very well with areas of dramatically different sizes which accounts for why the rural but extensive area to the west is badly overestimated.