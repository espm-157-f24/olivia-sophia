---
jupytext:
  formats: md:myst,ipynb
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.16.4
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
---

## Unit 3: Spatial Data

### Learning objectives
This module provides an introduction to the fundamentals of working with spatial vector and raster data in python while empirically exploring why systematic and structural racism is interwined with urban ecological processes.

+++

### Background 

In August 2020, [Christopher Schell](http://directory.tacoma.uw.edu/employee/cjschell) and collegues published a review in *Science* on ['The ecological and evolutionary consequences of systemic racism in urban environments'](https://science.sciencemag.org/content/early/2020/08/12/science.aay4497) (DOI: 10.1126/science.aay4497), showing how systematic racism and classism  has significant impacts on ecological and evolutionary processes within urban environments. Here we explore a subset of the data used to support these findings in this review and the broader literature.

+++

The [press release](https://www.washington.edu/news/2020/08/13/systemic-racism-has-consequences-for-all-life-in-cities/) on the paper is worth a read:

> “Racism is destroying our planet, and how we treat each other is essentially structural violence against our natural world,” said lead author Christopher Schell, an assistant professor of urban ecology at the University of Washington Tacoma. “Rather than just changing the conversation about how we treat each other, this paper will hopefully change the conversation about how we treat the natural world.”

In the paper, Schell writes: 

 > "In multiple cases, neighborhood racial composition can be a stronger predictor of urban socio-ecological patterns than wealth."

We are going to explore one metric for how structural racism and classism underpin landscape heterogeneity in cities.

+++

**Figure 2** in the Schell paper shows how NDVI (Normalized Difference Vegetation Index) tracks historical redlining.

```{image} attachment:23647e95-a23c-4eb8-a9bb-f0c91e45c1ac.png
:width: 300px
:align: center
```

+++

We are going to recreate some of these city maps, and plot the distributions and mean vegetation patterns across cities to explore the structural inequality and racism that Schell et al highlight in their paper.


To do this we are going to use the following spatial data:  

**1. Mapping Inequality:** (vector data)  
Please take the time to read the introduction to this dataset [here](https://dsl.richmond.edu/panorama/redlining/#loc=3/41.245/-105.469&text=intro)

**2.  Satellite Imagery from Sentinel-2 catalog:** (raster data), from [Planetary Computer](https://planetarycomputer.microsoft.com/dataset/sentinel-2-l2a)

```{code-cell} ipython3
from IPython.display import IFrame
```

```{code-cell} ipython3
import ibis
from ibis import _
con = ibis.duckdb.connect(extensions=["spatial"])
```

```{code-cell} ipython3
# slow
redlines = con.read_geo("/vsicurl/https://dsl.richmond.edu/panorama/redlining/static/mappinginequality.gpkg")
redlines.to_parquet("redlines.parquet")
```

```{code-cell} ipython3
redlines_local = con.read_parquet("redlines.parquet")
```

```{code-cell} ipython3
redlines.columns
```

```{code-cell} ipython3
redlines.select(_.city).distinct().head(10).execute() 
```

```{code-cell} ipython3
city = (redlines_local
        .filter(_.city == "New Haven")
        .execute()
       )
```

```{code-cell} ipython3
city = (con
    .read_geo("/vsicurl/https://dsl.richmond.edu/panorama/redlining/static/mappinginequality.gpkg")
    .filter(_.city == "New Haven")
    .execute()
)
```

```{code-cell} ipython3
city = redlines.filter(_.city == "New Haven")
```

```{code-cell} ipython3
city_gdf = city.head().execute()
city_gdf
```

```{code-cell} ipython3
city_gdf.geom[3]
```

```{code-cell} ipython3
str(city_gdf.geom[3])
```

```{code-cell} ipython3
city = (redlines
        .filter(_.city == "New Haven")
        .mutate(area = _.geom.area())
       )
```

```{code-cell} ipython3
import ibis
from ibis import _
con = ibis.duckdb.connect()

city = (con
    .read_geo("/vsicurl/https://dsl.richmond.edu/panorama/redlining/static/mappinginequality.gpkg")
    .filter(_.city == "New Haven")
    .execute()
)
```

```{code-cell} ipython3
import leafmap.maplibregl as leafmap

m = leafmap.Map(style="positron")
m.add_gdf(city)
m
```

```{code-cell} ipython3
m.to_html("docs/new_haven.html", overwrite = True)

# display map on course website by using an iframe to the output URL 
IFrame(src='https://espm-157.github.io/static-maps/nh1.html', width=700, height=400)
```

```{code-cell} ipython3
import ibis
from ibis import _
con = ibis.duckdb.connect(extensions=["spatial"])

redlines = (
    con
    .read_geo("/vsicurl/https://dsl.richmond.edu/panorama/redlining/static/mappinginequality.gpkg")
    .filter(_.city == "New Haven", _.residential)
   )
city =  redlines.execute()
box = city.total_bounds
box
```

```{code-cell} ipython3
from pystac_client import Client
import odc.stac
import rioxarray

 
items = (Client.open("https://earth-search.aws.element84.com/v1").search(collections = ['sentinel-2-l2a'], bbox = box, datetime = "2022-06-01/2022-08-01", query={"eo:cloud_cover": {"lt": 20}}).item_collection())
```

```{code-cell} ipython3
data = odc.stac.load(
    items,
    bands=["nir08", "red"],
    bbox=box
)
```

```{code-cell} ipython3
ndvi = (
    (data.nir08 - data.red) / (data.nir08 + data.red)
    .compute()
)
```

```{code-cell} ipython3
import matplotlib as plt
cmap = plt.colormaps.get_cmap('viridis')  # viridis is the default colormap for imshow
cmap.set_bad(color='black')
ndvi.plot.imshow(row="time", cmap=cmap, add_colorbar=False, size=4)
```

```{code-cell} ipython3
from pystac_client import Client
```

```{code-cell} ipython3
items = (
  Client.
  open("https://earth-search.aws.element84.com/v1").
  search(
    collections = ['sentinel-2-l2a'],
    bbox = box,
    datetime = "2024-06-01/2024-09-01",
    query={"eo:cloud_cover": {"lt": 20}}).
  item_collection()
)
```

```{code-cell} ipython3
items
```

```{code-cell} ipython3
import odc.stac
```

```{code-cell} ipython3
data = odc.stac.load(
    items,
    bands=["nir08", "red"],
    bbox=box,
    resolution=10, # the native resolution is already 10m.  Increase this to ~ 100m for larger cities.
    groupby="solar_day",
    chunks = {} # this tells odc to use dask
    
)

data
```

```{code-cell} ipython3
ndvi = (
    ((data.nir08 - data.red) / (data.red + data.nir08))
    .median("time", keep_attrs=True)
)

ndvi = ndvi.where(ndvi < 1).compute()

# ndvi calculation  then compute the time. this is used in the ndvi.tif file
```

```{code-cell} ipython3
ndvi
```

```{code-cell} ipython3
ndvi.plot.imshow()
```

```{code-cell} ipython3
import rioxarray
(ndvi
 .rio.reproject("EPSG:4326")
 .rio.to_raster(raster_path="ndvi.tif", 
                driver="COG")   
)
```

```{code-cell} ipython3
# city.to_file("new_haven.shp") # common legacy format, not ideal
# city.to_file("new_haven.gpkg") # Good open standard

# latest option, best performance but less widely known:
city.to_parquet("new_haven.parquet")
```

```{code-cell} ipython3
city = city.set_crs("EPSG:4326")
```

```{code-cell} ipython3
import exactextract

stats = exactextract.exact_extract("ndvi.tif", 
                                   city, 
                                   "mean", 
                                   output = "pandas", 
                                   include_cols = ["category", "label", "fill"],
                                   include_geom = True)
stats.to_parquet("new_haven_stats.parquet")

# show me that on average A values are higher than B values: show on the map or agragate statistics (bar chart or whatever)
```

```{code-cell} ipython3
con.read_parquet("new_haven_stats.parquet").execute()
```

```{code-cell} ipython3
city
```

```{code-cell} ipython3
(city
 .set_crs("EPSG:4326")
 .to_file("new_haven.gpkg") # common open standard
)


# latest option, best performance but less widely known:
city.set_crs("EPSG:4326").to_parquet("new_haven.parquet")
```

```{code-cell} ipython3
new_haven_stats =  con.read_parquet("new_haven.parquet").execute()
```

```{code-cell} ipython3
import leafmap.maplibregl as leafmap
m = leafmap.Map()
m.add_cog_layer("https://espm-157-f24.github.io/olivia-sophia/ndvi.tif", palette = "greens")
m.add_gdf(new_haven_stats)
m.add_layer_control()
m
```

```{code-cell} ipython3
from exactextract import exact_extract
import ibis
from ibis import _
```

```{code-cell} ipython3
con = ibis.duckdb.connect(extensions=["spatial"])

redlines = (
    con
    .read_geo("/vsicurl/https://dsl.richmond.edu/panorama/redlining/static/mappinginequality.gpkg")
    .filter(_.city == "New Haven", _.residential)
   )
```

```{code-cell} ipython3
city =  redlines.execute().set_crs("EPSG:4326")
```

```{code-cell} ipython3
city_stats = exact_extract("ndvi.tif", 
                           city, 
                           ["mean"], 
                           include_geom = True,
                           include_cols=["label", "grade", "city", "fill"],
                           output="pandas")

city_stats.head()
```

```{code-cell} ipython3
city_stats.to_parquet("new_haven_stats.parquet")
```

```{code-cell} ipython3
# construct the rest of the ibis code to compute the average NDVI by grade
# use groupby and aggregate, create table NDVI for all sections

city = con.read_parquet("new_haven_stats.parquet")

# Compute the average NDVI by grade
ndvi_by_grade = (
    city
    .group_by("grade")
    .aggregate(mean_ndvi = city["mean"].mean())
    .order_by("grade")
)

# Display the results
ndvi_by_grade.execute()
```

```{code-cell} ipython3

```
