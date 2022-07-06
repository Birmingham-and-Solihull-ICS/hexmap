# hexmap
Hex maps for ICS using Simple Features

A markdown file and necessary csv files to build hexmpas of ICS. 

A bit of a dump I am afraid  - need to tidy it up.

# What is the purpose of this blog? 

I am going to introduce how to use Simple Features to look at creating bespoke maps using co-ordinate system grids. 


# What is the problem? 

It all started with trying to do a map of ICS to show variation but being flummoxed by the different size of each of them. London is always swamped or has to be mapped seperately, and North East always draws the eye as it is so big.

![ICS regional map](3039386_regions_980766.png)

# So what can we do?  Hexmapping

One approached used to address such variation is to use a hex map. Hex maps have become the standard when visualizing data where the sizing of the geographical region is unimportant. It is not new, here is an example from 1895, The Unification of London: The Need and the Remedy. (https://mappinglondon.co.uk/2013/hexagonal-map-of-london/)

Here is an example using UK constituencies <https://open-innovations.org/projects/hexmaps/constituencies/>.  As you can see visual wieght of Scotland has shrunk whist the more populous London takes up more space, but the map retains traces of the orignal geography, unlike a bar chart. Or visit this dynamic example of regular to hex geography by ONS https://www.ons.gov.uk/visualisations/nesscontent/dvc237/hex.html.

![UK constituency hex map](map.svg)
  
### Why hexagons? 


Hexagons are a good shape to work with as the distances between its points are regular due to its internal symmetry.

There are two orientations you can use pointed or flat tops. Basically rotate the hexagon by 90degrees. Pointed-top hexes take up more vertical room, so can be better for portrait orientation.The choice is yours depending on your aesthetics. You can learn more about hexagon geomtery at https://hexnet.org/content/hexagonal-geometry.

![Heaxgon geometry](hexgeomRW.png)
  
## Mapping the ICS geography

There are no tools to build hex maps from shapefiles. So you have to get out paper and pencil/ipad/hex stickers and a wall to map out the relative positions of the objects you want to map. I used a hex paper image and ipad. One free resource for printing hex paper is https://incompetech.com/graphpaper/hexagonal/ . 

There is no science to this, it really is what works best for the geography you are working with.  This is an example of my rough working where to put STP/ICS.   
   
![Heaxgon geometry](hexworking.png)
      
The co-ordinates are kept quite simple by setting the width and height to be 4 units. Then using the row and column positions (starting from the origin - bottom left) to build the co-ordinates. So in my mapping Cornwall is in row 1 and column 1, and so on.  

The end result of all this plotting is a table of with a row for every hex and 7 not 6 points, you have to complete the shape.

![Heaxgon geometry](hexcoords.png)   
 
 
```{r ics polygons, results='asis', echo=TRUE}

library(tidyverse) #tidyverse
library(knitr) #tidyverse

library(kableExtra) #tidyverse

#https://www.england.nhs.uk/integratedcare/integrated-care-in-your-area/
ics<-read_csv("ics_coords2.csv")


ics$x1=ics$col
ics$x2=ics$x1
ics$x3=ics$x2+2
ics$x4=ics$x3+2
ics$x5=ics$x4
ics$x6=ics$x5-2
ics$x7=ics$x1

ics$y1=ics$row*3-2
ics$y2=ics$y1+2
ics$y3=ics$y2+1
ics$y4=ics$y2
ics$y5=ics$y1
ics$y6=ics$y5-1
ics$y7=ics$y1


ics[1:5,c(1:4,12:25)]%>%arrange(row,col)%>%kbl(caption="ICS geometery")%>%
  kable_styling()

```


## SF workflow

<https://r-spatial.github.io/sf/index.html>

The key to creating polygons is that the coordinates have to be in sequence to form a closed area (i.e., the last point is the same as the first point).

So your data will need a bit of manipulation to create the coordinates, and put them in order. I ran into difficulties with this and used this solution. (<https://stackoverflow.com/questions/63566637/rror-in-funxi-is-numericx-is-not-true-when-using-st-polygon>)

There are three steps to creating a SF

-   st_polygon() - Create simple feature from a numeric vector, matrix or list

-   st_sfc() - Create simple feature geometry list column

-   st_sf() - Create sf object

Merge a Spatial object having a data.frame (i.e. merging of non-spatial attributes).

sp::merge is located in package sp. Please install and load package sp before us

```{r data entry, echo=TRUE}

library(sf)  #simple features
library(sp)  #simple features

#https://stackoverflow.com/questions/63566637/rror-in-funxi-is-numericx-is-not-true-when-using-st-polygon
class(ics) <- 'data.frame'

icsdata<-read_csv("ics_data.csv")

#https://stackoverflow.com/questions/44335246/polygons-from-coordinates
lst <- lapply(1:nrow(ics), function(x){
  ## create a matrix of coordinates that also 'close' the polygon
  res <- matrix(c(ics[x, 'x1'], ics[x, 'y1'],
                  ics[x, 'x2'], ics[x, 'y2'],
                  ics[x, 'x3'], ics[x, 'y3'],
                  ics[x, 'x4'], ics[x, 'y4'],
                  ics[x, 'x5'], ics[x, 'y5'],
                  ics[x, 'x6'], ics[x, 'y6'],
                  ics[x, 'x7'], ics[x, 'y7'])  ## need to close the polygon
                , ncol =2, byrow = T
  )
  ## create polygon objects
  st_polygon(list(res))
  
})

## st_sfc : creates simple features collection
## st_sf : creates simple feature object
sfdf <- st_sf(ICS22NM= ics[, 'ICS22NM'], st_sfc(lst))

#merge SF object to ics data frame and the centroids of the hexagon for the labels

ics_map<-sp::merge(sfdf,ics) %>% 
  mutate(labelX = map_dbl(geometry, ~st_centroid(.x)[[1]]),
         labelY = map_dbl(geometry, ~st_centroid(.x)[[2]]))

icsdata<-icsdata%>%mutate(value=round(value,1))
```
 
