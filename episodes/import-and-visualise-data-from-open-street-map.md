---
title: 'Import vector data from Open Street Map'
teaching: 10
exercises: 2
---

:::::::::::::::::::::::::::::::::::::: questions 

- How to import and work with vector data from Open Street Map?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Import OSM vector data from the API
- Select and manipulate OSM vector data
- Visualise and map OSM Vector data
- Use Leaflet for interactive mapping

::::::::::::::::::::::::::::::::::::::::::::::::

## Introduction: What is Open Street Map?

Open Street Map (OSM) is a collaborative project which aims at mapping the world and sharing geospatial data in an open way. Anyone can contribute, by mapping geographical objects their encounter, by adding topical information on existing map objects (their name, function, capacity, etc.), or by mapping buildings and roads from satellite imagery (cf. [HOT: Humanitarian OpenStreetMap Team](https://www.hotosm.org/)).

This information is then validated by other users and eventually added to the common "map" or information system. This ensures that the information is accessible, open, verified, accurate and up-to-date.

The result looks like this:
![View of OSM web interface](episodes/fig/OSM1.png)

The geospatial data underlying this interface is made of geometrical objects (i.e. points, lines, polygons) and their associated tags (#building #height, #road #secondary #90kph, etc.).

## How to extract geospatial data from Open Street Map?


### Bounding-box

The first thing to do is to define the area within which you want to retrieve data, aka the *bounding box*. This can be defined easily using a place name and the package `nominatimlite` to access the free Nominatim API provided by OpenStreetMap. 

We are going to look at *Brielle* together, but you can also work with the small cities of *Naarden*, *Geertruidenberg*, *Gorinchem*, *Enkhuizen* or *Dokkum*.

/!\ This might not be needed, but ensures that your machine knows it has internet...

```{r osm-curl-internet}
assign("has_internet_via_proxy", TRUE, environment(curl::has_internet))
```


```{r osm-bounding-box}
nominatim_polygon <- nominatimlite::geo_lite_sf(address = "Brielle", points_only = FALSE)
bb <- sf::st_bbox(nominatim_polygon)
bb
```
- Word of caution

There might multiple responses from the API query, corresponding to different objects at the same location, or different objects at different locations.
For example: Brielle (Netherlands) and Brielle (New Jersey)

![Brielle, Netherlands](episodes/fig/Brielle_NL.jpeg){width=40%}

![Brielle, New Jersey](episodes/fig/Brielle_NJ.jpeg "Brielle, New Jersey"){width=40%}


We should therefore try to be as unambiguous as possible by adding a country code or district name.

```{r osm-bounding-box2}
nominatim_polygon <- nominatimlite::geo_lite_sf(address = "Brielle, NL", points_only = FALSE)
bb <- sf::st_bbox(nominatim_polygon)
bb

```


### Extracting features

A [feature](https://wiki.openstreetmap.org/wiki/Map_features) in the OSM language is a category or tag of a geospatial object. Features are described by general keys (e.g. "building", "boundary", "landuse", "highway"), themselves decomposed into sub-categories (values) such as "farm", "hotel" or "house" for `buildings`, "motorway", "secondary" and "residential" for `highway`. This determines how they are represented on the map.

### Searching documentation

Let's say we want to download data from OpenStreetMap and we know there is a package for it named `osmdata`, but we don't know which function to use and what arguments are needed. Where should we start?

> Let's check the documentation [online](https://docs.ropensci.org/osmdata/):

![The OSMdata Documentation page](episodes/fig/osmdata.png){width=80%}

It appears that there is a function to extract features, using the Overpass API. This function's name is `opq` (for OverPassQuery) which, in combination with `add_osm_feature`, seems to do the job. However it might not be crystal clear how  to apply it to our case. Let's click on the function name to know more.

![The Overpass Query Documentation page](episodes/fig/opq.png){width=80%}



On this page we can read about the arguments needed for each function: a bounding box for `opq()` and some `key` and `value` for `add_osm_feature()`. Thanks to the examples provided, we can assume that these keys and values correspond to different levels of tags from the OSM classification. In our case, we will keep it at the first level of classification, with "buildings" as `key`, and no value. We also see from the examples that another function is needed when working with the `sf` package: `osmdata_sf()`. This ensures that the type of object is suited for `sf`. With these tips and examples, we can write our feature extraction function as follows:

```{r osm-feature}
x <- opq(bbox = bb) %>%
   add_osm_feature(key = 'building') %>%
    osmdata_sf()
```

What is this x object made of? It is a table of all the buildings contained in the bounding box, which gives us their OSM id, their geometry and a range of attributes, such as their name, building material, building date, etc. The completion level of this table depends on user contributions and open resources (here for instance: BAG, different in other countries).

```{r osm-feature-preview}
head(x$osm_polygons)

```

## Mapping 


Let's map the building age of post-1900 Brielle buildings.


### Projections

First, we are going to select the polygons and reproject them with the Amersfoort/RD New projection, suited for maps centred on the Netherlands. This code for this projection is: 28992.

```{r reproject}
buildings <- x$osm_polygons %>%
  st_transform(.,crs=28992)
```

### Visualisation

Then we create a variable which a threshold at 1900. Every date prior to 1900 will be recoded 1900, so that buildings older than 1900 will be represented with the same shade.

Then we use the `ggplot` function to visualise the buildings by age. The specific function to represent information as a map is `geom_sf()`. The rest works like other graphs and visualisation, with `aes()` for the aesthetics.

```{r map-age}
buildings$build_date <- as.numeric(
  if_else(
    as.numeric(buildings$start_date) < 1900, 
          1900, 
          as.numeric(buildings$start_date)
          )
  )

 ggplot(data = buildings) +
   geom_sf(aes(fill = build_date, colour=build_date))  +
   scale_fill_viridis_c(option = "viridis")+
   scale_colour_viridis_c(option = "viridis")
```

So this reveals the historical centre of [city] and the various extensions.
Anything odd? what? around the centre? Why these limits / isolated points?



::::::::::::::::::::::::::::::::::::: challenge 

## Challenge: import an interactive basemap layer under the buildings with `Leaflet` (20min)

- Check out the [leaflet package documentation](https://rstudio.github.io/leaflet/)
- Plot a basemap in Leaflet and try different tiles. [Basemap documentation](https://rstudio.github.io/leaflet/basemaps.html)
- Transform the buildings into WGS84 projection and add them to the basemap layer with the `addPolygons` function.
- Have the `fillColor` of these polygons represent the `build_date` variable. [Choropleth documentation](https://rstudio.github.io/leaflet/choropleths.html). Using the example and replace the variable names where needed.


:::::::::::::::::::::::: solution 

## One solution
 
```{r, echo=FALSE}
buildings2 <- buildings %>%
  st_transform(.,crs=4326)

leaflet(buildings2) %>%
# addTiles()
  addProviderTiles(providers$CartoDB.Positron) %>%
  addPolygons(color = "#444444", weight = 0.1, smoothFactor = 0.5,
  opacity = 0.2, fillOpacity = 0.8,
  fillColor = ~colorQuantile("YlGnBu", -build_date)(-build_date),
  highlightOptions = highlightOptions(color = "white", weight = 2,
    bringToFront = TRUE))

```

:::::::::::::::::::::::::::::::::



:::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::: instructor

Inline instructor notes can help inform instructors of timing challenges
associated with the lessons. They appear in the "Instructor View"

::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::

## Figures

You can use pandoc markdown for static figures with the following syntax:

`![optional caption that appears below the figure](figure url){alt='alt text for
accessibility purposes'}`

![You belong in The Carpentries!](https://raw.githubusercontent.com/carpentries/logo/master/Badge_Carpentries.svg){alt='Blue Carpentries hex person logo with no text.'}

::::::::::::::::::::::::::::::::::::: keypoints 

- Use `.md` files for episodes when you want static content
- Use `.Rmd` files for episodes when you need to generate output
- Run `sandpaper::check_lesson()` to identify any issues with your lesson
- Run `sandpaper::build_lesson()` to preview your lesson locally

::::::::::::::::::::::::::::::::::::::::::::::::
