## Supplement 2: Model Selection

    targets::tar_load(suid)
    library(dplyr)
    library(magrittr)

This supplement lays out how we selected the final model for predicting
the number of SUID cases per census tract (variable `suid_count`).

From inception, we included in all models the variable `pop_under_five`
(population of all children under age 5 years per census tract), which
served as a quasi-exposure variable to account for variation in the
number of people susceptible to an SUID incident.

### Predictor Variable Candidates

To select predictor variable candidates, we sought to balance using
variables that were highly correlated with `suid_count`, but not too
highly correlated to each other to reduce
[multicollinearity](https://www.statology.org/multicollinearity-regression/).

We started by generating a correlation dataframe and filtering for
variables that had at least a weak association (&gt; 0.20) with
`suid_count`:

    suid_correlations <-
        suid |>
        as_tibble() |> 
        select(
            -fips, 
            -geometry, 
            -suid_present,
            -suid_count_factor
        ) |>
        relocate(suid_count) |> 
        corrr::correlate() |>
        filter(abs(suid_count) > 0.20) |>
        arrange(desc(abs(suid_count)))

    ## 
    ## Correlation method: 'pearson'
    ## Missing treated using: 'pairwise.complete.obs'

    suid_correlations

    ## # A tibble: 17 × 35
    ##    term        suid_…¹ count…² svi_s…³
    ##    <chr>         <dbl>   <dbl>   <dbl>
    ##  1 public_ins…   0.362   0.354   0.819
    ##  2 white        -0.349  -0.332  -0.721
    ##  3 black         0.347   0.335   0.561
    ##  4 private_in…  -0.323  -0.356  -0.915
    ##  5 svi_househ…   0.311   0.319   0.672
    ##  6 income_gt_…  -0.310  -0.342  -0.898
    ##  7 married_fe…  -0.296  -0.355  -0.619
    ##  8 svi_socioe…   0.294   0.334  NA    
    ##  9 income_lt_…   0.293   0.308   0.747
    ## 10 svi_summar…   0.291   0.349   0.941
    ## 11 employed     -0.291  -0.277  -0.625
    ## 12 income_lt_…   0.286   0.345   0.565
    ## 13 married_ma…  -0.286  -0.350  -0.621
    ## 14 college_di…  -0.272  -0.300  -0.856
    ## 15 count_opio…   0.248  NA       0.334
    ## 16 some_colle…   0.234   0.231   0.432
    ## 17 high_schoo…   0.217   0.243   0.714
    ## # … with 31 more variables:
    ## #   svi_household_composition_disability <dbl>,
    ## #   svi_minority_language <dbl>,
    ## #   svi_housing_transportation <dbl>,
    ## #   svi_summary_ranking <dbl>,
    ## #   foreign_born <dbl>,
    ## #   married_males <dbl>, …
    ## # ℹ Use `colnames()` to see all variable names

`publicinsurance`, the percentage of residents in each census tract on
public insurance, had the strongest correlation with `suid_count`.

Here is the relationship visualized:

    plot(correlation::cor_test(as_tibble(suid), "suid_count", "public_insurance"))

![](/home/riggins/cook_county_sids_mortality/markdown_output/53-model_selection_files/figure-markdown_strict/unnamed-chunk-3-1.png)

In order to minimize multicollinearity with additional predictor
variables, we next selected for variables that had some degree of
correlation with `suid_count`, but had no more than weak correlation
(&lt; 0.5) with `publicinsurance`:

    suid_correlations |>
        filter(abs(suid_count) > 0.20) |>
        filter(abs(public_insurance) < 0.5)

    ## # A tibble: 1 × 35
    ##   term         suid_…¹ count…² svi_s…³
    ##   <chr>          <dbl>   <dbl>   <dbl>
    ## 1 count_opioi…   0.248      NA   0.334
    ## # … with 31 more variables:
    ## #   svi_household_composition_disability <dbl>,
    ## #   svi_minority_language <dbl>,
    ## #   svi_housing_transportation <dbl>,
    ## #   svi_summary_ranking <dbl>,
    ## #   foreign_born <dbl>,
    ## #   married_males <dbl>, …
    ## # ℹ Use `colnames()` to see all variable names

This just left `count_opioid_death`, the count of opioid-related deaths
in each census tract.

    plot(correlation::cor_test(as_tibble(suid), "suid_count", "count_opioid_death"))

![](/home/riggins/cook_county_sids_mortality/markdown_output/53-model_selection_files/figure-markdown_strict/unnamed-chunk-5-1.png)

There was a large outlier of many opioid deaths in a tract with no SUID
deaths, but otherwise, there seemed to be a strong positive trend.

To fill out our candidates, we selected the next four variables that
correlated most with `suid_count`, without being fully redundant
(e.g. not selecting `black` when `white` was already in the list):

-   `white` = the percentage of residents in each census tract
    identifying their race as White
-   `svi_household_composition_disability` = percentile ranking for each
    census tract on the [Social Vulnerability
    Index](https://www.atsdr.cdc.gov/placeandhealth/svi/documentation/pdf/SVI2018Documentation_01192022_1.pdf)
    for Household Composition & Disability, which is a mash-up of
    information about households that include people older than 65,
    people younger than 17, people with disabilities, and/or
    single-parents
-   `income_gt_75` = the percentage of residents in each census tract
    whose income were greater than the national 75th percentile
-   `married_females` = the percentage of female residents who were
    married in each census tract

Here are the summary statistics and visualized distributions of our
candidate predictors:

    as_tibble(suid) |>
        select(
            pop_under_five,
            public_insurance,
            count_opioid_death,
            white,
            svi_household_composition_disability,
            income_gt_75,
            married_females
        ) %T>%
        DataExplorer::plot_histogram() |>
        summary()

![](/home/riggins/cook_county_sids_mortality/markdown_output/53-model_selection_files/figure-markdown_strict/unnamed-chunk-6-1.png)

    ##  pop_under_five   public_insurance
    ##  Min.   :   0.0   Min.   : 2.50   
    ##  1st Qu.: 127.0   1st Qu.:24.80   
    ##  Median : 213.5   Median :34.70   
    ##  Mean   : 237.7   Mean   :36.40   
    ##  3rd Qu.: 318.0   3rd Qu.:47.65   
    ##  Max.   :1011.0   Max.   :84.80   
    ##  NA's   :31                       
    ##  count_opioid_death     white      
    ##  Min.   : 0.000     Min.   : 0.00  
    ##  1st Qu.: 1.000     1st Qu.:24.00  
    ##  Median : 3.000     Median :59.50  
    ##  Mean   : 3.957     Mean   :52.84  
    ##  3rd Qu.: 5.000     3rd Qu.:81.00  
    ##  Max.   :70.000     Max.   :99.00  
    ##                                    
    ##  svi_household_composition_disability
    ##  Min.   :  0.10                      
    ##  1st Qu.: 19.30                      
    ##  Median : 43.60                      
    ##  Mean   : 46.02                      
    ##  3rd Qu.: 71.75                      
    ##  Max.   :100.00                      
    ##                                      
    ##   income_gt_75   married_females
    ##  Min.   : 0.70   Min.   : 1.90  
    ##  1st Qu.:23.50   1st Qu.:26.10  
    ##  Median :37.30   Median :39.60  
    ##  Mean   :39.33   Mean   :37.77  
    ##  3rd Qu.:54.50   3rd Qu.:48.90  
    ##  Max.   :90.60   Max.   :75.20  
    ## 

### Model Type

We explored the general family of models that expect an outcome
distribution to be a [count
variable](https://thomaselove.github.io/432-notes/modeling-a-count-outcome-in-ohio-smart.html#a-tobit-censored-regression-model).
The distribution types included Poisson, Negative Binomial, and their
zero-inflated variants.

First, we compared each model type on its predictive performance using
just the exposure variable and an intercept:

    list(
        poisson = 
            glm(
                suid_count ~ pop_under_five, 
                family = poisson(), 
                data = suid
            ),
        zero_infl_poisson = 
            pscl::zeroinfl(
                suid_count ~ pop_under_five,
                data = suid
            ),
        neg_bin = 
            MASS::glm.nb(
                suid_count ~ pop_under_five,
                data = suid
            ),
        zero_infl_neg_bin = 
            pscl::zeroinfl(
                suid_count ~ pop_under_five,
                dist = "negbin",
                data = suid
            )
    ) |> 
    compare_performance() |>  
    print_html()

<div id="ehvycgcjsr" style="overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>html {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Helvetica Neue', 'Fira Sans', 'Droid Sans', Arial, sans-serif;
}

#ehvycgcjsr .gt_table {
  display: table;
  border-collapse: collapse;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#ehvycgcjsr .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#ehvycgcjsr .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#ehvycgcjsr .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 0;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#ehvycgcjsr .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#ehvycgcjsr .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#ehvycgcjsr .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#ehvycgcjsr .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#ehvycgcjsr .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#ehvycgcjsr .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#ehvycgcjsr .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#ehvycgcjsr .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
}

#ehvycgcjsr .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#ehvycgcjsr .gt_from_md > :first-child {
  margin-top: 0;
}

