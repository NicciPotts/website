---
categories:
- R
date: "2021-11-10"
description: this is meta description
draft: false
image: images/post/police.jpg
tags:
- R
- Mapping
- GIS
title: UK police force representation
type: featured
---

Using R and open source data to look into how representative UK police forces are of the populations they serve.
 
<!--more-->

In this blog I'll be looking at the [Diversity in Data](https://data.world/diversityindata/diversityindata-the-countdown-to-christmas-and-new-year) UK Police Diversity data set, originally from [gov.uk](https://www.gov.uk/). The data has the breakdown of ethnicity as a percentage of the police force and the percentage of the population for each police force. I took the boundary shapefiles for the police force areas from the [ONS Open Geography Portal](https://geoportal.statistics.gov.uk/datasets/ons::police-force-areas-december-2016-full-clipped-boundaries-in-england-and-wales-1).


> Full disclosure, I tried to get an API set-up between data.world and RStudio, and while it worked for the test dataset it couldn't make it work with this dataset. So instead I downloaded the .xtl file and used an online converter to change to an .xlsx file.



## Representation matters

As you are reading this, chances are I don't have to tell you that representation matters - but, well it does! 


The [police](https://www.joiningthepolice.co.uk/why-join/representing-our-communities) themselves say *"the more our police officers represent the communities we serve, the more we understand their needs and concerns and the better we work together to make communities safer and stronger."*



In this post, I will show how no police force is full representative of the community it serves. This is not unknown or indeed a ground-breaking analysis, but hopefully someone may be able to use this guide to examiner the data to help come up with new solutions or ideas. 
A key feature missing from this data is gender. Of the two per cent of Chief Officers that are BAME, none are female. There are also no female BAME Chief Superintendents.

## Breaking down the data



1. Figure out what cut of the data you want to look at. In this blog I want to look at the breakdown of people with one or more long-term health conditions by output area. 


2. Go to the Scotland Census website, [search by topic](https://www.scotlandscensus.gov.uk/search-the-census#/topics/list?topic=Health&categoryId=3) and look for that breakdown. Under the title is an alpha-numeric code, this code relates to the name of the .csv file in the bulk data download.  

<div><img src="../../images/post/census_capture.PNG" width="60%"/></div>

3. Having searched for and found the relevant .csv file, upload this into R - remembering to move the file into your R folder!

```
library(tidyverse)
library(sf)
library(rmapshaper)
library(leaflet)

Health <- 
   read_csv("data/QS304SC.csv")

```

4. Eventually we are going to merge this file with a shapefile for the output areas, I'm cheating a little with this step-by-step guide as we haven't yet imported (or downloaded) the shapefile but *spoiler alert* we want to rename the column that contains the output area code.

```

Health <- 
   Health %>% 
   rename('code' = X1)

```

5. What I want to do is look at the absolute number of people with one or more long-term health conditions, and how this translates as a percentage of the output area population.

```
Health_df1 <-
  Health %>% 
  group_by(code) %>%
  mutate(abnum = `One or more conditions`,
        pct = round(abnum / `All people` *100, 1)) %>%
  select(code, abnum, pct)

```


6. Now the data is wrangled we will need some geographical coordinates for the output areas to make a map. I downloaded the coastline clipped version of the output area shapefile from the [National Records of Scotland.](https://www.nrscotland.gov.uk/statistics-and-data/geography/our-products/census-datasets/2011-census/2011-boundaries) 

```

OALayer <-
  st_read("data/OutputArea2011_MHW.shp",
          layer = "OutputArea2011_MHW")

OALayer <-
  st_transform(OALayer, "+init=epsg:4326")

```

7. I also use `rmapshaper` to simplify the shapefiles, just for quicker loading/rendering as the exact boundaries aren't too important to me at this stage. 


```
SimpleOA <-
  ms_simplify(OALayer) 
  
```

8. For this I am only going to look at the Ayrshires (North, South, and East) so rather than merge the entire shapefile and data, I filter the shapefile to the council codes of interest (I used [this table](https://www.opendata.nhs.scot/km/dataset/geography-codes-and-labels/resource/967937c4-8d67-4f39-974f-fd58c4acfda5) to find the council codes). When the two files are merged it'll only merge the output areas for the filtered councils. 


```

AyrshireOA <- 
    SimpleOA %>% 
    filter(council == c('S12000008', 
                    'S12000021',
                    'S12000028'))
                    
                    
Health_shp <- 
  merge(AyrshireOA, Health_df1, by='code')

```

And that's it! Nice and simple, right? 

I know this probably seems like a lot of work and prep to get to this stage, but once you've put in the initial effort you will be set up to just switch out the census data.

And I promise it'll all be worth the effort!


### Mapping those with long-term health conditions

Before adding in the polling station data I think we've earned a map to look at - so first we'll map the census data.

1. We'll need to define a palette -


```
bin <-
  colorBin('Oranges', domain = Health_shp$abnum)
```

2. Then we can build the map! I'm calling up the data in `addPolygon()` and `addLegend()` because my intention is to later add further layers from different data sources. 


> There are much better [resources](https://rstudio.github.io/leaflet/) out there for learning the in's and out's of the `leaflet` package, my intention is for you to be able to see the basics of this code being applied to this data set.




```
map <- leaflet() %>%
  addTiles() %>%
  addPolygons(data = Health_shp, 
              color= "grey", 
              weight = 0.75,  
              smoothFactor = 0.5, 
              opacity = 1.0,
              fillColor = ~ bin(abnum), 
              fillOpacity = 1, 
              popup = ~paste(abnum),
              highlightOptions = 
              highlightOptions(color = "white", 
                               weight = 1)
  ) %>%
  
  addLegend(data = Health_shp, 
            "bottomright", 
            pal = bin, 
            values = ~ abnum, 
            title = "Absolute number </br> 
            of long-term illness </br> population", 
            opacity = 0.6 
  )

map


```

<div><img src="../../images/post/ayrshire_lth.png" width="100%"/></div>


## Polling Station data

Now we know where people with a long-term health condition live across Ayrshire, let's see how this relates to polling stations from the 2021 Scottish Government election.


> Disabled people are more likely to vote by post (35%) compared to non-disabled people (19%). 

1. The polling station data for Scotland is downloaded from the [spatiial hub](https://data.spatialhub.scot/dataset/polling_places-is) and imported into R.


```

PollLayer <-
  st_read("data/pub_polpl.shp", layer = "pub_polpl")
  
PollLayer <-
  st_transform(PollLayer, "+init=epsg:4326") 

```

2. Then the polling stations are filtered to only those in Ayrshire. 


```
Ayrshire_Poll <- 
  PollLayer %>% 
  filter(local_auth %in% 
           c('North Ayrshire', 
             'South Ayrshire', 
             'East Ayrshire'))
```

3. And finally we can add these to our map.

```
map %>%
  addCircleMarkers(data = Ayrshire_Poll)
```




<div><img src="../../images/post/ayrshire_polls.png" width="100%"/></div>

### How do we use this?

We know that:

- Disabled people are less likely to use public transport;
- And those with long-term health conditions are more likely to make short public transport journeys only.
- People with a long-term health condition are also less likely to have access to a vehicle (46% no car/van compared to 19% of those with no illness).

> [Transport Scotland](https://www.transport.gov.scot/publication/going-further-scotland-s-accessible-travel-framework/j448711-05/) have an insightful summary on some of the barriers for disabled people using public transport 


Of course, not all disabled people encounter barriers to travelling; however, in areas with high numbers of people with long-term illness and no nearby polling station:

- what is the voting rate of the general output area population?
- what is the voting rate of those with long-term health conditions?


And then across Ayrshire - 

- do areas with high absolute and relative numbers of people with long-term health conditions that also have polling stations, have higher voting rates? 


If voting rates are comparably lower amongst those with long-term health conditions and no local polling stations, could this indicate the locations of the polling stations aren't accessible to all?

> In 2017, those with limiting health conditions  were less likely to confidently report they would vote in the upcoming UK General Election.

I should also state two other things:

1. I have no idea how the decision to locate polling stations is actually made, I imagine population figures and availability of suitable buildings plays a large role. I'm using this data as an example only!

2. Just because a polling station is physically located in an accessible place, it doesn't mean that the building nor the voting process itself is accessible


Polling stations can be replaced with any building, facility, or provision that is of interest, and the underlying census data can be a range of different demographics.


This post is intended to provide a guide on how census can be an insightful tool when we want to build opportunities that intend to serve communities; by taking that first step and asking where and how? Where are the people we want to engage? And how do we reduce/remove any barriers to ensure equitable access to our service?

*Summary image by <a href="https://unsplash.com/@shanerounce?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Shane Rounce</a> on <a href="https://unsplash.com/s/photos/diversity?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>*
  
Photo by <a href="https://unsplash.com/@scottrodgerson?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Scott Rodgerson</a> on <a href="https://unsplash.com/s/photos/police?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  
