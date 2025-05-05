---
title: 'GORUCK City Ruck: DC'
subtitle: Data, AI, and touching grass
date: 2025-05-06T00:30:00.000Z
tags:
  - ai
format: hugo-md
draft: false
lightbox: true
---


<link href="index_files/libs/htmltools-fill-0.5.8.1/fill.css" rel="stylesheet" />
<script src="index_files/libs/htmlwidgets-1.6.4/htmlwidgets.js"></script>
<script src="index_files/libs/jquery-3.6.0/jquery-3.6.0.min.js"></script>
<link href="index_files/libs/leaflet-1.3.1/leaflet.css" rel="stylesheet" />
<script src="index_files/libs/leaflet-1.3.1/leaflet.js"></script>
<link href="index_files/libs/leafletfix-1.0.0/leafletfix.css" rel="stylesheet" />
<script src="index_files/libs/proj4-2.6.2/proj4.min.js"></script>
<script src="index_files/libs/Proj4Leaflet-1.0.1/proj4leaflet.js"></script>
<link href="index_files/libs/rstudio_leaflet-1.3.1/rstudio_leaflet.css" rel="stylesheet" />
<script src="index_files/libs/leaflet-binding-2.2.2/leaflet.js"></script>


Over the weekend, I did the GORUCK 12 Mile City Ruck in DC.

<div class="strava-embed-placeholder" data-embed-type="activity" data-embed-id="14367953091" data-style="standard" data-from-embed="false"></div><script src="https://strava-embeds.com/embed.js"></script>

It was my first one so I didn't know anything about the route ahead of time, nor did anyone on my team.
At check-in, a list of waypoints was taped to a table.

<img src="../../..\img/goruck-dc/waypoints-1.jpg" data-group="waypoints" alt="waypoints-1" />
<img src="../../..\img/goruck-dc/waypoints-2.jpg" data-group="waypoints" alt="waypoints-2" />
<img src="../../..\img/goruck-dc/waypoints-3.jpg" data-group="waypoints" alt="waypoints-3" />
<img src="../../..\img/goruck-dc/waypoints-4.jpg" data-group="waypoints" alt="waypoints-4" />

During the briefing, we learned that you are allowed to visit the waypoints in any order.
The order they were listed in seemed like a recipe for needless mileage, so this was a relief.

``` r
library(dplyr)
```


    Attaching package: 'dplyr'

    The following objects are masked from 'package:stats':

        filter, lag

    The following objects are masked from 'package:base':

        intersect, setdiff, setequal, union

``` r
library(purrr)
library(tidygeocoder)
library(magrittr, include.only = "%T>%")
library(leaflet)

waypoints <- readr::read_delim("waypoints.csv", delim = "|", trim_ws = TRUE) |>
    select(Location, Address)
```

    New names:
    • `` -> `...1`
    • `` -> `...4`

    Rows: 13 Columns: 4
    ── Column specification ────────────────────────────────────────────────────────
    Delimiter: "|"
    chr (2): Location, Address
    lgl (2): ...1, ...4

    ℹ Use `spec()` to retrieve the full column specification for this data.
    ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
geo_providers <- c("google", "mapbox", "osm", "census", "arcgis")
geocoded <-
    map(
        .x = geo_providers,
        ~ geocode(waypoints, Address, method = .x)
        ) |> 
    list_rbind(names_to = "provider") %T>%
    readr::write_csv("waypoints_geocoded.csv")
```

    Passing 13 addresses to the Google single address geocoder
    Query completed in: 0.8 seconds
    Passing 13 addresses to the Mapbox single address geocoder
    Query completed in: 1.4 seconds
    Passing 13 addresses to the Nominatim single address geocoder
    Query completed in: 13.3 seconds
    Passing 13 addresses to the US Census batch geocoder
    Query completed in: 2.9 seconds
    Passing 13 addresses to the ArcGIS single address geocoder
    Query completed in: 6.2 seconds

``` r
geocoded_waypoints <-
    readr::read_csv("waypoints_geocoded.csv")
```

    Rows: 65 Columns: 5
    ── Column specification ────────────────────────────────────────────────────────
    Delimiter: ","
    chr (2): Location, Address
    dbl (3): provider, lat, long

    ℹ Use `spec()` to retrieve the full column specification for this data.
    ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
geocoded_waypoints |> 
    filter(provider == 2) |> 
    leaflet() |> 
        addTiles() |> 
        addMarkers(~long, ~lat, popup = ~Location)
```

