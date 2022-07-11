### Mapping

DPR generated maps using the
{[leaflet](https://rstudio.github.io/leaflet/)} package ecosystem in R.
His function `make_suid_count_map()`, makes a chloropleth map that
overlays onto CartoDB basemap tiles and colors census tracts with a
{[viridis](https://sjmgarnier.github.io/viridis/index.html)}-derived
palette.

    ## function(suid_sf) {
    ##     
    ##     # Configure color palette
    ##     suid_palette <- 
    ##         leaflet::colorFactor(
    ##             palette = "magma",
    ##             reverse = TRUE,
    ##             levels = c(
    ##                 "No Deaths", 
    ##                 "One Death", 
    ##                 "Two Deaths", 
    ##                 "Three Deaths", 
    ##                 "Four Deaths", 
    ##                 "Five Deaths", 
    ##                 "Six+ Deaths"
    ##             )
    ##         )
    ##     
    ##     obj1 <-
    ##         # Assign map to a widget object
    ##         leaflet(suid_sf) |>
    ##             # Use CartoDB's background tiles
    ##             addProviderTiles("CartoDB.Positron") |>
    ##             # Center and zoom the map to Cook County
    ##             setView(lat = 41.816544, lng = -87.749500, zoom = 9) |>
    ##             # Add button to enable fullscreen map
    ##             leaflet.extras::addFullscreenControl() |>
    ##             # Add census tract polygons colored to reflect the number of deaths
    ##             addPolygons(
    ##                 # No borders to the polygons, just fill
    ##                 stroke = FALSE,
    ##                 # Color according to palette above
    ##                 color = ~ suid_palette(suid_count_factor),
    ##                 # Group polygons by number of deaths for use in the layer control
    ##                 group = ~ suid_count_factor,
    ##                 # Make slightly transparent
    ##                 fillOpacity = 0.5,
    ##                 # Click on the polygon to get its ID
    ##                 popup = ~ paste0("<b>FIPS ID:</b> ", as.character(fips))
    ##             ) |>
    ##             #Add legend
    ##             addLegend(
    ##                 title = "Count of SUID <br> per census tract <br> in Cook County, IL <br> from 2015-2019",
    ##                 values = ~ suid_count_factor,
    ##                 pal = suid_palette,
    ##                 position = "topright"
    ##             ) |>
    ##             # Add ability to toggle each factor grouping on or off the map
    ##             addLayersControl(
    ##                 overlayGroups = c(
    ##                     "No Deaths", 
    ##                     "One Death", 
    ##                     "Two Deaths", 
    ##                     "Three Deaths", 
    ##                     "Four Deaths", 
    ##                     "Five Deaths", 
    ##                     "Six+ Deaths"
    ##                 ),
    ##                 position = "topleft"
    ##             )
    ##     
    ##     return(obj1)
    ##     
    ## }