#ehvycgcjsr .gt_from_md > :last-child {
  margin-bottom: 0;
}

#ehvycgcjsr .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#ehvycgcjsr .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#ehvycgcjsr .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#ehvycgcjsr .gt_row_group_first td {
  border-top-width: 2px;
}

#ehvycgcjsr .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#ehvycgcjsr .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#ehvycgcjsr .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#ehvycgcjsr .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#ehvycgcjsr .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#ehvycgcjsr .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#ehvycgcjsr .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#ehvycgcjsr .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#ehvycgcjsr .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#ehvycgcjsr .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-left: 4px;
  padding-right: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#ehvycgcjsr .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#ehvycgcjsr .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#ehvycgcjsr .gt_left {
  text-align: left;
}

#ehvycgcjsr .gt_center {
  text-align: center;
}

#ehvycgcjsr .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#ehvycgcjsr .gt_font_normal {
  font-weight: normal;
}

#ehvycgcjsr .gt_font_bold {
  font-weight: bold;
}

#ehvycgcjsr .gt_font_italic {
  font-style: italic;
}

#ehvycgcjsr .gt_super {
  font-size: 65%;
}

#ehvycgcjsr .gt_two_val_uncert {
  display: inline-block;
  line-height: 1em;
  text-align: right;
  font-size: 60%;
  vertical-align: -0.25em;
  margin-left: 0.1em;
}