<div class="leaflet html-widget html-fill-item" id="htmlwidget-a31b02aed3d8de1b387d" style="width:768px;height:480px;"></div>
<script type="application/json" data-for="htmlwidget-a31b02aed3d8de1b387d">{"x":{"options":{"crs":{"crsClass":"L.CRS.EPSG3857","code":null,"proj4def":null,"projectedBounds":null,"options":{}}},"calls":[{"method":"addTiles","args":["https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",null,null,{"minZoom":0,"maxZoom":18,"tileSize":256,"subdomains":"abc","errorTileUrl":"","tms":false,"noWrap":false,"zoomOffset":0,"zoomReverse":false,"opacity":1,"zIndex":1,"detectRetina":false,"attribution":"&copy; <a href=\"https://openstreetmap.org/copyright/\">OpenStreetMap<\/a>,  <a href=\"https://opendatacommons.org/licenses/odbl/\">ODbL<\/a>"}]},{"method":"addMarkers","args":[[38.855565,38.892108,38.889822,38.887914,38.89872,38.889264,38.888273,38.89288,38.895962,38.891326,38.884977,38.881571,38.889472],[-77.052138,-77.04298,-77.00242900000001,-77.04785,-77.035692,-77.050157,-77.044376,-77.022961,-77.017861,-77.030731,-77.06262099999999,-77.036834,-77.03526599999999],null,null,null,{"interactive":true,"draggable":false,"keyboard":true,"title":"","alt":"","zIndexOffset":0,"opacity":1,"riseOnHover":false,"riseOffset":250},["Gravelly Point","56 Signers of the Declaration of Independence Memorial","Capitol Building","Korean War Veterans Memorial","Lafayette Square","Lincoln Memorial","Martin Luther King, Jr. National Memorial","National Archives","National Law Enforcement Officers Memorial","National Museum of American History","National Seabee Memorial","Thomas Jefferson Memorial","Washington Monument"],null,null,null,null,{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},null]}],"limits":{"lat":[38.855565,38.89872],"lng":[-77.06262099999999,-77.00242900000001]}},"evals":[],"jsHooks":[]}</script>

If you're familiar with DC geography, a few of these points will appear wrong to you in subtle (or glaring!) ways. Let's clean it up:

``` r
lol <- tribble(
    ~Location, ~lat, ~long,
    "Gravelly Point", 38.865537513788496, -77.03777076583343,
    "Capitol Building", 38.89007191031991, -77.00827228253716,
    "Martin Luther King, Jr. National Memorial", 38.88616691159861, -77.04424409367498
)

geocoded_waypoints |> 
    filter(provider == 2) |>
    rows_update(lol) |> 
    leaflet() |> 
        addTiles() |> 
        addMarkers(~long, ~lat, popup = ~Location)
```

    Matching, by = "Location"

<div class="leaflet html-widget html-fill-item" id="htmlwidget-da1f47f4685857fd75c9" style="width:768px;height:480px;"></div>
<script type="application/json" data-for="htmlwidget-da1f47f4685857fd75c9">{"x":{"options":{"crs":{"crsClass":"L.CRS.EPSG3857","code":null,"proj4def":null,"projectedBounds":null,"options":{}}},"calls":[{"method":"addTiles","args":["https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",null,null,{"minZoom":0,"maxZoom":18,"tileSize":256,"subdomains":"abc","errorTileUrl":"","tms":false,"noWrap":false,"zoomOffset":0,"zoomReverse":false,"opacity":1,"zIndex":1,"detectRetina":false,"attribution":"&copy; <a href=\"https://openstreetmap.org/copyright/\">OpenStreetMap<\/a>,  <a href=\"https://opendatacommons.org/licenses/odbl/\">ODbL<\/a>"}]},{"method":"addMarkers","args":[[38.8655375137885,38.892108,38.89007191031991,38.887914,38.89872,38.889264,38.88616691159861,38.89288,38.895962,38.891326,38.884977,38.881571,38.889472],[-77.03777076583343,-77.04298,-77.00827228253716,-77.04785,-77.035692,-77.050157,-77.04424409367498,-77.022961,-77.017861,-77.030731,-77.06262099999999,-77.036834,-77.03526599999999],null,null,null,{"interactive":true,"draggable":false,"keyboard":true,"title":"","alt":"","zIndexOffset":0,"opacity":1,"riseOnHover":false,"riseOffset":250},["Gravelly Point","56 Signers of the Declaration of Independence Memorial","Capitol Building","Korean War Veterans Memorial","Lafayette Square","Lincoln Memorial","Martin Luther King, Jr. National Memorial","National Archives","National Law Enforcement Officers Memorial","National Museum of American History","National Seabee Memorial","Thomas Jefferson Memorial","Washington Monument"],null,null,null,null,{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},null]}],"limits":{"lat":[38.8655375137885,38.89872],"lng":[-77.06262099999999,-77.00827228253716]}},"evals":[],"jsHooks":[]}</script>

Better enough.

If you've never heard of rucking, it's walking around with a weighted backpack.
I needed to carry 20 pounds of weight and 3 liters of water--about 26 pounds.
While the suffering associated with marching this weight around was entirely voluntary, I nevertheless believed it was in my interest to hit all the relevant waypoints while covering as little distance as possible.

Classical:
I started entering points in Google maps, but it turns out that it won't let you navigate to more than 8 destinations.[^1]

My prompt:
\> Plan the optimal route between the waypoints. The course starts and ends at gravelly point

Gemini 2.5 flash https://g.co/gemini/share/fbe588880d33
Gemini 2.5 Pro https://g.co/gemini/share/6f8eec719ff4
Claude 3.7 Sonnet

Someone else in the party said he had looked it up with chatgpt at the time so I didn't use that. I tried to do it as part of drafting this post but apparently I exhausted my upload limit on the free tier.
(I have Gemini pro because my phone folds in half.)

[^1]: A limitation it turns out Gemini is not aware of.
