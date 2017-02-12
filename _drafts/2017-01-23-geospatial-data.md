---
layout: post
title:  "Mapping crowded houses"
date:   2017-01-23 10:33:42 +1300
categories: data-journalism
tags:
  - geospatial
  - shapefiles
  - census
  - maps
author: Caleb Tutty
git_repo: https://github.com/tuttinator/test
---

Explore overcrowding in Auckland housing using census data and R to plot a choropleth map. [WIP]


## Hey now, hey now


Some of the most interesting projects I've worked on in data journalism involved getting my head around New Zealand's wealth of geospatial data.

This blog post will overview where to find spatial data, and work
through building a simple choropleth in [QGIS](https://qgis.org),
replicating this process in [R](https://www.r-project.org/) and building for the web with [D3.js](https://d3js.org/).


#### Spatial data in New Zealand

- [LINZ Data Service](https://data.linz.govt.nz/)
- [koordinates.com](https://koordinates.com)
- [Census boundary shapes](https://stats.govt.nz)


### Census area shapes


## There's something in the way you're talking

Packages you'll need to install:

```R
library(rgdal)
library(readxl)
library(dplyr)
library(maptools)
library(ggmap)
library(scales)
```


[Statistics NZ](http://www.stats.govt.nz/browse_for_stats/people_and_communities/housing/auckland-housing-1991-2013.aspx) has compiled a tables of crowding using the Canadian National Occupancy Standard for Auckland and Christchurch over the 2006 and 2013 censuses.

This example will look at:

Table 7:	Crowded households and average household size by Auckland area unit and Canadian National Occupancy Standard, 2013 Census
Table 8:	People in crowded households by Auckland area unit and Canadian National Occupancy Standard 2013 Census

but I encourage you to examine other aspects of this spreadsheet.

Using the `readxl` R package we will explore and tidy these dataframes and create choropleth maps as part of exploratory analysis.

```R
download.file("http://www.stats.govt.nz/~/media/Statistics/browse-categories/people-and-communities/housing/auckland-housing-2013/housing-in-auckland-tables-1991-to-2013.xlsx", "auckland_housing_tables.xlsx")

crowded_household_counts <- read_excel("auckland_housing_tables.xlsx", sheet = "Table 7", col_names = FALSE)

clean_start_row <- which(crowded_household_counts$X0 == "Auckland")
clean_end_row   <- which(crowded_household_counts$X1 == "Total")

crowded_household_counts <- crowded_household_counts[(clean_start_row + 1):clean_end_row,2:15]

colnames(crowded_household_counts) <- c("area_unit_name", "severely_crowded", "crowded", "total_crowded", "percent_crowded", "no_spare_rooms", "one_spare_room", "two_or_more_spare_rooms", "total_not_crowded", "percent_not_crowded", "total_stated", "unknown", "total_households", "average_household_size")

crowded_household_counts[crowded_household_counts == "..C"] <- NA

crowded_household_counts
```

|area_unit_name|	severely_crowded|	crowded|	total_crowded|	percent_crowded|
|--------------|------------------|--------|---------------|-----------------|
|Ferguson      |	141	            | 168    |  309          | 42.2            |
|Viscount	     |  126             |	183    |	312          | 41.8            |
|Otara North	 |  48              |	66     |	114          | 40.9            |
|Otara West    |	93              |	138    |	231          | 39.3            |
|Rongomai      |	144             |	204    |	348	         | 38.5            |


```R
# Note: this is a 184.6 MB download
download.file("http://www3.stats.govt.nz/digitalboundaries/annual/ESRI_Shapefile_2016_Digital_Boundaries_Generalised_Clipped.zip", "2016_shapefiles.zip")

unzip("2016_shapefiles.zip")

area_units  <- readShapePoly("2016 Digital Boundaries Generalised Clipped/AU2016_GV_Clipped.shp")
proj4string(area_units) <- CRS("+proj=tmerc +lat_0=0 +lon_0=173 +k=0.9996 +x_0=1600000 +y_0=10000000 +ellps=GRS80 +towgs84=0,0,0,0,0,0,0 +units=m +no_defs")

area_units <- spTransform(area_units, CRS("+proj=longlat +datum=WGS84"))
```


```R

percent_crowded <- data_frame(name = crowded_household_counts$area_unit_name,
                              percent = crowded_household_counts$percent_crowded)

# You could filter to exclude certain islands
# excluded_islands = c("Rakitu Island", "Great Barrier Island", "Little Barrier Island", "Kaikoura and Rangiahua Islands")

crowded_area_units <- fortify(area_units, region = "AU2016_NAM") %>%
                      filter(id %in% percent_crowded$name)

crowded_area_units <- merge(crowded_area_units, percent_crowded, by.x = "id", by.y = "name")

crowded_area_units$percent <- as.numeric(crowded_area_units$percent)

glimpse(crowded_area_units)
```

```{r, fig.width = 11, fig.height = 11}
ggplot() +
  geom_polygon(data = crowded_area_units,
               aes(x = long, y = lat, group = group, fill = percent),
               color = "#444444", size = 0.05) +
  coord_map() +
  scale_fill_gradient(name = "Percent", limits = c(0, 45), low = "#fee0d2", high = "#de2d26") +
  theme_nothing(legend = TRUE) +
  labs(title = "Auckland household crowding by Area Unit")
```


```{r, fig.width = 11, fig.height = 11}

area_unit_labels <- aggregate(cbind(long, lat) ~ percent, data=crowded_area_units, FUN = function(x) mean(range(x)))


ggplot() +
  geom_polygon(data = crowded_area_units,
               aes(x = long, y = lat, group = group, fill = percent),
               color = "#444444", size = 0.05) +
  coord_map(xlim = c(174.533, 175.036), ylim = c(-36.704, -37.126)) +
  scale_fill_gradient(name = "Percent", low = "#fee0d2", high = "#de2d26") +
  theme_nothing(legend = TRUE) +
  labs(title = "Auckland household crowding by Area Unit") +
  geom_text(data = area_unit_labels, aes(long, lat, label = ifelse(percent > 40, paste(percent, "%"), "")), size = 5, fontface = "bold")
```


```{r}

number_of_people_crowded <- read_excel("auckland_housing_tables.xlsx", sheet = "Table 8", col_names = FALSE)

clean_start_row <- which(number_of_people_crowded$X0 == "Auckland")
clean_end_row   <- which(number_of_people_crowded$X1 == "Total")

number_of_people_crowded <- number_of_people_crowded[(clean_start_row + 1):clean_end_row, 2:14]

colnames(number_of_people_crowded) <- c("area_unit_name", "severely_crowded", "crowded", "total_crowded", "percent_crowded", "no_spare_rooms", "one_spare_room", "two_or_more_spare_rooms", "total_not_crowded", "percent_not_crowded", "total_stated", "unknown", "total_people")

number_of_people_crowded[number_of_people_crowded == "..C"] <- NA

number_of_people_crowded
```


```R
total_crowded <- data_frame(name = number_of_people_crowded$area_unit_name,
                            total_crowded = crowded_household_counts$total_crowded)

# You could filter to exclude certain islands
# excluded_islands = c("Rakitu Island", "Great Barrier Island", "Little Barrier Island", "Kaikoura and Rangiahua Islands")

crowded_area_units <- fortify(area_units, region = "AU2016_NAM") %>%
                      filter(id %in% total_crowded$name)

crowded_area_units <- merge(crowded_area_units, total_crowded, by.x = "id", by.y = "name")

crowded_area_units$total_crowded <- as.numeric(crowded_area_units$total_crowded)
```

```R
area_unit_labels <- aggregate(cbind(long, lat) ~ total_crowded, data = crowded_area_units, FUN = function(x) mean(range(x)))


ggplot() +
  geom_polygon(data = crowded_area_units,
               aes(x = long, y = lat, group = group, fill = total_crowded),
               color = "#444444", size = 0.05) +
  coord_map(xlim = c(174.533, 175.036), ylim = c(-36.704, -37.126)) +
  scale_fill_gradient(name = "Count", low = "#fee0d2", high = "#de2d26") +
  theme_nothing(legend = TRUE) +
  labs(title = "Number of people in crowded households by area unit") +
  geom_text(data = area_unit_labels, aes(long, lat, label = ifelse(total_crowded > 200, total_crowded, "")), size = 5)

```

## Don't dream it's over