#ehvycgcjsr .gt_footnote_marks {
  font-style: italic;
  font-weight: normal;
  font-size: 75%;
  vertical-align: 0.4em;
}

#ehvycgcjsr .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#ehvycgcjsr .gt_slash_mark {
  font-size: 0.7em;
  line-height: 0.7em;
  vertical-align: 0.15em;
}

#ehvycgcjsr .gt_fraction_numerator {
  font-size: 0.6em;
  line-height: 0.6em;
  vertical-align: 0.45em;
}

#ehvycgcjsr .gt_fraction_denominator {
  font-size: 0.6em;
  line-height: 0.6em;
  vertical-align: -0.05em;
}
</style>
<table class="gt_table">
  <thead class="gt_header">
    <tr>
      <th colspan="13" class="gt_heading gt_title gt_font_normal gt_bottom_border" style>Comparison of Model Performance Indices</th>
    </tr>
    
  </thead>
  <thead class="gt_col_headings">
    <tr>
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="1" colspan="1">Name</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">Model</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">AIC</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">AIC weights</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">BIC</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">BIC weights</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">RMSE</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">Sigma</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">Score_log</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">Score_spherical</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">R2</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">R2 (adj.)</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">Nagelkerke's R2</th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td class="gt_row gt_left">poisson</td>
<td class="gt_row gt_center">glm</td>
<td class="gt_row gt_center">1674.76</td>
<td class="gt_row gt_center">&lt; 0.001</td>
<td class="gt_row gt_center">1685.07</td>
<td class="gt_row gt_center">&lt; 0.001</td>
<td class="gt_row gt_center">0.62</td>
<td class="gt_row gt_center">0.95</td>
<td class="gt_row gt_center">-0.65</td>
<td class="gt_row gt_center">0.03</td>
<td class="gt_row gt_center"></td>
<td class="gt_row gt_center"></td>
<td class="gt_row gt_center">6.66e-04</td></tr>
    <tr><td class="gt_row gt_left">zero_infl_poisson</td>
<td class="gt_row gt_center">zeroinfl</td>
<td class="gt_row gt_center">1595.73</td>
<td class="gt_row gt_center">0.003</td>
<td class="gt_row gt_center">1616.37</td>
<td class="gt_row gt_center">&lt; 0.001</td>
<td class="gt_row gt_center">0.62</td>
<td class="gt_row gt_center">0.62</td>
<td class="gt_row gt_center">-0.62</td>
<td class="gt_row gt_center">0.03</td>
<td class="gt_row gt_center">1.01e-04</td>
<td class="gt_row gt_center">-1.46e-03</td>
<td class="gt_row gt_center"></td></tr>
    <tr><td class="gt_row gt_left">neg_bin</td>
<td class="gt_row gt_center">negbin</td>
<td class="gt_row gt_center">1584.44</td>
<td class="gt_row gt_center">0.829</td>
<td class="gt_row gt_center">1599.92</td>
<td class="gt_row gt_center">0.999</td>
<td class="gt_row gt_center">0.62</td>
<td class="gt_row gt_center">0.74</td>
<td class="gt_row gt_center">-0.63</td>
<td class="gt_row gt_center">0.03</td>
<td class="gt_row gt_center"></td>
<td class="gt_row gt_center"></td>
<td class="gt_row gt_center">6.21e-04</td></tr>
    <tr><td class="gt_row gt_left">zero_infl_neg_bin</td>
<td class="gt_row gt_center">zeroinfl</td>
<td class="gt_row gt_center">1587.63</td>
<td class="gt_row gt_center">0.168</td>
<td class="gt_row gt_center">1613.42</td>
<td class="gt_row gt_center">0.001</td>
<td class="gt_row gt_center">0.62</td>
<td class="gt_row gt_center">0.62</td>
<td class="gt_row gt_center">-0.62</td>
<td class="gt_row gt_center">0.03</td>
<td class="gt_row gt_center">1.22e-04</td>
<td class="gt_row gt_center">-1.44e-03</td>
<td class="gt_row gt_center"></td></tr>
  </tbody>
  <tfoot class="gt_sourcenotes">
    <tr>
      <td class="gt_sourcenote" colspan="13"></td>
    </tr>
  </tfoot>
  
