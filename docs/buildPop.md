Build Population Estimate Data
================
Christopher Prener, Ph.D.
(October 31, 2019)

## Introduction

This notebook creates the requested population estimates.

## Dependencies

This notebook requires a number of different `R` packages:

``` r
# tidyverse packages
library(dplyr)         # data wrangling
```

    ## 
    ## Attaching package: 'dplyr'

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

``` r
library(readr)         # working with csv data
library(stringr)       # string tools

# spatial packages
library(areal)         # interpolation
library(sf)            # working with spatial data
```

    ## Linking to GEOS 3.6.1, GDAL 2.1.3, PROJ 4.9.3

``` r
library(tidycensus)    # census api access
library(tigris)        # tiger/line api access
```

    ## To enable 
    ## caching of data, set `options(tigris_use_cache = TRUE)` in your R script or .Rprofile.

    ## 
    ## Attaching package: 'tigris'

    ## The following object is masked from 'package:graphics':
    ## 
    ##     plot

``` r
# other packages
library(here)          # file path management
```

    ## here() starts at /Users/prenercg/GitHub/STL_DEMOGRAPHY_NhoodPop

``` r
library(testthat)      # unit testing
```

    ## 
    ## Attaching package: 'testthat'

    ## The following object is masked from 'package:dplyr':
    ## 
    ##     matches

We also use a function for unit testing ID numbers:

``` r
source(here("source", "unique_id.R"))
```

## Create Demographic Data, 1940-2000

