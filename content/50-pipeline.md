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
    ##  [1] rmarkdown_2.14   knitr_1.39       lubridate_1.8.0  tidycensus_1.2.2
    ##  [5] sf_1.0-7         forcats_0.5.1    stringr_1.4.0    dplyr_1.0.9     
    ##  [9] purrr_0.3.4      readr_2.1.2      tidyr_1.2.0      tibble_3.1.7    
    ## [13] ggplot2_3.3.6    tidyverse_1.3.1  targets_0.12.1  
    ## 
    ## loaded via a namespace (and not attached):
    ##  [1] fs_1.5.2            webshot_0.5.3       httr_1.4.3          tools_4.2.1        
    ##  [5] backports_1.4.1     utf8_1.2.2          rgdal_1.5-32        R6_2.5.1           
    ##  [9] KernSmooth_2.23-20  DBI_1.1.3           colorspace_2.0-3    withr_2.5.0        
    ## [13] sp_1.5-0            tidyselect_1.1.2    processx_3.7.0      curl_4.3.2         
    ## [17] compiler_4.2.1      cli_3.3.0           rvest_1.0.2         gt_0.6.0           
    ## [21] xml2_1.3.3          sass_0.4.1          scales_1.2.0        checkmate_2.1.0    
    ## [25] classInt_0.4-7      corrr_0.4.3         callr_3.7.0         proxy_0.4-27       
    ## [29] rappdirs_0.3.3      commonmark_1.8.0    digest_0.6.29       foreign_0.8-82     
    ## [33] pkgconfig_2.0.3     htmltools_0.5.2     highr_0.9           dbplyr_2.2.1       
    ## [37] fastmap_1.1.0       htmlwidgets_1.5.4   rlang_1.0.3         readxl_1.4.0       
    ## [41] rstudioapi_0.13     visNetwork_2.1.0    generics_0.1.3      jsonlite_1.8.0     
    ## [45] magrittr_2.0.3      Rcpp_1.0.8.3        munsell_0.5.0       fansi_1.0.3        
    ## [49] lifecycle_1.0.1     stringi_1.7.6       yaml_2.3.5          gtsummary_1.6.1    
    ## [53] snakecase_0.11.0    grid_4.2.1          maptools_1.1-4      promises_1.2.0.1   
    ## [57] crayon_1.5.1        lattice_0.20-45     haven_2.5.0         chromote_0.1.0     
    ## [61] hms_1.1.1           ps_1.7.1            pillar_1.7.0        igraph_1.3.2       
    ## [65] uuid_1.1-0          base64url_1.4       codetools_0.2-18    reprex_2.0.1       
    ## [69] glue_1.6.2          evaluate_0.15       data.table_1.14.2   broom.helpers_1.8.0
    ## [73] modelr_0.1.8        vctrs_0.4.1         tzdb_0.3.0          cellranger_1.1.0   
    ## [77] webshot2_0.1.0      gtable_0.3.0        assertthat_0.2.1    xfun_0.31          
    ## [81] janitor_2.1.0       broom_1.0.0         e1071_1.7-11        later_1.3.0        
    ## [85] class_7.3-20        websocket_1.4.1     tigris_1.6.1        units_0.8-0        
    ## [89] ellipsis_0.3.2