</table>
</div>

Although the models seemed generally comparable, the negative binomial
model type had both the best Akaike Information Criterion (AIC) and
Bayesian Information Criterion (BIC) scores. Minimizing these two scores
theoretically optimizes balance between over- and under-fitting to the
observed data.

Next, we compared performance of a nested sequence of predictor
candidates:

    list(
        base_formula = suid_count ~ pop_under_five,
        base_formula_plus_one = suid_count ~ pop_under_five + public_insurance,
        base_formula_plus_two = suid_count ~ pop_under_five + public_insurance + count_opioid_death,
        base_formula_plus_three = suid_count ~ pop_under_five + public_insurance + count_opioid_death + white,
        base_formula_plus_four = suid_count ~ pop_under_five + public_insurance + count_opioid_death + white + svi_household_composition_disability,
        base_formula_plus_five = suid_count ~ pop_under_five + public_insurance + count_opioid_death + white + svi_household_composition_disability + income_gt_75,
        base_formula_plus_six = suid_count ~ pop_under_five + public_insurance + count_opioid_death + white + svi_household_composition_disability + income_gt_75 + married_females
    ) |> 
        purrr::map(~MASS::glm.nb(.x, data = suid)) |> 
        compare_performance(rank = TRUE) |>  
        print_html() 

<div id="xrkdwxnaxw" style="overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>html {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Helvetica Neue', 'Fira Sans', 'Droid Sans', Arial, sans-serif;
}

#xrkdwxnaxw .gt_table {
  display: table;
  border-collapse: collapse;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#xrkdwxnaxw .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#xrkdwxnaxw .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#xrkdwxnaxw .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 0;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#xrkdwxnaxw .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#xrkdwxnaxw .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#xrkdwxnaxw .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#xrkdwxnaxw .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#xrkdwxnaxw .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#xrkdwxnaxw .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#xrkdwxnaxw .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#xrkdwxnaxw .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
}

#xrkdwxnaxw .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#xrkdwxnaxw .gt_from_md > :first-child {
  margin-top: 0;
}

#xrkdwxnaxw .gt_from_md > :last-child {
  margin-bottom: 0;
}

#xrkdwxnaxw .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#xrkdwxnaxw .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#xrkdwxnaxw .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#xrkdwxnaxw .gt_row_group_first td {
  border-top-width: 2px;
}

#xrkdwxnaxw .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#xrkdwxnaxw .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#xrkdwxnaxw .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#xrkdwxnaxw .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#xrkdwxnaxw .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#xrkdwxnaxw .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#xrkdwxnaxw .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#xrkdwxnaxw .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#xrkdwxnaxw .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#xrkdwxnaxw .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-left: 4px;
  padding-right: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#xrkdwxnaxw .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#xrkdwxnaxw .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#xrkdwxnaxw .gt_left {
  text-align: left;
}

#xrkdwxnaxw .gt_center {
  text-align: center;
}

#xrkdwxnaxw .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#xrkdwxnaxw .gt_font_normal {
  font-weight: normal;
}

#xrkdwxnaxw .gt_font_bold {
  font-weight: bold;
}

#xrkdwxnaxw .gt_font_italic {
  font-style: italic;
}

#xrkdwxnaxw .gt_super {
  font-size: 65%;
}

#xrkdwxnaxw .gt_two_val_uncert {
  display: inline-block;
  line-height: 1em;
  text-align: right;
  font-size: 60%;
  vertical-align: -0.25em;
  margin-left: 0.1em;
}

#xrkdwxnaxw .gt_footnote_marks {
  font-style: italic;
  font-weight: normal;
  font-size: 75%;
  vertical-align: 0.4em;
}

#xrkdwxnaxw .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#xrkdwxnaxw .gt_slash_mark {
  font-size: 0.7em;
  line-height: 0.7em;
  vertical-align: 0.15em;
}

#xrkdwxnaxw .gt_fraction_numerator {
  font-size: 0.6em;
  line-height: 0.6em;
  vertical-align: 0.45em;
}

