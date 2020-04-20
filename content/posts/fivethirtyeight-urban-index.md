---
title: "the fivethirtyeight urban index"
date: 2020-04-19T23:10:00-04:00
draft: false
---

## tl;dr

I have attempted to recreate the fivethirtyeight urban index. [Here's the repo.](https://github.com/edavidaja/urbanindex)

## fivethirtyeight have a new urban index

fivethirtyeight have come up with a method for [quantifying urban or rural-ness.](https://fivethirtyeight.com/features/how-urban-or-rural-is-your-state-and-what-does-that-mean-for-the-2020-election/)
The data, quite helpfully, [are posted on GitHub](https://github.com/fivethirtyeight/data/tree/master/urbanization-index).
Less helpfully, [(and unlike one of the other sources they mention)](https://github.com/theatlantic/citylab-data/blob/master/citylab-congress/district_class.R) they don't show how their results are derived, so I thought I'd spend some time over the weekend reproducing the calculation.

## the process

1. get data for every census tract in the US
1. figure out which tracts are within five miles of each other
1. add up the population for tracts within five miles of each other

[Geocomputation with R](https://geocompr.robinlovelace.net/) by Robin Lovelace, Jakub Nowosad, and Jannes Muenchow is very helpful for getting up to speed on the fundamentals of analyzing geospatial data, of which I know precious little.

### getting the data

[{tidycensus}](https://walkerke.github.io/tidycensus/) by Kyle Walker makes pulling census data into R incredibly easy. 

Once you have the data, [{sf}](https://r-spatial.github.io/sf/) makes it easy to do spatial calculations. Judging from my [many](https://gis.stackexchange.com/questions/41956/summing-attributes-intersected-by-under-a-radius/41958#41958)
[searches](https://gis.stackexchange.com/questions/16637/how-to-use-buffers-to-estimate-the-attribute-data-for-a-given-polygon)
[through](https://gis.stackexchange.com/questions/16358/algorithm-for-finding-population-for-a-given-center-point-and-radius-in-us)
[stackoverflow](https://gis.stackexchange.com/questions/305888/calculating-population-within-buffer-in-qgis), it hasn't always been this easy.

### getting it wrong

> How many people live within 5 miles of you? If you calculate this number for each person (well, each Census Tract) in the state, take the natural logarithm, then average them together (weighed based on the Census Tract's population), you can come up with a nifty "urbanization index"
>
> [--Nate Silver](https://twitter.com/NateSilver538/status/1200498385125433344)

Did you think that meant you should compute five-mile buffers around each tract centroid and then perform an [areal weighted interpolation](https://cran.r-project.org/web/packages/areal/vignettes/areal-weighted-interpolation.html) of the tract populations into those buffers?
No? Good. [Neither did I.](https://gitlab.com/edavidaja/urbanindex/-/commit/eecfff72ae05895793cdc8848a95c0dc81b2a6a8#a1fd5ff31c985be6909cafb890edbeec07a52abe_0_33)

Here are the first ten rows of that attempt.

```
# A tibble: 52 x 2
   state          avg_pop_within_five
   <chr>                        <dbl>
 1 Idaho                         8.54
 2 Texas                         8.51
 3 Georgia                       8.51
 4 Utah                          8.50
 5 Washington                    8.48
 6 California                    8.45
 7 Oregon                        8.45
 8 Florida                       8.44
 9 Massachusetts                 8.41
10 North Carolina                8.40
```

🙅🏾‍♂️

### getting it less wrong

Using [spatial joins](https://cran.r-project.org/web/packages/sf/vignettes/sf4.html#joining_two_feature_sets_based_on_attributes), I created a dataset of intersections between each tract and a five-mile buffer around its centroid.
This is [surprisingly fast,](https://www.r-spatial.org/r/2017/06/22/spatial-index.html) and gives results that are close to the reference solution:

```r
inner_join(pop_within_5_mi, states) %>% 
  filter(pop_within_five != 0) %>% # this drops 18 or so tracts
  mutate(log_pop_within_five = log(pop_within_five)) %>% 
  group_by(state) %>% 
  summarise(avg_log_pop_within_five = mean(log_pop_within_five, na.rm = TRUE)) %>% 
  arrange(desc(avg_log_pop_within_five))
```

|state                | avg_log_pop_within_five|
|:--------------------|-----------------------:|
|District of Columbia |               13.609898|
|New York             |               12.923677|
|New Jersey           |               12.561995|
|California           |               12.555172|
|Massachusetts        |               12.250144|
|Maryland             |               12.223699|
|Nevada               |               12.176492|
|Illinois             |               12.132436|
|Rhode Island         |               12.114554|
|Florida              |               11.989963|
|Puerto Rico          |               11.987275|
|Arizona              |               11.907076|
|Connecticut          |               11.885447|
|Pennsylvania         |               11.796463|
|Texas                |               11.743707|
|Hawaii               |               11.690124|
|Utah                 |               11.684077|
|Colorado             |               11.679461|
|Virginia             |               11.669739|
|Ohio                 |               11.669211|
|Washington           |               11.661034|
|Delaware             |               11.646092|
|Michigan             |               11.545489|
|Georgia              |               11.408529|
|Oregon               |               11.371397|
|Indiana              |               11.267387|
|Louisiana            |               11.249018|
|North Carolina       |               11.240352|
|Minnesota            |               11.188313|
|Tennessee            |               11.160029|
|Wisconsin            |               11.149746|
|Missouri             |               11.146195|
|South Carolina       |               11.056779|
|New Hampshire        |               10.935114|
|Kentucky             |               10.910523|
|Oklahoma             |               10.900421|
|Alabama              |               10.791752|
|New Mexico           |               10.750496|
|Nebraska             |               10.725683|
|Kansas               |               10.713352|
|Idaho                |               10.535197|
|West Virginia        |               10.445101|
|Arkansas             |               10.360037|
|Mississippi          |               10.343037|
|Iowa                 |               10.298608|
|Maine                |               10.135474|
|Vermont              |               10.094946|
|Alaska               |                9.940388|
|Wyoming              |                9.773519|
|South Dakota         |                9.607456|
|Montana              |                9.514699|
|North Dakota         |                9.415500|

### one more alternative approach

Rather than intersecting tract boundaries with the five-mile buffer from their centroids, I tried using `st_is_within_distance` to join the data frame of tract centroids to itself. Since no index is built, this is not as fast. It also gives results that match the reference less well. You can install the package from the [`centroid-self-join`](https://github.com/edavidaja/urbanindex/tree/centroid-self-join) branch for a look at those.

## how to get even closer?

fivethirtyeight do some tweaking of their index for tracts whose centroid is further than five miles away from other tract centroids:

> For a census tract that is more than 5 miles away any other census tract (centroid to centroid), this number is decreased based on the minimum distance to the nearest census tract

I might be using the wrong projection for this kind of analysis.

There may be a better spatial join for expressing the relationship described in the article.