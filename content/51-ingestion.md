### Ingestion

At time of primary analysis, DPR combined a data file generated by
collaborator HZ and additional data pulled via the R package
{[tidycensus](https://walker-data.com/tidycensus/index.html)}.

DPR imported HZ’s excel file into the Targets pipeline using the
function `readxl::read_xls()`. He pulled from the tidycensus API using
his function `get_suid_from_tidycensus()`. This function asks for
variables from the American Community Survey’s 5-year estimate for
2015-2019. The variables describe count of total population, count of
population under age 5 years, average number of people per household,
and geospatial shape of census tracts in Cook County, IL. The function
also does some reshaping of data as noted in the code comments below:

    ## function() {
    ##     
    ##     df1 <-
    ##         # Call the census API
    ##         tidycensus::get_acs(
    ##             geography = "tract",
    ##             variables = c(
    ##                 "B01003_001", # TOTAL POPULATION
    ##                 "B06001_002",  # Total Under 5 years
    ##                 "B25010_001" # AVERAGE HOUSEHOLD SIZE OF OCCUPIED HOUSING UNITS
    ##             ),
    ##             state = "IL", # Illinois
    ##             county = 031, # Cook County
    ##             geometry = TRUE # Import census tract polygons too
    ##         ) |> 
    ##         st_transform(crs = 4326) |> 
    ##         # Drop margin of estimate and name variables
    ##         select(-moe, -NAME) |>
    ##         # Convert GEOID to numeric type
    ##         mutate(
    ##             GEOID = as.numeric(GEOID)
    ##         ) |>
    ##         # Reshape variable estimates into separate columns
    ##         tidyr::pivot_wider(
    ##             names_from = variable,
    ##             values_from = estimate
    ##         ) |>
    ##         # Clean names for consistency
    ##         rename(
    ##             fips = GEOID,
    ##             pop_total = B01003_001,
    ##             pop_under_five = B06001_002,
    ##             avg_peop_per_household = B25010_001
    ##         ) |> 
    ##         # Put geometry column at the end of the table
    ##         relocate(
    ##             geometry,
    ##             .after = avg_peop_per_household
    ##         ) |> 
    ##         st_as_sf()
    ##     
    ##     return(df1)
    ## }

DPR combined the two data sources with his function `assemble_suid()`,
which performs a left join on FIPS ID for observations from HZ’s excel
data, then does some wrangling around naming conventions, adds a boolean
variable for whether SUID is present in a given tract, and rounds
estimates to the first decimal point for consistency:

    ## function(suid_from_internal_raw_df, suid_from_tidycensus_raw_sf) {
    ##     
    ##     df1 <- 
    ##         left_join(
    ##             suid_from_internal_raw_df,
    ##             suid_from_tidycensus_raw_sf,
    ##             by = c("FIPS" = "fips")
    ##         ) |>
    ##         clean_names() |> 
    ##         # Rename variables for consistency
    ##         rename(
    ##             suid_count = count_asphyxia,
    ##             foreign_born = pe_foreignborn,
    ##             married_males = pe_marriedmales,
    ##             married_females = pe_marriedfemales,
    ##             divorced_widowed_males = pedivorcewidowedmale,
    ##             divorced_widowed_females = pedivorcewidowedfemale,
    ##             lt_high_school = pelessthanhighschool,
    ##             high_school_diploma = highschooldiploma,
    ##             some_college = somecollege,
    ##             college_diploma = collegediploma,
    ##             employed = percent_enployed,
    ##             income_lt_10 = incomelt10,
    ##             income_lt_25 = incomelt25,
    ##             income_lt_50 = incomelt50,
    ##             income_lt_75 = incomelt75,
    ##             income_gt_75 = incomegt75,
    ##             private_insurance = privateinsurance,
    ##             public_insurance = publicinsurance,
    ##             no_insurance = noinsurance
    ##         ) |> 
    ##         # Add new variables
    ##         mutate(
    ##             # Binary variable on whether suid is present in a tract
    ##             suid_present = case_when(
    ##                 suid_count > 0 ~ TRUE,
    ##                 TRUE ~ FALSE
    ##             ),
    ##             suid_count_factor = factor(
    ##                 case_when(
    ##                     suid_count == 0 ~ "No Deaths",
    ##                     suid_count == 1 ~ "One Death",
    ##                     suid_count == 2 ~ "Two Deaths",
    ##                     suid_count == 3 ~ "Three Deaths",
    ##                     suid_count == 4 ~ "Four Deaths",
    ##                     suid_count == 5 ~ "Five Deaths",
    ##                     suid_count > 5 ~ "Six+ Deaths"
    ##                 ),
    ##                 ordered = TRUE,
    ##                 levels = c(
    ##                     "No Deaths", 
    ##                     "One Death", 
    ##                     "Two Deaths", 
    ##                     "Three Deaths", 
    ##                     "Four Deaths", 
    ##                     "Five Deaths", 
    ##                     "Six+ Deaths"
    ##                 )
    ##             ),
    ##             approx_suid_incidence =
    ##                 round(
    ##                     suid_count / (pop_under_five / 5) * 1000,
    ##                     2
    ##                 ),
    ##             across(
    ##                 .cols = starts_with("svi_"),
    ##                 .fns = ~ round((.x * 100), digits = 1)
    ##             )
    ##         ) |> 
    ##         st_as_sf()
    ##     
    ##     return(df1)
    ## }