#xrkdwxnaxw .gt_fraction_denominator {
  font-size: 0.6em;
  line-height: 0.6em;
  vertical-align: -0.05em;
}
</style>
<table class="gt_table">
  <thead class="gt_header">
    <tr>
      <th colspan="10" class="gt_heading gt_title gt_font_normal gt_bottom_border" style>Comparison of Model Performance Indices</th>
    </tr>
    
  </thead>
  <thead class="gt_col_headings">
    <tr>
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="1" colspan="1">Name</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">Model</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">Nagelkerke's R2</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">RMSE</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">Sigma</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">Score_log</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">Score_spherical</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">AIC weights</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">BIC weights</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">Performance-Score</th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td class="gt_row gt_left">base_formula_plus_three</td>
<td class="gt_row gt_center">negbin</td>
<td class="gt_row gt_center">0.34</td>
<td class="gt_row gt_center">0.59</td>
<td class="gt_row gt_center">0.74</td>
<td class="gt_row gt_center">-0.53</td>
<td class="gt_row gt_center">0.03</td>
<td class="gt_row gt_center">0.387</td>
<td class="gt_row gt_center">0.963</td>
<td class="gt_row gt_center">82.02%</td></tr>
    <tr><td class="gt_row gt_left">base_formula_plus_five</td>
<td class="gt_row gt_center">negbin</td>
<td class="gt_row gt_center">0.35</td>
<td class="gt_row gt_center">0.58</td>
<td class="gt_row gt_center">0.74</td>
<td class="gt_row gt_center">-0.53</td>
<td class="gt_row gt_center">0.03</td>
<td class="gt_row gt_center">0.325</td>
<td class="gt_row gt_center">0.005</td>
<td class="gt_row gt_center">67.65%</td></tr>
    <tr><td class="gt_row gt_left">base_formula_plus_six</td>
<td class="gt_row gt_center">negbin</td>
<td class="gt_row gt_center">0.35</td>
<td class="gt_row gt_center">0.58</td>
<td class="gt_row gt_center">0.74</td>
<td class="gt_row gt_center">-0.53</td>
<td class="gt_row gt_center">0.03</td>
<td class="gt_row gt_center">0.122</td>
<td class="gt_row gt_center">&lt; 0.001</td>
<td class="gt_row gt_center">59.69%</td></tr>
    <tr><td class="gt_row gt_left">base_formula_plus_four</td>
<td class="gt_row gt_center">negbin</td>
<td class="gt_row gt_center">0.34</td>
<td class="gt_row gt_center">0.59</td>
<td class="gt_row gt_center">0.74</td>
<td class="gt_row gt_center">-0.53</td>
<td class="gt_row gt_center">0.03</td>
<td class="gt_row gt_center">0.165</td>
<td class="gt_row gt_center">0.031</td>
<td class="gt_row gt_center">59.69%</td></tr>
    <tr><td class="gt_row gt_left">base_formula_plus_two</td>
<td class="gt_row gt_center">negbin</td>
<td class="gt_row gt_center">0.32</td>
<td class="gt_row gt_center">0.62</td>
<td class="gt_row gt_center">0.74</td>
<td class="gt_row gt_center">-0.54</td>
<td class="gt_row gt_center">0.03</td>
<td class="gt_row gt_center">&lt; 0.001</td>
<td class="gt_row gt_center">&lt; 0.001</td>
<td class="gt_row gt_center">39.77%</td></tr>
    <tr><td class="gt_row gt_left">base_formula_plus_one</td>
<td class="gt_row gt_center">negbin</td>
<td class="gt_row gt_center">0.30</td>
<td class="gt_row gt_center">0.58</td>
<td class="gt_row gt_center">0.75</td>
<td class="gt_row gt_center">-0.54</td>
<td class="gt_row gt_center">0.03</td>
<td class="gt_row gt_center">&lt; 0.001</td>
<td class="gt_row gt_center">&lt; 0.001</td>
<td class="gt_row gt_center">39.47%</td></tr>
    <tr><td class="gt_row gt_left">base_formula</td>
<td class="gt_row gt_center">negbin</td>
<td class="gt_row gt_center">6.21e-04</td>
<td class="gt_row gt_center">0.62</td>
<td class="gt_row gt_center">0.74</td>
<td class="gt_row gt_center">-0.63</td>
<td class="gt_row gt_center">0.03</td>
<td class="gt_row gt_center">&lt; 0.001</td>
<td class="gt_row gt_center">&lt; 0.001</td>
<td class="gt_row gt_center">20.55%</td></tr>
  </tbody>
  <tfoot class="gt_sourcenotes">
    <tr>
      <td class="gt_sourcenote" colspan="10">NA</td>
    </tr>
  </tfoot>
  
</table>
</div>

Since all models were of the same type, in this comparison, we used the
`compare_performance()` function’s [ranking
algorithm](https://easystats.github.io/performance/reference/compare_performance.html#ranking-models),
which chose the model with the exposure variable plus 3 covariates to
perform the best.

Then we used the `select_parameters()` function’s [heuristic
algorithm](https://easystats.github.io/parameters/reference/select_parameters.html)
to check if any interaction terms were worth including in the model.

    MASS::glm.nb(
        suid_count ~ (pop_under_five + public_insurance + count_opioid_death + white)^2,
        data = suid
    ) |> 
    parameters::select_parameters() |> 
    parameters::parameters() |> 
    print_html()

<div id="ceokbshuzh" style="overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>html {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Helvetica Neue', 'Fira Sans', 'Droid Sans', Arial, sans-serif;
}

#ceokbshuzh .gt_table {
  display: table;
  border-collapse: collapse;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#ceokbshuzh .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#ceokbshuzh .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#ceokbshuzh .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 0;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#ceokbshuzh .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#ceokbshuzh .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#ceokbshuzh .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#ceokbshuzh .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#ceokbshuzh .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#ceokbshuzh .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#ceokbshuzh .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#ceokbshuzh .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
}

