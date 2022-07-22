## Supplement 1: Analytic Pipeline and Custom Functions

### Pipeline Overview

We hosted both the
[code](https://github.com/andtheWings/cook_county_sids_mortality) and
[manuscript](https://github.com/andtheWings/cook_county_suid_manuscript)
on Github.

We orchestrated the data pipeline for the data analysis with the package
{[targets](https://docs.ropensci.org/targets/)}.

    library(targets)

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
    ##  [1] LC_CTYPE=en_US.UTF-8      
    ##  [2] LC_NUMERIC=C              
    ##  [3] LC_TIME=en_US.UTF-8       
    ##  [4] LC_COLLATE=en_US.UTF-8    
    ##  [5] LC_MONETARY=en_US.UTF-8   
    ##  [6] LC_MESSAGES=en_US.UTF-8   
    ##  [7] LC_PAPER=en_US.UTF-8      
    ##  [8] LC_NAME=C                 
    ##  [9] LC_ADDRESS=C              
    ## [10] LC_TELEPHONE=C            
    ## [11] LC_MEASUREMENT=en_US.UTF-8
    ## [12] LC_IDENTIFICATION=C       
    ## 
    ## attached base packages:
    ## [1] stats     graphics  grDevices
    ## [4] utils     datasets  methods  
    ## [7] base     
    ## 
    ## other attached packages:
    ##  [1] magrittr_2.0.3   
    ##  [2] gt_0.6.0         
    ##  [3] lubridate_1.8.0  
    ##  [4] tidycensus_1.2.2 
    ##  [5] sf_1.0-8         
    ##  [6] forcats_0.5.1    
    ##  [7] stringr_1.4.0    
    ##  [8] dplyr_1.0.9      
    ##  [9] readr_2.1.2      
    ## [10] tidyr_1.2.0      
    ## [11] tibble_3.1.7     
    ## [12] tidyverse_1.3.2  
    ## [13] targets_0.12.1   
    ## [14] purrr_0.3.4      
    ## [15] ggplot2_3.3.6    
    ## [16] performance_0.9.1
    ## 
    ## loaded via a namespace (and not attached):
    ##   [1] googledrive_2.0.0  
    ##   [2] colorspace_2.0-3   
    ##   [3] ellipsis_0.3.2     
    ##   [4] class_7.3-20       
    ##   [5] leaflet_2.1.1      
    ##   [6] rgdal_1.5-32       
    ##   [7] estimability_1.4   
    ##   [8] snakecase_0.11.0   
    ##   [9] parameters_0.18.1  
    ##  [10] fs_1.5.2           
    ##  [11] rstudioapi_0.13    
    ##  [12] proxy_0.4-27       
    ##  [13] farver_2.1.1       
    ##  [14] gtsummary_1.6.1    
    ##  [15] fansi_1.0.3        
    ##  [16] mvtnorm_1.1-3      
    ##  [17] xml2_1.3.3         
    ##  [18] splines_4.2.1      
    ##  [19] codetools_0.2-18   
    ##  [20] pscl_1.5.5         
    ##  [21] knitr_1.39         
    ##  [22] jsonlite_1.8.0     
    ##  [23] broom_1.0.0        
    ##  [24] dbplyr_2.2.1       
    ##  [25] compiler_4.2.1     
    ##  [26] httr_1.4.3         
    ##  [27] emmeans_1.7.5      
    ##  [28] backports_1.4.1    
    ##  [29] Matrix_1.4-1       
    ##  [30] assertthat_0.2.1   
    ##  [31] fastmap_1.1.0      
    ##  [32] gargle_1.2.0       
    ##  [33] cli_3.3.0          
    ##  [34] htmltools_0.5.3    
    ##  [35] tools_4.2.1        
    ##  [36] igraph_1.3.4       
    ##  [37] gtable_0.3.0       
    ##  [38] glue_1.6.2         
    ##  [39] corrr_0.4.3        
    ##  [40] rappdirs_0.3.3     
    ##  [41] Rcpp_1.0.9         
    ##  [42] cellranger_1.1.0   
    ##  [43] vctrs_0.4.1        
    ##  [44] nlme_3.1-157       
    ##  [45] tigris_1.6.1       
    ##  [46] broom.helpers_1.8.0
    ##  [47] crosstalk_1.2.0    
    ##  [48] insight_0.18.0     
    ##  [49] xfun_0.31          
    ##  [50] networkD3_0.4      
    ##  [51] ps_1.7.1           
    ##  [52] rvest_1.0.2        
    ##  [53] lifecycle_1.0.1    
    ##  [54] googlesheets4_1.0.0
    ##  [55] MASS_7.3-58        
    ##  [56] scales_1.2.0       
    ##  [57] hms_1.1.1          
    ##  [58] parallel_4.2.1     
    ##  [59] yaml_2.3.5         
    ##  [60] gridExtra_2.3      
    ##  [61] see_0.7.1          
    ##  [62] sass_0.4.2         
    ##  [63] stringi_1.7.8      
    ##  [64] highr_0.9          
    ##  [65] bayestestR_0.12.1  
    ##  [66] maptools_1.1-4     
    ##  [67] e1071_1.7-11       
    ##  [68] checkmate_2.1.0    
    ##  [69] commonmark_1.8.0   
    ##  [70] rlang_1.0.4        
    ##  [71] pkgconfig_2.0.3    
    ##  [72] evaluate_0.15      
    ##  [73] lattice_0.20-45    
    ##  [74] htmlwidgets_1.5.4  
    ##  [75] labeling_0.4.2     
    ##  [76] processx_3.7.0     
    ##  [77] tidyselect_1.1.2   
    ##  [78] DataExplorer_0.8.2 
    ##  [79] R6_2.5.1           
    ##  [80] generics_0.1.3     
    ##  [81] base64url_1.4      
    ##  [82] DBI_1.1.3          
    ##  [83] mgcv_1.8-40        
    ##  [84] pillar_1.8.0       
    ##  [85] haven_2.5.0        
    ##  [86] foreign_0.8-82     
    ##  [87] withr_2.5.0        
    ##  [88] units_0.8-0        
    ##  [89] datawizard_0.4.1   
    ##  [90] sp_1.5-0           
    ##  [91] janitor_2.1.0      
    ##  [92] modelr_0.1.8       
    ##  [93] crayon_1.5.1       
    ##  [94] uuid_1.1-0         
    ##  [95] KernSmooth_2.23-20 
    ##  [96] utf8_1.2.2         
    ##  [97] correlation_0.8.1  
    ##  [98] tzdb_0.3.0         
    ##  [99] rmarkdown_2.14     
    ## [100] grid_4.2.1         
    ## [101] readxl_1.4.0       
    ## [102] data.table_1.14.2  
    ## [103] callr_3.7.1        
    ## [104] reprex_2.0.1       
    ## [105] digest_0.6.29      
    ## [106] classInt_0.4-7     
    ## [107] xtable_1.8-4       
    ## [108] munsell_0.5.0
