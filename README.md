
<!-- README.md is generated from README.Rmd. Please edit that file -->

# echor

[![CRAN
status](https://www.r-pkg.org/badges/version/echor)](https://cran.r-project.org/package=echor)

[![Travis build
status](https://travis-ci.org/mps9506/echor.svg?branch=master)](https://travis-ci.org/mps9506/echor)
[![AppVeyor build
status](https://ci.appveyor.com/api/projects/status/github/mps9506/echor?branch=master&svg=true)](https://ci.appveyor.com/project/mps9506/echor)
[![Coverage
status](https://codecov.io/gh/mps9506/echor/branch/master/graph/badge.svg)](https://codecov.io/github/mps9506/echor?branch=master)

## Overview

echor downloads wastewater discharge and air emission data for EPA
permitted facilities using the [EPA ECHO API](https://echo.epa.gov/).

## Installation

echor is on CRAN:

``` r
install.packages("echor")
```

Or install the development version from github:

``` r

devtools::install_github("mps9506/echor")
```

## Usage

[Getting
started](https://mps9506.github.io/echor/articles/introduction.html)

[Functions](https://mps9506.github.io/echor/reference/index.html)

## Examples

### Download information about facilities with an NPDES permit

We can look up plants by permit id, bounding box, and numerous other
parameters. I plan on providing documentation of available parameters.
However, arguments can be looked up here:
[get\_cwa\_rest\_services\_get\_facility\_info](https://echo.epa.gov/tools/web-services/facility-search-water#!/Facility_Information/get_cwa_rest_services_get_facility_info)

``` r
library(tidyverse)
library(echor)

## echoWaterGetFacilityInfo() will return a dataframe or simple features (sf) dataframe.

df <- echoWaterGetFacilityInfo(output = "df", 
                               xmin = '-96.387509', 
                               ymin = '30.583572', 
                               xmax = '-96.281422', 
                               ymax = '30.640008',
                               p_ptype = "NPD")

head(df)
#> # A tibble: 3 x 26
#>   CWPName SourceID CWPStreet CWPCity CWPState CWPStateDistrict CWPZip
#>   <chr>   <chr>    <chr>     <chr>   <chr>    <chr>            <chr> 
#> 1 CENTRA~ TX00027~ 222 IREL~ COLLEG~ TX       09               77843 
#> 2 HEAT T~ TX01065~ 0.25MI S~ COLLEG~ TX       09               77845 
#> 3 TURKEY~ TX00624~ 3000FT W~ BRYAN   TX       09               77807 
#> # ... with 19 more variables: MasterExternalPermitNmbr <chr>,
#> #   RegistryID <chr>, CWPCounty <chr>, CWPEPARegion <chr>,
#> #   FacDerivedHuc <chr>, FacLat <dbl>, FacLong <dbl>,
#> #   CWPTotalDesignFlowNmbr <dbl>, CWPActualAverageFlowNmbr <dbl>,
#> #   DschToMs4 <chr>, ExposedActivity <chr>, Subsector <chr>,
#> #   CWPVersionNmbr <dbl>, SubmittedDate <date>, CWPPermitTypeDesc <chr>,
#> #   CWPIssueDate <date>, CWPTerminationDate <date>, CWPSNCStatus <chr>,
#> #   CWPCsoOutfalls <dbl>
```

The ECHO database can provide over 270 different columns. echor returns
a subset of these columns that should work for most users. However, you
can specify what data you want returned. Use `echoWaterGetMeta()` to
return a dataframe with column numbers, names, and descriptions to
identify the columns you want returned. Then include the column numbers
as a comma seperated string in the `qcolumns` argument. In the example
below, the `qcolumns` argument indicates the dataframe will include
plant name, 8-digit HUC, latitute, longitude, and total design flow.

``` r
df <- echoWaterGetFacilityInfo(output = "df", 
                               xmin = '-96.387509', 
                               ymin = '30.583572', 
                               xmax = '-96.281422', 
                               ymax = '30.640008',
                               qcolumns = '1,14,23,24,25',
                               p_ptype = "NPD")
head(df)
#> # A tibble: 3 x 6
#>   CWPName         SourceID FacDerivedHuc FacLat FacLong CWPTotalDesignFlow~
#>   <chr>           <chr>    <chr>          <dbl>   <dbl>               <dbl>
#> 1 CENTRAL UTILIT~ TX00027~ 12070103        30.6   -96.3                0.93
#> 2 HEAT TRANSFER ~ TX01065~ 12070101        30.6   -96.4               NA   
#> 3 TURKEY CREEK W~ TX00624~ 12070101        30.6   -96.4                0.75
```

When returned as sf dataframes, the data is suitable for immediate
spatial plotting or analysis:

``` r
library(ggmap)
library(sf)
library(ggrepel)
## This example requires the development version of ggplot with support
## for geom_sf()
## and uses theme_ipsum_rc() from library(hrbrthemes)


df <- echoWaterGetFacilityInfo(output = "sf", 
                               xmin = '-96.387509', 
                               ymin = '30.583572', 
                               xmax = '-96.281422', 
                               ymax = '30.640008',
                               p_ptype = "NPD")

collegestation <- get_map(location = c(-96.387509, 30.583572,
                                       -96.281422, 30.640008), 
                          zoom = 14, maptype = "toner")

##to make labels, need to map the coords and use geom_text :(
## can't help but think there is an easier way to do this

df <- df %>%
  mutate(
    coords = map(geometry, st_coordinates),
    coords_x = map_dbl(coords, 1),
    coords_y = map_dbl(coords, 2)
  )

ggmap(collegestation) + 
  geom_sf(data = df, inherit.aes = FALSE, shape = 21, 
          color = "darkred", fill = "darkred", 
          size = 2, alpha = 0.25) +
  geom_label_repel(data = df, aes(x = coords_x, y = coords_y, label = SourceID),
                   point.padding = .5, min.segment.length = 0.1,
                   size = 2, color = "dodgerblue") +
  theme_ipsum_rc(plot_margin = margin(5, 5, 5, 5)) +
  labs(x = "Longitude", y = "Latitude", 
       title = "NPDES permits near Texas A&M",
       caption = "Source: EPA ECHO database")
```

<img src="man/figures/README-example3-1.png" width="672" />

### Download discharge/emissions data

Use `echoGetEffluent()` or `echoGetCAAPR()` to download tidy dataframes
of permitted water discharger Discharge Monitoring Report (DMR) or
permitted emitters Clean Air Act annual emissions reports. Please note
that all variables are returned as *character* vectors.

``` r
df <- echoGetEffluent(p_id = 'tx0119407', parameter_code = '00300')

df <- df %>%
  mutate(dmr_value_nmbr = as.numeric(dmr_value_nmbr),
         monitoring_period_end_date = as.Date(monitoring_period_end_date,
                                              "%m/%d/%Y")) %>%
  filter(!is.na(dmr_value_nmbr) & limit_value_type_code == "C1")

ggplot(df) +
  geom_line(aes(monitoring_period_end_date, dmr_value_nmbr)) +
  theme_ipsum_rc(grid = "Y") +
  labs(x = "Monitoring period date",
       y = "Dissolved oxygen concentration (mg/l)",
       title = "Reported minimum dissolved oxygen concentration",
       subtitle = "NPDES ID = TX119407",
       caption = "Source: EPA ECHO")
```

<img src="man/figures/README-unnamed-chunk-2-1.png" width="672" />

## Test Results

``` r
library(echor)

date()
#> [1] "Tue Nov 19 11:36:13 2019"

devtools::test()
#> v |  OK F W S | Context
#> 
/ |   0       | core functions return expected errors
v |   8       | core functions return expected errors
#> 
/ |   0       | core functions return expected objects
- |   1       | core functions return expected objects
\ |   2       | core functions return expected objects
| |   3       | core functions return expected objects
/ |   4       | core functions return expected objects
- |   5       | core functions return expected objects
\ |   6       | core functions return expected objects
| |   7       | core functions return expected objects
/ |   8       | core functions return expected objects
\ |  10       | core functions return expected objects
| |  11       | core functions return expected objects
/ |  12       | core functions return expected objects
v |  13       | core functions return expected objects [3.6 s]
#> 
#> == Results =============================================================================
#> Duration: 3.7 s
#> 
#> OK:       21
#> Failed:   0
#> Warnings: 0
#> Skipped:  0
```