#ceokbshuzh .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#ceokbshuzh .gt_from_md > :first-child {
  margin-top: 0;
}

#ceokbshuzh .gt_from_md > :last-child {
  margin-bottom: 0;
}

#ceokbshuzh .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#ceokbshuzh .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#ceokbshuzh .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#ceokbshuzh .gt_row_group_first td {
  border-top-width: 2px;
}

#ceokbshuzh .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#ceokbshuzh .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#ceokbshuzh .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#ceokbshuzh .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#ceokbshuzh .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#ceokbshuzh .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#ceokbshuzh .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#ceokbshuzh .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#ceokbshuzh .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#ceokbshuzh .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-left: 4px;
  padding-right: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#ceokbshuzh .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#ceokbshuzh .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#ceokbshuzh .gt_left {
  text-align: left;
}

#ceokbshuzh .gt_center {
  text-align: center;
}

#ceokbshuzh .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#ceokbshuzh .gt_font_normal {
  font-weight: normal;
}

#ceokbshuzh .gt_font_bold {
  font-weight: bold;
}

#ceokbshuzh .gt_font_italic {
  font-style: italic;
}

#ceokbshuzh .gt_super {
  font-size: 65%;
}

#ceokbshuzh .gt_two_val_uncert {
  display: inline-block;
  line-height: 1em;
  text-align: right;
  font-size: 60%;
  vertical-align: -0.25em;
  margin-left: 0.1em;
}

#ceokbshuzh .gt_footnote_marks {
  font-style: italic;
  font-weight: normal;
  font-size: 75%;
  vertical-align: 0.4em;
}

#ceokbshuzh .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#ceokbshuzh .gt_slash_mark {
  font-size: 0.7em;
  line-height: 0.7em;
  vertical-align: 0.15em;
}

#ceokbshuzh .gt_fraction_numerator {
  font-size: 0.6em;
  line-height: 0.6em;
  vertical-align: 0.45em;
}

#ceokbshuzh .gt_fraction_denominator {
  font-size: 0.6em;
  line-height: 0.6em;
  vertical-align: -0.05em;
}
</style>
<table class="gt_table">
  <thead class="gt_header">
    <tr>
      <th colspan="6" class="gt_heading gt_title gt_font_normal gt_bottom_border" style>Model Summary</th>
    </tr>
    
  </thead>
  <thead class="gt_col_headings">
    <tr>
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="1" colspan="1">Parameter</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">Coefficient</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">SE</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">95% CI</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">z</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">p</th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td class="gt_row gt_left">(Intercept)</td>
<td class="gt_row gt_center">-2.10</td>
<td class="gt_row gt_center">0.44</td>
<td class="gt_row gt_center">(-2.98, -1.24)</td>
<td class="gt_row gt_center">-4.75</td>
<td class="gt_row gt_center">&lt; .001</td></tr>
    <tr><td class="gt_row gt_left">pop under five</td>
<td class="gt_row gt_center">9.07e-04</td>
<td class="gt_row gt_center">3.81e-04</td>
<td class="gt_row gt_center">(1.23e-04, 1.68e-03)</td>
<td class="gt_row gt_center">2.38</td>
<td class="gt_row gt_center">0.017 </td></tr>
    <tr><td class="gt_row gt_left">public insurance</td>
<td class="gt_row gt_center">0.03</td>
<td class="gt_row gt_center">7.38e-03</td>
<td class="gt_row gt_center">(0.01, 0.04)</td>
<td class="gt_row gt_center">3.45</td>
<td class="gt_row gt_center">&lt; .001</td></tr>
    <tr><td class="gt_row gt_left">count opioid death</td>
<td class="gt_row gt_center">2.89e-03</td>
<td class="gt_row gt_center">0.01</td>
<td class="gt_row gt_center">(-0.02, 0.03)</td>
<td class="gt_row gt_center">0.26</td>
<td class="gt_row gt_center">0.798 </td></tr>
    <tr><td class="gt_row gt_left">white</td>
<td class="gt_row gt_center">-0.03</td>
<td class="gt_row gt_center">7.83e-03</td>
<td class="gt_row gt_center">(-0.05, -0.02)</td>
<td class="gt_row gt_center">-4.12</td>
<td class="gt_row gt_center">&lt; .001</td></tr>
    <tr><td class="gt_row gt_left">public insurance * white</td>
