## Supplement 1: Analytic Pipeline

### Pipeline Overview

    library(targets)

DPR orchestrated the data pipeline for the primary data analysis with
the package {[targets](https://docs.ropensci.org/targets/)}.

If working with this repository and you want a visual overview of the
pipeline, you can execute:

    tar_visnetwork()

Here is information on the R session and its dependencies:

    sessionInfo()

    ## R version 4.2.1 (2022-06-23)
    ## Platform: x86_64-pc-linux-gnu (64-bit)
    ## Running under: Pop!_OS 22.04 LTS
    ## 
    ## Matrix products: default
    ## BLAS:   /usr/lib/x86_64-linux-gnu/openblas-pthread/libblas.so.3
    ## LAPACK: /usr/lib/x86_64-linux-gnu/openblas-pthread/libopenblasp-r0.3.20.so
    ## 
    ## locale:
    ##  [1] LC_CTYPE=en_US.UTF-8       LC_NUMERIC=C               LC_TIME=en_US.UTF-8       
    ##  [4] LC_COLLATE=en_US.UTF-8     LC_MONETARY=en_US.UTF-8    LC_MESSAGES=en_US.UTF-8   
    ##  [7] LC_PAPER=en_US.UTF-8       LC_NAME=C                  LC_ADDRESS=C              
    ## [10] LC_TELEPHONE=C             LC_MEASUREMENT=en_US.UTF-8 LC_IDENTIFICATION=C       
    ## 
    ## attached base packages:
    ## [1] stats     graphics  grDevices utils     datasets  methods   base     
    ## 
    ## other attached packages:
    ## [1] targets_0.12.1
    ## 
    ## loaded via a namespace (and not attached):
    ##  [1] Rcpp_1.0.9              tidyr_1.2.0             class_7.3-20            ps_1.7.1               
    ##  [5] leaflet.providers_1.9.0 assertthat_0.2.1        digest_0.6.29           utf8_1.2.2             
    ##  [9] R6_2.5.1                backports_1.4.1         leaflet.extras_1.0.0    evaluate_0.15          
    ## [13] e1071_1.7-11            ggplot2_3.3.6           pillar_1.8.0            rlang_1.0.4            
    ## [17] rstudioapi_0.13         data.table_1.14.2       callr_3.7.1             jquerylib_0.1.4        
    ## [21] checkmate_2.1.0         rmarkdown_2.14          stringr_1.4.0           htmlwidgets_1.5.4      
    ## [25] igraph_1.3.4            munsell_0.5.0           proxy_0.4-27            compiler_4.2.1         
    ## [29] xfun_0.31               pkgconfig_2.0.3         htmltools_0.5.3         tidyselect_1.1.2       
    ## [33] tibble_3.1.7            gridExtra_2.3           corrr_0.4.3             codetools_0.2-18       
    ## [37] fansi_1.0.3             viridisLite_0.4.0       crayon_1.5.1            dplyr_1.0.9            
    ## [41] withr_2.5.0             sf_1.0-8                commonmark_1.8.0        grid_4.2.1             
    ## [45] jsonlite_1.8.0          gtable_0.3.0            lifecycle_1.0.1         DBI_1.1.3              
    ## [49] magrittr_2.0.3          units_0.8-0             scales_1.2.0            KernSmooth_2.23-20     
    ## [53] cachem_1.0.6            cli_3.3.0               stringi_1.7.8           farver_2.1.1           
    ## [57] viridis_0.6.2           broom.helpers_1.8.0     leaflet_2.1.1           bslib_0.4.0            
    ## [61] ellipsis_0.3.2          generics_0.1.3          vctrs_0.4.1             RColorBrewer_1.1-3     
    ## [65] tools_4.2.1             glue_1.6.2              purrr_0.3.4             crosstalk_1.2.0        
    ## [69] processx_3.7.0          fastmap_1.1.0           yaml_2.3.5              colorspace_2.0-3       
    ## [73] gtsummary_1.6.1         gt_0.6.0                base64url_1.4           classInt_0.4-7         
    ## [77] knitr_1.39              sass_0.4.2