These decennial census data were obtained from two sources. The
tract-level shapefiles were obtained from IPUMS’
[NHGIS](https://www.nhgis.org) database. They come for the entire U.S.
(or as much of the U.S. as was tracted at that point - full tract
coverage is relatively recent). They were merged with tract-level data
obtained from [Social Explorer](http://socialexplorer.com) that was
already clean and ready to use for each decade.

### 1940

First, we need to load the shapefile
geometry:

``` r
st_read(here("data", "spatial", "STL_DEMOGRAPHICS_tracts40", "STL_DEMOGRAPHICS_tracts40.shp"),
        stringsAsFactors = FALSE) %>%
  st_transform(crs = 26915) -> stl40
```

    ## Reading layer `STL_DEMOGRAPHICS_tracts40' from data source `/Users/prenercg/GitHub/STL_DEMOGRAPHY_NhoodPop/data/spatial/STL_DEMOGRAPHICS_tracts40/STL_DEMOGRAPHICS_tracts40.shp' using driver `ESRI Shapefile'
    ## Simple feature collection with 128 features and 1 field
    ## geometry type:  POLYGON
    ## dimension:      XY
    ## bbox:           xmin: -90.32052 ymin: 38.53185 xmax: -90.16641 ymax: 38.77435
    ## epsg (SRID):    NA
    ## proj4string:    +proj=longlat +ellps=GRS80 +no_defs

Next, we need to load the census data and combine it with the spatial
data. We need to create the `TRACTID` variable out of a larger variable
named `Geo_Name`. A unit test is included to ensure that the `TRACTID`
variable we are creating uniquely identifies observations:

``` r
read_csv(here("data", "tabular", "STL_DEMOGRAPHICS_pop40.csv")) %>%
  select(Geo_Name, SE_T001_001) %>%
  rename(TRACTID = Geo_Name, 
         pop40 = SE_T001_001) %>%
  mutate(TRACTID = str_pad(string = TRACTID, width = 5, side = "left", pad = "0")) -> pop40
```

    ## Parsed with column specification:
    ## cols(
    ##   Geo_Name = col_character(),
    ##   Geo_QName = col_character(),
    ##   Geo_SUMLEV = col_double(),
    ##   Geo_FIPS = col_double(),
    ##   Geo_state = col_double(),
    ##   Geo_county = col_double(),
    ##   SE_T001_001 = col_double()
    ## )

``` r
# unit test
pop40 %>% unique_id(TRACTID) -> idUnique
expect_equal(idUnique, TRUE)

# join data
stl40 <- left_join(stl40, pop40, by = "TRACTID")
```

Finally, we’ll use a technique called [areal weighted
interpolation](https://slu-opengis.github.io/areal/articles/areal-weighted-interpolation.html)
to produce estimates at the neighborhood level. We’ll import the
neighborhood data, re-project it so that it matches the projection used
for the 1940 tract boundaries, subset it so that we have only the needed
columns and only residential neighborhoods (large parks removed), and
then interpolate all of the tract data into neighborhoods.

``` r
# interpolate
st_read(here("data", "spatial", "nhood", "BND_Nhd88_cw.shp"), stringsAsFactors = FALSE) %>%
  st_transform(crs = 26915) %>%
  select(NHD_NAME, NHD_NUM) %>%
  filter(NHD_NUM <= 79) %>%
  aw_interpolate(tid = NHD_NUM, source = stl40, sid = TRACTID, 
                 weight = "sum", output = "tibble", 
                 extensive = "pop40") -> nhood40
```

    ## Reading layer `BND_Nhd88_cw' from data source `/Users/prenercg/GitHub/STL_DEMOGRAPHY_NhoodPop/data/spatial/nhood/BND_Nhd88_cw.shp' using driver `ESRI Shapefile'
    ## Simple feature collection with 88 features and 6 fields
    ## geometry type:  MULTIPOLYGON
    ## dimension:      XY
    ## bbox:           xmin: 871512.3 ymin: 982994.4 xmax: 912850.5 ymax: 1070957
    ## epsg (SRID):    NA
    ## proj4string:    +proj=tmerc +lat_0=35.83333333333334 +lon_0=-90.5 +k=0.9999333333333333 +x_0=250000 +y_0=0 +datum=NAD83 +units=us-ft +no_defs

``` r
# unit test
expect_equal(aw_verify(source = stl40, sourceValue = pop40, result = nhood40, resultValue = pop40), TRUE)

# clean-up enviornment
rm(pop40, stl40)
```

For tracts that straddle one of the large parks, their entire population
is allocated into the appropriate adjacent neighborhood. We confirm that
the entire city’s population using a unit test with the `aw_verify()`
function. As long as `aw_verify()` returns `TRUE`, we know that each
resident has been allocated. We wrap this in a unit test so that the
code errors out if this assumption is not met.

### 1950

For the remainder of the decennial census data, I’m going to use the
same workflow but condense the code.

The only major change with this decade is that the tract ID numbers
already have the appropriate number of leading zeros, so the code to add
them can be omitted.

``` r
# read in 1950 era tract boundaries, re-project
st_read(here("data", "spatial", "STL_DEMOGRAPHICS_tracts50", "STL_DEMOGRAPHICS_tracts50.shp"),
        stringsAsFactors = FALSE) %>%
  st_transform(crs = 26915) -> stl50
```

    ## Reading layer `STL_DEMOGRAPHICS_tracts50' from data source `/Users/prenercg/GitHub/STL_DEMOGRAPHY_NhoodPop/data/spatial/STL_DEMOGRAPHICS_tracts50/STL_DEMOGRAPHICS_tracts50.shp' using driver `ESRI Shapefile'
    ## Simple feature collection with 128 features and 1 field
    ## geometry type:  POLYGON
    ## dimension:      XY
    ## bbox:           xmin: -90.32051 ymin: 38.53185 xmax: -90.16641 ymax: 38.77435
    ## epsg (SRID):    NA
    ## proj4string:    +proj=longlat +ellps=GRS80 +no_defs

``` r
# read in 1950 census counts, clean
read_csv(here("data", "tabular", "STL_DEMOGRAPHICS_pop50.csv")) %>%
  select(Geo_Name, SE_T001_001) %>%
  rename(TRACTID = Geo_Name, 
         pop50 = SE_T001_001) -> pop50
```

    ## Parsed with column specification:
    ## cols(
    ##   Geo_Name = col_character(),
    ##   Geo_QName = col_character(),
    ##   Geo_SUMLEV = col_double(),
    ##   Geo_FIPS = col_double(),
    ##   Geo_state = col_double(),
    ##   Geo_county = col_double(),
    ##   SE_T001_001 = col_double()
    ## )

``` r
# unit test
pop50 %>% unique_id(TRACTID) -> idUnique
expect_equal(idUnique, TRUE)

# join data
stl50 <- left_join(stl50, pop50, by = "TRACTID")

# interpolate to neighborhoods
st_read(here("data", "spatial", "nhood", "BND_Nhd88_cw.shp"), stringsAsFactors = FALSE) %>%
  st_transform(crs = 26915) %>%
  select(NHD_NUM) %>%
  filter(NHD_NUM <= 79) %>%
  aw_interpolate(tid = NHD_NUM, source = stl50, sid = TRACTID, 
                 weight = "sum", output = "tibble", 
                 extensive = "pop50") -> nhood50
```

    ## Reading layer `BND_Nhd88_cw' from data source `/Users/prenercg/GitHub/STL_DEMOGRAPHY_NhoodPop/data/spatial/nhood/BND_Nhd88_cw.shp' using driver `ESRI Shapefile'
    ## Simple feature collection with 88 features and 6 fields
    ## geometry type:  MULTIPOLYGON
    ## dimension:      XY
    ## bbox:           xmin: 871512.3 ymin: 982994.4 xmax: 912850.5 ymax: 1070957
    ## epsg (SRID):    NA
    ## proj4string:    +proj=tmerc +lat_0=35.83333333333334 +lon_0=-90.5 +k=0.9999333333333333 +x_0=250000 +y_0=0 +datum=NAD83 +units=us-ft +no_defs

``` r
# unit test
expect_equal(aw_verify(source = stl50, sourceValue = pop50, result = nhood50, resultValue = pop50), TRUE)

# clean-up enviornment
rm(pop50, stl50)
```

### 1960

We’ll repeat this process for the 1960s era data.

One issue with the 1960s data is that the `TRACTID` in the 1960 census
data needs to be parsed out of a larger variable `Geo_FIPS`. This is
addressed in the second pipeline.

``` r
# read in 1960 era tract boundaries, re-project
st_read(here("data", "spatial", "STL_DEMOGRAPHICS_tracts60", "STL_DEMOGRAPHICS_tracts60.shp"),
        stringsAsFactors = FALSE) %>%
  st_transform(crs = 26915) -> stl60
```

    ## Reading layer `STL_DEMOGRAPHICS_tracts60' from data source `/Users/prenercg/GitHub/STL_DEMOGRAPHY_NhoodPop/data/spatial/STL_DEMOGRAPHICS_tracts60/STL_DEMOGRAPHICS_tracts60.shp' using driver `ESRI Shapefile'
    ## Simple feature collection with 128 features and 1 field
    ## geometry type:  POLYGON
    ## dimension:      XY
    ## bbox:           xmin: -90.32051 ymin: 38.53185 xmax: -90.16641 ymax: 38.77435
    ## epsg (SRID):    NA
    ## proj4string:    +proj=longlat +ellps=GRS80 +no_defs

``` r
# read in 1960 census counts, clean
read_csv(here("data", "tabular", "STL_DEMOGRAPHICS_pop60.csv")) %>%
  select(Geo_FIPS, SE_T001_001) %>%
  rename(TRACTID = Geo_FIPS, 
         pop60 = SE_T001_001) %>%
  mutate(TRACTID = str_sub(TRACTID, 7, 11)) -> pop60
```

    ## Parsed with column specification:
    ## cols(
    ##   .default = col_double(),
    ##   Geo_Name = col_character(),
    ##   Geo_QName = col_character(),
    ##   Geo_FIPS = col_character(),
    ##   Geo_tractcd = col_character(),
    ##   Geo_GISJOIN = col_character(),
    ##   Geo_GISJoin2 = col_character(),
    ##   Geo_county60 = col_character()
    ## )

    ## See spec(...) for full column specifications.

``` r
# unit test
pop60 %>% unique_id(TRACTID) -> idUnique
expect_equal(idUnique, TRUE)

# join data
stl60 <- left_join(stl60, pop60, by = "TRACTID")

# interpolate to neighborhoods
st_read(here("data", "spatial", "nhood", "BND_Nhd88_cw.shp"), stringsAsFactors = FALSE) %>%
  st_transform(crs = 26915) %>%
  select(NHD_NUM) %>%
  filter(NHD_NUM <= 79) %>%
  aw_interpolate(tid = NHD_NUM, source = stl60, sid = TRACTID, 
                 weight = "sum", output = "tibble", 
                 extensive = "pop60") -> nhood60
```

    ## Reading layer `BND_Nhd88_cw' from data source `/Users/prenercg/GitHub/STL_DEMOGRAPHY_NhoodPop/data/spatial/nhood/BND_Nhd88_cw.shp' using driver `ESRI Shapefile'
    ## Simple feature collection with 88 features and 6 fields
    ## geometry type:  MULTIPOLYGON
    ## dimension:      XY
    ## bbox:           xmin: 871512.3 ymin: 982994.4 xmax: 912850.5 ymax: 1070957
    ## epsg (SRID):    NA
    ## proj4string:    +proj=tmerc +lat_0=35.83333333333334 +lon_0=-90.5 +k=0.9999333333333333 +x_0=250000 +y_0=0 +datum=NAD83 +units=us-ft +no_defs

``` r
# unit test
expect_equal(aw_verify(source = stl60, sourceValue = pop60, result = nhood60, resultValue = pop60), TRUE)

# clean-up enviornment
rm(pop60, stl60)
```

### 1970

We’ll repeat this process for the 1970s era data.

``` r
# read in 1970 era tract boundaries, re-project
st_read(here("data", "spatial", "STL_DEMOGRAPHICS_tracts70", "STL_DEMOGRAPHICS_tracts70.shp"),
        stringsAsFactors = FALSE) %>%
  st_transform(crs = 26915) -> stl70
```

    ## Reading layer `STL_DEMOGRAPHICS_tracts70' from data source `/Users/prenercg/GitHub/STL_DEMOGRAPHY_NhoodPop/data/spatial/STL_DEMOGRAPHICS_tracts70/STL_DEMOGRAPHICS_tracts70.shp' using driver `ESRI Shapefile'
    ## Simple feature collection with 126 features and 1 field
    ## geometry type:  POLYGON
    ## dimension:      XY
    ## bbox:           xmin: -90.32051 ymin: 38.53185 xmax: -90.16641 ymax: 38.77435
    ## epsg (SRID):    NA
    ## proj4string:    +proj=longlat +ellps=GRS80 +no_defs

``` r
# read in 1970 census counts, clean
read_csv(here("data", "tabular", "STL_DEMOGRAPHICS_pop70.csv")) %>%
  select(Geo_TractCode, SE_T001_001) %>%
  rename(TRACTID = Geo_TractCode, 
         pop70 = SE_T001_001) -> pop70
```

    ## Parsed with column specification:
    ## cols(
    ##   Geo_FIPS = col_double(),
    ##   Geo_NAME = col_double(),
    ##   Geo_QName = col_character(),
    ##   Geo_State = col_double(),
    ##   Geo_COUNTY = col_double(),
    ##   Geo_TractCode = col_double(),
    ##   Geo_TRACT = col_double(),
    ##   Geo_SUFFTRT = col_double(),
    ##   Geo_METROARA = col_double(),
    ##   Geo_PLACE = col_logical(),
    ##   Geo_URBANARA = col_logical(),
    ##   SE_T001_001 = col_double()
    ## )

``` r
# unit test
pop70 %>% unique_id(TRACTID) -> idUnique
expect_equal(idUnique, TRUE)

# join data
stl70 <- left_join(stl70, pop70, by = "TRACTID")

# interpolate to neighborhoods
st_read(here("data", "spatial", "nhood", "BND_Nhd88_cw.shp"), stringsAsFactors = FALSE) %>%
  st_transform(crs = 26915) %>%
  select(NHD_NUM) %>%
  filter(NHD_NUM <= 79) %>%
  aw_interpolate(tid = NHD_NUM, source = stl70, sid = TRACTID, 
                 weight = "sum", output = "tibble", 
                 extensive = "pop70") -> nhood70
```

    ## Reading layer `BND_Nhd88_cw' from data source `/Users/prenercg/GitHub/STL_DEMOGRAPHY_NhoodPop/data/spatial/nhood/BND_Nhd88_cw.shp' using driver `ESRI Shapefile'
    ## Simple feature collection with 88 features and 6 fields
    ## geometry type:  MULTIPOLYGON
    ## dimension:      XY
    ## bbox:           xmin: 871512.3 ymin: 982994.4 xmax: 912850.5 ymax: 1070957
    ## epsg (SRID):    NA
    ## proj4string:    +proj=tmerc +lat_0=35.83333333333334 +lon_0=-90.5 +k=0.9999333333333333 +x_0=250000 +y_0=0 +datum=NAD83 +units=us-ft +no_defs

``` r
# unit test
expect_equal(aw_verify(source = stl70, sourceValue = pop70, result = nhood70, resultValue = pop70), TRUE)

# clean-up enviornment
rm(pop70, stl70)
```

### 1980

We’ll repeat this process for the 1980s era data.

``` r
# read in 1980 era tract boundaries, re-project
st_read(here("data", "spatial", "STL_DEMOGRAPHICS_tracts80", "STL_DEMOGRAPHICS_tracts80.shp"),
        stringsAsFactors = FALSE) %>%
  st_transform(crs = 26915) -> stl80
```

    ## Reading layer `STL_DEMOGRAPHICS_tracts80' from data source `/Users/prenercg/GitHub/STL_DEMOGRAPHY_NhoodPop/data/spatial/STL_DEMOGRAPHICS_tracts80/STL_DEMOGRAPHICS_tracts80.shp' using driver `ESRI Shapefile'
    ## Simple feature collection with 113 features and 1 field
    ## geometry type:  POLYGON
    ## dimension:      XY
    ## bbox:           xmin: -90.32051 ymin: 38.53185 xmax: -90.16641 ymax: 38.77435
    ## epsg (SRID):    NA
    ## proj4string:    +proj=longlat +ellps=GRS80 +no_defs

``` r
# read in 1980 census counts, clean
read_csv(here("data", "tabular", "STL_DEMOGRAPHICS_pop80.csv")) %>%
  select(Geo_TRACT6, SE_T001_001) %>%
  rename(TRACTID = Geo_TRACT6, 
         pop80 = SE_T001_001) -> pop80
```

    ## Parsed with column specification:
    ## cols(
    ##   Geo_Name = col_character(),
    ##   Geo_QName = col_character(),
    ##   Geo_GISJOIN = col_character(),
    ##   Geo_GISJOIN2 = col_double(),
    ##   Geo_SUMLEV = col_double(),
    ##   Geo_shape_area = col_double(),
    ##   Geo_FIPS = col_double(),
    ##   Geo_STATE = col_double(),
    ##   Geo_COUNTY = col_double(),
    ##   Geo_TRACT6 = col_double(),
    ##   SE_T001_001 = col_double()
    ## )

``` r
# unit test
pop80 %>% unique_id(TRACTID) -> idUnique
expect_equal(idUnique, TRUE)

# join data
stl80 <- left_join(stl80, pop80, by = "TRACTID")

# interpolate to neighborhoods
st_read(here("data", "spatial", "nhood", "BND_Nhd88_cw.shp"), stringsAsFactors = FALSE) %>%
  st_transform(crs = 26915) %>%
  select(NHD_NUM) %>%
  filter(NHD_NUM <= 79) %>%
  aw_interpolate(tid = NHD_NUM, source = stl80, sid = TRACTID, 
                 weight = "sum", output = "tibble", 
                 extensive = "pop80") -> nhood80
```

    ## Reading layer `BND_Nhd88_cw' from data source `/Users/prenercg/GitHub/STL_DEMOGRAPHY_NhoodPop/data/spatial/nhood/BND_Nhd88_cw.shp' using driver `ESRI Shapefile'
    ## Simple feature collection with 88 features and 6 fields
    ## geometry type:  MULTIPOLYGON
    ## dimension:      XY
    ## bbox:           xmin: 871512.3 ymin: 982994.4 xmax: 912850.5 ymax: 1070957
    ## epsg (SRID):    NA
    ## proj4string:    +proj=tmerc +lat_0=35.83333333333334 +lon_0=-90.5 +k=0.9999333333333333 +x_0=250000 +y_0=0 +datum=NAD83 +units=us-ft +no_defs

``` r
# unit test
expect_equal(aw_verify(source = stl80, sourceValue = pop80, result = nhood80, resultValue = pop80), TRUE)

# clean-up enviornment
rm(pop80, stl80)
```

### 1990

These data are from the decennial census. Note that the total population
in neighborhoods is 12 people less than the total population in tracts,
because Census Tract 1018.99 does not have any geometry.

``` r
# read in 1990 census counts, clean
get_decennial(geography = "tract", variable = "P0010001", year = 1990, state = 29, county = 510, geometry = TRUE) %>%
  st_transform(crs = 26915) %>%
  select(GEOID, value) %>%
  rename(pop90 = value) -> pop90
```

    ## Getting data from the 1990 decennial Census

    ## Downloading feature geometry from the Census website.  To cache shapefiles for use in future sessions, set `options(tigris_use_cache = TRUE)`.

    ## 
      |                                                                       
      |                                                                 |   0%
      |                                                                       
      |===                                                              |   5%
      |                                                                       
      |=======                                                          |  10%
      |                                                                       
      |==========                                                       |  15%
      |                                                                       
      |===========                                                      |  16%
      |                                                                       
      |=============                                                    |  20%
      |                                                                       
      |=================                                                |  25%
      |                                                                       
      |====================                                             |  31%
      |                                                                       
      |======================                                           |  33%
      |                                                                       
      |=========================                                        |  38%
      |                                                                       
      |===========================                                      |  41%
      |                                                                       
      |==============================                                   |  46%
      |                                                                       
      |=================================                                |  51%
      |                                                                       
      |===================================                              |  54%
      |                                                                       
      |======================================                           |  59%
      |                                                                       
      |========================================                         |  61%
      |                                                                       
      |===========================================                      |  67%
      |                                                                       
      |===============================================                  |  72%
      |                                                                       
      |================================================                 |  74%
      |                                                                       
      |====================================================             |  79%
      |                                                                       
      |=====================================================            |  82%
      |                                                                       
      |=========================================================        |  87%
      |                                                                       
      |============================================================     |  92%
      |                                                                       
      |==============================================================   |  95%
      |                                                                       
      |=================================================================| 100%

``` r
# interpolate to neighborhoods
st_read(here("data", "spatial", "nhood", "BND_Nhd88_cw.shp"), stringsAsFactors = FALSE) %>%
  st_transform(crs = 26915) %>%
  select(NHD_NUM) %>%
  filter(NHD_NUM <= 79) %>%
  aw_interpolate(tid = NHD_NUM, source = pop90, sid = GEOID, 
                 weight = "sum", output = "tibble", 
                 extensive = "pop90") -> nhood90
```

    ## Reading layer `BND_Nhd88_cw' from data source `/Users/prenercg/GitHub/STL_DEMOGRAPHY_NhoodPop/data/spatial/nhood/BND_Nhd88_cw.shp' using driver `ESRI Shapefile'
    ## Simple feature collection with 88 features and 6 fields
    ## geometry type:  MULTIPOLYGON
    ## dimension:      XY
    ## bbox:           xmin: 871512.3 ymin: 982994.4 xmax: 912850.5 ymax: 1070957
    ## epsg (SRID):    NA
    ## proj4string:    +proj=tmerc +lat_0=35.83333333333334 +lon_0=-90.5 +k=0.9999333333333333 +x_0=250000 +y_0=0 +datum=NAD83 +units=us-ft +no_defs

``` r
# unit test
expect_equal(aw_verify(source = pop90, sourceValue = pop90, result = nhood90, resultValue = pop90), FALSE)
expect_equal(sum(nhood90$pop90), sum(pop90$pop90)-12)

# clean-up enviornment
rm(pop90)
```

### 2000

We’ll repeat this process for the 2000s era data.

``` r
# read in 2000 census counts, clean
get_decennial(geography = "tract", variable = "P001001", year = 2000, state = 29, county = 510, geometry = TRUE) %>%
  st_transform(crs = 26915) %>%
  select(GEOID, value) %>%
  rename(pop00 = value) -> pop00
```

    ## Getting data from the 2000 decennial Census

    ## Downloading feature geometry from the Census website.  To cache shapefiles for use in future sessions, set `options(tigris_use_cache = TRUE)`.

    ## 
    Downloading: 16 kB     
    Downloading: 16 kB     
    Downloading: 25 kB     
    Downloading: 25 kB     
    Downloading: 41 kB     
    Downloading: 41 kB     
    Downloading: 49 kB     
    Downloading: 49 kB     
    Downloading: 49 kB     
    Downloading: 49 kB     
    Downloading: 57 kB     
    Downloading: 57 kB     
    Downloading: 57 kB     
    Downloading: 57 kB     
    Downloading: 65 kB     
    Downloading: 65 kB     
    Downloading: 65 kB     
    Downloading: 65 kB     
    Downloading: 81 kB     
    Downloading: 81 kB     
    Downloading: 89 kB     
    Downloading: 89 kB     
    Downloading: 89 kB     
    Downloading: 89 kB     
    Downloading: 110 kB     
    Downloading: 110 kB     
    Downloading: 110 kB     
    Downloading: 110 kB     
    Downloading: 110 kB     
    Downloading: 110 kB     
    Downloading: 120 kB     
    Downloading: 120 kB     
    Downloading: 120 kB     
    Downloading: 120 kB     
    Downloading: 140 kB     
    Downloading: 140 kB     
    Downloading: 140 kB     
    Downloading: 140 kB     
    Downloading: 150 kB     
    Downloading: 150 kB     
    Downloading: 150 kB     
    Downloading: 150 kB     
    Downloading: 160 kB     
    Downloading: 160 kB     
    Downloading: 170 kB     
    Downloading: 170 kB     
    Downloading: 170 kB     
    Downloading: 170 kB     
    Downloading: 190 kB     
    Downloading: 190 kB     
    Downloading: 190 kB     
    Downloading: 190 kB     
    Downloading: 190 kB     
    Downloading: 190 kB     
    Downloading: 190 kB     
    Downloading: 190 kB     
    Downloading: 200 kB     
    Downloading: 200 kB     
    Downloading: 210 kB     
    Downloading: 210 kB     
    Downloading: 210 kB     
    Downloading: 210 kB     
    Downloading: 230 kB     
    Downloading: 230 kB     
    Downloading: 240 kB     
    Downloading: 240 kB     
    Downloading: 240 kB     
    Downloading: 240 kB     
    Downloading: 240 kB     
    Downloading: 240 kB     
    Downloading: 240 kB     
    Downloading: 240 kB     
    Downloading: 250 kB     
    Downloading: 250 kB     
    Downloading: 250 kB     
    Downloading: 250 kB     
    Downloading: 270 kB     
    Downloading: 270 kB     
    Downloading: 280 kB     
    Downloading: 280 kB     
    Downloading: 280 kB     
    Downloading: 280 kB     
    Downloading: 290 kB     
    Downloading: 290 kB     
    Downloading: 300 kB     
    Downloading: 300 kB     
    Downloading: 300 kB     
    Downloading: 300 kB     
    Downloading: 310 kB     
    Downloading: 310 kB     
    Downloading: 310 kB     
    Downloading: 310 kB     
    Downloading: 320 kB     
    Downloading: 320 kB     
    Downloading: 320 kB     
    Downloading: 320 kB     
    Downloading: 330 kB     
    Downloading: 330 kB     
    Downloading: 340 kB     
    Downloading: 340 kB     
    Downloading: 340 kB     
    Downloading: 340 kB     
    Downloading: 360 kB     
    Downloading: 360 kB     
    Downloading: 360 kB     
    Downloading: 360 kB     
    Downloading: 360 kB     
    Downloading: 360 kB     
    Downloading: 370 kB     
    Downloading: 370 kB     
    Downloading: 370 kB     
    Downloading: 370 kB     
    Downloading: 390 kB     
    Downloading: 390 kB     
    Downloading: 400 kB     
    Downloading: 400 kB     
    Downloading: 420 kB     
    Downloading: 420 kB     
    Downloading: 420 kB     
    Downloading: 420 kB     
    Downloading: 420 kB     
    Downloading: 420 kB     
    Downloading: 420 kB     
    Downloading: 420 kB     
    Downloading: 420 kB     
    Downloading: 420 kB     
    Downloading: 440 kB     
    Downloading: 440 kB     
    Downloading: 440 kB     
    Downloading: 440 kB     
    Downloading: 440 kB     
    Downloading: 440 kB     
    Downloading: 440 kB     
    Downloading: 440 kB     
    Downloading: 450 kB     
    Downloading: 450 kB     
    Downloading: 450 kB     
    Downloading: 450 kB     
    Downloading: 450 kB     
    Downloading: 450 kB     
    Downloading: 450 kB     
    Downloading: 450 kB     
    Downloading: 470 kB     
    Downloading: 470 kB     
    Downloading: 470 kB     
    Downloading: 470 kB     
    Downloading: 470 kB     
    Downloading: 470 kB     
    Downloading: 470 kB     
    Downloading: 470 kB     
    Downloading: 480 kB     
    Downloading: 480 kB     
    Downloading: 480 kB     
    Downloading: 480 kB     
    Downloading: 500 kB     
    Downloading: 500 kB     
    Downloading: 500 kB     
    Downloading: 500 kB     
    Downloading: 500 kB     
    Downloading: 500 kB     
    Downloading: 500 kB     
    Downloading: 500 kB     
    Downloading: 500 kB     
    Downloading: 500 kB     
    Downloading: 520 kB     
    Downloading: 520 kB     
    Downloading: 520 kB     
    Downloading: 520 kB     
    Downloading: 520 kB     
    Downloading: 520 kB     
    Downloading: 520 kB     
    Downloading: 520 kB     
    Downloading: 530 kB     
    Downloading: 530 kB     
    Downloading: 530 kB     
    Downloading: 530 kB     
    Downloading: 550 kB     
    Downloading: 550 kB     
    Downloading: 550 kB     
    Downloading: 550 kB     
    Downloading: 550 kB     
    Downloading: 550 kB     
    Downloading: 550 kB     
    Downloading: 550 kB     
    Downloading: 550 kB     
    Downloading: 550 kB     
    Downloading: 570 kB     
    Downloading: 570 kB     
    Downloading: 580 kB     
    Downloading: 580 kB     
    Downloading: 580 kB     
    Downloading: 580 kB     
    Downloading: 580 kB     
    Downloading: 580 kB     
    Downloading: 590 kB     
    Downloading: 590 kB     
    Downloading: 600 kB     
    Downloading: 600 kB     
    Downloading: 600 kB     
    Downloading: 600 kB     
    Downloading: 600 kB     
    Downloading: 600 kB     
    Downloading: 610 kB     
    Downloading: 610 kB     
    Downloading: 610 kB     
    Downloading: 610 kB     
    Downloading: 620 kB     
    Downloading: 620 kB     
    Downloading: 620 kB     
    Downloading: 620 kB     
    Downloading: 630 kB     
    Downloading: 630 kB     
    Downloading: 640 kB     
    Downloading: 640 kB     
    Downloading: 640 kB     
    Downloading: 640 kB     
    Downloading: 640 kB     
    Downloading: 640 kB     
    Downloading: 660 kB     
    Downloading: 660 kB     
    Downloading: 660 kB     
    Downloading: 660 kB     
    Downloading: 660 kB     
    Downloading: 660 kB     
    Downloading: 660 kB     
    Downloading: 660 kB     
    Downloading: 680 kB     
    Downloading: 680 kB     
    Downloading: 680 kB     
    Downloading: 680 kB     
    Downloading: 680 kB     
    Downloading: 680 kB     
    Downloading: 680 kB     
    Downloading: 680 kB     
    Downloading: 680 kB     
    Downloading: 680 kB     
    Downloading: 700 kB     
    Downloading: 700 kB     
    Downloading: 700 kB     
    Downloading: 700 kB     
    Downloading: 700 kB     
    Downloading: 700 kB     
    Downloading: 700 kB     
    Downloading: 700 kB     
    Downloading: 700 kB     
    Downloading: 700 kB     
    Downloading: 710 kB     
    Downloading: 710 kB     
    Downloading: 720 kB     
    Downloading: 720 kB     
    Downloading: 720 kB     
    Downloading: 720 kB     
    Downloading: 720 kB     
    Downloading: 720 kB     
    Downloading: 740 kB     
    Downloading: 740 kB     
    Downloading: 740 kB     
    Downloading: 740 kB     
    Downloading: 740 kB     
    Downloading: 740 kB     
    Downloading: 740 kB     
    Downloading: 740 kB     
    Downloading: 740 kB     
    Downloading: 740 kB     
    Downloading: 740 kB     
    Downloading: 740 kB     
    Downloading: 760 kB     
    Downloading: 760 kB     
    Downloading: 760 kB     
    Downloading: 760 kB     
    Downloading: 760 kB     
    Downloading: 760 kB     
    Downloading: 760 kB     
    Downloading: 760 kB     
    Downloading: 760 kB     
    Downloading: 760 kB     
    Downloading: 780 kB     
    Downloading: 780 kB     
    Downloading: 790 kB     
    Downloading: 790 kB     
    Downloading: 790 kB     
    Downloading: 790 kB     
    Downloading: 790 kB     
    Downloading: 790 kB     
    Downloading: 790 kB     
    Downloading: 790 kB     
    Downloading: 790 kB     
    Downloading: 790 kB     
    Downloading: 810 kB     
    Downloading: 810 kB     
    Downloading: 810 kB     
    Downloading: 810 kB     
    Downloading: 810 kB     
    Downloading: 810 kB     
    Downloading: 810 kB     
    Downloading: 810 kB     
    Downloading: 810 kB     
    Downloading: 810 kB     
    Downloading: 830 kB     
    Downloading: 830 kB     
    Downloading: 830 kB     
    Downloading: 830 kB     
    Downloading: 830 kB     
    Downloading: 830 kB     
    Downloading: 830 kB     
    Downloading: 830 kB     
    Downloading: 840 kB     
    Downloading: 840 kB     
    Downloading: 840 kB     
    Downloading: 840 kB     
    Downloading: 860 kB     
    Downloading: 860 kB     
    Downloading: 860 kB     
    Downloading: 860 kB     
    Downloading: 870 kB     
    Downloading: 870 kB     
    Downloading: 870 kB     
    Downloading: 870 kB     
    Downloading: 880 kB     
    Downloading: 880 kB     
    Downloading: 880 kB     
    Downloading: 880 kB     
    Downloading: 880 kB     
    Downloading: 880 kB     
    Downloading: 880 kB     
    Downloading: 880 kB     
    Downloading: 900 kB     
    Downloading: 900 kB     
    Downloading: 910 kB     
    Downloading: 910 kB     
    Downloading: 910 kB     
    Downloading: 910 kB     
    Downloading: 910 kB     
    Downloading: 910 kB     
    Downloading: 920 kB     
    Downloading: 920 kB     
    Downloading: 930 kB     
    Downloading: 930 kB     
    Downloading: 930 kB     
    Downloading: 930 kB     
    Downloading: 930 kB     
    Downloading: 930 kB     
    Downloading: 930 kB     
    Downloading: 930 kB     
    Downloading: 950 kB     
    Downloading: 950 kB     
    Downloading: 960 kB     
    Downloading: 960 kB     
    Downloading: 960 kB     
    Downloading: 960 kB     
    Downloading: 960 kB     
    Downloading: 960 kB     
    Downloading: 970 kB     
    Downloading: 970 kB     
    Downloading: 980 kB     
    Downloading: 980 kB     
    Downloading: 980 kB     
    Downloading: 980 kB     
    Downloading: 990 kB     
    Downloading: 990 kB     
    Downloading: 990 kB     
    Downloading: 990 kB     
    Downloading: 1 MB     
    Downloading: 1 MB     
    Downloading: 1 MB     
    Downloading: 1 MB     
    Downloading: 1 MB     
    Downloading: 1 MB

``` r
# interpolate to neighborhoods
st_read(here("data", "spatial", "nhood", "BND_Nhd88_cw.shp"), stringsAsFactors = FALSE) %>%
  st_transform(crs = 26915) %>%
  select(NHD_NUM) %>%
  filter(NHD_NUM <= 79) %>%
  aw_interpolate(tid = NHD_NUM, source = pop00, sid = GEOID, 
                 weight = "sum", output = "tibble", 
                 extensive = "pop00") -> nhood00
```

    ## Reading layer `BND_Nhd88_cw' from data source `/Users/prenercg/GitHub/STL_DEMOGRAPHY_NhoodPop/data/spatial/nhood/BND_Nhd88_cw.shp' using driver `ESRI Shapefile'
    ## Simple feature collection with 88 features and 6 fields
    ## geometry type:  MULTIPOLYGON
    ## dimension:      XY
    ## bbox:           xmin: 871512.3 ymin: 982994.4 xmax: 912850.5 ymax: 1070957
    ## epsg (SRID):    NA
    ## proj4string:    +proj=tmerc +lat_0=35.83333333333334 +lon_0=-90.5 +k=0.9999333333333333 +x_0=250000 +y_0=0 +datum=NAD83 +units=us-ft +no_defs

``` r
# unit test
expect_equal(aw_verify(source = pop00, sourceValue = pop00, result = nhood00, resultValue = pop00), TRUE)

# clean-up enviornment
rm(pop00)
```

### Combine 1940s-2000s Data

Next, we’ll join all of the neighborhood estimates we’ve created so far
together into a single object:

``` r
left_join(nhood40, nhood50, by = "NHD_NUM") %>%
  left_join(., nhood60, by = "NHD_NUM") %>%
  left_join(., nhood70, by = "NHD_NUM") %>%
  left_join(., nhood80, by = "NHD_NUM") %>%
  left_join(., nhood90, by = "NHD_NUM") %>%
  left_join(., nhood00, by = "NHD_NUM") -> nhoodPop_40_00

# clean up enviornment
rm(nhood40, nhood50, nhood60, nhood70, nhood80, nhood90, nhood00, idUnique, unique_id)
```

## Download Demographic Data, 2010-2017

All data for this era are downloaded from the Census Bureau’s API via
`tidycensus`.

### 2010

These data are from the decennial
census.

``` r
get_decennial(geography = "tract", variables = "P001001", state = 29, county = 510, geometry = TRUE) %>% 
  select(GEOID, NAME, value) %>%
  rename(pop10 = value) -> stl10
```

    ## Getting data from the 2010 decennial Census

    ## Downloading feature geometry from the Census website.  To cache shapefiles for use in future sessions, set `options(tigris_use_cache = TRUE)`.

### 2011

These data are from the 2007-2011 5-year American Community Survey
estimates.

``` r
get_acs(geography = "tract", year = 2011, variables = "B01003_001", state = 29, county = 510) %>%
  select(GEOID, estimate, moe) %>%
  rename(pop11 = estimate,
         pop11_m = moe) -> stl11
```

    ## Getting data from the 2007-2011 5-year ACS

### 2012

These data are from the 2008-2012 5-year American Community Survey
estimates.

``` r
get_acs(geography = "tract", year = 2012, variables = "B01003_001", state = 29, county = 510) %>%
  select(GEOID, estimate, moe) %>%
  rename(pop12 = estimate,
         pop12_m = moe) -> stl12
```

    ## Getting data from the 2008-2012 5-year ACS

### 2013

These data are from the 2009-2013 5-year American Community Survey
estimates.

``` r
get_acs(geography = "tract", year = 2013, variables = "B01003_001", state = 29, county = 510) %>%
  select(GEOID, estimate, moe) %>%
  rename(pop13 = estimate,
         pop13_m = moe) -> stl13
```

    ## Getting data from the 2009-2013 5-year ACS

### 2014

These data are from the 2010-2014 5-year American Community Survey
estimates.

``` r
get_acs(geography = "tract", year = 2014, variables = "B01003_001", state = 29, county = 510) %>%
  select(GEOID, estimate, moe) %>%
  rename(pop14 = estimate,
         pop14_m = moe) -> stl14
```

    ## Getting data from the 2010-2014 5-year ACS

### 2015

These data are from the 2011-2015 5-year American Community Survey
estimates.

``` r
get_acs(geography = "tract", year = 2015, variables = "B01003_001", state = 29, county = 510) %>%
  select(GEOID, estimate, moe) %>%
  rename(pop15 = estimate,
         pop15_m = moe) -> stl15
```

    ## Getting data from the 2011-2015 5-year ACS

### 2016

These data are from the 2012-2016 5-year American Community Survey
estimates.

``` r
get_acs(geography = "tract", year = 2016, variables = "B01003_001", state = 29, county = 510) %>%
  select(GEOID, estimate, moe) %>%
  rename(pop16 = estimate,
         pop16_m = moe) -> stl16
```

    ## Getting data from the 2012-2016 5-year ACS

### 2017

These data are from the 2013-2017 5-year American Community Survey
estimates.

``` r
get_acs(geography = "tract", year = 2017, variables = "B01003_001", state = 29, county = 510) %>%
  select(GEOID, estimate, moe) %>%
  rename(pop17 = estimate,
         pop17_m = moe) -> stl17
```

    ## Getting data from the 2013-2017 5-year ACS

### Combine Data

We have these data in a number of different tables, so the next step is
to join them together by `GEOID`.

``` r
left_join(stl10, stl11, by = "GEOID") %>%
  left_join(., stl12, by = "GEOID") %>%
  left_join(., stl13, by = "GEOID") %>%
  left_join(., stl14, by = "GEOID") %>%
  left_join(., stl15, by = "GEOID") %>%
  left_join(., stl16, by = "GEOID") %>%
  left_join(., stl17, by = "GEOID") %>%
  st_transform(crs = 26915) -> tractPop

# clean up enviornment
rm(stl10, stl11, stl12, stl13, stl14, stl15, stl16, stl17)
```

### Interpolate Neighborhood Data

Next, we’ll use the same estimation process we used before on the
2010-2017 data:

``` r
# read neighborhood data, re-project, and interpolate
st_read(here("data", "spatial", "nhood", "BND_Nhd88_cw.shp"), stringsAsFactors = FALSE) %>%
  st_transform(crs = 26915) %>%
  select(NHD_NUM) %>%
  filter(NHD_NUM <= 79) %>%
  aw_interpolate(tid = NHD_NUM, source = tractPop, sid = GEOID, 
                 weight = "sum", output = "tibble", 
                 extensive = c("pop10", "pop11", "pop11_m", 
                               "pop12", "pop12_m", "pop13", "pop13_m", 
                               "pop14", "pop14_m", "pop15", "pop15_m", 
                               "pop16", "pop16_m", "pop17", "pop17_m")
                 ) -> nhoodPop_10_17
```

    ## Reading layer `BND_Nhd88_cw' from data source `/Users/prenercg/GitHub/STL_DEMOGRAPHY_NhoodPop/data/spatial/nhood/BND_Nhd88_cw.shp' using driver `ESRI Shapefile'
    ## Simple feature collection with 88 features and 6 fields
    ## geometry type:  MULTIPOLYGON
    ## dimension:      XY
    ## bbox:           xmin: 871512.3 ymin: 982994.4 xmax: 912850.5 ymax: 1070957
    ## epsg (SRID):    NA
    ## proj4string:    +proj=tmerc +lat_0=35.83333333333334 +lon_0=-90.5 +k=0.9999333333333333 +x_0=250000 +y_0=0 +datum=NAD83 +units=us-ft +no_defs

``` r
# unit test
expect_equal(aw_verify(source = tractPop, sourceValue = pop10, result = nhoodPop_10_17, resultValue = pop10), TRUE)
expect_equal(aw_verify(source = tractPop, sourceValue = pop11, result = nhoodPop_10_17, resultValue = pop11), TRUE)
expect_equal(aw_verify(source = tractPop, sourceValue = pop12, result = nhoodPop_10_17, resultValue = pop12), TRUE)
expect_equal(aw_verify(source = tractPop, sourceValue = pop13, result = nhoodPop_10_17, resultValue = pop13), TRUE)
expect_equal(aw_verify(source = tractPop, sourceValue = pop14, result = nhoodPop_10_17, resultValue = pop14), TRUE)
expect_equal(aw_verify(source = tractPop, sourceValue = pop15, result = nhoodPop_10_17, resultValue = pop15), TRUE)
expect_equal(aw_verify(source = tractPop, sourceValue = pop16, result = nhoodPop_10_17, resultValue = pop16), TRUE)
expect_equal(aw_verify(source = tractPop, sourceValue = pop17, result = nhoodPop_10_17, resultValue = pop17), TRUE)

# clean up enviornment
rm(tractPop)
```

## Combine Historical and Modern Census Data

Next, we’ll combine the two data objects to create a single table of
census estimates:

``` r
nhoodPop <- left_join(nhoodPop_40_00, nhoodPop_10_17, by = "NHD_NUM")

# clean up enviornment
rm(nhoodPop_40_00, nhoodPop_10_17)
```

## Export

Finally, we’ll write the data to a `.csv` file for future analysis.

``` r
# write output
write_csv(nhoodPop, here("data", "clean", "STL_PopByNhood.csv"))
```