<td class="gt_row gt_center">2.44e-04</td>
<td class="gt_row gt_center">1.73e-04</td>
<td class="gt_row gt_center">(-8.97e-05, 5.88e-04)</td>
<td class="gt_row gt_center">1.41</td>
<td class="gt_row gt_center">0.158 </td></tr>
    <tr><td class="gt_row gt_left">count opioid death * white</td>
<td class="gt_row gt_center">1.67e-03</td>
<td class="gt_row gt_center">3.86e-04</td>
<td class="gt_row gt_center">(8.59e-04, 2.47e-03)</td>
<td class="gt_row gt_center">4.32</td>
<td class="gt_row gt_center">&lt; .001</td></tr>
  </tbody>
  <tfoot class="gt_sourcenotes">
    <tr>
      <td class="gt_sourcenote" colspan="6"></td>
    </tr>
  </tfoot>
  
</table>
</div>

The interaction between `ccount_opioid_death` and `white` was proposed
by the algorithm and statistically significant, so we included it as the
only interaction term.

Next, we compared our final set of predictors in the panel of model
types again:

    final_models <-
        list(
            poisson = 
                glm(
                    suid_count ~ pop_under_five + public_insurance + count_opioid_death + white + count_opioid_death:white, 
                    family = poisson(), 
                    data = suid
                ),
            zero_infl_poisson = 
                pscl::zeroinfl(
                    suid_count ~ pop_under_five + public_insurance + count_opioid_death + white + count_opioid_death:white,
                    data = suid
                ),
            neb_bin = 
                MASS::glm.nb(
                    suid_count ~ pop_under_five + public_insurance + count_opioid_death + white + count_opioid_death:white,
                    data = suid
                ),
            zero_infl_neg_bin = 
                pscl::zeroinfl(
                    suid_count ~ pop_under_five + public_insurance + count_opioid_death + white + count_opioid_death:white,
                    dist = "negbin",
                    data = suid
                )
        ) 

    final_models |> 
        compare_performance() |>  
        print_html()

<div id="ssycowmnvk" style="overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>html {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Helvetica Neue', 'Fira Sans', 'Droid Sans', Arial, sans-serif;
}

#ssycowmnvk .gt_table {
  display: table;
  border-collapse: collapse;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#ssycowmnvk .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#ssycowmnvk .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#ssycowmnvk .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 0;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#ssycowmnvk .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#ssycowmnvk .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#ssycowmnvk .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#ssycowmnvk .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#ssycowmnvk .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#ssycowmnvk .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#ssycowmnvk .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#ssycowmnvk .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
}

#ssycowmnvk .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#ssycowmnvk .gt_from_md > :first-child {
  margin-top: 0;
}

#ssycowmnvk .gt_from_md > :last-child {
  margin-bottom: 0;
}

#ssycowmnvk .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#ssycowmnvk .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#ssycowmnvk .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#ssycowmnvk .gt_row_group_first td {
  border-top-width: 2px;
}

#ssycowmnvk .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#ssycowmnvk .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#ssycowmnvk .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#ssycowmnvk .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#ssycowmnvk .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#ssycowmnvk .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#ssycowmnvk .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#ssycowmnvk .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#ssycowmnvk .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#ssycowmnvk .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-left: 4px;
  padding-right: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#ssycowmnvk .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#ssycowmnvk .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#ssycowmnvk .gt_left {
  text-align: left;
}

#ssycowmnvk .gt_center {
  text-align: center;
}

#ssycowmnvk .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#ssycowmnvk .gt_font_normal {
  font-weight: normal;
}

#ssycowmnvk .gt_font_bold {
  font-weight: bold;
}

#ssycowmnvk .gt_font_italic {
  font-style: italic;
}

#ssycowmnvk .gt_super {
  font-size: 65%;
}

#ssycowmnvk .gt_two_val_uncert {
  display: inline-block;
  line-height: 1em;
  text-align: right;
  font-size: 60%;
  vertical-align: -0.25em;
  margin-left: 0.1em;
}

#ssycowmnvk .gt_footnote_marks {
  font-style: italic;
  font-weight: normal;
  font-size: 75%;
  vertical-align: 0.4em;
}

#ssycowmnvk .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#ssycowmnvk .gt_slash_mark {
  font-size: 0.7em;
  line-height: 0.7em;
  vertical-align: 0.15em;
}

#ssycowmnvk .gt_fraction_numerator {
  font-size: 0.6em;
  line-height: 0.6em;
  vertical-align: 0.45em;
}

#ssycowmnvk .gt_fraction_denominator {
  font-size: 0.6em;
  line-height: 0.6em;
  vertical-align: -0.05em;
}
</style>
<table class="gt_table">
  <thead class="gt_header">
    <tr>
      <th colspan="13" class="gt_heading gt_title gt_font_normal gt_bottom_border" style>Comparison of Model Performance Indices</th>
    </tr>
    
  </thead>
  <thead class="gt_col_headings">
    <tr>
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="1" colspan="1">Name</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">Model</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">AIC</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">AIC weights</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">BIC</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">BIC weights</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">RMSE</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">Sigma</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">Score_log</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">Score_spherical</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">R2</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">R2 (adj.)</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1">Nagelkerke's R2</th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td class="gt_row gt_left">poisson</td>
<td class="gt_row gt_center">glm</td>
<td class="gt_row gt_center">1364.76</td>
<td class="gt_row gt_center">&lt; 0.001</td>
<td class="gt_row gt_center">1395.71</td>
<td class="gt_row gt_center">0.009</td>
<td class="gt_row gt_center">0.56</td>
<td class="gt_row gt_center">0.81</td>
<td class="gt_row gt_center">-0.53</td>
<td class="gt_row gt_center">0.03</td>
<td class="gt_row gt_center"></td>
<td class="gt_row gt_center"></td>
<td class="gt_row gt_center">0.37</td></tr>
    <tr><td class="gt_row gt_left">zero_infl_poisson</td>
<td class="gt_row gt_center">zeroinfl</td>
<td class="gt_row gt_center">1349.84</td>
<td class="gt_row gt_center">0.310</td>
<td class="gt_row gt_center">1411.73</td>
<td class="gt_row gt_center">&lt; 0.001</td>
<td class="gt_row gt_center">0.56</td>
<td class="gt_row gt_center">0.56</td>
<td class="gt_row gt_center">-0.52</td>
<td class="gt_row gt_center">0.03</td>
<td class="gt_row gt_center">0.07</td>
<td class="gt_row gt_center">0.07</td>
<td class="gt_row gt_center"></td></tr>
    <tr><td class="gt_row gt_left">neb_bin</td>
<td class="gt_row gt_center">negbin</td>
<td class="gt_row gt_center">1350.11</td>
<td class="gt_row gt_center">0.270</td>
<td class="gt_row gt_center">1386.21</td>
<td class="gt_row gt_center">0.991</td>
<td class="gt_row gt_center">0.56</td>
<td class="gt_row gt_center">0.74</td>
<td class="gt_row gt_center">-0.52</td>
<td class="gt_row gt_center">0.03</td>
<td class="gt_row gt_center"></td>
<td class="gt_row gt_center"></td>
<td class="gt_row gt_center">0.37</td></tr>
    <tr><td class="gt_row gt_left">zero_infl_neg_bin</td>
<td class="gt_row gt_center">zeroinfl</td>
<td class="gt_row gt_center">1349.23</td>
<td class="gt_row gt_center">0.420</td>
<td class="gt_row gt_center">1416.28</td>
<td class="gt_row gt_center">&lt; 0.001</td>
<td class="gt_row gt_center">0.56</td>
<td class="gt_row gt_center">0.56</td>
<td class="gt_row gt_center">-0.52</td>
<td class="gt_row gt_center">0.03</td>
<td class="gt_row gt_center">0.07</td>
<td class="gt_row gt_center">0.07</td>
<td class="gt_row gt_center"></td></tr>
  </tbody>
  <tfoot class="gt_sourcenotes">
    <tr>
      <td class="gt_sourcenote" colspan="13"></td>
    </tr>
  </tfoot>
  
</table>
</div>

Precision-related scores were fairly similar and the R^2 terms were not
directly comparable, so we again paid most attention to AIC and BIC.
These two scores disagreed on which model type had the best fit, but BIC
more emphatically chose the negative binomial model, so we stuck with
this type.

Another way to compare the goodness of fit was with a visual check of
[rootograms](https://www.r-bloggers.com/2016/06/rootograms/):

    source("R/plot_rootogram.R")

    final_models |> 
        purrr::map(plot_rootogram)

    ## $poisson

![](/home/riggins/cook_county_sids_mortality/markdown_output/53-model_selection_files/figure-markdown_strict/unnamed-chunk-11-1.png)

    ## 
    ## $zero_infl_poisson

![](/home/riggins/cook_county_sids_mortality/markdown_output/53-model_selection_files/figure-markdown_strict/unnamed-chunk-11-2.png)

    ## 
    ## $neb_bin

![](/home/riggins/cook_county_sids_mortality/markdown_output/53-model_selection_files/figure-markdown_strict/unnamed-chunk-11-3.png)

    ## 
    ## $zero_infl_neg_bin

![](/home/riggins/cook_county_sids_mortality/markdown_output/53-model_selection_files/figure-markdown_strict/unnamed-chunk-11-4.png)

Visually speaking, the negative binomial model type and its
zero-inflated variant looked almost identical, so we felt further
reassured in selecting the plain, non-zero-inflated type.
