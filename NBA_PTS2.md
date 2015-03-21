# NBA:  Basketball-Reference.com: PTS regression:: PTS2
bdanalytics  

**  **    
**Date: (Wed) Jun 17, 2015**    

# Introduction:  

Data: 
Source: 
    Training:   https://courses.edx.org/asset-v1:MITx+15.071x_2a+2T2015+type@asset+block/NBA_train.csv  
    New:        https://courses.edx.org/asset-v1:MITx+15.071x_2a+2T2015+type@asset+block/NBA_test.csv  
Time period: 



# Synopsis:

Based on analysis utilizing <> techniques, <conclusion heading>:  

Regression results:
First run:
    <glb_sel_mdl_id>: 
        OOB_RMSE=<0.4f>; new_RMSE=<0.4f>; <feat1>=<imp>; <feat2>=<imp>

First run:
    All.X.lm: 
        OOB_RMSE=0.0000; new_RMSE=0.0000; FG=100.00; FT=99.66; X2P=33.22
-W-SeasonEnd:
    All.X.lm: 
        OOB_RMSE=0.0000; new_RMSE=0.0000; FG=100.00; FT=93.30; X2P=31.61        

Classification results:
First run:
    <glb_sel_mdl_id>: 
        OOB_conf_mtrx=[NY, YN]; <feat1>=<imp>; <feat2>=<imp>

### ![](<filename>.png)

## Potential next steps include:
- Organization:
    - Categorize by chunk
    - Priority criteria:
        0. Ease of change
        1. Impacts report
        2. Cleans innards
        3. Bug report
        
- all chunks:
    - at chunk-end rm(!glb_<var>)
    
- manage.missing.data chunk:
    - cleaner way to manage re-splitting of training vs. new entity

- extract.features chunk:
    - Add n-grams for glb_txt_vars
        - "RTextTools", "tau", "RWeka", and "textcat" packages
    - Convert user-specified mutate code to config specs
    
- fit.models chunk:
    - Prediction accuracy scatter graph:
    -   Add tiles (raw vs. PCA)
    -   Use shiny for drop-down of "important" features
    -   Use plot.ly for interactive plots ?
    
    - Change .fit suffix of model metrics to .mdl if it's data independent (e.g. AIC, Adj.R.Squared - is it truly data independent ?, etc.)
    - move model_type parameter to myfit_mdl before indep_vars_vctr (keep all model_* together)
    - create a custom model for rpart that has minbucket as a tuning parameter
    - varImp for randomForest crashes in caret version:6.0.41 -> submit bug report

- Probability handling for multinomials vs. desired binomial outcome
-   ROCR currently supports only evaluation of binary classification tasks (version 1.0.7)
-   extensions toward multiclass classification are scheduled for the next release

- Skip trControl.method="cv" for dummy classifier ?
- Add custom model to caret for a dummy (baseline) classifier (binomial & multinomial) that generates proba/outcomes which mimics the freq distribution of glb_rsp_var values; Right now glb_dmy_glm_mdl always generates most frequent outcome in training data
- glm_dmy_mdl should use the same method as glm_sel_mdl until custom dummy classifer is implemented

- fit.all.training chunk:
    - myplot_prediction_classification: displays 'x' instead of '+' when there are no prediction errors 
- Compare glb_sel_mdl vs. glb_fin_mdl:
    - varImp
    - Prediction differences (shd be minimal ?)

- Move glb_analytics_diag_plots to mydsutils.R: (+) Easier to debug (-) Too many glb vars used
- Add print(ggplot.petrinet(glb_analytics_pn) + coord_flip()) at the end of every major chunk
- Parameterize glb_analytics_pn
- Move glb_impute_missing_data to mydsutils.R: (-) Too many glb vars used; glb_<>_df reassigned
- Replicate myfit_mdl_classification features in myfit_mdl_regression
- Do non-glm methods handle interaction terms ?
- f-score computation for classifiers should be summation across outcomes (not just the desired one ?)
- Add accuracy computation to glb_dmy_mdl in predict.data.new chunk
- Why does splitting fit.data.training.all chunk into separate chunks add an overhead of ~30 secs ? It's not rbind b/c other chunks have lower elapsed time. Is it the number of plots ?
- Incorporate code chunks in print_sessionInfo
- Test against 
    - projects in github.com/bdanalytics
    - lectures in jhu-datascience track

# Analysis: 

```r
rm(list=ls())
set.seed(12345)
options(stringsAsFactors=FALSE)
source("~/Dropbox/datascience/R/myscript.R")
source("~/Dropbox/datascience/R/mydsutils.R")
```

```
## Loading required package: caret
## Loading required package: lattice
## Loading required package: ggplot2
```

```r
source("~/Dropbox/datascience/R/myplot.R")
source("~/Dropbox/datascience/R/mypetrinet.R")
source("~/Dropbox/datascience/R/myplclust.R")
# Gather all package requirements here
suppressPackageStartupMessages(require(doMC))
registerDoMC(4) # max(length(glb_txt_vars), glb_n_cv_folds) + 1
#packageVersion("snow")
#require(sos); findFn("cosine", maxPages=2, sortby="MaxScore")

# Analysis control global variables
glb_trnng_url <- "https://courses.edx.org/asset-v1:MITx+15.071x_2a+2T2015+type@asset+block/NBA_train.csv"
glb_newdt_url <- "https://courses.edx.org/asset-v1:MITx+15.071x_2a+2T2015+type@asset+block/NBA_test.csv"
glb_out_pfx <- "PTS2_"
glb_save_envir <- FALSE # or TRUE

glb_is_separate_newent_dataset <- TRUE    # or TRUE
    glb_split_entity_newent_datasets <- TRUE   # or FALSE
    glb_split_newdata_method <- "sample"          # "condition" or "sample" or "copy"
    glb_split_newdata_condition <- NULL # or "is.na(<var>)"; "<var> <condition_operator> <value>"
    glb_split_newdata_size_ratio <- 0.3               # > 0 & < 1
    glb_split_sample.seed <- 123               # or any integer

glb_max_fitent_obs <- NULL # or any integer                         
glb_is_regression <- TRUE; glb_is_classification <- !glb_is_regression; 
    glb_is_binomial <- TRUE # or TRUE or FALSE

glb_rsp_var_raw <- "PTS"

# for classification, the response variable has to be a factor
glb_rsp_var <- glb_rsp_var_raw # or "<glb_rsp_var_raw>.fctr"

# if the response factor is based on numbers/logicals e.g (0/1 OR TRUE/FALSE vs. "A"/"B"), 
#   or contains spaces (e.g. "Not in Labor Force")
#   caret predict(..., type="prob") crashes
glb_map_rsp_raw_to_var <- NULL # or function(raw) {
#     relevel(factor(ifelse(raw == 1, "Y", "N")), as.factor(c("Y", "N")), ref="N")
#     #as.factor(paste0("B", raw))
#     #as.factor(gsub(" ", "\\.", raw))    
# }
# glb_map_rsp_raw_to_var(c(1, 1, 0, 0, 0))

glb_map_rsp_var_to_raw <- NULL # or function(var) {
#     as.numeric(var) - 1
#     #as.numeric(var)
#     #gsub("\\.", " ", levels(var)[as.numeric(var)])
#     #c(" <=50K", " >50K")[as.numeric(var)]
#     #c(FALSE, TRUE)[as.numeric(var)]
# }
# glb_map_rsp_var_to_raw(glb_map_rsp_raw_to_var(c(1, 1, 0, 0, 0)))

if ((glb_rsp_var != glb_rsp_var_raw) & is.null(glb_map_rsp_raw_to_var))
    stop("glb_map_rsp_raw_to_var function expected")
glb_rsp_var_out <- paste0(glb_rsp_var, ".predict.") # model_id is appended later

# List info gathered for various columns
# <col_name>:   <description>; <notes>
#   SeasonEnd:  is the year the season ended.
#   Team:       Team ID
#   Playoffs:   1 if team made it to the playoffs that year else 0
#   W:          # of regular season wins.
#   PTS:        # of points scored during the regular season.
#   oppPTS:     # of opponent points scored during the regular season.
#   FG:         # of successful field goals, including two and three pointers.
#   FGA:        # of FG attempts
#   X2P:        # of 2 point field goals made
#   X2PA:       # of 2 point field goals attmpted
#   X3P, X3PA:  # of 3 point field goals
#   FT, FTA:    # of free throws
#   ORB, DRB:   # of offensive and defensive rebounds.
#   AST:        # of assists.
#   STL:        # of steals.
#   BLK:        # of blocks.
#   TOV:        # of turnovers.

# If multiple vars are parts of id, consider concatenating them to create one id var
# If glb_id_var == NULL, ".rownames <- row.names()" is the default
glb_id_var <- NULL # or c("<var1>")
glb_category_vars <- NULL # or c("<var1>", "<var2>")
glb_drop_vars <- c(NULL) # or c("<col_name>")

glb_map_vars <- NULL # or c("<var1>", "<var2>")
glb_map_urls <- list();
# glb_map_urls[["<var1>"]] <- "<var1.url>"

glb_assign_pairs_lst <- NULL; 
# glb_assign_pairs_lst[["<var1>"]] <- list(from=c(NA),
#                                            to=c("NA.my"))
glb_assign_vars <- names(glb_assign_pairs_lst)

glb_transform_lst <- NULL;
# glb_transform_lst[["<var>"]] <- list(
#     mapfn=function(raw) { tfr_raw <- as.character(cut(raw, 5)); 
#                           tfr_raw[is.na(tfr_raw)] <- "NA.my";
#                           return(as.factor(tfr_raw)) }
#     , sfx=".my.fctr")
# mapfn(glb_allobs_df$<var>)
# glb_transform_lst[["<var1>"]] <- glb_transform_lst[["<var2>"]]
# Add logs of numerics that are not distributed normally ->  do automatically ???
glb_transform_vars <- names(glb_transform_lst)

glb_date_vars <- NULL # or c("<date_var>")
glb_date_fmts <- list(); #glb_date_fmts[["<date_var>"]] <- "%m/%e/%y"
glb_date_tzs <- list();  #glb_date_tzs[["<date_var>"]] <- "America/New_York"
#grep("America/New", OlsonNames(), value=TRUE)

glb_txt_vars <- NULL # or c("<txt_var1>", "<txt_var2>")   
#Sys.setlocale("LC_ALL", "C") # For english

glb_append_stop_words <- list()
# Remember to use unstemmed words
#orderBy(~ -cor.y.abs, subset(glb_feats_df, grepl("[HSA]\\.T\\.", id) & !is.na(cor.high.X)))
#dsp_obs(Headline.contains="polit")
#subset(glb_allobs_df, H.T.compani > 0)[, c("UniqueID", "Headline", "H.T.compani")]
# glb_append_stop_words[["<txt_var1>"]] <- c(NULL
# #                             ,"<word1>" # <reason1>
#                             )
#subset(glb_allobs_df, S.T.newyorktim > 0)[, c("UniqueID", "Snippet", "S.T.newyorktim")]
#glb_txt_lst[["Snippet"]][which(glb_allobs_df$UniqueID %in% c(8394, 8317, 8339, 8350, 8307))]

glb_important_terms <- list()
# Remember to use stemmed terms 

glb_sprs_thresholds <- NULL # or c(0.988, 0.970, 0.970) # Generates 29, 22, 22 terms
# Properties:
#   numrows(glb_feats_df) << numrows(glb_fitobs_df)
#   Select terms that appear in at least 0.2 * O(FP/FN(glb_OOBobs_df))
#       numrows(glb_OOBobs_df) = 1.1 * numrows(glb_newobs_df)
names(glb_sprs_thresholds) <- glb_txt_vars

# Derived features (consolidate this with transform features ???)
glb_derive_lst <- NULL;
glb_derive_lst[["PTS.diff"]] <- list(
    mapfn=function(PTS, oppPTS) { return(PTS - oppPTS) }
    , args=c("PTS", "oppPTS"))
# args_lst <- NULL; for (arg in glb_derive_lst[["PTS.diff"]]$args) args_lst[[arg]] <- glb_allobs_df[, arg]; do.call(mapfn, args_lst)
# glb_derive_lst[["<var1>"]] <- glb_derive_lst[["<var2>"]]
glb_derive_vars <- names(glb_derive_lst)

# User-specified exclusions  
glb_exclude_vars_as_features <- c("Team.fctr", "SeasonEnd",
                                  "Playoffs", "W", "oppPTS", "PTS.diff") 
if (glb_rsp_var_raw != glb_rsp_var)
    glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, 
                                            glb_rsp_var_raw)

# List feats that shd be excluded due to known causation by prediction variable
glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, 
                                      c(NULL)) # or c("<col_name>")

glb_impute_na_data <- FALSE # or TRUE
glb_mice_complete.seed <- 144 # or any integer

glb_cluster <- FALSE # or TRUE

glb_interaction_only_features <- NULL # or ???

glb_models_lst <- list(); glb_models_df <- data.frame()
# Regression
if (glb_is_regression)
    glb_models_method_vctr <- c("lm", "glm", "bayesglm", "rpart", "rf") else
# Classification
    if (glb_is_binomial)
        glb_models_method_vctr <- c("glm", "bayesglm", "rpart", "rf") else  
        glb_models_method_vctr <- c("rpart", "rf")

# Baseline prediction model feature(s)
glb_Baseline_mdl_var <- NULL # or c("<col_name>")

glb_model_metric_terms <- NULL # or matrix(c(
#                               0,1,2,3,4,
#                               2,0,1,2,3,
#                               4,2,0,1,2,
#                               6,4,2,0,1,
#                               8,6,4,2,0
#                           ), byrow=TRUE, nrow=5)
glb_model_metric <- NULL # or "<metric_name>"
glb_model_metric_maximize <- NULL # or FALSE (TRUE is not the default for both classification & regression) 
glb_model_metric_smmry <- NULL # or function(data, lev=NULL, model=NULL) {
#     confusion_mtrx <- t(as.matrix(confusionMatrix(data$pred, data$obs)))
#     #print(confusion_mtrx)
#     #print(confusion_mtrx * glb_model_metric_terms)
#     metric <- sum(confusion_mtrx * glb_model_metric_terms) / nrow(data)
#     names(metric) <- glb_model_metric
#     return(metric)
# }

glb_tune_models_df <- 
   rbind(
    #data.frame(parameter="cp", min=0.00005, max=0.00005, by=0.000005),
                            #seq(from=0.01,  to=0.01, by=0.01)
    #data.frame(parameter="mtry",  min=080, max=100, by=10),
    #data.frame(parameter="mtry",  min=08, max=10, by=1),    
    data.frame(parameter="dummy", min=2, max=4, by=1)
        ) 
# or NULL
glb_n_cv_folds <- 3 # or NULL

glb_clf_proba_threshold <- NULL # 0.5

# Model selection criteria
if (glb_is_regression)
    glb_model_evl_criteria <- c("min.RMSE.OOB", "max.R.sq.OOB", "max.Adj.R.sq.fit")
if (glb_is_classification) {
    if (glb_is_binomial)
        glb_model_evl_criteria <- 
            c("max.Accuracy.OOB", "max.auc.OOB", "max.Kappa.OOB", "min.aic.fit") else
        glb_model_evl_criteria <- c("max.Accuracy.OOB", "max.Kappa.OOB")
}

glb_sel_mdl_id <- NULL # or "<model_id_prefix>.<model_method>"
glb_fin_mdl_id <- glb_sel_mdl_id # or "Final"

# Depict process
glb_analytics_pn <- petrinet(name="glb_analytics_pn",
                        trans_df=data.frame(id=1:6,
    name=c("data.training.all","data.new",
           "model.selected","model.final",
           "data.training.all.prediction","data.new.prediction"),
    x=c(   -5,-5,-15,-25,-25,-35),
    y=c(   -5, 5,  0,  0, -5,  5)
                        ),
                        places_df=data.frame(id=1:4,
    name=c("bgn","fit.data.training.all","predict.data.new","end"),
    x=c(   -0,   -20,                    -30,               -40),
    y=c(    0,     0,                      0,                 0),
    M0=c(   3,     0,                      0,                 0)
                        ),
                        arcs_df=data.frame(
    begin=c("bgn","bgn","bgn",        
            "data.training.all","model.selected","fit.data.training.all",
            "fit.data.training.all","model.final",    
            "data.new","predict.data.new",
            "data.training.all.prediction","data.new.prediction"),
    end  =c("data.training.all","data.new","model.selected",
            "fit.data.training.all","fit.data.training.all","model.final",
            "data.training.all.prediction","predict.data.new",
            "predict.data.new","data.new.prediction",
            "end","end")
                        ))
#print(ggplot.petrinet(glb_analytics_pn))
print(ggplot.petrinet(glb_analytics_pn) + coord_flip())
```

```
## Loading required package: grid
```

![](NBA_PTS2_files/figure-html/set_global_options-1.png) 

```r
glb_analytics_avl_objs <- NULL

glb_chunks_df <- myadd_chunk(NULL, "import.data")
```

```
##         label step_major step_minor   bgn end elapsed
## 1 import.data          1          0 9.041  NA      NA
```

## Step `1.0: import data`
#### chunk option: eval=<r condition>

```r
#glb_chunks_df <- myadd_chunk(NULL, "import.data")

glb_trnobs_df <- myimport_data(url=glb_trnng_url, comment="glb_trnobs_df", 
                                force_header=TRUE)
```

```
## [1] "Reading file ./data/NBA_train.csv..."
## [1] "dimensions of data in ./data/NBA_train.csv: 835 rows x 20 cols"
##   SeasonEnd                Team Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 1      1980       Atlanta Hawks        1 50 8573   8334 3261 7027 3248
## 2      1980      Boston Celtics        1 61 9303   8664 3617 7387 3455
## 3      1980       Chicago Bulls        0 30 8813   9035 3362 6943 3292
## 4      1980 Cleveland Cavaliers        0 37 9360   9332 3811 8041 3775
## 5      1980      Denver Nuggets        0 30 8878   9240 3462 7470 3379
## 6      1980     Detroit Pistons        0 16 8933   9609 3643 7596 3586
##   X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV
## 1 6952  13   75 2038 2645 1369 2406 1913 782 539 1495
## 2 6965 162  422 1907 2449 1227 2457 2198 809 308 1539
## 3 6668  70  275 2019 2592 1115 2465 2152 704 392 1684
## 4 7854  36  187 1702 2205 1307 2381 2108 764 342 1370
## 5 7215  83  255 1871 2539 1311 2524 2079 746 404 1533
## 6 7377  57  219 1590 2149 1226 2415 1950 783 562 1742
##     SeasonEnd                 Team Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 29       1981      Detroit Pistons        0 21 8174   8692 3236 6986 3223
## 127      1985   Los Angeles Lakers        1 62 9696   9093 3952 7254 3862
## 426      1997        Chicago Bulls        1 69 8458   7572 3277 6923 2754
## 607      2004 Los Angeles Clippers        0 28 7771   8147 2817 6579 2488
## 611      2004      Milwaukee Bucks        1 41 8039   7952 2970 6650 2569
## 825      2011      New York Knicks        1 42 8734   8670 3140 6867 2375
##     X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV
## 29  6902  13   84 1689 2330 1201 2111 1819 884 492 1759
## 127 6959  90  295 1702 2232 1063 2550 2575 695 481 1537
## 426 5520 523 1403 1381 1848 1235 2461 2142 716 332 1109
## 607 5555 329 1024 1808 2302 1149 2416 1653 594 376 1344
## 611 5505 401 1145 1698 2192  960 2502 1872 554 383 1110
## 825 4786 765 2081 1689 2087  847 2470 1757 625 475 1123
##     SeasonEnd                   Team Playoffs  W  PTS oppPTS   FG  FGA
## 830      2011 Portland Trail Blazers        1 48 7896   7771 2951 6599
## 831      2011       Sacramento Kings        0 24 8151   8589 3134 6979
## 832      2011      San Antonio Spurs        1 61 8502   8034 3148 6628
## 833      2011        Toronto Raptors        0 22 8124   8639 3144 6755
## 834      2011              Utah Jazz        0 39 8153   8303 3064 6590
## 835      2011     Washington Wizards        0 23 7977   8584 3048 6888
##      X2P X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV
## 830 2433 5096 518 1503 1476 1835  996 2230 1736 660 358 1070
## 831 2706 5702 428 1277 1455 1981 1071 2526 1675 608 391 1324
## 832 2463 4901 685 1727 1521 1984  829 2603 1836 602 372 1101
## 833 2799 5664 345 1091 1491 1976  963 2343 1795 581 350 1206
## 834 2629 5334 435 1256 1590 2061  898 2338 1921 629 484 1175
## 835 2656 5706 392 1182 1489 1999 1013 2374 1592 665 502 1258
## 'data.frame':	835 obs. of  20 variables:
##  $ SeasonEnd: int  1980 1980 1980 1980 1980 1980 1980 1980 1980 1980 ...
##  $ Team     : chr  "Atlanta Hawks" "Boston Celtics" "Chicago Bulls" "Cleveland Cavaliers" ...
##  $ Playoffs : int  1 1 0 0 0 0 0 1 0 1 ...
##  $ W        : int  50 61 30 37 30 16 24 41 37 47 ...
##  $ PTS      : int  8573 9303 8813 9360 8878 8933 8493 9084 9119 8860 ...
##  $ oppPTS   : int  8334 8664 9035 9332 9240 9609 8853 9070 9176 8603 ...
##  $ FG       : int  3261 3617 3362 3811 3462 3643 3527 3599 3639 3582 ...
##  $ FGA      : int  7027 7387 6943 8041 7470 7596 7318 7496 7689 7489 ...
##  $ X2P      : int  3248 3455 3292 3775 3379 3586 3500 3495 3551 3557 ...
##  $ X2PA     : int  6952 6965 6668 7854 7215 7377 7197 7117 7375 7375 ...
##  $ X3P      : int  13 162 70 36 83 57 27 104 88 25 ...
##  $ X3PA     : int  75 422 275 187 255 219 121 379 314 114 ...
##  $ FT       : int  2038 1907 2019 1702 1871 1590 1412 1782 1753 1671 ...
##  $ FTA      : int  2645 2449 2592 2205 2539 2149 1914 2326 2333 2250 ...
##  $ ORB      : int  1369 1227 1115 1307 1311 1226 1155 1394 1398 1187 ...
##  $ DRB      : int  2406 2457 2465 2381 2524 2415 2437 2217 2326 2429 ...
##  $ AST      : int  1913 2198 2152 2108 2079 1950 2028 2149 2148 2123 ...
##  $ STL      : int  782 809 704 764 746 783 779 782 900 863 ...
##  $ BLK      : int  539 308 392 342 404 562 339 373 530 356 ...
##  $ TOV      : int  1495 1539 1684 1370 1533 1742 1492 1565 1517 1439 ...
##  - attr(*, "comment")= chr "glb_trnobs_df"
## NULL
```

```r
# glb_trnobs_df <- data.frame()
# for (symbol in c("Boeing", "CocaCola", "GE", "IBM", "ProcterGamble")) {
#     sym_trnobs_df <- 
#         myimport_data(url=gsub("IBM", symbol, glb_trnng_url), comment="glb_trnobs_df", 
#                                     force_header=TRUE)
#     sym_trnobs_df$Symbol <- symbol
#     glb_trnobs_df <- myrbind_df(glb_trnobs_df, sym_trnobs_df)
# }
                                
# glb_trnobs_df <- 
#     glb_trnobs_df %>% dplyr::filter(Year >= 1999)
                                
if (glb_is_separate_newent_dataset) {
    glb_newobs_df <- myimport_data(url=glb_newdt_url, comment="glb_newobs_df", 
                                   force_header=TRUE)
    
    # To make plots / stats / checks easier in chunk:inspectORexplore.data
    glb_allobs_df <- myrbind_df(glb_trnobs_df, glb_newobs_df); 
    comment(glb_allobs_df) <- "glb_allobs_df"
} else {
    glb_allobs_df <- glb_trnobs_df; comment(glb_allobs_df) <- "glb_allobs_df"
    if (!glb_split_entity_newent_datasets) {
        stop("Not implemented yet") 
        glb_newobs_df <- glb_trnobs_df[sample(1:nrow(glb_trnobs_df),
                                          max(2, nrow(glb_trnobs_df) / 1000)),]                    
    } else      if (glb_split_newdata_method == "condition") {
            glb_newobs_df <- do.call("subset", 
                list(glb_trnobs_df, parse(text=glb_split_newdata_condition)))
            glb_trnobs_df <- do.call("subset", 
                list(glb_trnobs_df, parse(text=paste0("!(", 
                                                      glb_split_newdata_condition,
                                                      ")"))))
        } else if (glb_split_newdata_method == "sample") {
                require(caTools)
                
                set.seed(glb_split_sample.seed)
                split <- sample.split(glb_trnobs_df[, glb_rsp_var_raw], 
                                      SplitRatio=(1-glb_split_newdata_size_ratio))
                glb_newobs_df <- glb_trnobs_df[!split, ] 
                glb_trnobs_df <- glb_trnobs_df[split ,]
        } else if (glb_split_newdata_method == "copy") {  
            glb_trnobs_df <- glb_allobs_df
            comment(glb_trnobs_df) <- "glb_trnobs_df"
            glb_newobs_df <- glb_allobs_df
            comment(glb_newobs_df) <- "glb_newobs_df"
        } else stop("glb_split_newdata_method should be %in% c('condition', 'sample', 'copy')")   

    comment(glb_newobs_df) <- "glb_newobs_df"
    myprint_df(glb_newobs_df)
    str(glb_newobs_df)

    if (glb_split_entity_newent_datasets) {
        myprint_df(glb_trnobs_df)
        str(glb_trnobs_df)        
    }
}         
```

```
## [1] "Reading file ./data/NBA_test.csv..."
## [1] "dimensions of data in ./data/NBA_test.csv: 28 rows x 20 cols"
##   SeasonEnd                Team Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 1      2013       Atlanta Hawks        1 44 8032   7999 3084 6644 2378
## 2      2013       Brooklyn Nets        1 49 7944   7798 2942 6544 2314
## 3      2013   Charlotte Bobcats        0 21 7661   8418 2823 6649 2354
## 4      2013       Chicago Bulls        1 45 7641   7615 2926 6698 2480
## 5      2013 Cleveland Cavaliers        0 24 7913   8297 2993 6901 2446
## 6      2013    Dallas Mavericks        0 41 8293   8342 3182 6892 2576
##   X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV
## 1 4743 706 1901 1158 1619  758 2593 2007 664 369 1219
## 2 4784 628 1760 1432 1958 1047 2460 1668 599 391 1206
## 3 5250 469 1399 1546 2060  917 2389 1587 591 479 1153
## 4 5433 446 1265 1343 1738 1026 2514 1886 588 417 1171
## 5 5320 547 1581 1380 1826 1004 2359 1694 647 334 1149
## 6 5264 606 1628 1323 1669  767 2670 1906 648 454 1144
##    SeasonEnd                 Team Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 1       2013        Atlanta Hawks        1 44 8032   7999 3084 6644 2378
## 5       2013  Cleveland Cavaliers        0 24 7913   8297 2993 6901 2446
## 10      2013      Houston Rockets        1 45 8688   8403 3124 6782 2257
## 11      2013 Los Angeles Clippers        1 56 8289   7760 3160 6608 2533
## 13      2013    Memphis Grizzlies        1 56 7659   7319 2964 6679 2582
## 25      2013    San Antonio Spurs        1 58 8448   7923 3210 6675 2547
##    X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV
## 1  4743 706 1901 1158 1619  758 2593 2007 664 369 1219
## 5  5320 547 1581 1380 1826 1004 2359 1694 647 334 1149
## 10 4413 867 2369 1573 2087  909 2652 1902 679 359 1348
## 11 4856 627 1752 1342 1888  938 2475 1958 784 461 1197
## 13 5572 382 1107 1349 1746 1059 2445 1715 703 436 1144
## 25 4911 663 1764 1365 1725  666 2721 2058 695 446 1206
##    SeasonEnd                   Team Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 23      2013 Portland Trail Blazers        0 33 7995   8255 3009 6715 2336
## 24      2013       Sacramento Kings        0 28 8219   8619 3086 6904 2476
## 25      2013      San Antonio Spurs        1 58 8448   7923 3210 6675 2547
## 26      2013        Toronto Raptors        0 34 7971   8092 2979 6685 2408
## 27      2013              Utah Jazz        0 43 8038   8045 3046 6710 2539
## 28      2013     Washington Wizards        0 29 7644   7852 2910 6693 2365
##    X2PA X3P X3PA   FT  FTA ORB  DRB  AST STL BLK  TOV
## 23 4811 673 1904 1304 1680 874 2474 1784 538 353 1203
## 24 5223 610 1681 1437 1869 943 2385 1708 671 342 1199
## 25 4911 663 1764 1365 1725 666 2721 2058 695 446 1206
## 26 5020 571 1665 1442 1831 871 2426 1765 595 392 1124
## 27 5325 507 1385 1439 1883 989 2457 1859 690 515 1210
## 28 5198 545 1495 1279 1746 887 2652 1775 598 376 1238
## 'data.frame':	28 obs. of  20 variables:
##  $ SeasonEnd: int  2013 2013 2013 2013 2013 2013 2013 2013 2013 2013 ...
##  $ Team     : chr  "Atlanta Hawks" "Brooklyn Nets" "Charlotte Bobcats" "Chicago Bulls" ...
##  $ Playoffs : int  1 1 0 1 0 0 1 0 1 1 ...
##  $ W        : int  44 49 21 45 24 41 57 29 47 45 ...
##  $ PTS      : int  8032 7944 7661 7641 7913 8293 8704 7778 8296 8688 ...
##  $ oppPTS   : int  7999 7798 8418 7615 8297 8342 8287 8105 8223 8403 ...
##  $ FG       : int  3084 2942 2823 2926 2993 3182 3339 2979 3130 3124 ...
##  $ FGA      : int  6644 6544 6649 6698 6901 6892 6983 6638 6840 6782 ...
##  $ X2P      : int  2378 2314 2354 2480 2446 2576 2818 2466 2472 2257 ...
##  $ X2PA     : int  4743 4784 5250 5433 5320 5264 5465 5198 5208 4413 ...
##  $ X3P      : int  706 628 469 446 547 606 521 513 658 867 ...
##  $ X3PA     : int  1901 1760 1399 1265 1581 1628 1518 1440 1632 2369 ...
##  $ FT       : int  1158 1432 1546 1343 1380 1323 1505 1307 1378 1573 ...
##  $ FTA      : int  1619 1958 2060 1738 1826 1669 2148 1870 1744 2087 ...
##  $ ORB      : int  758 1047 917 1026 1004 767 1092 991 885 909 ...
##  $ DRB      : int  2593 2460 2389 2514 2359 2670 2601 2463 2801 2652 ...
##  $ AST      : int  2007 1668 1587 1886 1694 1906 2002 1742 1845 1902 ...
##  $ STL      : int  664 599 591 588 647 648 762 574 567 679 ...
##  $ BLK      : int  369 391 479 417 334 454 533 400 346 359 ...
##  $ TOV      : int  1219 1206 1153 1171 1149 1144 1253 1241 1236 1348 ...
##  - attr(*, "comment")= chr "glb_newobs_df"
## NULL
```

```r
if ((num_nas <- sum(is.na(glb_trnobs_df[, glb_rsp_var_raw]))) > 0)
    stop("glb_trnobs_df$", glb_rsp_var_raw, " contains NAs for ", num_nas, " obs")

if (nrow(glb_trnobs_df) == nrow(glb_allobs_df))
    warning("glb_trnobs_df same as glb_allobs_df")
if (nrow(glb_newobs_df) == nrow(glb_allobs_df))
    warning("glb_newobs_df same as glb_allobs_df")

if (length(glb_drop_vars) > 0) {
    warning("dropping vars: ", paste0(glb_drop_vars, collapse=", "))
    glb_allobs_df <- glb_allobs_df[, setdiff(names(glb_allobs_df), glb_drop_vars)]
    glb_trnobs_df <- glb_trnobs_df[, setdiff(names(glb_trnobs_df), glb_drop_vars)]    
    glb_newobs_df <- glb_newobs_df[, setdiff(names(glb_newobs_df), glb_drop_vars)]    
}

#stop(here"); sav_allobs_df <- glb_allobs_df # glb_allobs_df <- sav_allobs_df
# Check for duplicates in glb_id_var
if (length(glb_id_var) == 0) {
    warning("using .rownames as identifiers for observations")
    glb_allobs_df$.rownames <- rownames(glb_allobs_df)
    glb_trnobs_df$.rownames <- rownames(glb_trnobs_df)
    glb_newobs_df$.rownames <- rownames(glb_newobs_df)    
    glb_id_var <- ".rownames"
}
```

```
## Warning: using .rownames as identifiers for observations
```

```r
if (sum(duplicated(glb_allobs_df[, glb_id_var, FALSE])) > 0)
    stop(glb_id_var, " duplicated in glb_allobs_df")
glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, glb_id_var)

# Combine trnent & newent into glb_allobs_df for easier manipulation
glb_trnobs_df$.src <- "Train"; glb_newobs_df$.src <- "Test"; 
glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, ".src")
glb_allobs_df <- myrbind_df(glb_trnobs_df, glb_newobs_df)
comment(glb_allobs_df) <- "glb_allobs_df"
glb_allobs_df <- orderBy(reformulate(glb_id_var), glb_allobs_df)
glb_trnobs_df <- glb_newobs_df <- NULL

glb_chunks_df <- myadd_chunk(glb_chunks_df, "inspect.data", major.inc=TRUE)
```

```
##          label step_major step_minor   bgn   end elapsed
## 1  import.data          1          0 9.041 9.476   0.435
## 2 inspect.data          2          0 9.477    NA      NA
```

## Step `2.0: inspect data`

```r
#print(str(glb_allobs_df))
#View(glb_allobs_df)

dsp_class_dstrb <- function(var) {
    xtab_df <- mycreate_xtab_df(glb_allobs_df, c(".src", var))
    rownames(xtab_df) <- xtab_df$.src
    xtab_df <- subset(xtab_df, select=-.src)
    print(xtab_df)
    print(xtab_df / rowSums(xtab_df, na.rm=TRUE))    
}    

# Performed repeatedly in other chunks
glb_chk_data <- function() {
    # Histogram of predictor in glb_trnobs_df & glb_newobs_df
    print(myplot_histogram(glb_allobs_df, glb_rsp_var_raw) + facet_wrap(~ .src))
    
    if (glb_is_classification) 
        dsp_class_dstrb(var=ifelse(glb_rsp_var %in% names(glb_allobs_df), 
                                   glb_rsp_var, glb_rsp_var_raw))
    mycheck_problem_data(glb_allobs_df)
}
glb_chk_data()
```

```
## Warning in loop_apply(n, do.ply): position_stack requires constant width:
## output may be incorrect
```

```
## Warning in loop_apply(n, do.ply): position_stack requires constant width:
## output may be incorrect
```

![](NBA_PTS2_files/figure-html/inspect.data-1.png) 

```
## [1] "numeric data missing in glb_allobs_df: "
## named integer(0)
## [1] "numeric data w/ 0s in glb_allobs_df: "
## Playoffs 
##      369 
## [1] "numeric data w/ Infs in glb_allobs_df: "
## named integer(0)
## [1] "numeric data w/ NaNs in glb_allobs_df: "
## named integer(0)
## [1] "string data missing in glb_allobs_df: "
##      Team .rownames 
##         0         0
```

```r
# Create new features that help diagnostics
if (!is.null(glb_map_rsp_raw_to_var)) {
    glb_allobs_df[, glb_rsp_var] <- 
        glb_map_rsp_raw_to_var(glb_allobs_df[, glb_rsp_var_raw])
    mycheck_map_results(mapd_df=glb_allobs_df, 
                        from_col_name=glb_rsp_var_raw, to_col_name=glb_rsp_var)
        
    if (glb_is_classification) dsp_class_dstrb(glb_rsp_var)
}

#   Convert dates to numbers 
#       typically, dates come in as chars; 
#           so this must be done before converting chars to factors

myextract_dates_df <- function(df, vars, id_vars, rsp_var) {
    keep_feats <- c(NULL)
    for (var in vars) {
        dates_df            <- df[, id_vars, FALSE]        
        dates_df[, rsp_var] <- df[, rsp_var, FALSE]
        #dates_df <- data.frame(.date=strptime(df[, var], "%Y-%m-%d %H:%M:%S"))
        dates_df <- cbind(dates_df, data.frame(.date=strptime(df[, var], 
            glb_date_fmts[[var]], tz=glb_date_tzs[[var]])))
#         print(dates_df[is.na(dates_df$.date), c("ID", "Arrest.fctr", ".date")])
#         print(glb_allobs_df[is.na(dates_df$.date), c("ID", "Arrest.fctr", "Date")])     
#         print(head(glb_allobs_df[grepl("4/7/02 .:..", glb_allobs_df$Date), c("ID", "Arrest.fctr", "Date")]))
#         print(head(strptime(glb_allobs_df[grepl("4/7/02 .:..", glb_allobs_df$Date), "Date"], "%m/%e/%y %H:%M"))
        # Wrong data during EST->EDT transition
#         tmp <- strptime("4/7/02 2:00","%m/%e/%y %H:%M:%S"); print(tmp); print(is.na(tmp))
#         dates_df[dates_df$ID == 2068197, .date] <- tmp
#         grep("(.*?) 2:(.*)", glb_allobs_df[is.na(dates_df$.date), "Date"], value=TRUE)
#         dates_df[is.na(dates_df$.date), ".date"] <- 
#             data.frame(.date=strptime(gsub("(.*?) 2:(.*)", "\\1 3:\\2",
#                 glb_allobs_df[is.na(dates_df$.date), "Date"]), "%m/%e/%y %H:%M"))$.date
        if (sum(is.na(dates_df$.date)) > 0) {
            stop("NA POSIX dates for ", var)
            print(df[is.na(dates_df$.date), c(id_vars, rsp_var, var)])
        }    
        
        .date <- dates_df$.date
        dates_df[, paste0(var, ".POSIX")] <- .date
        dates_df[, paste0(var, ".year")] <- as.numeric(format(.date, "%Y"))
        dates_df[, paste0(var, ".year.fctr")] <- as.factor(format(.date, "%Y")) 
        dates_df[, paste0(var, ".month")] <- as.numeric(format(.date, "%m"))
        dates_df[, paste0(var, ".month.fctr")] <- as.factor(format(.date, "%m"))
        dates_df[, paste0(var, ".date")] <- as.numeric(format(.date, "%d"))
        dates_df[, paste0(var, ".date.fctr")] <- 
            cut(as.numeric(format(.date, "%d")), 5) # by month week  
        dates_df[, paste0(var, ".juliandate")] <- as.numeric(format(.date, "%j"))        
        
        # wkday Sun=0; Mon=1; ...; Sat=6
        dates_df[, paste0(var, ".wkday")] <- as.numeric(format(.date, "%w"))
        dates_df[, paste0(var, ".wkday.fctr")] <- as.factor(format(.date, "%w"))
        
        # Get US Federal Holidays for relevant years
        require(XML)
        doc.html = htmlTreeParse('http://about.usps.com/news/events-calendar/2012-federal-holidays.htm', useInternal = TRUE)
        
#         # Extract all the paragraphs (HTML tag is p, starting at
#         # the root of the document). Unlist flattens the list to
#         # create a character vector.
#         doc.text = unlist(xpathApply(doc.html, '//p', xmlValue))
#         # Replace all \n by spaces
#         doc.text = gsub('\\n', ' ', doc.text)
#         # Join all the elements of the character vector into a single
#         # character string, separated by spaces
#         doc.text = paste(doc.text, collapse = ' ')
        
        # parse the tree by tables
        txt <- unlist(strsplit(xpathSApply(doc.html, "//*/table", xmlValue), "\n"))
        # do some clean up with regular expressions
        txt <- grep("day, ", txt, value=TRUE)
        txt <- trimws(gsub("(.*?)day, (.*)", "\\2", txt))
#         txt <- gsub("\t","",txt)
#         txt <- sub("^[[:space:]]*(.*?)[[:space:]]*$", "\\1", txt, perl=TRUE)
#         txt <- txt[!(txt %in% c("", "|"))]
        hldays <- strptime(paste(txt, ", 2012", sep=""), "%B %e, %Y")
        dates_df[, paste0(var, ".hlday")] <- 
            ifelse(format(.date, "%Y-%m-%d") %in% hldays, 1, 0)
        
        # NYState holidays 1.9., 13.10., 11.11., 27.11., 25.12.
        
        dates_df[, paste0(var, ".wkend")] <- as.numeric(
            (dates_df[, paste0(var, ".wkday")] %in% c(0, 6)) | 
            dates_df[, paste0(var, ".hlday")] )
        
        dates_df[, paste0(var, ".hour")] <- as.numeric(format(.date, "%H"))
        dates_df[, paste0(var, ".hour.fctr")] <- 
            if (length(unique(vals <- as.numeric(format(.date, "%H")))) <= 1)
                   vals else cut(vals, 3) # by work-shift    
        dates_df[, paste0(var, ".minute")] <- as.numeric(format(.date, "%M")) 
        dates_df[, paste0(var, ".minute.fctr")] <- 
            if (length(unique(vals <- as.numeric(format(.date, "%M")))) <= 1)
                   vals else cut(vals, 4) # by quarter-hours    
        dates_df[, paste0(var, ".second")] <- as.numeric(format(.date, "%S")) 
        dates_df[, paste0(var, ".second.fctr")] <- 
            if (length(unique(vals <- as.numeric(format(.date, "%S")))) <= 1)
                   vals else cut(vals, 4) # by quarter-minutes

        dates_df[, paste0(var, ".day.minutes")] <- 
            60 * dates_df[, paste0(var, ".hour")] + 
                 dates_df[, paste0(var, ".minute")]
        if ((unq_vals_n <- length(unique(dates_df[, paste0(var, ".day.minutes")]))) > 1) {
            max_degree <- min(unq_vals_n, 5)
            dates_df[, paste0(var, ".day.minutes.poly.", 1:max_degree)] <- 
                as.matrix(poly(dates_df[, paste0(var, ".day.minutes")], max_degree))
        } else max_degree <- 0   
        
#         print(gp <- myplot_box(df=dates_df, ycol_names="PubDate.day.minutes", 
#                                xcol_name=rsp_var))
#         print(gp <- myplot_scatter(df=dates_df, xcol_name=".rownames", 
#                         ycol_name="PubDate.day.minutes", colorcol_name=rsp_var))
#         print(gp <- myplot_scatter(df=dates_df, xcol_name="PubDate.juliandate", 
#                         ycol_name="PubDate.day.minutes.poly.1", colorcol_name=rsp_var))
#         print(gp <- myplot_scatter(df=dates_df, xcol_name="PubDate.day.minutes", 
#                         ycol_name="PubDate.day.minutes.poly.4", colorcol_name=rsp_var))
# 
#         print(gp <- myplot_scatter(df=dates_df, xcol_name="PubDate.juliandate", 
#                         ycol_name="PubDate.day.minutes", colorcol_name=rsp_var, smooth=TRUE))
#         print(gp <- myplot_scatter(df=dates_df, xcol_name="PubDate.juliandate", 
#                         ycol_name="PubDate.day.minutes.poly.4", colorcol_name=rsp_var, smooth=TRUE))
#         print(gp <- myplot_scatter(df=dates_df, xcol_name="PubDate.juliandate", 
#                         ycol_name=c("PubDate.day.minutes", "PubDate.day.minutes.poly.4"), 
#                         colorcol_name=rsp_var))
        
#         print(gp <- myplot_scatter(df=subset(dates_df, Popular.fctr=="Y"), 
#                                    xcol_name=paste0(var, ".juliandate"), 
#                         ycol_name=paste0(var, ".day.minutes", colorcol_name=rsp_var))
#         print(gp <- myplot_box(df=dates_df, ycol_names=paste0(var, ".hour"), 
#                                xcol_name=rsp_var))
#         print(gp <- myplot_bar(df=dates_df, ycol_names=paste0(var, ".hour.fctr"), 
#                                xcol_name=rsp_var, 
#                                colorcol_name=paste0(var, ".hour.fctr")))                
        keep_feats <- paste(var, 
            c(".POSIX", ".year.fctr", ".month.fctr", ".date.fctr", ".wkday.fctr", 
              ".wkend", ".hour.fctr", ".minute.fctr", ".second.fctr"), sep="")
        if (max_degree > 0)
            keep_feats <- union(keep_feats, paste(var, 
              paste0(".day.minutes.poly.", 1:max_degree), sep=""))
        keep_feats <- intersect(keep_feats, names(dates_df))        
    }
    #myprint_df(dates_df)
    return(dates_df[, keep_feats])
}

if (!is.null(glb_date_vars)) {
    glb_allobs_df <- cbind(glb_allobs_df, 
        myextract_dates_df(df=glb_allobs_df, vars=glb_date_vars, 
                           id_vars=glb_id_var, rsp_var=glb_rsp_var))
    glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, 
                                          paste(glb_date_vars, c("", ".POSIX"), sep=""))

    for (feat in glb_date_vars) {
        glb_allobs_df <- orderBy(reformulate(paste0(feat, ".POSIX")), glb_allobs_df)
#         print(myplot_scatter(glb_allobs_df, xcol_name=paste0(feat, ".POSIX"),
#                              ycol_name=glb_rsp_var, colorcol_name=glb_rsp_var))
        print(myplot_scatter(glb_allobs_df[glb_allobs_df[, paste0(feat, ".POSIX")] >=
                                               strptime("2012-12-01", "%Y-%m-%d"), ], 
                             xcol_name=paste0(feat, ".POSIX"),
                             ycol_name=glb_rsp_var, colorcol_name=paste0(feat, ".wkend")))

        # Create features that measure the gap between previous timestamp in the data
        require(zoo)
        z <- zoo(as.numeric(as.POSIXlt(glb_allobs_df[, paste0(feat, ".POSIX")])))
        glb_allobs_df[, paste0(feat, ".zoo")] <- z
        print(head(glb_allobs_df[, c(glb_id_var, feat, paste0(feat, ".zoo"))]))
        print(myplot_scatter(glb_allobs_df[glb_allobs_df[,  paste0(feat, ".POSIX")] >
                                            strptime("2012-10-01", "%Y-%m-%d"), ], 
                            xcol_name=paste0(feat, ".zoo"), ycol_name=glb_rsp_var,
                            colorcol_name=glb_rsp_var))
        b <- zoo(, seq(nrow(glb_allobs_df)))
        
        last1 <- as.numeric(merge(z-lag(z, -1), b, all=TRUE)); last1[is.na(last1)] <- 0
        glb_allobs_df[, paste0(feat, ".last1.log")] <- log(1 + last1)
        print(gp <- myplot_box(df=glb_allobs_df[glb_allobs_df[, 
                                                    paste0(feat, ".last1.log")] > 0, ], 
                               ycol_names=paste0(feat, ".last1.log"), 
                               xcol_name=glb_rsp_var))
        
        last10 <- as.numeric(merge(z-lag(z, -10), b, all=TRUE)); last10[is.na(last10)] <- 0
        glb_allobs_df[, paste0(feat, ".last10.log")] <- log(1 + last10)
        print(gp <- myplot_box(df=glb_allobs_df[glb_allobs_df[, 
                                                    paste0(feat, ".last10.log")] > 0, ], 
                               ycol_names=paste0(feat, ".last10.log"), 
                               xcol_name=glb_rsp_var))
        
        last100 <- as.numeric(merge(z-lag(z, -100), b, all=TRUE)); last100[is.na(last100)] <- 0
        glb_allobs_df[, paste0(feat, ".last100.log")] <- log(1 + last100)
        print(gp <- myplot_box(df=glb_allobs_df[glb_allobs_df[, 
                                                    paste0(feat, ".last100.log")] > 0, ], 
                               ycol_names=paste0(feat, ".last100.log"), 
                               xcol_name=glb_rsp_var))
        
        glb_allobs_df <- orderBy(reformulate(glb_id_var), glb_allobs_df)
        glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, 
                                                c(paste0(feat, ".zoo")))
        # all2$last3 = as.numeric(merge(z-lag(z, -3), b, all = TRUE))
        # all2$last5 = as.numeric(merge(z-lag(z, -5), b, all = TRUE))
        # all2$last10 = as.numeric(merge(z-lag(z, -10), b, all = TRUE))
        # all2$last20 = as.numeric(merge(z-lag(z, -20), b, all = TRUE))
        # all2$last50 = as.numeric(merge(z-lag(z, -50), b, all = TRUE))
        # 
        # 
        # # order table
        # all2 = all2[order(all2$id),]
        # 
        # ## fill in NAs
        # # count averages
        # na.avg = all2 %>% group_by(weekend, hour) %>% dplyr::summarise(
        #     last1=mean(last1, na.rm=TRUE),
        #     last3=mean(last3, na.rm=TRUE),
        #     last5=mean(last5, na.rm=TRUE),
        #     last10=mean(last10, na.rm=TRUE),
        #     last20=mean(last20, na.rm=TRUE),
        #     last50=mean(last50, na.rm=TRUE)
        # )
        # 
        # # fill in averages
        # na.merge = merge(all2, na.avg, by=c("weekend","hour"))
        # na.merge = na.merge[order(na.merge$id),]
        # for(i in c("last1", "last3", "last5", "last10", "last20", "last50")) {
        #     y = paste0(i, ".y")
        #     idx = is.na(all2[[i]])
        #     all2[idx,][[i]] <- na.merge[idx,][[y]]
        # }
        # rm(na.avg, na.merge, b, i, idx, n, pd, sec, sh, y, z)
    }
}

# check distribution of all numeric data
dsp_numeric_feats_dstrb <- function(feats_vctr) {
    for (feat in feats_vctr) {
        print(sprintf("feat: %s", feat))
        if (glb_is_regression)
            gp <- myplot_scatter(df=glb_allobs_df, ycol_name=glb_rsp_var, xcol_name=feat,
                                 smooth=TRUE)
        if (glb_is_classification)
            gp <- myplot_box(df=glb_allobs_df, ycol_names=feat, xcol_name=glb_rsp_var)
        if (inherits(glb_allobs_df[, feat], "factor"))
            gp <- gp + facet_wrap(reformulate(feat))
        print(gp)
    }
}
# dsp_numeric_vars_dstrb(setdiff(names(glb_allobs_df), 
#                                 union(myfind_chr_cols_df(glb_allobs_df), 
#                                       c(glb_rsp_var_raw, glb_rsp_var))))                                      

add_new_diag_feats <- function(obs_df, ref_df=glb_allobs_df) {
    require(plyr)
    
    obs_df <- mutate(obs_df,
#         <col_name>.NA=is.na(<col_name>),

#         <col_name>.fctr=factor(<col_name>, 
#                     as.factor(union(obs_df$<col_name>, obs_twin_df$<col_name>))), 
#         <col_name>.fctr=relevel(factor(<col_name>, 
#                     as.factor(union(obs_df$<col_name>, obs_twin_df$<col_name>))),
#                                   "<ref_val>"), 
#         <col2_name>.fctr=relevel(factor(ifelse(<col1_name> == <val>, "<oth_val>", "<ref_val>")), 
#                               as.factor(c("R", "<ref_val>")),
#                               ref="<ref_val>"),

          # This doesn't work - use sapply instead
#         <col_name>.fctr_num=grep(<col_name>, levels(<col_name>.fctr)), 
#         
#         Date.my=as.Date(strptime(Date, "%m/%d/%y %H:%M")),
#         Year=year(Date.my),
#         Month=months(Date.my),
#         Weekday=weekdays(Date.my)

#         <col_name>=<table>[as.character(<col2_name>)],
#         <col_name>=as.numeric(<col2_name>),

#         <col_name> = trunc(<col2_name> / 100),

        .rnorm = rnorm(n=nrow(obs_df))
                        )

    # If levels of a factor are different across obs_df & glb_newobs_df; predict.glm fails  
    # Transformations not handled by mutate
#     obs_df$<col_name>.fctr.num <- sapply(1:nrow(obs_df), 
#         function(row_ix) grep(obs_df[row_ix, "<col_name>"],
#                               levels(obs_df[row_ix, "<col_name>.fctr"])))
    
    #print(summary(obs_df))
    #print(sapply(names(obs_df), function(col) sum(is.na(obs_df[, col]))))
    return(obs_df)
}
glb_allobs_df <- add_new_diag_feats(glb_allobs_df)
```

```
## Loading required package: plyr
```

```r
require(dplyr)
```

```
## Loading required package: dplyr
## 
## Attaching package: 'dplyr'
## 
## The following objects are masked from 'package:plyr':
## 
##     arrange, count, desc, failwith, id, mutate, rename, summarise,
##     summarize
## 
## The following object is masked from 'package:stats':
## 
##     filter
## 
## The following objects are masked from 'package:base':
## 
##     intersect, setdiff, setequal, union
```

```r
#stop(here"); sav_allobs_df <- glb_allobs_df # glb_allobs_df <- sav_allobs_df
# Merge some <descriptor>
# glb_allobs_df$<descriptor>.my <- glb_allobs_df$<descriptor>
# glb_allobs_df[grepl("\\bAIRPORT\\b", glb_allobs_df$<descriptor>.my),
#               "<descriptor>.my"] <- "AIRPORT"
# glb_allobs_df$<descriptor>.my <-
#     plyr::revalue(glb_allobs_df$<descriptor>.my, c(
#         "ABANDONED BUILDING" = "OTHER",
#         "##"                      = "##"
#     ))
# print(<descriptor>_freq_df <- mycreate_sqlxtab_df(glb_allobs_df, c("<descriptor>.my")))
# # print(dplyr::filter(<descriptor>_freq_df, grepl("(MEDICAL|DENTAL|OFFICE)", <descriptor>.my)))
# # print(dplyr::filter(dplyr::select(glb_allobs_df, -<var.zoo>), 
# #                     grepl("STORE", <descriptor>.my)))
# glb_exclude_vars_as_features <- c(glb_exclude_vars_as_features, "<descriptor>")

# Check distributions of newly transformed / extracted vars
#   Enhancement: remove vars that were displayed ealier
dsp_numeric_feats_dstrb(feats_vctr=setdiff(names(glb_allobs_df), 
        c(myfind_chr_cols_df(glb_allobs_df), glb_rsp_var_raw, glb_rsp_var, 
          glb_exclude_vars_as_features)))
```

```
## [1] "feat: FG"
```

![](NBA_PTS2_files/figure-html/inspect.data-2.png) 

```
## [1] "feat: FGA"
```

![](NBA_PTS2_files/figure-html/inspect.data-3.png) 

```
## [1] "feat: X2P"
```

![](NBA_PTS2_files/figure-html/inspect.data-4.png) 

```
## [1] "feat: X2PA"
```

![](NBA_PTS2_files/figure-html/inspect.data-5.png) 

```
## [1] "feat: X3P"
```

![](NBA_PTS2_files/figure-html/inspect.data-6.png) 

```
## [1] "feat: X3PA"
```

![](NBA_PTS2_files/figure-html/inspect.data-7.png) 

```
## [1] "feat: FT"
```

![](NBA_PTS2_files/figure-html/inspect.data-8.png) 

```
## [1] "feat: FTA"
```

![](NBA_PTS2_files/figure-html/inspect.data-9.png) 

```
## [1] "feat: ORB"
```

![](NBA_PTS2_files/figure-html/inspect.data-10.png) 

```
## [1] "feat: DRB"
```

![](NBA_PTS2_files/figure-html/inspect.data-11.png) 

```
## [1] "feat: AST"
```

![](NBA_PTS2_files/figure-html/inspect.data-12.png) 

```
## [1] "feat: STL"
```

![](NBA_PTS2_files/figure-html/inspect.data-13.png) 

```
## [1] "feat: BLK"
```

![](NBA_PTS2_files/figure-html/inspect.data-14.png) 

```
## [1] "feat: TOV"
```

![](NBA_PTS2_files/figure-html/inspect.data-15.png) 

```
## [1] "feat: .rnorm"
```

![](NBA_PTS2_files/figure-html/inspect.data-16.png) 

```r
#   Convert factors to dummy variables
#   Build splines   require(splines); bsBasis <- bs(training$age, df=3)

#pairs(subset(glb_trnobs_df, select=-c(col_symbol)))
# Check for glb_newobs_df & glb_trnobs_df features range mismatches

# Other diagnostics:
# print(subset(glb_trnobs_df, <col1_name> == max(glb_trnobs_df$<col1_name>, na.rm=TRUE) & 
#                         <col2_name> <= mean(glb_trnobs_df$<col1_name>, na.rm=TRUE)))

# print(glb_trnobs_df[which.max(glb_trnobs_df$<col_name>),])

# print(<col_name>_freq_glb_trnobs_df <- mycreate_tbl_df(glb_trnobs_df, "<col_name>"))
# print(which.min(table(glb_trnobs_df$<col_name>)))
# print(which.max(table(glb_trnobs_df$<col_name>)))
# print(which.max(table(glb_trnobs_df$<col1_name>, glb_trnobs_df$<col2_name>)[, 2]))
# print(table(glb_trnobs_df$<col1_name>, glb_trnobs_df$<col2_name>))
# print(table(is.na(glb_trnobs_df$<col1_name>), glb_trnobs_df$<col2_name>))
# print(table(sign(glb_trnobs_df$<col1_name>), glb_trnobs_df$<col2_name>))
# print(mycreate_xtab_df(glb_trnobs_df, <col1_name>))
# print(mycreate_xtab_df(glb_trnobs_df, c(<col1_name>, <col2_name>)))
# print(<col1_name>_<col2_name>_xtab_glb_trnobs_df <- 
#   mycreate_xtab_df(glb_trnobs_df, c("<col1_name>", "<col2_name>")))
# <col1_name>_<col2_name>_xtab_glb_trnobs_df[is.na(<col1_name>_<col2_name>_xtab_glb_trnobs_df)] <- 0
# print(<col1_name>_<col2_name>_xtab_glb_trnobs_df <- 
#   mutate(<col1_name>_<col2_name>_xtab_glb_trnobs_df, 
#             <col3_name>=(<col1_name> * 1.0) / (<col1_name> + <col2_name>))) 
# print(mycreate_sqlxtab_df(glb_allobs_df, c("<col1_name>", "<col2_name>")))

# print(<col2_name>_min_entity_arr <- 
#    sort(tapply(glb_trnobs_df$<col1_name>, glb_trnobs_df$<col2_name>, min, na.rm=TRUE)))
# print(<col1_name>_na_by_<col2_name>_arr <- 
#    sort(tapply(glb_trnobs_df$<col1_name>.NA, glb_trnobs_df$<col2_name>, mean, na.rm=TRUE)))

# Other plots:
# print(myplot_box(df=glb_trnobs_df, ycol_names="<col1_name>"))
# print(myplot_box(df=glb_trnobs_df, ycol_names="<col1_name>", xcol_name="<col2_name>"))
# print(myplot_line(subset(glb_trnobs_df, Symbol %in% c("CocaCola", "ProcterGamble")), 
#                   "Date.POSIX", "StockPrice", facet_row_colnames="Symbol") + 
#     geom_vline(xintercept=as.numeric(as.POSIXlt("2003-03-01"))) +
#     geom_vline(xintercept=as.numeric(as.POSIXlt("1983-01-01")))        
#         )
# print(myplot_line(subset(glb_trnobs_df, Date.POSIX > as.POSIXct("2004-01-01")), 
#                   "Date.POSIX", "StockPrice") +
#     geom_line(aes(color=Symbol)) + 
#     coord_cartesian(xlim=c(as.POSIXct("1990-01-01"),
#                            as.POSIXct("2000-01-01"))) +     
#     coord_cartesian(ylim=c(0, 250)) +     
#     geom_vline(xintercept=as.numeric(as.POSIXlt("1997-09-01"))) +
#     geom_vline(xintercept=as.numeric(as.POSIXlt("1997-11-01")))        
#         )
# print(myplot_scatter(glb_allobs_df, "<col1_name>", "<col2_name>", smooth=TRUE))
# print(myplot_scatter(glb_allobs_df, "<col1_name>", "<col2_name>", colorcol_name="<Pred.fctr>") + 
#         geom_point(data=subset(glb_allobs_df, <condition>), 
#                     mapping=aes(x=<x_var>, y=<y_var>), color="red", shape=4, size=5) +
#         geom_vline(xintercept=84))

rm(srt_allobs_df, last1, last10, last100, pd)
```

```
## Warning in rm(srt_allobs_df, last1, last10, last100, pd): object
## 'srt_allobs_df' not found
```

```
## Warning in rm(srt_allobs_df, last1, last10, last100, pd): object 'last1'
## not found
```

```
## Warning in rm(srt_allobs_df, last1, last10, last100, pd): object 'last10'
## not found
```

```
## Warning in rm(srt_allobs_df, last1, last10, last100, pd): object 'last100'
## not found
```

```
## Warning in rm(srt_allobs_df, last1, last10, last100, pd): object 'pd' not
## found
```

```r
glb_chunks_df <- myadd_chunk(glb_chunks_df, "scrub.data", major.inc=FALSE)
```

```
##          label step_major step_minor    bgn    end elapsed
## 2 inspect.data          2          0  9.477 16.557    7.08
## 3   scrub.data          2          1 16.557     NA      NA
```

### Step `2.1: scrub data`

```r
# Options:
#   1. Not fill missing vars
#   2. Fill missing numerics with a different algorithm
#   3. Fill missing chars with data based on clusters 

mycheck_problem_data(glb_allobs_df)
```

```
## [1] "numeric data missing in glb_allobs_df: "
## named integer(0)
## [1] "numeric data w/ 0s in glb_allobs_df: "
## Playoffs 
##      369 
## [1] "numeric data w/ Infs in glb_allobs_df: "
## named integer(0)
## [1] "numeric data w/ NaNs in glb_allobs_df: "
## named integer(0)
## [1] "string data missing in glb_allobs_df: "
##      Team .rownames 
##         0         0
```

```r
# if (!is.null(glb_force_0_to_NA_vars)) {
#     for (feat in glb_force_0_to_NA_vars) {
#         warning("Forcing ", sum(glb_allobs_df[, feat] == 0),
#                 " obs with ", feat, " 0s to NAs")
#         glb_allobs_df[glb_allobs_df[, feat] == 0, feat] <- NA
#     }
# }

mycheck_problem_data(glb_allobs_df)
```

```
## [1] "numeric data missing in glb_allobs_df: "
## named integer(0)
## [1] "numeric data w/ 0s in glb_allobs_df: "
## Playoffs 
##      369 
## [1] "numeric data w/ Infs in glb_allobs_df: "
## named integer(0)
## [1] "numeric data w/ NaNs in glb_allobs_df: "
## named integer(0)
## [1] "string data missing in glb_allobs_df: "
##      Team .rownames 
##         0         0
```

```r
dsp_catgs <- function() {
    print("NewsDesk:")
    print(table(glb_allobs_df$NewsDesk))
    print("SectionName:")    
    print(table(glb_allobs_df$SectionName))
    print("SubsectionName:")        
    print(table(glb_allobs_df$SubsectionName))
}

# sel_obs <- function(Popular=NULL, 
#                     NewsDesk=NULL, SectionName=NULL, SubsectionName=NULL,
#         Headline.contains=NULL, Snippet.contains=NULL, Abstract.contains=NULL,
#         Headline.pfx=NULL, NewsDesk.nb=NULL, .clusterid=NULL, myCategory=NULL,
#         perl=FALSE) {
sel_obs <- function(vars_lst) {
    tmp_df <- glb_allobs_df
    # Does not work for Popular == NAs ???
    if (!is.null(Popular)) {
        if (is.na(Popular))
            tmp_df <- tmp_df[is.na(tmp_df$Popular), ] else   
            tmp_df <- tmp_df[tmp_df$Popular == Popular, ]    
    }    
    if (!is.null(NewsDesk)) 
        tmp_df <- tmp_df[tmp_df$NewsDesk == NewsDesk, ]
    if (!is.null(SectionName)) 
        tmp_df <- tmp_df[tmp_df$SectionName == SectionName, ]
    if (!is.null(SubsectionName)) 
        tmp_df <- tmp_df[tmp_df$SubsectionName == SubsectionName, ]
    if (!is.null(Headline.contains))
        tmp_df <- 
            tmp_df[grep(Headline.contains, tmp_df$Headline, perl=perl), ]
    if (!is.null(Snippet.contains))
        tmp_df <- 
            tmp_df[grep(Snippet.contains, tmp_df$Snippet, perl=perl), ]
    if (!is.null(Abstract.contains))
        tmp_df <- 
            tmp_df[grep(Abstract.contains, tmp_df$Abstract, perl=perl), ]
    if (!is.null(Headline.pfx)) {
        if (length(grep("Headline.pfx", names(tmp_df), fixed=TRUE, value=TRUE))
            > 0) tmp_df <- 
                tmp_df[tmp_df$Headline.pfx == Headline.pfx, ] else
        warning("glb_allobs_df does not contain Headline.pfx; ignoring that filter")                    
    }    
    if (!is.null(NewsDesk.nb)) {
        if (any(grepl("NewsDesk.nb", names(tmp_df), fixed=TRUE)) > 0) 
            tmp_df <- 
                tmp_df[tmp_df$NewsDesk.nb == NewsDesk.nb, ] else
        warning("glb_allobs_df does not contain NewsDesk.nb; ignoring that filter")                    
    }    
    if (!is.null(.clusterid)) {
        if (any(grepl(".clusterid", names(tmp_df), fixed=TRUE)) > 0) 
            tmp_df <- 
                tmp_df[tmp_df$clusterid == clusterid, ] else
        warning("glb_allobs_df does not contain clusterid; ignoring that filter")                       }
    if (!is.null(myCategory)) {    
        if (!(myCategory %in% names(glb_allobs_df)))
            tmp_df <-
                tmp_df[tmp_df$myCategory == myCategory, ] else
        warning("glb_allobs_df does not contain myCategory; ignoring that filter")                    
    }    
    
    return(glb_allobs_df$UniqueID %in% tmp_df$UniqueID)
}

dsp_obs <- function(..., cols=c(NULL), all=FALSE) {
    tmp_df <- glb_allobs_df[sel_obs(...), 
                            union(c("UniqueID", "Popular", "myCategory", "Headline"), cols), FALSE]
    if(all) { print(tmp_df) } else { myprint_df(tmp_df) }
}
#dsp_obs(Popular=1, NewsDesk="", SectionName="", Headline.contains="Boehner")
# dsp_obs(Popular=1, NewsDesk="", SectionName="")
# dsp_obs(Popular=NA, NewsDesk="", SectionName="")

dsp_tbl <- function(...) {
    tmp_entity_df <- glb_allobs_df[sel_obs(...), ]
    tmp_tbl <- table(tmp_entity_df$NewsDesk, 
                     tmp_entity_df$SectionName,
                     tmp_entity_df$SubsectionName, 
                     tmp_entity_df$Popular, useNA="ifany")
    #print(names(tmp_tbl))
    #print(dimnames(tmp_tbl))
    print(tmp_tbl)
}

dsp_hdlxtab <- function(str) 
    print(mycreate_sqlxtab_df(glb_allobs_df[sel_obs(Headline.contains=str), ],
                           c("Headline.pfx", "Headline", glb_rsp_var)))
#dsp_hdlxtab("(1914)|(1939)")

dsp_catxtab <- function(str) 
    print(mycreate_sqlxtab_df(glb_allobs_df[sel_obs(Headline.contains=str), ],
        c("Headline.pfx", "NewsDesk", "SectionName", "SubsectionName", glb_rsp_var)))
# dsp_catxtab("1914)|(1939)")
# dsp_catxtab("19(14|39|64):")
# dsp_catxtab("19..:")

# Create myCategory <- NewsDesk#SectionName#SubsectionName
#   Fix some data before merging categories
# glb_allobs_df[sel_obs(Headline.contains="Your Turn:", NewsDesk=""),
#               "NewsDesk"] <- "Styles"
# glb_allobs_df[sel_obs(Headline.contains="School", NewsDesk="", SectionName="U.S.",
#                       SubsectionName=""),
#               "SubsectionName"] <- "Education"
# glb_allobs_df[sel_obs(Headline.contains="Today in Small Business:", NewsDesk="Business"),
#               "SectionName"] <- "Business Day"
# glb_allobs_df[sel_obs(Headline.contains="Today in Small Business:", NewsDesk="Business"),
#               "SubsectionName"] <- "Small Business"
# glb_allobs_df[sel_obs(Headline.contains="Readers Respond:"),
#               "SectionName"] <- "Opinion"
# glb_allobs_df[sel_obs(Headline.contains="Readers Respond:"),
#               "SubsectionName"] <- "Room For Debate"

# glb_allobs_df[sel_obs(NewsDesk="Business", SectionName="", SubsectionName="", Popular=NA),
#               "SubsectionName"] <- "Small Business"
# print(glb_allobs_df[glb_allobs_df$UniqueID %in% c(7973), 
#     c("UniqueID", "Headline", "myCategory", "NewsDesk", "SectionName", "SubsectionName")])
# 
# glb_allobs_df[sel_obs(NewsDesk="Business", SectionName="", SubsectionName=""),
#               "SectionName"] <- "Technology"
# print(glb_allobs_df[glb_allobs_df$UniqueID %in% c(5076, 5736, 5924, 5911, 6532), 
#     c("UniqueID", "Headline", "myCategory", "NewsDesk", "SectionName", "SubsectionName")])
# 
# glb_allobs_df[sel_obs(SectionName="Health"),
#               "NewsDesk"] <- "Science"
# glb_allobs_df[sel_obs(SectionName="Travel"),
#               "NewsDesk"] <- "Travel"
# 
# glb_allobs_df[sel_obs(SubsectionName="Fashion & Style"),
#               "SectionName"] <- ""
# glb_allobs_df[sel_obs(SubsectionName="Fashion & Style"),
#               "SubsectionName"] <- ""
# glb_allobs_df[sel_obs(NewsDesk="Styles", SectionName="", SubsectionName="", Popular=1),
#               "SectionName"] <- "U.S."
# print(glb_allobs_df[glb_allobs_df$UniqueID %in% c(5486), 
#     c("UniqueID", "Headline", "myCategory", "NewsDesk", "SectionName", "SubsectionName")])
# 
# glb_allobs_df$myCategory <- paste(glb_allobs_df$NewsDesk, 
#                                   glb_allobs_df$SectionName,
#                                   glb_allobs_df$SubsectionName,
#                                   sep="#")

# dsp_obs( Headline.contains="Music:"
#         #,NewsDesk=""
#         #,SectionName=""  
#         #,SubsectionName="Fashion & Style"
#         #,Popular=1 #NA
#         ,cols= c("UniqueID", "Headline", "Popular", "myCategory", 
#                 "NewsDesk", "SectionName", "SubsectionName"),
#         all=TRUE)
# dsp_obs( Headline.contains="."
#         ,NewsDesk=""
#         ,SectionName="Opinion"  
#         ,SubsectionName=""
#         #,Popular=1 #NA
#         ,cols= c("UniqueID", "Headline", "Popular", "myCategory", 
#                 "NewsDesk", "SectionName", "SubsectionName"),
#         all=TRUE)
                                        
# Merge some categories
# glb_allobs_df$myCategory <-
#     plyr::revalue(glb_allobs_df$myCategory, c(      
#         "#Business Day#Dealbook"            = "Business#Business Day#Dealbook",
#         "#Business Day#Small Business"      = "Business#Business Day#Small Business",
#         "#Crosswords/Games#"                = "Business#Crosswords/Games#",
#         "Business##"                        = "Business#Technology#",
#         "#Open#"                            = "Business#Technology#",
#         "#Technology#"                      = "Business#Technology#",
#         
#         "#Arts#"                            = "Culture#Arts#",        
#         "Culture##"                         = "Culture#Arts#",        
#         
#         "#World#Asia Pacific"               = "Foreign#World#Asia Pacific",        
#         "Foreign##"                         = "Foreign#World#",    
#         
#         "#N.Y. / Region#"                   = "Metro#N.Y. / Region#",  
#         
#         "#Opinion#"                         = "OpEd#Opinion#",                
#         "OpEd##"                            = "OpEd#Opinion#",        
# 
#         "#Health#"                          = "Science#Health#",
#         "Science##"                         = "Science#Health#",        
#         
#         "Styles##"                          = "Styles##Fashion",                        
#         "Styles#Health#"                    = "Science#Health#",                
#         "Styles#Style#Fashion & Style"      = "Styles##Fashion",        
# 
#         "#Travel#"                          = "Travel#Travel#",                
#         
#         "Magazine#Magazine#"                = "myOther",
#         "National##"                        = "myOther",
#         "National#U.S.#Politics"            = "myOther",        
#         "Sports##"                          = "myOther",
#         "Sports#Sports#"                    = "myOther",
#         "#U.S.#"                            = "myOther",        
#         
# 
# #         "Business##Small Business"        = "Business#Business Day#Small Business",        
# #         
# #         "#Opinion#"                       = "#Opinion#Room For Debate",        
#         "##"                                = "##"
# #         "Business##" = "Business#Business Day#Dealbook",
# #         "Foreign#World#" = "Foreign##",
# #         "#Open#" = "Other",
# #         "#Opinion#The Public Editor" = "OpEd#Opinion#",
# #         "Styles#Health#" = "Styles##",
# #         "Styles#Style#Fashion & Style" = "Styles##",
# #         "#U.S.#" = "#U.S.#Education",
#     ))

# ctgry_xtab_df <- orderBy(reformulate(c("-", ".n")),
#                           mycreate_sqlxtab_df(glb_allobs_df,
#     c("myCategory", "NewsDesk", "SectionName", "SubsectionName", glb_rsp_var)))
# myprint_df(ctgry_xtab_df)
# write.table(ctgry_xtab_df, paste0(glb_out_pfx, "ctgry_xtab.csv"), 
#             row.names=FALSE)

# ctgry_cast_df <- orderBy(~ -Y -NA, dcast(ctgry_xtab_df, 
#                        myCategory + NewsDesk + SectionName + SubsectionName ~ 
#                            Popular.fctr, sum, value.var=".n"))
# myprint_df(ctgry_cast_df)
# write.table(ctgry_cast_df, paste0(glb_out_pfx, "ctgry_cast.csv"), 
#             row.names=FALSE)

# print(ctgry_sum_tbl <- table(glb_allobs_df$myCategory, glb_allobs_df[, glb_rsp_var], 
#                              useNA="ifany"))

dsp_chisq.test <- function(...) {
    sel_df <- glb_allobs_df[sel_obs(...) & 
                            !is.na(glb_allobs_df$Popular), ]
    sel_df$.marker <- 1
    ref_df <- glb_allobs_df[!is.na(glb_allobs_df$Popular), ]
    mrg_df <- merge(ref_df[, c(glb_id_var, "Popular")],
                    sel_df[, c(glb_id_var, ".marker")], all.x=TRUE)
    mrg_df[is.na(mrg_df)] <- 0
    print(mrg_tbl <- table(mrg_df$.marker, mrg_df$Popular))
    print("Rows:Selected; Cols:Popular")
    #print(mrg_tbl)
    print(chisq.test(mrg_tbl))
}
# dsp_chisq.test(Headline.contains="[Ee]bola")
# dsp_chisq.test(Snippet.contains="[Ee]bola")
# dsp_chisq.test(Abstract.contains="[Ee]bola")

# print(mycreate_sqlxtab_df(glb_allobs_df[sel_obs(Headline.contains="[Ee]bola"), ], 
#                           c(glb_rsp_var, "NewsDesk", "SectionName", "SubsectionName")))

# print(table(glb_allobs_df$NewsDesk, glb_allobs_df$SectionName))
# print(table(glb_allobs_df$SectionName, glb_allobs_df$SubsectionName))
# print(table(glb_allobs_df$NewsDesk, glb_allobs_df$SectionName, glb_allobs_df$SubsectionName))

# glb_allobs_df$myCategory.fctr <- as.factor(glb_allobs_df$myCategory)
# glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, 
#                                       c("myCategory", "NewsDesk", "SectionName", "SubsectionName"))

# Copy Headline into Snipper & Abstract if they are empty
# print(glb_allobs_df[nchar(glb_allobs_df[, "Snippet"]) == 0, c("Headline", "Snippet")])
# print(glb_allobs_df[glb_allobs_df$Headline == glb_allobs_df$Snippet, 
#                     c("UniqueID", "Headline", "Snippet")])
# glb_allobs_df[nchar(glb_allobs_df[, "Snippet"]) == 0, "Snippet"] <- 
#     glb_allobs_df[nchar(glb_allobs_df[, "Snippet"]) == 0, "Headline"]
# 
# print(glb_allobs_df[nchar(glb_allobs_df[, "Abstract"]) == 0, c("Headline", "Abstract")])
# print(glb_allobs_df[glb_allobs_df$Headline == glb_allobs_df$Abstract, 
#                     c("UniqueID", "Headline", "Abstract")])
# glb_allobs_df[nchar(glb_allobs_df[, "Abstract"]) == 0, "Abstract"] <- 
#     glb_allobs_df[nchar(glb_allobs_df[, "Abstract"]) == 0, "Headline"]

# WordCount_0_df <- subset(glb_allobs_df, WordCount == 0)
# table(WordCount_0_df$Popular, WordCount_0_df$WordCount, useNA="ifany")
# myprint_df(WordCount_0_df[, 
#                 c("UniqueID", "Popular", "WordCount", "Headline")])
```

### Step `2.1: scrub data`

```r
glb_chunks_df <- myadd_chunk(glb_chunks_df, "transform.data", major.inc=FALSE)
```

```
##            label step_major step_minor    bgn    end elapsed
## 3     scrub.data          2          1 16.557 18.741   2.184
## 4 transform.data          2          2 18.741     NA      NA
```

```r
### Mapping dictionary
#sav_allobs_df <- glb_allobs_df; glb_allobs_df <- sav_allobs_df
if (!is.null(glb_map_vars)) {
    for (feat in glb_map_vars) {
        map_df <- myimport_data(url=glb_map_urls[[feat]], 
                                            comment="map_df", 
                                           print_diagn=TRUE)
        glb_allobs_df <- mymap_codes(glb_allobs_df, feat, names(map_df)[2], 
                                     map_df, map_join_col_name=names(map_df)[1], 
                                     map_tgt_col_name=names(map_df)[2])
    }
    glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, glb_map_vars)
}

### Forced Assignments
#stop(here"); sav_allobs_df <- glb_allobs_df; glb_allobs_df <- sav_allobs_df
for (feat in glb_assign_vars) {
    new_feat <- paste0(feat, ".my")
    print(sprintf("Forced Assignments for: %s -> %s...", feat, new_feat))
    glb_allobs_df[, new_feat] <- glb_allobs_df[, feat]
    
    pairs <- glb_assign_pairs_lst[[feat]]
    for (pair_ix in 1:length(pairs$from)) {
        if (is.na(pairs$from[pair_ix]))
            nobs <- nrow(filter(glb_allobs_df, 
                                is.na(eval(parse(text=feat),
                                            envir=glb_allobs_df)))) else
            nobs <- sum(glb_allobs_df[, feat] == pairs$from[pair_ix])
        #nobs <- nrow(filter(glb_allobs_df, is.na(Married.fctr)))    ; print(nobs)
        
        if ((is.na(pairs$from[pair_ix])) && (is.na(pairs$to[pair_ix])))
            stop("what are you trying to do ???")
        if (is.na(pairs$from[pair_ix]))
            glb_allobs_df[is.na(glb_allobs_df[, feat]), new_feat] <- 
                pairs$to[pair_ix] else
            glb_allobs_df[glb_allobs_df[, feat] == pairs$from[pair_ix], new_feat] <- 
                pairs$to[pair_ix]
                    
        print(sprintf("    %s -> %s for %s obs", 
                      pairs$from[pair_ix], pairs$to[pair_ix], format(nobs, big.mark=",")))
    }

    glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, glb_assign_vars)
}

### Transformations using mapping functions
#stop(here"); sav_allobs_df <- glb_allobs_df; glb_allobs_df <- sav_allobs_df
for (feat in glb_transform_vars) {
    new_feat <- paste0(feat, glb_transform_lst[[feat]]$sfx)
    print(sprintf("Applying mapping function for: %s -> %s...", feat, new_feat))
    glb_allobs_df[, new_feat] <- glb_transform_lst[[feat]]$mapfn(glb_allobs_df[, feat])

    glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, glb_transform_vars)
}

### Derivations using mapping functions
#stop(here"); sav_allobs_df <- glb_allobs_df; glb_allobs_df <- sav_allobs_df
for (new_feat in glb_derive_vars) {
    print(sprintf("Creating new feature: %s...", new_feat))
    args_lst <- NULL 
    for (arg in glb_derive_lst[[new_feat]]$args) 
        args_lst[[arg]] <- glb_allobs_df[, arg]
    glb_allobs_df[, new_feat] <- do.call(glb_derive_lst[[new_feat]]$mapfn, args_lst)
}
```

```
## [1] "Creating new feature: PTS.diff..."
```

### Step `2.2: transform data`

```r
glb_chunks_df <- myadd_chunk(glb_chunks_df, "manage.missing.data", major.inc=FALSE)
```

```
##                 label step_major step_minor    bgn    end elapsed
## 4      transform.data          2          2 18.741 18.778   0.037
## 5 manage.missing.data          2          3 18.778     NA      NA
```

```r
# print(sapply(names(glb_trnobs_df), function(col) sum(is.na(glb_trnobs_df[, col]))))
# print(sapply(names(glb_newobs_df), function(col) sum(is.na(glb_newobs_df[, col]))))
# glb_trnobs_df <- na.omit(glb_trnobs_df)
# glb_newobs_df <- na.omit(glb_newobs_df)
# df[is.na(df)] <- 0

mycheck_problem_data(glb_allobs_df)
```

```
## [1] "numeric data missing in glb_allobs_df: "
## named integer(0)
## [1] "numeric data w/ 0s in glb_allobs_df: "
## Playoffs 
##      369 
## [1] "numeric data w/ Infs in glb_allobs_df: "
## named integer(0)
## [1] "numeric data w/ NaNs in glb_allobs_df: "
## named integer(0)
## [1] "string data missing in glb_allobs_df: "
##      Team .rownames 
##         0         0
```

```r
# Not refactored into mydsutils.R since glb_*_df might be reassigned
glb_impute_missing_data <- function() {
    
    require(mice)
    set.seed(glb_mice_complete.seed)
    inp_impent_df <- glb_allobs_df[, setdiff(names(glb_allobs_df), 
                                union(glb_exclude_vars_as_features, glb_rsp_var))]
    print("Summary before imputation: ")
    print(summary(inp_impent_df))
    out_impent_df <- complete(mice(inp_impent_df))
    print(summary(out_impent_df))
    
    # complete(mice()) changes attributes of factors even though values don't change
    ret_vars <- sapply(names(out_impent_df), 
                       function(col) ifelse(!identical(out_impent_df[, col], inp_impent_df[, col]), 
                                            col, ""))
    ret_vars <- ret_vars[ret_vars != ""]
    return(out_impent_df[, ret_vars])
}

if (glb_impute_na_data && 
    (length(myfind_numerics_missing(glb_allobs_df)) > 0) &&
    (ncol(nonna_df <- glb_impute_missing_data()) > 0)) {
    for (col in names(nonna_df)) {
        glb_allobs_df[, paste0(col, ".nonNA")] <- nonna_df[, col]
        glb_exclude_vars_as_features <- c(glb_exclude_vars_as_features, col)        
    }
}    
    
mycheck_problem_data(glb_allobs_df, terminate = TRUE)
```

```
## [1] "numeric data missing in glb_allobs_df: "
## named integer(0)
## [1] "numeric data w/ 0s in glb_allobs_df: "
## Playoffs 
##      369 
## [1] "numeric data w/ Infs in glb_allobs_df: "
## named integer(0)
## [1] "numeric data w/ NaNs in glb_allobs_df: "
## named integer(0)
## [1] "string data missing in glb_allobs_df: "
##      Team .rownames 
##         0         0
```

## Step `2.3: manage missing data`

```r
#```{r extract_features, cache=FALSE, eval=!is.null(glb_txt_vars)}
glb_chunks_df <- myadd_chunk(glb_chunks_df, "extract.features", major.inc=TRUE)
```

```
##                 label step_major step_minor    bgn    end elapsed
## 5 manage.missing.data          2          3 18.778 18.852   0.074
## 6    extract.features          3          0 18.852     NA      NA
```

```r
extract.features_chunk_df <- myadd_chunk(NULL, "extract.features_bgn")
```

```
##                  label step_major step_minor    bgn end elapsed
## 1 extract.features_bgn          1          0 18.858  NA      NA
```

```r
# Options:
#   Select Tf, log(1 + Tf), Tf-IDF or BM25Tf-IDf

# Create new features that help prediction
# <col_name>.lag.2 <- lag(zoo(glb_trnobs_df$<col_name>), -2, na.pad=TRUE)
# glb_trnobs_df[, "<col_name>.lag.2"] <- coredata(<col_name>.lag.2)
# <col_name>.lag.2 <- lag(zoo(glb_newobs_df$<col_name>), -2, na.pad=TRUE)
# glb_newobs_df[, "<col_name>.lag.2"] <- coredata(<col_name>.lag.2)
# 
# glb_newobs_df[1, "<col_name>.lag.2"] <- glb_trnobs_df[nrow(glb_trnobs_df) - 1, 
#                                                    "<col_name>"]
# glb_newobs_df[2, "<col_name>.lag.2"] <- glb_trnobs_df[nrow(glb_trnobs_df), 
#                                                    "<col_name>"]
                                                   
# glb_allobs_df <- mutate(glb_allobs_df,
#     A.P.http=ifelse(grepl("http",Added,fixed=TRUE), 1, 0)
#                     )
# 
# glb_trnobs_df <- mutate(glb_trnobs_df,
#                     )
# 
# glb_newobs_df <- mutate(glb_newobs_df,
#                     )

#   Create factors of string variables
extract.features_chunk_df <- myadd_chunk(extract.features_chunk_df, 
            paste0("extract.features_", "factorize.str.vars"), major.inc=TRUE)
```

```
##                                 label step_major step_minor    bgn    end
## 1                extract.features_bgn          1          0 18.858 18.866
## 2 extract.features_factorize.str.vars          2          0 18.866     NA
##   elapsed
## 1   0.008
## 2      NA
```

```r
#stop(here"); sav_allobs_df <- glb_allobs_df; #glb_allobs_df <- sav_allobs_df
print(str_vars <- myfind_chr_cols_df(glb_allobs_df))
```

```
##        Team   .rownames        .src 
##      "Team" ".rownames"      ".src"
```

```r
if (length(str_vars <- setdiff(str_vars, 
                               glb_exclude_vars_as_features)) > 0) {
    for (var in str_vars) {
        warning("Creating factors of string variable: ", var, 
                ": # of unique values: ", length(unique(glb_allobs_df[, var])))
        glb_allobs_df[, paste0(var, ".fctr")] <- factor(glb_allobs_df[, var], 
                        as.factor(unique(glb_allobs_df[, var])))
#         glb_trnobs_df[, paste0(var, ".fctr")] <- factor(glb_trnobs_df[, var], 
#                         as.factor(unique(glb_allobs_df[, var])))
#         glb_newobs_df[, paste0(var, ".fctr")] <- factor(glb_newobs_df[, var], 
#                         as.factor(unique(glb_allobs_df[, var])))
    }
    glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, str_vars)
}
```

```
## Warning: Creating factors of string variable: Team: # of unique values: 38
```

```r
if (!is.null(glb_txt_vars)) {
    require(foreach)
    require(gsubfn)
    require(stringr)
    require(tm)
    
    extract.features_chunk_df <- myadd_chunk(extract.features_chunk_df, 
            paste0("extract.features_", "process.text"), major.inc=TRUE)
    
    chk_pattern_freq <- function(re_str, ignore.case=TRUE) {
        match_mtrx <- str_extract_all(txt_vctr, regex(re_str, ignore_case=ignore.case), 
                                      simplify=TRUE)
        match_df <- as.data.frame(match_mtrx[match_mtrx != ""])
        names(match_df) <- "pattern"
        return(mycreate_sqlxtab_df(match_df, "pattern"))        
    }
    #tmp_freq_df <- chk_pattern_freq("\\bNew (\\w)+", ignore.case=FALSE)
    #subset(chk_pattern_freq("\\bNew (\\w)+", ignore.case=FALSE), grepl("New [[:upper:]]", pattern))
    #chk_pattern_freq("\\bnew (\\W)+")

    chk_subfn <- function(pos_ix) {
        re_str <- gsubfn_args_lst[["re_str"]][[pos_ix]]
        print("re_str:"); print(re_str)
        rp_frmla <- gsubfn_args_lst[["rp_frmla"]][[pos_ix]]        
        print("rp_frmla:"); print(rp_frmla, showEnv=FALSE)
        tmp_vctr <- grep(re_str, txt_vctr, value=TRUE, ignore.case=TRUE)[1:5]
        print("Before:")
        print(tmp_vctr)
        print("After:")            
        print(gsubfn(re_str, rp_frmla, tmp_vctr, ignore.case=TRUE))
    }
    #chk_subfn(1)

    myapply_gsub <- function(...) {
        if ((length_lst <- length(names(gsub_map_lst))) == 0)
            return(txt_vctr)
        for (ptn_ix in 1:length_lst) {
            print(sprintf("running gsub for %02d (of %02d): #%s#...", ptn_ix, 
                            length(names(gsub_map_lst)), names(gsub_map_lst)[ptn_ix]))
            txt_vctr <- gsub(names(gsub_map_lst)[ptn_ix], gsub_map_lst[[ptn_ix]], 
                               txt_vctr, ...)
        }
        return(txt_vctr)
    }    

    myapply_txtmap <- function(txt_vctr, ...) {
        nrows <- nrow(glb_txt_map_df)
        for (ptn_ix in 1:nrows) {
            print(sprintf("running gsub for %02d (of %02d): #%s#...", ptn_ix, 
                            nrows, glb_txt_map_df[ptn_ix, "rex_str"]))
            txt_vctr <- gsub(glb_txt_map_df[ptn_ix, "rex_str"], 
                             glb_txt_map_df[ptn_ix, "rpl_str"], 
                               txt_vctr, ...)
        }
        return(txt_vctr)
    }    

    chk.equal <- function(bgn, end) {
        print(all.equal(sav_txt_lst[["Headline"]][bgn:end], glb_txt_lst[["Headline"]][bgn:end]))
    }    
    dsp.equal <- function(bgn, end) {
        print(sav_txt_lst[["Headline"]][bgn:end])
        print(glb_txt_lst[["Headline"]][bgn:end])
    }    
#sav_txt_lst <- glb_txt_lst; all.equal(sav_txt_lst, glb_txt_lst)
#all.equal(sav_txt_lst[["Headline"]][1:4200], glb_txt_lst[["Headline"]][1:4200])
#all.equal(sav_txt_lst[["Headline"]][1:2000], glb_txt_lst[["Headline"]][1:2000])
#all.equal(sav_txt_lst[["Headline"]][1:1000], glb_txt_lst[["Headline"]][1:1000])
#all.equal(sav_txt_lst[["Headline"]][1:500], glb_txt_lst[["Headline"]][1:500])
#all.equal(sav_txt_lst[["Headline"]][1:200], glb_txt_lst[["Headline"]][1:200])
#all.equal(sav_txt_lst[["Headline"]][1:100], glb_txt_lst[["Headline"]][1:100])
#chk.equal( 1, 100)
#chk.equal(51, 100)
#chk.equal(81, 100)
#chk.equal(81,  90)
#chk.equal(81,  85)
#chk.equal(86,  90)
#chk.equal(96, 100)

#dsp.equal(86, 90)
    
    glb_txt_map_df <- read.csv("mytxt_map.csv", comment.char="#", strip.white=TRUE)
    glb_txt_lst <- list(); 
    print(sprintf("Building glb_txt_lst..."))
    glb_txt_lst <- foreach(txt_var=glb_txt_vars) %dopar% {   
#     for (txt_var in glb_txt_vars) {
        txt_vctr <- glb_allobs_df[, txt_var]
        
        # myapply_txtmap shd be created as a tm_map::content_transformer ?
        #print(glb_txt_map_df)
        #txt_var=glb_txt_vars[3]; txt_vctr <- glb_txt_lst[[txt_var]]
        #print(rex_str <- glb_txt_map_df[glb_txt_map_df$rex_str == "\\bWall St\\.", "rex_str"])
        #print(rex_str <- glb_txt_map_df[grepl("du Pont", glb_txt_map_df$rex_str), "rex_str"])        
        #print(rex_str <- glb_txt_map_df[glb_txt_map_df$rpl_str == "versus", "rex_str"])             
        #print(tmp_vctr <- grep(rex_str, txt_vctr, value=TRUE, ignore.case=FALSE))
        #ret_lst <- regexec(rex_str, txt_vctr, ignore.case=FALSE); ret_lst <- regmatches(txt_vctr, ret_lst); ret_vctr <- sapply(1:length(ret_lst), function(pos_ix) ifelse(length(ret_lst[[pos_ix]]) > 0, ret_lst[[pos_ix]], "")); print(ret_vctr <- ret_vctr[ret_vctr != ""])
        #gsub(rex_str, glb_txt_map_df[glb_txt_map_df$rex_str == rex_str, "rpl_str"], tmp_vctr, ignore.case=FALSE)
        #grep("Hong Hong", txt_vctr, value=TRUE)
    
        txt_vctr <- myapply_txtmap(txt_vctr, ignore.case=FALSE)    
    }
    names(glb_txt_lst) <- glb_txt_vars

    for (txt_var in glb_txt_vars) {
        print(sprintf("Remaining Acronyms in %s:", txt_var))
        txt_vctr <- glb_txt_lst[[txt_var]]
        print(tmp_vctr <- grep("[[:upper:]]\\.", txt_vctr, value=TRUE, ignore.case=FALSE))
    }

    for (txt_var in glb_txt_vars) {
        re_str <- "\\b(Fort|Ft\\.|Hong|Las|Los|New|Puerto|Saint|San|St\\.)( |-)(\\w)+"
        print(sprintf("Remaining #%s# terms in %s: ", re_str, txt_var))
        txt_vctr <- glb_txt_lst[[txt_var]]        
        print(orderBy(~ -.n +pattern, subset(chk_pattern_freq(re_str, ignore.case=FALSE), 
                                             grepl("( |-)[[:upper:]]", pattern))))
        print("    consider cleaning if relevant to problem domain; geography name; .n > 1")
        #grep("New G", txt_vctr, value=TRUE, ignore.case=FALSE)
        #grep("St\\. Wins", txt_vctr, value=TRUE, ignore.case=FALSE)
    }        
        
    for (txt_var in glb_txt_vars) {
        re_str <- "\\b(N|S|E|W|C)( |\\.)(\\w)+"
        print(sprintf("Remaining #%s# terms in %s: ", re_str, txt_var))        
        txt_vctr <- glb_txt_lst[[txt_var]]                
        print(orderBy(~ -.n +pattern, subset(chk_pattern_freq(re_str, ignore.case=FALSE), 
                                             grepl(".", pattern))))
        #grep("N Weaver", txt_vctr, value=TRUE, ignore.case=FALSE)        
    }    

    for (txt_var in glb_txt_vars) {
        re_str <- "\\b(North|South|East|West|Central)( |\\.)(\\w)+"
        print(sprintf("Remaining #%s# terms in %s: ", re_str, txt_var))        
        txt_vctr <- glb_txt_lst[[txt_var]]                        
        print(orderBy(~ -.n +pattern, subset(chk_pattern_freq(re_str, ignore.case=FALSE), 
                                             grepl(".", pattern))))
        #grep("Central (African|Bankers|Cast|Italy|Role|Spring)", txt_vctr, value=TRUE, ignore.case=FALSE)
        #grep("East (Africa|Berlin|London|Poland|Rivals|Spring)", txt_vctr, value=TRUE, ignore.case=FALSE)
        #grep("North (American|Korean|West)", txt_vctr, value=TRUE, ignore.case=FALSE)        
        #grep("South (Pacific|Street)", txt_vctr, value=TRUE, ignore.case=FALSE)
        #grep("St\\. Martins", txt_vctr, value=TRUE, ignore.case=FALSE)
    }    

    find_cmpnd_wrds <- function(txt_vctr) {
        txt_corpus <- Corpus(VectorSource(txt_vctr))
        txt_corpus <- tm_map(txt_corpus, tolower)
        txt_corpus <- tm_map(txt_corpus, PlainTextDocument)
        txt_corpus <- tm_map(txt_corpus, removePunctuation, 
                             preserve_intra_word_dashes=TRUE)
        full_Tf_DTM <- DocumentTermMatrix(txt_corpus, 
                                          control=list(weighting=weightTf))
        print("   Full TermMatrix:"); print(full_Tf_DTM)
        full_Tf_mtrx <- as.matrix(full_Tf_DTM)
        rownames(full_Tf_mtrx) <- rownames(glb_allobs_df) # print undreadable otherwise
        full_Tf_vctr <- colSums(full_Tf_mtrx)
        names(full_Tf_vctr) <- dimnames(full_Tf_DTM)[[2]]
        #grep("year", names(full_Tf_vctr), value=TRUE)
        #which.max(full_Tf_mtrx[, "yearlong"])
        full_Tf_df <- as.data.frame(full_Tf_vctr)
        names(full_Tf_df) <- "Tf.full"
        full_Tf_df$term <- rownames(full_Tf_df)
        #full_Tf_df$freq.full <- colSums(full_Tf_mtrx != 0)
        full_Tf_df <- orderBy(~ -Tf.full, full_Tf_df)
        cmpnd_Tf_df <- full_Tf_df[grep("-", full_Tf_df$term, value=TRUE) ,]
        
        filter_df <- read.csv("mytxt_compound.csv", comment.char="#", strip.white=TRUE)
        cmpnd_Tf_df$filter <- FALSE
        for (row_ix in 1:nrow(filter_df))
            cmpnd_Tf_df[!cmpnd_Tf_df$filter, "filter"] <- 
            grepl(filter_df[row_ix, "rex_str"], 
                  cmpnd_Tf_df[!cmpnd_Tf_df$filter, "term"], ignore.case=TRUE)
        cmpnd_Tf_df <- subset(cmpnd_Tf_df, !filter)
        # Bug in tm_map(txt_corpus, removePunctuation, preserve_intra_word_dashes=TRUE) ???
        #   "net-a-porter" gets converted to "net-aporter"
        #grep("net-a-porter", txt_vctr, ignore.case=TRUE, value=TRUE)
        #grep("maser-laser", txt_vctr, ignore.case=TRUE, value=TRUE)
        #txt_corpus[[which(grepl("net-a-porter", txt_vctr, ignore.case=TRUE))]]
        #grep("\\b(across|longer)-(\\w)", cmpnd_Tf_df$term, ignore.case=TRUE, value=TRUE)
        #grep("(\\w)-(affected|term)\\b", cmpnd_Tf_df$term, ignore.case=TRUE, value=TRUE)
        
        print(sprintf("nrow(cmpnd_Tf_df): %d", nrow(cmpnd_Tf_df)))
        myprint_df(cmpnd_Tf_df)
    }

    extract.features_chunk_df <- myadd_chunk(extract.features_chunk_df, 
            paste0("extract.features_", "process.text_reporting_compound_terms"), major.inc=FALSE)
    
    for (txt_var in glb_txt_vars) {
        print(sprintf("Remaining compound terms in %s: ", txt_var))        
        txt_vctr <- glb_txt_lst[[txt_var]]                        
#         find_cmpnd_wrds(txt_vctr)
        #grep("thirty-five", txt_vctr, ignore.case=TRUE, value=TRUE)
        #rex_str <- glb_txt_map_df[grepl("hirty", glb_txt_map_df$rex_str), "rex_str"]
    }

    extract.features_chunk_df <- myadd_chunk(extract.features_chunk_df, 
            paste0("extract.features_", "build.corpus"), major.inc=TRUE)
    
    glb_corpus_lst <- list()
    print(sprintf("Building glb_corpus_lst..."))
    glb_corpus_lst <- foreach(txt_var=glb_txt_vars) %dopar% {   
#     for (txt_var in glb_txt_vars) {
        txt_corpus <- Corpus(VectorSource(glb_txt_lst[[txt_var]]))
        txt_corpus <- tm_map(txt_corpus, tolower) #nuppr
        txt_corpus <- tm_map(txt_corpus, PlainTextDocument)
        txt_corpus <- tm_map(txt_corpus, removePunctuation) #npnct<chr_ix>
#         txt-corpus <- tm_map(txt_corpus, content_transformer(function(x, pattern) gsub(pattern, "", x))   

        # Not to be run in production
        inspect_terms <- function() {
            full_Tf_DTM <- DocumentTermMatrix(txt_corpus, 
                                              control=list(weighting=weightTf))
            print("   Full TermMatrix:"); print(full_Tf_DTM)
            full_Tf_mtrx <- as.matrix(full_Tf_DTM)
            rownames(full_Tf_mtrx) <- rownames(glb_allobs_df) # print undreadable otherwise
            full_Tf_vctr <- colSums(full_Tf_mtrx)
            names(full_Tf_vctr) <- dimnames(full_Tf_DTM)[[2]]
            #grep("year", names(full_Tf_vctr), value=TRUE)
            #which.max(full_Tf_mtrx[, "yearlong"])
            full_Tf_df <- as.data.frame(full_Tf_vctr)
            names(full_Tf_df) <- "Tf.full"
            full_Tf_df$term <- rownames(full_Tf_df)
            #full_Tf_df$freq.full <- colSums(full_Tf_mtrx != 0)
            full_Tf_df <- orderBy(~ -Tf.full +term, full_Tf_df)
            print(myplot_histogram(full_Tf_df, "Tf.full"))
            myprint_df(full_Tf_df)
            #txt_corpus[[which(grepl("zun", txt_vctr, ignore.case=TRUE))]]
            digit_terms_df <- subset(full_Tf_df, grepl("[[:digit:]]", term))
            myprint_df(digit_terms_df)
            return(full_Tf_df)
        }    
        #print("RemovePunct:"); remove_punct_Tf_df <- inspect_terms()

        txt_corpus <- tm_map(txt_corpus, removeWords, 
                             c(glb_append_stop_words[[txt_var]], 
                               stopwords("english"))) #nstopwrds
        #print("StoppedWords:"); stopped_words_Tf_df <- inspect_terms()
        txt_corpus <- tm_map(txt_corpus, stemDocument) #Features for lost information: Difference/ratio in density of full_TfIdf_DTM ???
        #txt_corpus <- tm_map(txt_corpus, content_transformer(stemDocument))        
        #print("StemmedWords:"); stemmed_words_Tf_df <- inspect_terms()
        #stemmed_stopped_Tf_df <- merge(stemmed_words_Tf_df, stopped_words_Tf_df, by="term", all=TRUE, suffixes=c(".stem", ".stop"))
        #myprint_df(stemmed_stopped_Tf_df)
        #print(subset(stemmed_stopped_Tf_df, grepl("compan", term)))
        #glb_corpus_lst[[txt_var]] <- txt_corpus
    }
    names(glb_corpus_lst) <- glb_txt_vars
        
    extract.features_chunk_df <- myadd_chunk(extract.features_chunk_df, 
            paste0("extract.features_", "extract.DTM"), major.inc=TRUE)

    glb_full_DTM_lst <- list(); glb_sprs_DTM_lst <- list();
    for (txt_var in glb_txt_vars) {
        print(sprintf("Extracting TfIDf terms for %s...", txt_var))        
        txt_corpus <- glb_corpus_lst[[txt_var]]
        
#         full_Tf_DTM <- DocumentTermMatrix(txt_corpus, 
#                                           control=list(weighting=weightTf))
        full_TfIdf_DTM <- DocumentTermMatrix(txt_corpus, 
                                          control=list(weighting=weightTfIdf))
        sprs_TfIdf_DTM <- removeSparseTerms(full_TfIdf_DTM, 
                                            glb_sprs_thresholds[txt_var])
        
#         glb_full_DTM_lst[[txt_var]] <- full_Tf_DTM
#         glb_sprs_DTM_lst[[txt_var]] <- sprs_Tf_DTM
        glb_full_DTM_lst[[txt_var]] <- full_TfIdf_DTM
        glb_sprs_DTM_lst[[txt_var]] <- sprs_TfIdf_DTM
    }

    extract.features_chunk_df <- myadd_chunk(extract.features_chunk_df, 
            paste0("extract.features_", "report.DTM"), major.inc=TRUE)
    
    for (txt_var in glb_txt_vars) {
        print(sprintf("Reporting TfIDf terms for %s...", txt_var))        
        full_TfIdf_DTM <- glb_full_DTM_lst[[txt_var]]
        sprs_TfIdf_DTM <- glb_sprs_DTM_lst[[txt_var]]        

        print("   Full TermMatrix:"); print(full_TfIdf_DTM)
        full_TfIdf_mtrx <- as.matrix(full_TfIdf_DTM)
        rownames(full_TfIdf_mtrx) <- rownames(glb_allobs_df) # print undreadable otherwise
        full_TfIdf_vctr <- colSums(full_TfIdf_mtrx)
        names(full_TfIdf_vctr) <- dimnames(full_TfIdf_DTM)[[2]]
        #grep("scene", names(full_TfIdf_vctr), value=TRUE)
        #which.max(full_TfIdf_mtrx[, "yearlong"])
        full_TfIdf_df <- as.data.frame(full_TfIdf_vctr)
        names(full_TfIdf_df) <- "TfIdf.full"
        full_TfIdf_df$term <- rownames(full_TfIdf_df)
        full_TfIdf_df$freq.full <- colSums(full_TfIdf_mtrx != 0)
        full_TfIdf_df <- orderBy(~ -TfIdf.full, full_TfIdf_df)

        print("   Sparse TermMatrix:"); print(sprs_TfIdf_DTM)
        sprs_TfIdf_vctr <- colSums(as.matrix(sprs_TfIdf_DTM))
        names(sprs_TfIdf_vctr) <- dimnames(sprs_TfIdf_DTM)[[2]]
        sprs_TfIdf_df <- as.data.frame(sprs_TfIdf_vctr)
        names(sprs_TfIdf_df) <- "TfIdf.sprs"
        sprs_TfIdf_df$term <- rownames(sprs_TfIdf_df)
        sprs_TfIdf_df$freq.sprs <- colSums(as.matrix(sprs_TfIdf_DTM) != 0)        
        sprs_TfIdf_df <- orderBy(~ -TfIdf.sprs, sprs_TfIdf_df)
        
        terms_TfIdf_df <- merge(full_TfIdf_df, sprs_TfIdf_df, all.x=TRUE)
        terms_TfIdf_df$in.sprs <- !is.na(terms_TfIdf_df$freq.sprs)
        plt_TfIdf_df <- subset(terms_TfIdf_df, 
                               TfIdf.full >= min(terms_TfIdf_df$TfIdf.sprs, na.rm=TRUE))
        plt_TfIdf_df$label <- ""
        plt_TfIdf_df[is.na(plt_TfIdf_df$TfIdf.sprs), "label"] <- 
            plt_TfIdf_df[is.na(plt_TfIdf_df$TfIdf.sprs), "term"]
        glb_important_terms[[txt_var]] <- union(glb_important_terms[[txt_var]],
            plt_TfIdf_df[is.na(plt_TfIdf_df$TfIdf.sprs), "term"])
        print(myplot_scatter(plt_TfIdf_df, "freq.full", "TfIdf.full", 
                             colorcol_name="in.sprs") + 
                  geom_text(aes(label=label), color="Black", size=3.5))
        
        melt_TfIdf_df <- orderBy(~ -value, melt(terms_TfIdf_df, id.var="term"))
        print(ggplot(melt_TfIdf_df, aes(value, color=variable)) + stat_ecdf() + 
                  geom_hline(yintercept=glb_sprs_thresholds[txt_var], 
                             linetype = "dotted"))
        
        melt_TfIdf_df <- orderBy(~ -value, 
                        melt(subset(terms_TfIdf_df, !is.na(TfIdf.sprs)), id.var="term"))
        print(myplot_hbar(melt_TfIdf_df, "term", "value", 
                          colorcol_name="variable"))
        
        melt_TfIdf_df <- orderBy(~ -value, 
                        melt(subset(terms_TfIdf_df, is.na(TfIdf.sprs)), id.var="term"))
        print(myplot_hbar(head(melt_TfIdf_df, 10), "term", "value", 
                          colorcol_name="variable"))
    }

#     sav_full_DTM_lst <- glb_full_DTM_lst
#     sav_sprs_DTM_lst <- glb_sprs_DTM_lst
#     print(identical(sav_glb_corpus_lst, glb_corpus_lst))
#     print(all.equal(length(sav_glb_corpus_lst), length(glb_corpus_lst)))
#     print(all.equal(names(sav_glb_corpus_lst), names(glb_corpus_lst)))
#     print(all.equal(sav_glb_corpus_lst[["Headline"]], glb_corpus_lst[["Headline"]]))

#     print(identical(sav_full_DTM_lst, glb_full_DTM_lst))
#     print(identical(sav_sprs_DTM_lst, glb_sprs_DTM_lst))
        
    rm(full_TfIdf_mtrx, full_TfIdf_df, melt_TfIdf_df, terms_TfIdf_df)

    # Create txt features
    if ((length(glb_txt_vars) > 1) &&
        (length(unique(pfxs <- sapply(glb_txt_vars, 
                    function(txt) toupper(substr(txt, 1, 1))))) < length(glb_txt_vars)))
            stop("Prefixes for corpus freq terms not unique: ", pfxs)
    
    extract.features_chunk_df <- myadd_chunk(extract.features_chunk_df, 
                            paste0("extract.features_", "bind.DTM"), 
                                         major.inc=TRUE)
    for (txt_var in glb_txt_vars) {
        print(sprintf("Binding DTM for %s...", txt_var))
        txt_var_pfx <- toupper(substr(txt_var, 1, 1))
        txt_X_df <- as.data.frame(as.matrix(glb_sprs_DTM_lst[[txt_var]]))
        colnames(txt_X_df) <- paste(txt_var_pfx, ".T.",
                                    make.names(colnames(txt_X_df)), sep="")
        rownames(txt_X_df) <- rownames(glb_allobs_df) # warning otherwise
#         plt_X_df <- cbind(txt_X_df, glb_allobs_df[, c(glb_id_var, glb_rsp_var)])
#         print(myplot_box(df=plt_X_df, ycol_names="H.T.today", xcol_name=glb_rsp_var))

#         log_X_df <- log(1 + txt_X_df)
#         colnames(log_X_df) <- paste(colnames(txt_X_df), ".log", sep="")
#         plt_X_df <- cbind(log_X_df, glb_allobs_df[, c(glb_id_var, glb_rsp_var)])
#         print(myplot_box(df=plt_X_df, ycol_names="H.T.today.log", xcol_name=glb_rsp_var))
        glb_allobs_df <- cbind(glb_allobs_df, txt_X_df) # TfIdf is normalized
        #glb_allobs_df <- cbind(glb_allobs_df, log_X_df) # if using non-normalized metrics 
    }
    #identical(chk_entity_df, glb_allobs_df)
    #chk_entity_df <- glb_allobs_df

    extract.features_chunk_df <- myadd_chunk(extract.features_chunk_df, 
                            paste0("extract.features_", "bind.DXM"), 
                                         major.inc=TRUE)

#sav_allobs_df <- glb_allobs_df
    glb_punct_vctr <- c("!", "\"", "#", "\\$", "%", "&", "'", 
                        "\\(|\\)",# "\\(", "\\)", 
                        "\\*", "\\+", ",", "-", "\\.", "/", ":", ";", 
                        "<|>", # "<", 
                        "=", 
                        # ">", 
                        "\\?", "@", "\\[", "\\\\", "\\]", "^", "_", "`", 
                        "\\{", "\\|", "\\}", "~")
    txt_X_df <- glb_allobs_df[, c(glb_id_var, ".rnorm"), FALSE]
    txt_X_df <- foreach(txt_var=glb_txt_vars, .combine=cbind) %dopar% {   
    #for (txt_var in glb_txt_vars) {
        print(sprintf("Binding DXM for %s...", txt_var))
        txt_var_pfx <- toupper(substr(txt_var, 1, 1))        
        #txt_X_df <- glb_allobs_df[, c(glb_id_var, ".rnorm"), FALSE]
        
        txt_full_DTM_mtrx <- as.matrix(glb_full_DTM_lst[[txt_var]])
        rownames(txt_full_DTM_mtrx) <- rownames(glb_allobs_df) # print undreadable otherwise
        #print(txt_full_DTM_mtrx[txt_full_DTM_mtrx[, "ebola"] != 0, "ebola"])
        
        # Create <txt_var>.T.<term> for glb_important_terms
        for (term in glb_important_terms[[txt_var]])
            txt_X_df[, paste0(txt_var_pfx, ".T.", make.names(term))] <- 
                txt_full_DTM_mtrx[, term]
                
        # Create <txt_var>.nwrds.log & .nwrds.unq.log
        txt_X_df[, paste0(txt_var_pfx, ".nwrds.log")] <- 
            log(1 + mycount_pattern_occ("\\w+", glb_txt_lst[[txt_var]]))
        txt_X_df[, paste0(txt_var_pfx, ".nwrds.unq.log")] <- 
            log(1 + rowSums(txt_full_DTM_mtrx != 0))
        txt_X_df[, paste0(txt_var_pfx, ".sum.TfIdf")] <- 
            rowSums(txt_full_DTM_mtrx) 
        txt_X_df[, paste0(txt_var_pfx, ".ratio.sum.TfIdf.nwrds")] <- 
            txt_X_df[, paste0(txt_var_pfx, ".sum.TfIdf")] / 
            (exp(txt_X_df[, paste0(txt_var_pfx, ".nwrds.log")]) - 1)
        txt_X_df[is.nan(txt_X_df[, paste0(txt_var_pfx, ".ratio.sum.TfIdf.nwrds")]),
                 paste0(txt_var_pfx, ".ratio.sum.TfIdf.nwrds")] <- 0

        # Create <txt_var>.nchrs.log
        txt_X_df[, paste0(txt_var_pfx, ".nchrs.log")] <- 
            log(1 + mycount_pattern_occ(".", glb_allobs_df[, txt_var]))
        txt_X_df[, paste0(txt_var_pfx, ".nuppr.log")] <- 
            log(1 + mycount_pattern_occ("[[:upper:]]", glb_allobs_df[, txt_var]))
        txt_X_df[, paste0(txt_var_pfx, ".ndgts.log")] <- 
            log(1 + mycount_pattern_occ("[[:digit:]]", glb_allobs_df[, txt_var]))

        # Create <txt_var>.npnct?.log
        # would this be faster if it's iterated over each row instead of 
        #   each created column ???
        for (punct_ix in 1:length(glb_punct_vctr)) { 
#             smp0 <- " "
#             smp1 <- "! \" # $ % & ' ( ) * + , - . / : ; < = > ? @ [ \ ] ^ _ ` { | } ~"
#             smp2 <- paste(smp1, smp1, sep=" ")
#             print(sprintf("Testing %s pattern:", glb_punct_vctr[punct_ix])) 
#             results <- mycount_pattern_occ(glb_punct_vctr[punct_ix], c(smp0, smp1, smp2))
#             names(results) <- NULL; print(results)
            txt_X_df[, 
                paste0(txt_var_pfx, ".npnct", sprintf("%02d", punct_ix), ".log")] <-
                log(1 + mycount_pattern_occ(glb_punct_vctr[punct_ix], 
                                            glb_allobs_df[, txt_var]))
        }
#         print(head(glb_allobs_df[glb_allobs_df[, "A.npnct23.log"] > 0, 
#                                     c("UniqueID", "Popular", "Abstract", "A.npnct23.log")]))    
        
        # Create <txt_var>.nstopwrds.log & <txt_var>ratio.nstopwrds.nwrds
        stop_words_rex_str <- paste0("\\b(", paste0(c(glb_append_stop_words[[txt_var]], 
                                       stopwords("english")), collapse="|"),
                                     ")\\b")
        txt_X_df[, paste0(txt_var_pfx, ".nstopwrds", ".log")] <-
            log(1 + mycount_pattern_occ(stop_words_rex_str, glb_txt_lst[[txt_var]]))
        txt_X_df[, paste0(txt_var_pfx, ".ratio.nstopwrds.nwrds")] <-
            exp(txt_X_df[, paste0(txt_var_pfx, ".nstopwrds", ".log")] - 
                txt_X_df[, paste0(txt_var_pfx, ".nwrds", ".log")])

        # Create <txt_var>.P.http
        txt_X_df[, paste(txt_var_pfx, ".P.http", sep="")] <- 
            as.integer(0 + mycount_pattern_occ("http", glb_allobs_df[, txt_var]))    
    
        # Create user-specified pattern vectors 
        #   <txt_var>.P.year.colon
        txt_X_df[, paste0(txt_var_pfx, ".P.year.colon")] <-
            as.integer(0 + mycount_pattern_occ("[0-9]{4}:", glb_allobs_df[, txt_var]))
        txt_X_df[, paste0(txt_var_pfx, ".P.daily.clip.report")] <-
            as.integer(0 + mycount_pattern_occ("Daily Clip Report", glb_allobs_df[, txt_var]))
        txt_X_df[, paste0(txt_var_pfx, ".P.fashion.week")] <-
            as.integer(0 + mycount_pattern_occ("Fashion Week", glb_allobs_df[, txt_var]))
        txt_X_df[, paste0(txt_var_pfx, ".P.first.draft")] <-
            as.integer(0 + mycount_pattern_occ("First Draft", glb_allobs_df[, txt_var]))

#sum(mycount_pattern_occ("Metropolitan Diary:", glb_allobs_df$Abstract) > 0)
        if (txt_var %in% c("Snippet", "Abstract")) {
            txt_X_df[, paste0(txt_var_pfx, ".P.metropolitan.diary.colon")] <-
                as.integer(0 + mycount_pattern_occ("Metropolitan Diary:", 
                                                   glb_allobs_df[, txt_var]))
        }

#sum(mycount_pattern_occ("[0-9]{4}:", glb_allobs_df$Headline) > 0)
#sum(mycount_pattern_occ("Quandary(.*)(?=:)", glb_allobs_df$Headline, perl=TRUE) > 0)
#sum(mycount_pattern_occ("No Comment(.*):", glb_allobs_df$Headline) > 0)
#sum(mycount_pattern_occ("Friday Night Music:", glb_allobs_df$Headline) > 0)
        if (txt_var %in% c("Headline")) {
            txt_X_df[, paste0(txt_var_pfx, ".P.facts.figures")] <-
                as.integer(0 + mycount_pattern_occ("Facts & Figures:", glb_allobs_df[, txt_var]))            
            txt_X_df[, paste0(txt_var_pfx, ".P.friday.night.music")] <-
                as.integer(0 + mycount_pattern_occ("Friday Night Music", glb_allobs_df[, txt_var]))            
            txt_X_df[, paste0(txt_var_pfx, ".P.no.comment.colon")] <-
                as.integer(0 + mycount_pattern_occ("No Comment(.*):", glb_allobs_df[, txt_var]))            
            txt_X_df[, paste0(txt_var_pfx, ".P.on.this.day")] <-
                as.integer(0 + mycount_pattern_occ("On This Day", glb_allobs_df[, txt_var]))            
            txt_X_df[, paste0(txt_var_pfx, ".P.quandary")] <-
                as.integer(0 + mycount_pattern_occ("Quandary(.*)(?=:)", glb_allobs_df[, txt_var], perl=TRUE))
            txt_X_df[, paste0(txt_var_pfx, ".P.readers.respond")] <-
                as.integer(0 + mycount_pattern_occ("Readers Respond", glb_allobs_df[, txt_var]))            
            txt_X_df[, paste0(txt_var_pfx, ".P.recap.colon")] <-
                as.integer(0 + mycount_pattern_occ("Recap:", glb_allobs_df[, txt_var]))
            txt_X_df[, paste0(txt_var_pfx, ".P.s.notebook")] <-
                as.integer(0 + mycount_pattern_occ("s Notebook", glb_allobs_df[, txt_var]))
            txt_X_df[, paste0(txt_var_pfx, ".P.today.in.politic")] <-
                as.integer(0 + mycount_pattern_occ("Today in Politic", glb_allobs_df[, txt_var]))            
            txt_X_df[, paste0(txt_var_pfx, ".P.today.in.smallbusiness")] <-
                as.integer(0 + mycount_pattern_occ("Today in Small Business:", glb_allobs_df[, txt_var]))
            txt_X_df[, paste0(txt_var_pfx, ".P.verbatim.colon")] <-
                as.integer(0 + mycount_pattern_occ("Verbatim:", glb_allobs_df[, txt_var]))
            txt_X_df[, paste0(txt_var_pfx, ".P.what.we.are")] <-
                as.integer(0 + mycount_pattern_occ("What We're", glb_allobs_df[, txt_var]))
        }

#summary(glb_allobs_df[ ,grep("P.on.this.day", names(glb_allobs_df), value=TRUE)])
        txt_X_df <- subset(txt_X_df, select=-.rnorm)
        txt_X_df <- txt_X_df[, -grep(glb_id_var, names(txt_X_df), fixed=TRUE), FALSE]
        #glb_allobs_df <- cbind(glb_allobs_df, txt_X_df)
    }
    glb_allobs_df <- cbind(glb_allobs_df, txt_X_df)
    #myplot_box(glb_allobs_df, "A.sum.TfIdf", glb_rsp_var)

    # Generate summaries
#     print(summary(glb_allobs_df))
#     print(sapply(names(glb_allobs_df), function(col) sum(is.na(glb_allobs_df[, col]))))
#     print(summary(glb_trnobs_df))
#     print(sapply(names(glb_trnobs_df), function(col) sum(is.na(glb_trnobs_df[, col]))))
#     print(summary(glb_newobs_df))
#     print(sapply(names(glb_newobs_df), function(col) sum(is.na(glb_newobs_df[, col]))))

    glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, 
                                          glb_txt_vars)
    rm(log_X_df, txt_X_df)
}

# print(sapply(names(glb_trnobs_df), function(col) sum(is.na(glb_trnobs_df[, col]))))
# print(sapply(names(glb_newobs_df), function(col) sum(is.na(glb_newobs_df[, col]))))

# print(myplot_scatter(glb_trnobs_df, "<col1_name>", "<col2_name>", smooth=TRUE))

rm(corpus_lst, full_TfIdf_DTM, full_TfIdf_vctr, 
   glb_full_DTM_lst, glb_sprs_DTM_lst, txt_corpus, txt_vctr)
```

```
## Warning in rm(corpus_lst, full_TfIdf_DTM, full_TfIdf_vctr,
## glb_full_DTM_lst, : object 'corpus_lst' not found
```

```
## Warning in rm(corpus_lst, full_TfIdf_DTM, full_TfIdf_vctr,
## glb_full_DTM_lst, : object 'full_TfIdf_DTM' not found
```

```
## Warning in rm(corpus_lst, full_TfIdf_DTM, full_TfIdf_vctr,
## glb_full_DTM_lst, : object 'full_TfIdf_vctr' not found
```

```
## Warning in rm(corpus_lst, full_TfIdf_DTM, full_TfIdf_vctr,
## glb_full_DTM_lst, : object 'glb_full_DTM_lst' not found
```

```
## Warning in rm(corpus_lst, full_TfIdf_DTM, full_TfIdf_vctr,
## glb_full_DTM_lst, : object 'glb_sprs_DTM_lst' not found
```

```
## Warning in rm(corpus_lst, full_TfIdf_DTM, full_TfIdf_vctr,
## glb_full_DTM_lst, : object 'txt_corpus' not found
```

```
## Warning in rm(corpus_lst, full_TfIdf_DTM, full_TfIdf_vctr,
## glb_full_DTM_lst, : object 'txt_vctr' not found
```

```r
extract.features_chunk_df <- myadd_chunk(extract.features_chunk_df, "extract.features_end", 
                                     major.inc=TRUE)
```

```
##                                 label step_major step_minor    bgn    end
## 2 extract.features_factorize.str.vars          2          0 18.866 18.883
## 3                extract.features_end          3          0 18.883     NA
##   elapsed
## 2   0.017
## 3      NA
```

```r
myplt_chunk(extract.features_chunk_df)
```

```
##                                 label step_major step_minor    bgn    end
## 2 extract.features_factorize.str.vars          2          0 18.866 18.883
## 1                extract.features_bgn          1          0 18.858 18.866
##   elapsed duration
## 2   0.017    0.017
## 1   0.008    0.008
## [1] "Total Elapsed Time: 18.883 secs"
```

![](NBA_PTS2_files/figure-html/extract.features-1.png) 

```r
# if (glb_save_envir)
#     save(glb_feats_df, 
#          glb_allobs_df, #glb_trnobs_df, glb_fitobs_df, glb_OOBobs_df, glb_newobs_df,
#          file=paste0(glb_out_pfx, "extract_features_dsk.RData"))
# load(paste0(glb_out_pfx, "extract_features_dsk.RData"))

replay.petrisim(pn=glb_analytics_pn, 
    replay.trans=(glb_analytics_avl_objs <- c(glb_analytics_avl_objs, 
        "data.training.all","data.new")), flip_coord=TRUE)
```

```
## time	trans	 "bgn " "fit.data.training.all " "predict.data.new " "end " 
## 0.0000 	multiple enabled transitions:  data.training.all data.new model.selected 	firing:  data.training.all 
## 1.0000 	 1 	 2 1 0 0 
## 1.0000 	multiple enabled transitions:  data.training.all data.new model.selected model.final data.training.all.prediction 	firing:  data.new 
## 2.0000 	 2 	 1 1 1 0
```

![](NBA_PTS2_files/figure-html/extract.features-2.png) 

```r
glb_chunks_df <- myadd_chunk(glb_chunks_df, "cluster.data", major.inc=TRUE)
```

```
##              label step_major step_minor    bgn    end elapsed
## 6 extract.features          3          0 18.852 20.168   1.316
## 7     cluster.data          4          0 20.168     NA      NA
```

## Step `4.0: cluster data`

```r
if (glb_cluster) {
    require(proxy)
    #require(hash)
    require(dynamicTreeCut)

#     glb_hash <- hash(key=unique(glb_allobs_df$myCategory), 
#                      values=1:length(unique(glb_allobs_df$myCategory)))
#     glb_hash_lst <- hash(key=unique(glb_allobs_df$myCategory), 
#                      values=1:length(unique(glb_allobs_df$myCategory)))
#stophere; sav_allobs_df <- glb_allobs_df; 
    print("Clustering features: ")
    print(cluster_vars <- grep("[HSA]\\.[PT]\\.", names(glb_allobs_df), value=TRUE))
    #print(cluster_vars <- grep("[HSA]\\.", names(glb_allobs_df), value=TRUE))
    glb_allobs_df$.clusterid <- 1    
    #print(max(table(glb_allobs_df$myCategory.fctr) / 20))
    for (myCategory in c("##", "Business#Business Day#Dealbook", "OpEd#Opinion#", 
                         "Styles#U.S.#", "Business#Technology#", "Science#Health#",
                         "Culture#Arts#")) {
        ctgry_allobs_df <- glb_allobs_df[glb_allobs_df$myCategory == myCategory, ]
        
        dstns_dist <- dist(ctgry_allobs_df[, cluster_vars], method = "cosine")
        dstns_mtrx <- as.matrix(dstns_dist)
        print(sprintf("max distance(%0.4f) pair:", max(dstns_mtrx)))
        row_ix <- ceiling(which.max(dstns_mtrx) / ncol(dstns_mtrx))
        col_ix <- which.max(dstns_mtrx[row_ix, ])
        print(ctgry_allobs_df[c(row_ix, col_ix), 
            c("UniqueID", "Popular", "myCategory", "Headline", cluster_vars)])
    
        min_dstns_mtrx <- dstns_mtrx
        diag(min_dstns_mtrx) <- 1
        print(sprintf("min distance(%0.4f) pair:", min(min_dstns_mtrx)))
        row_ix <- ceiling(which.min(min_dstns_mtrx) / ncol(min_dstns_mtrx))
        col_ix <- which.min(min_dstns_mtrx[row_ix, ])
        print(ctgry_allobs_df[c(row_ix, col_ix), 
            c("UniqueID", "Popular", "myCategory", "Headline", cluster_vars)])                          
    
        clusters <- hclust(dstns_dist, method = "ward.D2")
        #plot(clusters, labels=NULL, hang=-1)
        myplclust(clusters, lab.col=unclass(ctgry_allobs_df[, glb_rsp_var]))
        
        #clusterGroups = cutree(clusters, k=7)
        clusterGroups <- cutreeDynamic(clusters, minClusterSize=20, method="tree", deepSplit=0)
        # Unassigned groups are labeled 0; the largest group has label 1
        table(clusterGroups, ctgry_allobs_df[, glb_rsp_var], useNA="ifany")   
        #print(ctgry_allobs_df[which(clusterGroups == 1), c("UniqueID", "Popular", "Headline")])
        #print(ctgry_allobs_df[(clusterGroups == 1) & !is.na(ctgry_allobs_df$Popular) & (ctgry_allobs_df$Popular==1), c("UniqueID", "Popular", "Headline")])
        clusterGroups[clusterGroups == 0] <- 1
        table(clusterGroups, ctgry_allobs_df[, glb_rsp_var], useNA="ifany")        
        #summary(factor(clusterGroups))
#         clusterGroups <- clusterGroups + 
#                 100 * # has to be > max(table(glb_allobs_df$myCategory.fctr) / minClusterSize=20)
#                             which(levels(glb_allobs_df$myCategory.fctr) == myCategory)
#         table(clusterGroups, ctgry_allobs_df[, glb_rsp_var], useNA="ifany")        
    
        # add to glb_allobs_df - then split the data again
        glb_allobs_df[glb_allobs_df$myCategory==myCategory,]$.clusterid <- clusterGroups
        #print(unique(glb_allobs_df$.clusterid))
        #print(glb_feats_df[glb_feats_df$id == ".clusterid.fctr", ])
    }
    
    ctgry_xtab_df <- orderBy(reformulate(c("-", ".n")),
                              mycreate_sqlxtab_df(glb_allobs_df,
        c("myCategory", ".clusterid", glb_rsp_var)))
    ctgry_cast_df <- orderBy(~ -Y -NA, dcast(ctgry_xtab_df, 
                           myCategory + .clusterid ~ 
                               Popular.fctr, sum, value.var=".n"))
    print(ctgry_cast_df)
    #print(orderBy(~ myCategory -Y -NA, ctgry_cast_df))
    # write.table(ctgry_cast_df, paste0(glb_out_pfx, "ctgry_clst.csv"), 
    #             row.names=FALSE)
    
    print(ctgry_sum_tbl <- table(glb_allobs_df$myCategory, glb_allobs_df$.clusterid, 
                                 glb_allobs_df[, glb_rsp_var], 
                                 useNA="ifany"))
#     dsp_obs(.clusterid=1, myCategory="OpEd#Opinion#", 
#             cols=c("UniqueID", "Popular", "myCategory", ".clusterid", "Headline"),
#             all=TRUE)
    
    glb_allobs_df$.clusterid.fctr <- as.factor(glb_allobs_df$.clusterid)
    glb_exclude_vars_as_features <- c(glb_exclude_vars_as_features, 
                                      ".clusterid")
    glb_interaction_only_features["myCategory.fctr"] <- c(".clusterid.fctr")
    glb_exclude_vars_as_features <- c(glb_exclude_vars_as_features, 
                                      cluster_vars)
}

# Re-partition
glb_trnobs_df <- subset(glb_allobs_df, .src == "Train")
glb_newobs_df <- subset(glb_allobs_df, .src == "Test")

glb_chunks_df <- myadd_chunk(glb_chunks_df, "select.features", major.inc=TRUE)
```

```
##             label step_major step_minor    bgn    end elapsed
## 7    cluster.data          4          0 20.168 20.441   0.273
## 8 select.features          5          0 20.441     NA      NA
```

## Step `5.0: select features`

```r
print(glb_feats_df <- myselect_features(entity_df=glb_trnobs_df, 
                       exclude_vars_as_features=glb_exclude_vars_as_features, 
                       rsp_var=glb_rsp_var))
```

```
##                  id        cor.y exclude.as.feat   cor.y.abs
## FG               FG  0.941954732               0 0.941954732
## X2P             X2P  0.825724570               0 0.825724570
## FGA             FGA  0.793979466               0 0.793979466
## oppPTS       oppPTS  0.789074714               1 0.789074714
## AST             AST  0.759889152               0 0.759889152
## X2PA           X2PA  0.708361398               0 0.708361398
## FT               FT  0.697483850               0 0.697483850
## FTA             FTA  0.655058756               0 0.655058756
## SeasonEnd SeasonEnd -0.639527733               1 0.639527733
## X3PA           X3PA -0.515198125               0 0.515198125
## ORB             ORB  0.496920726               0 0.496920726
## X3P             X3P -0.489948908               0 0.489948908
## STL             STL  0.430990261               0 0.430990261
## TOV             TOV  0.427138315               0 0.427138315
## PTS.diff   PTS.diff  0.309378875               1 0.309378875
## W                 W  0.298825614               1 0.298825614
## Playoffs   Playoffs  0.270601328               1 0.270601328
## Team.fctr Team.fctr -0.169301718               1 0.169301718
## BLK             BLK  0.152055366               0 0.152055366
## DRB             DRB  0.090291366               0 0.090291366
## .rnorm       .rnorm  0.009627489               0 0.009627489
```

```r
# sav_feats_df <- glb_feats_df; glb_feats_df <- sav_feats_df
print(glb_feats_df <- orderBy(~-cor.y, 
          myfind_cor_features(feats_df=glb_feats_df, obs_df=glb_trnobs_df, 
                              rsp_var=glb_rsp_var)))
```

```
## Loading required package: reshape2
```

```
## [1] "cor(X3P, X3PA)=0.9945"
## [1] "cor(PTS, X3P)=-0.4899"
## [1] "cor(PTS, X3PA)=-0.5152"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified X3P as highly correlated with X3PA
```

```
## [1] "cor(X2P, X2PA)=0.9653"
## [1] "cor(PTS, X2P)=0.8257"
## [1] "cor(PTS, X2PA)=0.7084"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified X2PA as highly correlated with X2P
```

```
## [1] "cor(FT, FTA)=0.9505"
## [1] "cor(PTS, FT)=0.6975"
## [1] "cor(PTS, FTA)=0.6551"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified FTA as highly correlated with FT
```

```
## [1] "cor(FG, X2P)=0.9429"
## [1] "cor(PTS, FG)=0.9420"
## [1] "cor(PTS, X2P)=0.8257"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified X2P as highly correlated with FG
```

```
## [1] "cor(FG, FGA)=0.8797"
## [1] "cor(PTS, FG)=0.9420"
## [1] "cor(PTS, FGA)=0.7940"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified FGA as highly correlated with FG
```

```
## [1] "cor(AST, FG)=0.8123"
## [1] "cor(PTS, AST)=0.7599"
## [1] "cor(PTS, FG)=0.9420"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified AST as highly correlated with FG
```

```
##           id        cor.y exclude.as.feat   cor.y.abs cor.high.X freqRatio
## 5         FG  0.941954732               0 0.941954732       <NA>  1.000000
## 18       X2P  0.825724570               0 0.825724570         FG  1.200000
## 6        FGA  0.793979466               0 0.793979466         FG  1.000000
## 9     oppPTS  0.789074714               1 0.789074714       <NA>  1.000000
## 2        AST  0.759889152               0 0.759889152         FG  1.000000
## 19      X2PA  0.708361398               0 0.708361398        X2P  1.000000
## 7         FT  0.697483850               0 0.697483850       <NA>  1.166667
## 8        FTA  0.655058756               0 0.655058756         FT  1.000000
## 10       ORB  0.496920726               0 0.496920726       <NA>  1.000000
## 14       STL  0.430990261               0 0.430990261       <NA>  1.000000
## 16       TOV  0.427138315               0 0.427138315       <NA>  1.166667
## 12  PTS.diff  0.309378875               1 0.309378875       <NA>  1.200000
## 17         W  0.298825614               1 0.298825614       <NA>  1.000000
## 11  Playoffs  0.270601328               1 0.270601328       <NA>  1.352113
## 3        BLK  0.152055366               0 0.152055366       <NA>  1.125000
## 4        DRB  0.090291366               0 0.090291366       <NA>  1.428571
## 1     .rnorm  0.009627489               0 0.009627489       <NA>  1.000000
## 15 Team.fctr -0.169301718               1 0.169301718       <NA>  1.000000
## 20       X3P -0.489948908               0 0.489948908       X3PA  1.500000
## 21      X3PA -0.515198125               0 0.515198125       <NA>  1.000000
## 13 SeasonEnd -0.639527733               1 0.639527733       <NA>  1.000000
##    percentUnique zeroVar   nzv myNearZV is.cor.y.abs.low
## 5      70.898204   FALSE FALSE    FALSE            FALSE
## 18     72.934132   FALSE FALSE    FALSE            FALSE
## 6      76.287425   FALSE FALSE    FALSE            FALSE
## 9      83.353293   FALSE FALSE    FALSE            FALSE
## 2      62.874251   FALSE FALSE    FALSE            FALSE
## 19     85.748503   FALSE FALSE    FALSE            FALSE
## 7      57.964072   FALSE FALSE    FALSE            FALSE
## 8      63.473054   FALSE FALSE    FALSE            FALSE
## 10     51.736527   FALSE FALSE    FALSE            FALSE
## 14     38.562874   FALSE FALSE    FALSE            FALSE
## 16     53.772455   FALSE FALSE    FALSE            FALSE
## 12     75.089820   FALSE FALSE    FALSE            FALSE
## 17      7.065868   FALSE FALSE    FALSE            FALSE
## 11      0.239521   FALSE FALSE    FALSE            FALSE
## 3      36.526946   FALSE FALSE    FALSE            FALSE
## 4      49.700599   FALSE FALSE    FALSE            FALSE
## 1     100.000000   FALSE FALSE    FALSE            FALSE
## 15      4.431138   FALSE FALSE    FALSE            FALSE
## 20     57.724551   FALSE FALSE    FALSE            FALSE
## 21     77.844311   FALSE FALSE    FALSE            FALSE
## 13      3.712575   FALSE FALSE    FALSE            FALSE
```

```r
#subset(glb_feats_df, id %in% c("A.nuppr.log", "S.nuppr.log"))
print(myplot_scatter(glb_feats_df, "percentUnique", "freqRatio", 
                     colorcol_name="myNearZV", jitter=TRUE) + 
          geom_point(aes(shape=nzv)) + xlim(-5, 25))
```

```
## Warning in myplot_scatter(glb_feats_df, "percentUnique", "freqRatio",
## colorcol_name = "myNearZV", : converting myNearZV to class:factor
```

```
## Warning in loop_apply(n, do.ply): Removed 17 rows containing missing values
## (geom_point).
```

```
## Warning in loop_apply(n, do.ply): Removed 17 rows containing missing values
## (geom_point).
```

```
## Warning in loop_apply(n, do.ply): Removed 17 rows containing missing values
## (geom_point).
```

![](NBA_PTS2_files/figure-html/select.features-1.png) 

```r
print(subset(glb_feats_df, myNearZV))
```

```
##  [1] id               cor.y            exclude.as.feat  cor.y.abs       
##  [5] cor.high.X       freqRatio        percentUnique    zeroVar         
##  [9] nzv              myNearZV         is.cor.y.abs.low
## <0 rows> (or 0-length row.names)
```

```r
glb_allobs_df <- glb_allobs_df[, setdiff(names(glb_allobs_df), 
                                         subset(glb_feats_df, myNearZV)$id)]

if (!is.null(glb_interaction_only_features))
    glb_feats_df[glb_feats_df$id %in% glb_interaction_only_features, "interaction.feat"] <-
        names(glb_interaction_only_features) else
    glb_feats_df$interaction.feat <- NA        

mycheck_problem_data(glb_allobs_df, terminate = TRUE)
```

```
## [1] "numeric data missing in : "
## named integer(0)
## [1] "numeric data w/ 0s in : "
## Playoffs 
##      369 
## [1] "numeric data w/ Infs in : "
## named integer(0)
## [1] "numeric data w/ NaNs in : "
## named integer(0)
## [1] "string data missing in : "
##      Team .rownames 
##         0         0
```

```r
# glb_allobs_df %>% filter(is.na(Married.fctr)) %>% tbl_df()
# glb_allobs_df %>% count(Married.fctr)
# levels(glb_allobs_df$Married.fctr)

glb_chunks_df <- myadd_chunk(glb_chunks_df, "partition.data.training", major.inc=TRUE)
```

```
##                     label step_major step_minor    bgn    end elapsed
## 8         select.features          5          0 20.441 21.119   0.678
## 9 partition.data.training          6          0 21.119     NA      NA
```

## Step `6.0: partition data training`

```r
if (all(is.na(glb_newobs_df[, glb_rsp_var]))) {
    require(caTools)
    
    set.seed(glb_split_sample.seed)
    split <- sample.split(glb_trnobs_df[, glb_rsp_var_raw], 
        SplitRatio=1 - (nrow(glb_newobs_df) * 1.1 / nrow(glb_trnobs_df)))
    glb_fitobs_df <- glb_trnobs_df[split, ] 
    glb_OOBobs_df <- glb_trnobs_df[!split ,]    
} else {
    print(sprintf("Newdata contains non-NA data for %s; setting OOB to Newdata", 
                  glb_rsp_var))
    glb_fitobs_df <- glb_trnobs_df; glb_OOBobs_df <- glb_newobs_df
}
```

```
## [1] "Newdata contains non-NA data for PTS; setting OOB to Newdata"
```

```r
if (!is.null(glb_max_fitent_obs) && (nrow(glb_fitobs_df) > glb_max_fitent_obs)) {
    warning("glb_fitobs_df restricted to glb_max_fitent_obs: ", 
            format(glb_max_fitent_obs, big.mark=","))
    org_fitent_df <- glb_fitobs_df
    glb_fitobs_df <- 
        org_fitent_df[split <- sample.split(org_fitent_df[, glb_rsp_var_raw], 
                                            SplitRatio=glb_max_fitent_obs), ]
    org_fitent_df <- NULL
}

glb_allobs_df$.lcn <- ""
glb_allobs_df[glb_allobs_df[, glb_id_var] %in% 
              glb_fitobs_df[, glb_id_var], ".lcn"] <- "Fit"
glb_allobs_df[glb_allobs_df[, glb_id_var] %in% 
              glb_OOBobs_df[, glb_id_var], ".lcn"] <- "OOB"

dsp_class_dstrb <- function(obs_df, location_var, partition_var) {
    xtab_df <- mycreate_xtab_df(obs_df, c(location_var, partition_var))
    rownames(xtab_df) <- xtab_df[, location_var]
    xtab_df <- xtab_df[, -grepl(location_var, names(xtab_df))]
    print(xtab_df)
    print(xtab_df / rowSums(xtab_df, na.rm=TRUE))    
}    

# Ensure proper splits by glb_rsp_var_raw & user-specified feature for OOB vs. new
if (!is.null(glb_category_vars)) {
    if (glb_is_classification)
        dsp_class_dstrb(glb_allobs_df, ".lcn", glb_rsp_var_raw)
    newent_ctgry_df <- mycreate_sqlxtab_df(subset(glb_allobs_df, .src == "Test"), 
                                           glb_category_vars)
    OOBobs_ctgry_df <- mycreate_sqlxtab_df(subset(glb_allobs_df, .lcn == "OOB"), 
                                           glb_category_vars)
    glb_ctgry_df <- merge(newent_ctgry_df, OOBobs_ctgry_df, by=glb_category_vars
                          , all=TRUE, suffixes=c(".Tst", ".OOB"))
    glb_ctgry_df$.freqRatio.Tst <- glb_ctgry_df$.n.Tst / sum(glb_ctgry_df$.n.Tst, na.rm=TRUE)
    glb_ctgry_df$.freqRatio.OOB <- glb_ctgry_df$.n.OOB / sum(glb_ctgry_df$.n.OOB, na.rm=TRUE)
    print(orderBy(~-.freqRatio.Tst-.freqRatio.OOB, glb_ctgry_df))
}

# Run this line by line
print("glb_feats_df:");   print(dim(glb_feats_df))
```

```
## [1] "glb_feats_df:"
```

```
## [1] 21 12
```

```r
sav_feats_df <- glb_feats_df
glb_feats_df <- sav_feats_df

glb_feats_df[, "rsp_var_raw"] <- FALSE
glb_feats_df[glb_feats_df$id == glb_rsp_var_raw, "rsp_var_raw"] <- TRUE 
glb_feats_df$exclude.as.feat <- (glb_feats_df$exclude.as.feat == 1)
if (!is.null(glb_id_var) && glb_id_var != ".rownames")
    glb_feats_df[glb_feats_df$id %in% glb_id_var, "id_var"] <- TRUE 
add_feats_df <- data.frame(id=glb_rsp_var, exclude.as.feat=TRUE, rsp_var=TRUE)
row.names(add_feats_df) <- add_feats_df$id; print(add_feats_df)
```

```
##      id exclude.as.feat rsp_var
## PTS PTS            TRUE    TRUE
```

```r
glb_feats_df <- myrbind_df(glb_feats_df, add_feats_df)
if (glb_id_var != ".rownames")
    print(subset(glb_feats_df, rsp_var_raw | rsp_var | id_var)) else
    print(subset(glb_feats_df, rsp_var_raw | rsp_var))    
```

```
##      id cor.y exclude.as.feat cor.y.abs cor.high.X freqRatio percentUnique
## PTS PTS    NA            TRUE        NA       <NA>        NA            NA
##     zeroVar nzv myNearZV is.cor.y.abs.low interaction.feat rsp_var_raw
## PTS      NA  NA       NA               NA               NA          NA
##     rsp_var
## PTS    TRUE
```

```r
print("glb_feats_df vs. glb_allobs_df: "); 
```

```
## [1] "glb_feats_df vs. glb_allobs_df: "
```

```r
print(setdiff(glb_feats_df$id, names(glb_allobs_df)))
```

```
## character(0)
```

```r
print("glb_allobs_df vs. glb_feats_df: "); 
```

```
## [1] "glb_allobs_df vs. glb_feats_df: "
```

```r
# Ensure these are only chr vars
print(setdiff(setdiff(names(glb_allobs_df), glb_feats_df$id), 
                myfind_chr_cols_df(glb_allobs_df)))
```

```
## character(0)
```

```r
#print(setdiff(setdiff(names(glb_allobs_df), glb_exclude_vars_as_features), 
#                glb_feats_df$id))

print("glb_allobs_df: "); print(dim(glb_allobs_df))
```

```
## [1] "glb_allobs_df: "
```

```
## [1] 863  26
```

```r
print("glb_trnobs_df: "); print(dim(glb_trnobs_df))
```

```
## [1] "glb_trnobs_df: "
```

```
## [1] 835  25
```

```r
print("glb_fitobs_df: "); print(dim(glb_fitobs_df))
```

```
## [1] "glb_fitobs_df: "
```

```
## [1] 835  25
```

```r
print("glb_OOBobs_df: "); print(dim(glb_OOBobs_df))
```

```
## [1] "glb_OOBobs_df: "
```

```
## [1] 28 25
```

```r
print("glb_newobs_df: "); print(dim(glb_newobs_df))
```

```
## [1] "glb_newobs_df: "
```

```
## [1] 28 25
```

```r
# # Does not handle NULL or length(glb_id_var) > 1
# glb_allobs_df$.src.trn <- 0
# glb_allobs_df[glb_allobs_df[, glb_id_var] %in% glb_trnobs_df[, glb_id_var], 
#                 ".src.trn"] <- 1 
# glb_allobs_df$.src.fit <- 0
# glb_allobs_df[glb_allobs_df[, glb_id_var] %in% glb_fitobs_df[, glb_id_var], 
#                 ".src.fit"] <- 1 
# glb_allobs_df$.src.OOB <- 0
# glb_allobs_df[glb_allobs_df[, glb_id_var] %in% glb_OOBobs_df[, glb_id_var], 
#                 ".src.OOB"] <- 1 
# glb_allobs_df$.src.new <- 0
# glb_allobs_df[glb_allobs_df[, glb_id_var] %in% glb_newobs_df[, glb_id_var], 
#                 ".src.new"] <- 1 
# #print(unique(glb_allobs_df[, ".src.trn"]))
# write_cols <- c(glb_feats_df$id, 
#                 ".src.trn", ".src.fit", ".src.OOB", ".src.new")
# glb_allobs_df <- glb_allobs_df[, write_cols]
# 
# tmp_feats_df <- glb_feats_df
# tmp_entity_df <- glb_allobs_df

if (glb_save_envir)
    save(glb_feats_df, 
         glb_allobs_df, #glb_trnobs_df, glb_fitobs_df, glb_OOBobs_df, glb_newobs_df,
         file=paste0(glb_out_pfx, "blddfs_dsk.RData"))
# load(paste0(glb_out_pfx, "blddfs_dsk.RData"))

# if (!all.equal(tmp_feats_df, glb_feats_df))
#     stop("glb_feats_df r/w not working")
# if (!all.equal(tmp_entity_df, glb_allobs_df))
#     stop("glb_allobs_df r/w not working")

rm(split)
```

```
## Warning in rm(split): object 'split' not found
```

```r
glb_chunks_df <- myadd_chunk(glb_chunks_df, "fit.models", major.inc=TRUE)
```

```
##                      label step_major step_minor    bgn    end elapsed
## 9  partition.data.training          6          0 21.119 21.412   0.293
## 10              fit.models          7          0 21.412     NA      NA
```

## Step `7.0: fit models`

```r
# load(paste0(glb_out_pfx, "dsk.RData"))
# keep_cols <- setdiff(names(glb_allobs_df), 
#                      grep("^.src", names(glb_allobs_df), value=TRUE))
# glb_trnobs_df <- glb_allobs_df[glb_allobs_df$.src.trn == 1, keep_cols]
# glb_fitobs_df <- glb_allobs_df[glb_allobs_df$.src.fit == 1, keep_cols]
# glb_OOBobs_df <- glb_allobs_df[glb_allobs_df$.src.OOB == 1, keep_cols]
# glb_newobs_df <- glb_allobs_df[glb_allobs_df$.src.new == 1, keep_cols]
# 
# glb_models_lst <- list(); glb_models_df <- data.frame()
# 
if (glb_is_classification && glb_is_binomial && 
        (length(unique(glb_fitobs_df[, glb_rsp_var])) < 2))
    stop("glb_fitobs_df$", glb_rsp_var, ": contains less than 2 unique values: ",
         paste0(unique(glb_fitobs_df[, glb_rsp_var]), collapse=", "))

max_cor_y_x_vars <- orderBy(~ -cor.y.abs, 
        subset(glb_feats_df, (exclude.as.feat == 0) & !is.cor.y.abs.low & 
                                is.na(cor.high.X)))[1:2, "id"]
# while(length(max_cor_y_x_vars) < 2) {
#     max_cor_y_x_vars <- c(max_cor_y_x_vars, orderBy(~ -cor.y.abs, 
#             subset(glb_feats_df, (exclude.as.feat == 0) & !is.cor.y.abs.low))[3, "id"])    
# }
if (!is.null(glb_Baseline_mdl_var)) {
    if ((max_cor_y_x_vars[1] != glb_Baseline_mdl_var) & 
        (glb_feats_df[max_cor_y_x_vars[1], "cor.y.abs"] > 
         glb_feats_df[glb_Baseline_mdl_var, "cor.y.abs"]))
        stop(max_cor_y_x_vars[1], " has a lower correlation with ", glb_rsp_var, 
             " than the Baseline var: ", glb_Baseline_mdl_var)
}

glb_model_type <- ifelse(glb_is_regression, "regression", "classification")
    
# Baseline
if (!is.null(glb_Baseline_mdl_var)) 
    ret_lst <- myfit_mdl_fn(model_id="Baseline", model_method="mybaseln_classfr",
                            indep_vars_vctr=glb_Baseline_mdl_var,
                            rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                            fit_df=glb_fitobs_df, OOB_df=glb_OOBobs_df)

# Most Frequent Outcome "MFO" model: mean(y) for regression
#   Not using caret's nullModel since model stats not avl
#   Cannot use rpart for multinomial classification since it predicts non-MFO
ret_lst <- myfit_mdl(model_id="MFO", 
                     model_method=ifelse(glb_is_regression, "lm", "myMFO_classfr"), 
                     model_type=glb_model_type,
                        indep_vars_vctr=".rnorm",
                        rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                        fit_df=glb_fitobs_df, OOB_df=glb_OOBobs_df)
```

```
## [1] "fitting model: MFO.lm"
## [1] "    indep_vars: .rnorm"
## Fitting parameter = none on full training set
```

![](NBA_PTS2_files/figure-html/fit.models_0-1.png) ![](NBA_PTS2_files/figure-html/fit.models_0-2.png) ![](NBA_PTS2_files/figure-html/fit.models_0-3.png) ![](NBA_PTS2_files/figure-html/fit.models_0-4.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##      Min       1Q   Median       3Q      Max 
## -1468.75  -433.93   -56.55   416.87  1993.12 
## 
## Coefficients:
##             Estimate Std. Error t value Pr(>|t|)    
## (Intercept) 8370.085     20.127 415.874   <2e-16 ***
## .rnorm         5.746     20.679   0.278    0.781    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 581.4 on 833 degrees of freedom
## Multiple R-squared:  9.269e-05,	Adjusted R-squared:  -0.001108 
## F-statistic: 0.07722 on 1 and 833 DF,  p-value: 0.7812
## 
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##   model_id model_method  feats max.nTuningRuns min.elapsedtime.everything
## 1   MFO.lm           lm .rnorm               0                      0.682
##   min.elapsedtime.final max.R.sq.fit min.RMSE.fit max.R.sq.OOB
## 1                 0.003 9.268854e-05     580.6652 0.0003645831
##   min.RMSE.OOB max.Adj.R.sq.fit
## 1      453.679      -0.00110768
```

```r
if (glb_is_classification)
    # "random" model - only for classification; 
    #   none needed for regression since it is same as MFO
    ret_lst <- myfit_mdl(model_id="Random", model_method="myrandom_classfr",
                            model_type=glb_model_type,                         
                            indep_vars_vctr=".rnorm",
                            rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                            fit_df=glb_fitobs_df, OOB_df=glb_OOBobs_df)

# Any models that have tuning parameters has "better" results with cross-validation
#   (except rf) & "different" results for different outcome metrics

# Max.cor.Y
#   Check impact of cv
#       rpart is not a good candidate since caret does not optimize cp (only tuning parameter of rpart) well
ret_lst <- myfit_mdl(model_id="Max.cor.Y.cv.0", 
                        model_method="rpart",
                     model_type=glb_model_type,
                        indep_vars_vctr=max_cor_y_x_vars,
                        rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                        fit_df=glb_fitobs_df, OOB_df=glb_OOBobs_df)
```

```
## [1] "fitting model: Max.cor.Y.cv.0.rpart"
## [1] "    indep_vars: FG, FT"
```

```
## Loading required package: rpart
```

```
## Fitting cp = 0.625 on full training set
```

```
## Loading required package: rpart.plot
```

![](NBA_PTS2_files/figure-html/fit.models_0-5.png) 

```
## Call:
## rpart(formula = .outcome ~ ., control = list(minsplit = 20, minbucket = 7, 
##     cp = 0, maxcompete = 4, maxsurrogate = 5, usesurrogate = 2, 
##     surrogatestyle = 0, maxdepth = 30, xval = 0))
##   n= 835 
## 
##         CP nsplit rel error
## 1 0.624723      0         1
## 
## Node number 1: 835 observations
##   mean=8370.24, MSE=337203.3 
## 
## n= 835 
## 
## node), split, n, deviance, yval
##       * denotes terminal node
## 
## 1) root 835 281564800 8370.24 *
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##               model_id model_method  feats max.nTuningRuns
## 1 Max.cor.Y.cv.0.rpart        rpart FG, FT               0
##   min.elapsedtime.everything min.elapsedtime.final max.R.sq.fit
## 1                      0.621                 0.019            0
##   min.RMSE.fit max.R.sq.OOB min.RMSE.OOB
## 1     580.6921            0     453.7617
```

```r
ret_lst <- myfit_mdl(model_id="Max.cor.Y.cv.0.cp.0", 
                        model_method="rpart",
                     model_type=glb_model_type,
                        indep_vars_vctr=max_cor_y_x_vars,
                        rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                        fit_df=glb_fitobs_df, OOB_df=glb_OOBobs_df,
                        n_cv_folds=0, 
            tune_models_df=data.frame(parameter="cp", min=0.0, max=0.0, by=0.1))
```

```
## [1] "fitting model: Max.cor.Y.cv.0.cp.0.rpart"
## [1] "    indep_vars: FG, FT"
## Fitting cp = 0 on full training set
```

![](NBA_PTS2_files/figure-html/fit.models_0-6.png) 

```
## Call:
## rpart(formula = .outcome ~ ., control = list(minsplit = 20, minbucket = 7, 
##     cp = 0, maxcompete = 4, maxsurrogate = 5, usesurrogate = 2, 
##     surrogatestyle = 0, maxdepth = 30, xval = 0))
##   n= 835 
## 
##              CP nsplit  rel error
## 1  6.247230e-01      0 1.00000000
## 2  1.211189e-01      1 0.37527697
## 3  6.735895e-02      2 0.25415806
## 4  3.101652e-02      3 0.18679911
## 5  1.418080e-02      4 0.15578259
## 6  1.304650e-02      5 0.14160179
## 7  1.288298e-02      6 0.12855529
## 8  8.336997e-03      7 0.11567231
## 9  6.016133e-03      8 0.10733531
## 10 4.897272e-03      9 0.10131918
## 11 4.040325e-03     10 0.09642191
## 12 3.976143e-03     11 0.09238159
## 13 3.622821e-03     12 0.08840544
## 14 2.741576e-03     13 0.08478262
## 15 2.686175e-03     14 0.08204105
## 16 2.595021e-03     15 0.07935487
## 17 2.370434e-03     16 0.07675985
## 18 2.032246e-03     17 0.07438942
## 19 1.914936e-03     18 0.07235717
## 20 1.724190e-03     19 0.07044223
## 21 1.705552e-03     20 0.06871804
## 22 1.646214e-03     21 0.06701249
## 23 1.627072e-03     22 0.06536628
## 24 1.416667e-03     23 0.06373921
## 25 1.110854e-03     24 0.06232254
## 26 1.087106e-03     25 0.06121169
## 27 9.536758e-04     26 0.06012458
## 28 7.216056e-04     27 0.05917090
## 29 6.961895e-04     28 0.05844930
## 30 6.850019e-04     29 0.05775311
## 31 6.424703e-04     30 0.05706811
## 32 6.407194e-04     31 0.05642564
## 33 5.859531e-04     32 0.05578492
## 34 5.257552e-04     33 0.05519896
## 35 5.002518e-04     34 0.05467321
## 36 4.984735e-04     37 0.05317245
## 37 4.944719e-04     38 0.05267398
## 38 4.734047e-04     39 0.05217951
## 39 4.349422e-04     40 0.05170610
## 40 4.109703e-04     41 0.05127116
## 41 4.053581e-04     42 0.05086019
## 42 3.981980e-04     43 0.05045483
## 43 3.911168e-04     44 0.05005663
## 44 3.849480e-04     45 0.04966552
## 45 3.832867e-04     47 0.04889562
## 46 3.689954e-04     48 0.04851234
## 47 3.550673e-04     49 0.04814334
## 48 3.388961e-04     50 0.04778827
## 49 3.114718e-04     51 0.04744938
## 50 2.821174e-04     52 0.04713790
## 51 2.638783e-04     53 0.04685579
## 52 2.422940e-04     54 0.04659191
## 53 2.087328e-04     55 0.04634961
## 54 1.985641e-04     56 0.04614088
## 55 1.782783e-04     57 0.04594232
## 56 1.752921e-04     58 0.04576404
## 57 1.624231e-04     60 0.04541346
## 58 1.444766e-04     61 0.04525103
## 59 1.437844e-04     62 0.04510656
## 60 1.282225e-04     63 0.04496277
## 61 1.044346e-04     64 0.04483455
## 62 9.071191e-05     65 0.04473011
## 63 7.728998e-05     66 0.04463940
## 64 7.174930e-05     67 0.04456211
## 65 6.033509e-05     68 0.04449036
## 66 5.546150e-05     69 0.04443003
## 67 4.019060e-05     70 0.04437457
## 68 0.000000e+00     71 0.04433438
## 
## Variable importance
## FG FT 
## 75 25 
## 
## Node number 1: 835 observations,    complexity param=0.624723
##   mean=8370.24, MSE=337203.3 
##   left son=2 (527 obs) right son=3 (308 obs)
##   Primary splits:
##       FG < 3306.5 to the left,  improve=0.6247230, (0 missing)
##       FT < 1697.5 to the left,  improve=0.3517337, (0 missing)
##   Surrogate splits:
##       FT < 1778.5 to the left,  agree=0.766, adj=0.367, (0 split)
## 
## Node number 2: 527 observations,    complexity param=0.1211189
##   mean=8019.359, MSE=125200.1 
##   left son=4 (171 obs) right son=5 (356 obs)
##   Primary splits:
##       FG < 2945   to the left,  improve=0.5168624, (0 missing)
##       FT < 1482.5 to the left,  improve=0.1553086, (0 missing)
##   Surrogate splits:
##       FT < 1241.5 to the left,  agree=0.681, adj=0.018, (0 split)
## 
## Node number 3: 308 observations,    complexity param=0.06735895
##   mean=8970.61, MSE=128845.2 
##   left son=6 (198 obs) right son=7 (110 obs)
##   Primary splits:
##       FG < 3558   to the left,  improve=0.4779195, (0 missing)
##       FT < 1955.5 to the left,  improve=0.2171089, (0 missing)
##   Surrogate splits:
##       FT < 2001   to the left,  agree=0.675, adj=0.091, (0 split)
## 
## Node number 4: 171 observations,    complexity param=0.0130465
##   mean=7652.316, MSE=58182.63 
##   left son=8 (54 obs) right son=9 (117 obs)
##   Primary splits:
##       FG < 2819.5 to the left,  improve=0.3692180, (0 missing)
##       FT < 1425   to the left,  improve=0.2782324, (0 missing)
##   Surrogate splits:
##       FT < 1300   to the left,  agree=0.708, adj=0.074, (0 split)
## 
## Node number 5: 356 observations,    complexity param=0.03101652
##   mean=8195.663, MSE=61596.67 
##   left son=10 (163 obs) right son=11 (193 obs)
##   Primary splits:
##       FG < 3057.5 to the left,  improve=0.3982576, (0 missing)
##       FT < 1804.5 to the left,  improve=0.2056137, (0 missing)
##   Surrogate splits:
##       FT < 1362.5 to the left,  agree=0.548, adj=0.012, (0 split)
## 
## Node number 6: 198 observations,    complexity param=0.01288298
##   mean=8785.652, MSE=54485.65 
##   left son=12 (123 obs) right son=13 (75 obs)
##   Primary splits:
##       FT < 1816.5 to the left,  improve=0.3362384, (0 missing)
##       FG < 3408.5 to the left,  improve=0.2257927, (0 missing)
## 
## Node number 7: 110 observations,    complexity param=0.0141808
##   mean=9303.536, MSE=90274.94 
##   left son=14 (85 obs) right son=15 (25 obs)
##   Primary splits:
##       FG < 3751   to the left,  improve=0.4020862, (0 missing)
##       FT < 1941   to the left,  improve=0.3395614, (0 missing)
##   Surrogate splits:
##       FT < 2170.5 to the left,  agree=0.8, adj=0.12, (0 split)
## 
## Node number 8: 54 observations,    complexity param=0.002741576
##   mean=7436.574, MSE=44594.47 
##   left son=16 (8 obs) right son=17 (46 obs)
##   Primary splits:
##       FT < 1347   to the left,  improve=0.3205559, (0 missing)
##       FG < 2741   to the left,  improve=0.3203774, (0 missing)
## 
## Node number 9: 117 observations,    complexity param=0.003976143
##   mean=7751.889, MSE=33057.21 
##   left son=18 (26 obs) right son=19 (91 obs)
##   Primary splits:
##       FT < 1432.5 to the left,  improve=0.2894598, (0 missing)
##       FG < 2901.5 to the left,  improve=0.2204246, (0 missing)
## 
## Node number 10: 163 observations,    complexity param=0.004897272
##   mean=8025.233, MSE=25834.22 
##   left son=20 (57 obs) right son=21 (106 obs)
##   Primary splits:
##       FT < 1502.5 to the left,  improve=0.3274535, (0 missing)
##       FG < 3012.5 to the left,  improve=0.1121400, (0 missing)
## 
## Node number 11: 193 observations,    complexity param=0.008336997
##   mean=8339.601, MSE=46550.66 
##   left son=22 (105 obs) right son=23 (88 obs)
##   Primary splits:
##       FT < 1604   to the left,  improve=0.2612791, (0 missing)
##       FG < 3139   to the left,  improve=0.1047791, (0 missing)
##   Surrogate splits:
##       FG < 3178.5 to the left,  agree=0.591, adj=0.102, (0 split)
## 
## Node number 12: 123 observations,    complexity param=0.003622821
##   mean=8679.959, MSE=40345.63 
##   left son=24 (54 obs) right son=25 (69 obs)
##   Primary splits:
##       FG < 3391.5 to the left,  improve=0.2055528, (0 missing)
##       FT < 1629.5 to the left,  improve=0.1349141, (0 missing)
##   Surrogate splits:
##       FT < 1768.5 to the right, agree=0.602, adj=0.093, (0 split)
## 
## Node number 13: 75 observations,    complexity param=0.002595021
##   mean=8958.987, MSE=29310.04 
##   left son=26 (25 obs) right son=27 (50 obs)
##   Primary splits:
##       FG < 3410.5 to the left,  improve=0.3323850, (0 missing)
##       FT < 2050.5 to the left,  improve=0.1963458, (0 missing)
## 
## Node number 14: 85 observations,    complexity param=0.006016133
##   mean=9200.212, MSE=45559.11 
##   left son=28 (23 obs) right son=29 (62 obs)
##   Primary splits:
##       FT < 1724.5 to the left,  improve=0.4374229, (0 missing)
##       FG < 3648   to the left,  improve=0.1716765, (0 missing)
##   Surrogate splits:
##       FG < 3740.5 to the right, agree=0.741, adj=0.043, (0 split)
## 
## Node number 15: 25 observations,    complexity param=0.004040325
##   mean=9654.84, MSE=82596.21 
##   left son=30 (15 obs) right son=31 (10 obs)
##   Primary splits:
##       FT < 1892.5 to the left,  improve=0.5509275, (0 missing)
##       FG < 3899.5 to the left,  improve=0.2288336, (0 missing)
##   Surrogate splits:
##       FG < 3855   to the left,  agree=0.64, adj=0.1, (0 split)
## 
## Node number 16: 8 observations
##   mean=7149.875, MSE=23505.11 
## 
## Node number 17: 46 observations,    complexity param=0.00172419
##   mean=7486.435, MSE=31481.07 
##   left son=34 (25 obs) right son=35 (21 obs)
##   Primary splits:
##       FG < 2772.5 to the left,  improve=0.3352401, (0 missing)
##       FT < 1566.5 to the left,  improve=0.2446409, (0 missing)
##   Surrogate splits:
##       FT < 1566.5 to the left,  agree=0.717, adj=0.381, (0 split)
## 
## Node number 18: 26 observations,    complexity param=0.0009536758
##   mean=7568.885, MSE=22414.18 
##   left son=36 (17 obs) right son=37 (9 obs)
##   Primary splits:
##       FG < 2916.5 to the left,  improve=0.4607686, (0 missing)
##       FT < 1360.5 to the left,  improve=0.1412395, (0 missing)
##   Surrogate splits:
##       FT < 1411   to the left,  agree=0.808, adj=0.444, (0 split)
## 
## Node number 19: 91 observations,    complexity param=0.002032246
##   mean=7804.176, MSE=23795.42 
##   left son=38 (54 obs) right son=39 (37 obs)
##   Primary splits:
##       FG < 2901.5 to the left,  improve=0.2642529, (0 missing)
##       FT < 1540.5 to the left,  improve=0.1005385, (0 missing)
##   Surrogate splits:
##       FT < 1769.5 to the left,  agree=0.604, adj=0.027, (0 split)
## 
## Node number 20: 57 observations,    complexity param=0.0004984735
##   mean=7899.807, MSE=12190.72 
##   left son=40 (37 obs) right son=41 (20 obs)
##   Primary splits:
##       FG < 3012.5 to the left,  improve=0.20198370, (0 missing)
##       FT < 1450.5 to the left,  improve=0.08134442, (0 missing)
##   Surrogate splits:
##       FT < 1292   to the right, agree=0.684, adj=0.1, (0 split)
## 
## Node number 21: 106 observations,    complexity param=0.001627072
##   mean=8092.679, MSE=20162.33 
##   left son=42 (54 obs) right son=43 (52 obs)
##   Primary splits:
##       FG < 2994   to the left,  improve=0.2143574, (0 missing)
##       FT < 1748   to the left,  improve=0.1827706, (0 missing)
##   Surrogate splits:
##       FT < 1632.5 to the right, agree=0.566, adj=0.115, (0 split)
## 
## Node number 22: 105 observations,    complexity param=0.002686175
##   mean=8238.638, MSE=38721.79 
##   left son=44 (36 obs) right son=45 (69 obs)
##   Primary splits:
##       FG < 3126.5 to the left,  improve=0.1860235, (0 missing)
##       FT < 1489.5 to the left,  improve=0.1406284, (0 missing)
##   Surrogate splits:
##       FT < 1251.5 to the left,  agree=0.676, adj=0.056, (0 split)
## 
## Node number 23: 88 observations,    complexity param=0.001914936
##   mean=8460.068, MSE=29216.88 
##   left son=46 (55 obs) right son=47 (33 obs)
##   Primary splits:
##       FT < 1781.5 to the left,  improve=0.20970850, (0 missing)
##       FG < 3212   to the left,  improve=0.08950988, (0 missing)
##   Surrogate splits:
##       FG < 3075.5 to the right, agree=0.67, adj=0.121, (0 split)
## 
## Node number 24: 54 observations,    complexity param=0.001416667
##   mean=8577.019, MSE=46295.91 
##   left son=48 (22 obs) right son=49 (32 obs)
##   Primary splits:
##       FT < 1619.5 to the left,  improve=0.15955470, (0 missing)
##       FG < 3338.5 to the left,  improve=0.04347864, (0 missing)
##   Surrogate splits:
##       FG < 3382.5 to the right, agree=0.63, adj=0.091, (0 split)
## 
## Node number 25: 69 observations,    complexity param=0.001110854
##   mean=8760.522, MSE=20905.44 
##   left son=50 (32 obs) right son=51 (37 obs)
##   Primary splits:
##       FT < 1641.5 to the left,  improve=0.2168338, (0 missing)
##       FG < 3456   to the left,  improve=0.0916439, (0 missing)
##   Surrogate splits:
##       FG < 3416.5 to the left,  agree=0.609, adj=0.156, (0 split)
## 
## Node number 26: 25 observations,    complexity param=0.000242294
##   mean=8819.4, MSE=15423.2 
##   left son=52 (8 obs) right son=53 (17 obs)
##   Primary splits:
##       FT < 1889.5 to the left,  improve=0.17693200, (0 missing)
##       FG < 3370   to the left,  improve=0.02317186, (0 missing)
##   Surrogate splits:
##       FG < 3387   to the right, agree=0.72, adj=0.125, (0 split)
## 
## Node number 27: 50 observations,    complexity param=0.001087106
##   mean=9028.78, MSE=21640.13 
##   left son=54 (41 obs) right son=55 (9 obs)
##   Primary splits:
##       FT < 1988   to the left,  improve=0.2828917, (0 missing)
##       FG < 3483.5 to the left,  improve=0.2034928, (0 missing)
## 
## Node number 28: 23 observations,    complexity param=0.0006850019
##   mean=8968.435, MSE=12933.2 
##   left son=56 (9 obs) right son=57 (14 obs)
##   Primary splits:
##       FG < 3628   to the left,  improve=0.6483897, (0 missing)
##       FT < 1602   to the left,  improve=0.1276290, (0 missing)
##   Surrogate splits:
##       FT < 1562.5 to the left,  agree=0.652, adj=0.111, (0 split)
## 
## Node number 29: 62 observations,    complexity param=0.002370434
##   mean=9286.194, MSE=30340.8 
##   left son=58 (45 obs) right son=59 (17 obs)
##   Primary splits:
##       FT < 1970.5 to the left,  improve=0.3548031, (0 missing)
##       FG < 3647.5 to the left,  improve=0.2673739, (0 missing)
##   Surrogate splits:
##       FG < 3735   to the left,  agree=0.758, adj=0.118, (0 split)
## 
## Node number 30: 15 observations
##   mean=9480.667, MSE=28879.16 
## 
## Node number 31: 10 observations
##   mean=9916.1, MSE=49410.49 
## 
## Node number 34: 25 observations,    complexity param=0.0003388961
##   mean=7392.28, MSE=22872.28 
##   left son=68 (13 obs) right son=69 (12 obs)
##   Primary splits:
##       FG < 2741   to the left,  improve=0.1668766, (0 missing)
##       FT < 1525   to the left,  improve=0.1161270, (0 missing)
##   Surrogate splits:
##       FT < 1504.5 to the left,  agree=0.6, adj=0.167, (0 split)
## 
## Node number 35: 21 observations,    complexity param=0.0003832867
##   mean=7598.524, MSE=18611.96 
##   left son=70 (7 obs) right son=71 (14 obs)
##   Primary splits:
##       FT < 1508.5 to the left,  improve=0.2761153, (0 missing)
##       FG < 2803   to the right, improve=0.1766865, (0 missing)
##   Surrogate splits:
##       FG < 2806   to the right, agree=0.762, adj=0.286, (0 split)
## 
## Node number 36: 17 observations
##   mean=7494.941, MSE=8190.997 
## 
## Node number 37: 9 observations
##   mean=7708.556, MSE=19444.47 
## 
## Node number 38: 54 observations,    complexity param=0.0004109703
##   mean=7738.537, MSE=14717.32 
##   left son=76 (25 obs) right son=77 (29 obs)
##   Primary splits:
##       FT < 1571   to the left,  improve=0.14560160, (0 missing)
##       FG < 2841.5 to the left,  improve=0.07620825, (0 missing)
##   Surrogate splits:
##       FG < 2885.5 to the right, agree=0.593, adj=0.12, (0 split)
## 
## Node number 39: 37 observations,    complexity param=0.0005257552
##   mean=7899.973, MSE=21579.43 
##   left son=78 (11 obs) right son=79 (26 obs)
##   Primary splits:
##       FT < 1531.5 to the left,  improve=0.1854044, (0 missing)
##       FG < 2924.5 to the left,  improve=0.1265167, (0 missing)
##   Surrogate splits:
##       FG < 2943.5 to the right, agree=0.757, adj=0.182, (0 split)
## 
## Node number 40: 37 observations,    complexity param=0.0001782783
##   mean=7863.324, MSE=10055.14 
##   left son=80 (19 obs) right son=81 (18 obs)
##   Primary splits:
##       FT < 1450.5 to the left,  improve=0.1349233, (0 missing)
##       FG < 2976   to the left,  improve=0.1161383, (0 missing)
##   Surrogate splits:
##       FG < 2989   to the left,  agree=0.622, adj=0.222, (0 split)
## 
## Node number 41: 20 observations,    complexity param=9.071191e-05
##   mean=7967.3, MSE=9123.91 
##   left son=82 (13 obs) right son=83 (7 obs)
##   Primary splits:
##       FT < 1453   to the left,  improve=0.13996890, (0 missing)
##       FG < 3037.5 to the left,  improve=0.02721713, (0 missing)
##   Surrogate splits:
##       FG < 3024.5 to the right, agree=0.75, adj=0.286, (0 split)
## 
## Node number 42: 54 observations,    complexity param=0.0006961895
##   mean=8028.167, MSE=14377.92 
##   left son=84 (47 obs) right son=85 (7 obs)
##   Primary splits:
##       FT < 1786   to the left,  improve=0.25247360, (0 missing)
##       FG < 2969   to the left,  improve=0.05653201, (0 missing)
## 
## Node number 43: 52 observations,    complexity param=0.0006407194
##   mean=8159.673, MSE=17359.1 
##   left son=86 (39 obs) right son=87 (13 obs)
##   Primary splits:
##       FT < 1710.5 to the left,  improve=0.19985520, (0 missing)
##       FG < 3029.5 to the left,  improve=0.05248081, (0 missing)
## 
## Node number 44: 36 observations,    complexity param=0.0006424703
##   mean=8121.139, MSE=29317.4 
##   left son=88 (22 obs) right son=89 (14 obs)
##   Primary splits:
##       FT < 1513   to the left,  improve=0.1713971, (0 missing)
##       FG < 3086.5 to the left,  improve=0.1289080, (0 missing)
##   Surrogate splits:
##       FG < 3068   to the right, agree=0.639, adj=0.071, (0 split)
## 
## Node number 45: 69 observations,    complexity param=0.001646214
##   mean=8299.942, MSE=32667.1 
##   left son=90 (26 obs) right son=91 (43 obs)
##   Primary splits:
##       FT < 1464   to the left,  improve=0.20563880, (0 missing)
##       FG < 3271.5 to the left,  improve=0.08297167, (0 missing)
##   Surrogate splits:
##       FG < 3139   to the left,  agree=0.638, adj=0.038, (0 split)
## 
## Node number 46: 55 observations,    complexity param=0.0003911168
##   mean=8399.436, MSE=24714.21 
##   left son=92 (31 obs) right son=93 (24 obs)
##   Primary splits:
##       FG < 3196   to the left,  improve=0.08101684, (0 missing)
##       FT < 1619.5 to the right, improve=0.02053522, (0 missing)
##   Surrogate splits:
##       FT < 1624   to the right, agree=0.618, adj=0.125, (0 split)
## 
## Node number 47: 33 observations,    complexity param=0.0004349422
##   mean=8561.121, MSE=20382.59 
##   left son=94 (22 obs) right son=95 (11 obs)
##   Primary splits:
##       FG < 3230.5 to the left,  improve=0.1820692, (0 missing)
##       FT < 1858.5 to the left,  improve=0.1176989, (0 missing)
##   Surrogate splits:
##       FT < 1823.5 to the right, agree=0.727, adj=0.182, (0 split)
## 
## Node number 48: 22 observations,    complexity param=0.0003114718
##   mean=8473.364, MSE=55021.5 
##   left son=96 (10 obs) right son=97 (12 obs)
##   Primary splits:
##       FT < 1538.5 to the left,  improve=0.07245059, (0 missing)
##       FG < 3351.5 to the right, improve=0.02809420, (0 missing)
##   Surrogate splits:
##       FG < 3363   to the right, agree=0.682, adj=0.3, (0 split)
## 
## Node number 49: 32 observations,    complexity param=0.0005859531
##   mean=8648.281, MSE=27831.95 
##   left son=98 (12 obs) right son=99 (20 obs)
##   Primary splits:
##       FG < 3336   to the left,  improve=0.18524540, (0 missing)
##       FT < 1761.5 to the left,  improve=0.02673973, (0 missing)
##   Surrogate splits:
##       FT < 1809   to the right, agree=0.688, adj=0.167, (0 split)
## 
## Node number 50: 32 observations,    complexity param=0.0001752921
##   mean=8688.125, MSE=22921.98 
##   left son=100 (10 obs) right son=101 (22 obs)
##   Primary splits:
##       FT < 1611   to the right, improve=0.05162869, (0 missing)
##       FG < 3479   to the left,  improve=0.03205772, (0 missing)
##   Surrogate splits:
##       FG < 3406.5 to the left,  agree=0.719, adj=0.1, (0 split)
## 
## Node number 51: 37 observations,    complexity param=0.0002821174
##   mean=8823.135, MSE=10707.95 
##   left son=102 (10 obs) right son=103 (27 obs)
##   Primary splits:
##       FG < 3456   to the left,  improve=0.20049330, (0 missing)
##       FT < 1755.5 to the left,  improve=0.07076068, (0 missing)
## 
## Node number 52: 8 observations
##   mean=8743.25, MSE=6919.188 
## 
## Node number 53: 17 observations
##   mean=8855.235, MSE=15412.06 
## 
## Node number 54: 41 observations,    complexity param=0.0007216056
##   mean=8992.122, MSE=12872.79 
##   left son=108 (24 obs) right son=109 (17 obs)
##   Primary splits:
##       FG < 3491.5 to the left,  improve=0.3849654, (0 missing)
##       FT < 1943.5 to the left,  improve=0.0540851, (0 missing)
##   Surrogate splits:
##       FT < 1849.5 to the right, agree=0.659, adj=0.176, (0 split)
## 
## Node number 55: 9 observations
##   mean=9195.778, MSE=27570.17 
## 
## Node number 56: 9 observations
##   mean=8854.222, MSE=4537.284 
## 
## Node number 57: 14 observations
##   mean=9041.857, MSE=4553.98 
## 
## Node number 58: 45 observations,    complexity param=0.001705552
##   mean=9222.422, MSE=21456.33 
##   left son=116 (31 obs) right son=117 (14 obs)
##   Primary splits:
##       FG < 3684.5 to the left,  improve=0.4973650, (0 missing)
##       FT < 1884.5 to the left,  improve=0.0961802, (0 missing)
##   Surrogate splits:
##       FT < 1738.5 to the right, agree=0.733, adj=0.143, (0 split)
## 
## Node number 59: 17 observations
##   mean=9455, MSE=14597.88 
## 
## Node number 68: 13 observations
##   mean=7332.923, MSE=29694.07 
## 
## Node number 69: 12 observations
##   mean=7456.583, MSE=7530.243 
## 
## Node number 70: 7 observations
##   mean=7497.143, MSE=16891.27 
## 
## Node number 71: 14 observations
##   mean=7649.214, MSE=11763.74 
## 
## Node number 76: 25 observations,    complexity param=5.54615e-05
##   mean=7688.68, MSE=10405.66 
##   left son=152 (12 obs) right son=153 (13 obs)
##   Primary splits:
##       FG < 2865.5 to the right, improve=0.06002890, (0 missing)
##       FT < 1535   to the left,  improve=0.03191741, (0 missing)
##   Surrogate splits:
##       FT < 1472   to the left,  agree=0.68, adj=0.333, (0 split)
## 
## Node number 77: 29 observations,    complexity param=0.0004053581
##   mean=7781.517, MSE=14444.11 
##   left son=154 (8 obs) right son=155 (21 obs)
##   Primary splits:
##       FG < 2840   to the left,  improve=0.27247600, (0 missing)
##       FT < 1655   to the left,  improve=0.04275723, (0 missing)
## 
## Node number 78: 11 observations
##   mean=7802.727, MSE=14302.74 
## 
## Node number 79: 26 observations,    complexity param=0.0004734047
##   mean=7941.115, MSE=18964.41 
##   left son=158 (15 obs) right son=159 (11 obs)
##   Primary splits:
##       FG < 2924.5 to the left,  improve=0.27033240, (0 missing)
##       FT < 1678.5 to the left,  improve=0.06963449, (0 missing)
##   Surrogate splits:
##       FT < 1744.5 to the left,  agree=0.654, adj=0.182, (0 split)
## 
## Node number 80: 19 observations
##   mean=7827.474, MSE=6088.144 
## 
## Node number 81: 18 observations
##   mean=7901.167, MSE=11453.81 
## 
## Node number 82: 13 observations
##   mean=7941.077, MSE=9038.533 
## 
## Node number 83: 7 observations
##   mean=8016, MSE=5633.714 
## 
## Node number 84: 47 observations,    complexity param=0.0001985641
##   mean=8004.915, MSE=10906.12 
##   left son=168 (34 obs) right son=169 (13 obs)
##   Primary splits:
##       FT < 1662.5 to the left,  improve=0.10907140, (0 missing)
##       FG < 2969   to the left,  improve=0.05380949, (0 missing)
## 
## Node number 85: 7 observations
##   mean=8184.286, MSE=9685.347 
## 
## Node number 86: 39 observations,    complexity param=0.0002087328
##   mean=8125.667, MSE=13785.97 
##   left son=172 (8 obs) right son=173 (31 obs)
##   Primary splits:
##       FG < 3002   to the left,  improve=0.10931190, (0 missing)
##       FT < 1569.5 to the left,  improve=0.02790829, (0 missing)
##   Surrogate splits:
##       FT < 1691.5 to the right, agree=0.846, adj=0.25, (0 split)
## 
## Node number 87: 13 observations
##   mean=8261.692, MSE=14201.29 
## 
## Node number 88: 22 observations,    complexity param=0.0003689954
##   mean=8064.591, MSE=19418.33 
##   left son=176 (14 obs) right son=177 (8 obs)
##   Primary splits:
##       FG < 3107   to the left,  improve=0.24320060, (0 missing)
##       FT < 1419   to the left,  improve=0.06116228, (0 missing)
##   Surrogate splits:
##       FT < 1431   to the right, agree=0.682, adj=0.125, (0 split)
## 
## Node number 89: 14 observations
##   mean=8210, MSE=31951.86 
## 
## Node number 90: 26 observations,    complexity param=0.0001437844
##   mean=8194.538, MSE=20367.17 
##   left son=180 (12 obs) right son=181 (14 obs)
##   Primary splits:
##       FG < 3175.5 to the left,  improve=0.07645149, (0 missing)
##       FT < 1365.5 to the left,  improve=0.05355152, (0 missing)
##   Surrogate splits:
##       FT < 1276.5 to the left,  agree=0.615, adj=0.167, (0 split)
## 
## Node number 91: 43 observations,    complexity param=0.0005002518
##   mean=8363.674, MSE=29324.82 
##   left son=182 (36 obs) right son=183 (7 obs)
##   Primary splits:
##       FG < 3271.5 to the left,  improve=0.10797880, (0 missing)
##       FT < 1529.5 to the right, improve=0.05967323, (0 missing)
## 
## Node number 92: 31 observations,    complexity param=0.000384948
##   mean=8360.065, MSE=19067.42 
##   left son=184 (7 obs) right son=185 (24 obs)
##   Primary splits:
##       FG < 3174.5 to the right, improve=0.1432701, (0 missing)
##       FT < 1714.5 to the right, improve=0.1281557, (0 missing)
## 
## Node number 93: 24 observations,    complexity param=0.0002638783
##   mean=8450.292, MSE=27419.46 
##   left son=186 (17 obs) right son=187 (7 obs)
##   Primary splits:
##       FT < 1733.5 to the left,  improve=0.11290470, (0 missing)
##       FG < 3247   to the left,  improve=0.01888201, (0 missing)
## 
## Node number 94: 22 observations,    complexity param=0.0003550673
##   mean=8518.045, MSE=18644.13 
##   left son=188 (10 obs) right son=189 (12 obs)
##   Primary splits:
##       FT < 1858.5 to the left,  improve=0.2437385, (0 missing)
##       FG < 3166.5 to the right, improve=0.1077695, (0 missing)
##   Surrogate splits:
##       FG < 3099   to the left,  agree=0.682, adj=0.3, (0 split)
## 
## Node number 95: 11 observations
##   mean=8647.273, MSE=12726.38 
## 
## Node number 96: 10 observations
##   mean=8404.2, MSE=60434.76 
## 
## Node number 97: 12 observations
##   mean=8531, MSE=43202.17 
## 
## Node number 98: 12 observations
##   mean=8555.583, MSE=15572.41 
## 
## Node number 99: 20 observations,    complexity param=7.728998e-05
##   mean=8703.9, MSE=26938.49 
##   left son=198 (8 obs) right son=199 (12 obs)
##   Primary splits:
##       FG < 3354   to the left,  improve=0.04039227, (0 missing)
##       FT < 1778   to the right, improve=0.01736136, (0 missing)
##   Surrogate splits:
##       FT < 1778   to the right, agree=0.75, adj=0.375, (0 split)
## 
## Node number 100: 10 observations
##   mean=8637.1, MSE=12604.49 
## 
## Node number 101: 22 observations,    complexity param=0.0001752921
##   mean=8711.318, MSE=25890.4 
##   left son=202 (8 obs) right son=203 (14 obs)
##   Primary splits:
##       FT < 1508   to the left,  improve=0.10681800, (0 missing)
##       FG < 3525   to the right, improve=0.02471914, (0 missing)
##   Surrogate splits:
##       FG < 3526.5 to the right, agree=0.727, adj=0.25, (0 split)
## 
## Node number 102: 10 observations
##   mean=8747, MSE=8864.2 
## 
## Node number 103: 27 observations,    complexity param=0.0001282225
##   mean=8851.333, MSE=8448.815 
##   left son=206 (20 obs) right son=207 (7 obs)
##   Primary splits:
##       FG < 3529   to the left,  improve=0.15826430, (0 missing)
##       FT < 1762.5 to the left,  improve=0.04095387, (0 missing)
## 
## Node number 108: 24 observations,    complexity param=4.01906e-05
##   mean=8932.875, MSE=5623.193 
##   left son=216 (17 obs) right son=217 (7 obs)
##   Primary splits:
##       FT < 1948   to the left,  improve=0.08385105, (0 missing)
##       FG < 3463.5 to the left,  improve=0.03093433, (0 missing)
## 
## Node number 109: 17 observations
##   mean=9075.765, MSE=11155.83 
## 
## Node number 116: 31 observations,    complexity param=0.0004944719
##   mean=9153, MSE=11279.94 
##   left son=232 (22 obs) right son=233 (9 obs)
##   Primary splits:
##       FT < 1906.5 to the left,  improve=0.3981545, (0 missing)
##       FG < 3575.5 to the left,  improve=0.1001856, (0 missing)
## 
## Node number 117: 14 observations
##   mean=9376.143, MSE=9688.122 
## 
## Node number 152: 12 observations
##   mean=7662.667, MSE=9795.222 
## 
## Node number 153: 13 observations
##   mean=7712.692, MSE=9767.905 
## 
## Node number 154: 8 observations
##   mean=7679.875, MSE=4551.109 
## 
## Node number 155: 21 observations,    complexity param=0.0001624231
##   mean=7820.238, MSE=12777.9 
##   left son=310 (8 obs) right son=311 (13 obs)
##   Primary splits:
##       FT < 1643.5 to the left,  improve=0.1704305, (0 missing)
##       FG < 2881.5 to the left,  improve=0.1259378, (0 missing)
##   Surrogate splits:
##       FG < 2853.5 to the left,  agree=0.667, adj=0.125, (0 split)
## 
## Node number 158: 15 observations
##   mean=7879.8, MSE=6472.027 
## 
## Node number 159: 11 observations
##   mean=8024.727, MSE=23881.83 
## 
## Node number 168: 34 observations,    complexity param=6.033509e-05
##   mean=7983.588, MSE=9165.595 
##   left son=336 (17 obs) right son=337 (17 obs)
##   Primary splits:
##       FG < 2971   to the left,  improve=0.05451408, (0 missing)
##       FT < 1597   to the left,  improve=0.04467731, (0 missing)
##   Surrogate splits:
##       FT < 1611   to the right, agree=0.588, adj=0.176, (0 split)
## 
## Node number 169: 13 observations
##   mean=8060.692, MSE=11157.6 
## 
## Node number 172: 8 observations
##   mean=8049.25, MSE=6244.438 
## 
## Node number 173: 31 observations,    complexity param=0.0001444766
##   mean=8145.387, MSE=13836.3 
##   left son=346 (16 obs) right son=347 (15 obs)
##   Primary splits:
##       FT < 1570.5 to the left,  improve=0.09484055, (0 missing)
##       FG < 3020.5 to the right, improve=0.01170292, (0 missing)
##   Surrogate splits:
##       FG < 3014   to the right, agree=0.677, adj=0.333, (0 split)
## 
## Node number 176: 14 observations
##   mean=8012.643, MSE=14217.09 
## 
## Node number 177: 8 observations
##   mean=8155.5, MSE=15533.5 
## 
## Node number 180: 12 observations
##   mean=8151.917, MSE=16272.74 
## 
## Node number 181: 14 observations
##   mean=8231.071, MSE=20984.92 
## 
## Node number 182: 36 observations,    complexity param=0.0005002518
##   mean=8338.861, MSE=28305.68 
##   left son=364 (9 obs) right son=365 (27 obs)
##   Primary splits:
##       FG < 3220.5 to the right, improve=0.07552479, (0 missing)
##       FT < 1529.5 to the right, improve=0.04943597, (0 missing)
## 
## Node number 183: 7 observations
##   mean=8491.286, MSE=15115.06 
## 
## Node number 184: 7 observations
##   mean=8263.286, MSE=9758.49 
## 
## Node number 185: 24 observations,    complexity param=0.000384948
##   mean=8388.292, MSE=18253.96 
##   left son=370 (8 obs) right son=371 (16 obs)
##   Primary splits:
##       FT < 1694.5 to the right, improve=0.30151020, (0 missing)
##       FG < 3138.5 to the left,  improve=0.04322579, (0 missing)
## 
## Node number 186: 17 observations
##   mean=8414.588, MSE=19536.83 
## 
## Node number 187: 7 observations
##   mean=8537, MSE=35948.86 
## 
## Node number 188: 10 observations
##   mean=8444.2, MSE=12939.56 
## 
## Node number 189: 12 observations
##   mean=8579.583, MSE=15066.74 
## 
## Node number 198: 8 observations
##   mean=8663.5, MSE=29547.5 
## 
## Node number 199: 12 observations
##   mean=8730.833, MSE=23385.64 
## 
## Node number 202: 8 observations
##   mean=8641.75, MSE=12872.44 
## 
## Node number 203: 14 observations
##   mean=8751.071, MSE=28983.35 
## 
## Node number 206: 20 observations,    complexity param=7.17493e-05
##   mean=8829.7, MSE=9222.21 
##   left son=412 (12 obs) right son=413 (8 obs)
##   Primary splits:
##       FT < 1755.5 to the left,  improve=0.10952950, (0 missing)
##       FG < 3475.5 to the right, improve=0.05984217, (0 missing)
##   Surrogate splits:
##       FG < 3488.5 to the left,  agree=0.65, adj=0.125, (0 split)
## 
## Node number 207: 7 observations
##   mean=8913.143, MSE=1081.551 
## 
## Node number 216: 17 observations
##   mean=8918.941, MSE=6109.467 
## 
## Node number 217: 7 observations
##   mean=8966.714, MSE=2825.633 
## 
## Node number 232: 22 observations,    complexity param=0.0001044346
##   mean=9110.136, MSE=6805.936 
##   left son=464 (14 obs) right son=465 (8 obs)
##   Primary splits:
##       FT < 1845   to the left,  improve=0.1963867, (0 missing)
##       FG < 3598   to the left,  improve=0.1842221, (0 missing)
## 
## Node number 233: 9 observations
##   mean=9257.778, MSE=6746.84 
## 
## Node number 310: 8 observations
##   mean=7760.75, MSE=4111.438 
## 
## Node number 311: 13 observations
##   mean=7856.846, MSE=14593.21 
## 
## Node number 336: 17 observations
##   mean=7961.235, MSE=10151.24 
## 
## Node number 337: 17 observations
##   mean=8005.941, MSE=7180.644 
## 
## Node number 346: 16 observations
##   mean=8110.312, MSE=11894.34 
## 
## Node number 347: 15 observations
##   mean=8182.8, MSE=13195.76 
## 
## Node number 364: 9 observations
##   mean=8258.778, MSE=39526.84 
## 
## Node number 365: 27 observations,    complexity param=0.0005002518
##   mean=8365.556, MSE=21714.91 
##   left son=730 (20 obs) right son=731 (7 obs)
##   Primary splits:
##       FG < 3201   to the left,  improve=0.3572251, (0 missing)
##       FT < 1556.5 to the right, improve=0.1171928, (0 missing)
## 
## Node number 370: 8 observations
##   mean=8283.375, MSE=3702.484 
## 
## Node number 371: 16 observations
##   mean=8440.75, MSE=17274.06 
## 
## Node number 412: 12 observations
##   mean=8803.75, MSE=10794.69 
## 
## Node number 413: 8 observations
##   mean=8868.625, MSE=4338.234 
## 
## Node number 464: 14 observations
##   mean=9082.5, MSE=5479.679 
## 
## Node number 465: 8 observations
##   mean=9158.5, MSE=5451.25 
## 
## Node number 730: 20 observations,    complexity param=0.000398198
##   mean=8313.45, MSE=16626.65 
##   left son=1460 (8 obs) right son=1461 (12 obs)
##   Primary splits:
##       FG < 3162.5 to the right, improve=0.3371652, (0 missing)
##       FT < 1528.5 to the right, improve=0.2170524, (0 missing)
##   Surrogate splits:
##       FT < 1528.5 to the right, agree=0.7, adj=0.25, (0 split)
## 
## Node number 731: 7 observations
##   mean=8514.429, MSE=6332.531 
## 
## Node number 1460: 8 observations
##   mean=8221.75, MSE=5177.438 
## 
## Node number 1461: 12 observations
##   mean=8374.583, MSE=14916.24 
## 
## n= 835 
## 
## node), split, n, deviance, yval
##       * denotes terminal node
## 
##    1) root 835 2.815648e+08 8370.240  
##      2) FG< 3306.5 527 6.598046e+07 8019.359  
##        4) FG< 2945 171 9.949229e+06 7652.316  
##          8) FG< 2819.5 54 2.408101e+06 7436.574  
##           16) FT< 1347 8 1.880409e+05 7149.875 *
##           17) FT>=1347 46 1.448129e+06 7486.435  
##             34) FG< 2772.5 25 5.718070e+05 7392.280  
##               68) FG< 2741 13 3.860229e+05 7332.923 *
##               69) FG>=2741 12 9.036292e+04 7456.583 *
##             35) FG>=2772.5 21 3.908512e+05 7598.524  
##               70) FT< 1508.5 7 1.182389e+05 7497.143 *
##               71) FT>=1508.5 14 1.646924e+05 7649.214 *
##          9) FG>=2819.5 117 3.867694e+06 7751.889  
##           18) FT< 1432.5 26 5.827687e+05 7568.885  
##             36) FG< 2916.5 17 1.392469e+05 7494.941 *
##             37) FG>=2916.5 9 1.750002e+05 7708.556 *
##           19) FT>=1432.5 91 2.165383e+06 7804.176  
##             38) FG< 2901.5 54 7.947354e+05 7738.537  
##               76) FT< 1571 25 2.601414e+05 7688.680  
##                152) FG>=2865.5 12 1.175427e+05 7662.667 *
##                153) FG< 2865.5 13 1.269828e+05 7712.692 *
##               77) FT>=1571 29 4.188792e+05 7781.517  
##                154) FG< 2840 8 3.640888e+04 7679.875 *
##                155) FG>=2840 21 2.683358e+05 7820.238  
##                  310) FT< 1643.5 8 3.289150e+04 7760.750 *
##                  311) FT>=1643.5 13 1.897117e+05 7856.846 *
##             39) FG>=2901.5 37 7.984390e+05 7899.973  
##               78) FT< 1531.5 11 1.573302e+05 7802.727 *
##               79) FT>=1531.5 26 4.930747e+05 7941.115  
##                158) FG< 2924.5 15 9.708040e+04 7879.800 *
##                159) FG>=2924.5 11 2.627002e+05 8024.727 *
##        5) FG>=2945 356 2.192841e+07 8195.663  
##         10) FG< 3057.5 163 4.210977e+06 8025.233  
##           20) FT< 1502.5 57 6.948709e+05 7899.807  
##             40) FG< 3012.5 37 3.720401e+05 7863.324  
##               80) FT< 1450.5 19 1.156747e+05 7827.474 *
##               81) FT>=1450.5 18 2.061685e+05 7901.167 *
##             41) FG>=3012.5 20 1.824782e+05 7967.300  
##               82) FT< 1453 13 1.175009e+05 7941.077 *
##               83) FT>=1453 7 3.943600e+04 8016.000 *
##           21) FT>=1502.5 106 2.137207e+06 8092.679  
##             42) FG< 2994 54 7.764075e+05 8028.167  
##               84) FT< 1786 47 5.125877e+05 8004.915  
##                168) FT< 1662.5 34 3.116302e+05 7983.588  
##                  336) FG< 2971 17 1.725711e+05 7961.235 *
##                  337) FG>=2971 17 1.220709e+05 8005.941 *
##                169) FT>=1662.5 13 1.450488e+05 8060.692 *
##               85) FT>=1786 7 6.779743e+04 8184.286 *
##             43) FG>=2994 52 9.026734e+05 8159.673  
##               86) FT< 1710.5 39 5.376527e+05 8125.667  
##                172) FG< 3002 8 4.995550e+04 8049.250 *
##                173) FG>=3002 31 4.289254e+05 8145.387  
##                  346) FT< 1570.5 16 1.903094e+05 8110.312 *
##                  347) FT>=1570.5 15 1.979364e+05 8182.800 *
##               87) FT>=1710.5 13 1.846168e+05 8261.692 *
##         11) FG>=3057.5 193 8.984278e+06 8339.601  
##           22) FT< 1604 105 4.065788e+06 8238.638  
##             44) FG< 3126.5 36 1.055426e+06 8121.139  
##               88) FT< 1513 22 4.272033e+05 8064.591  
##                176) FG< 3107 14 1.990392e+05 8012.643 *
##                177) FG>=3107 8 1.242680e+05 8155.500 *
##               89) FT>=1513 14 4.473260e+05 8210.000 *
##             45) FG>=3126.5 69 2.254030e+06 8299.942  
##               90) FT< 1464 26 5.295465e+05 8194.538  
##                180) FG< 3175.5 12 1.952729e+05 8151.917 *
##                181) FG>=3175.5 14 2.937889e+05 8231.071 *
##               91) FT>=1464 43 1.260967e+06 8363.674  
##                182) FG< 3271.5 36 1.019004e+06 8338.861  
##                  364) FG>=3220.5 9 3.557416e+05 8258.778 *
##                  365) FG< 3220.5 27 5.863027e+05 8365.556  
##                    730) FG< 3201 20 3.325330e+05 8313.450  
##                     1460) FG>=3162.5 8 4.141950e+04 8221.750 *
##                     1461) FG< 3162.5 12 1.789949e+05 8374.583 *
##                    731) FG>=3201 7 4.432771e+04 8514.429 *
##                183) FG>=3271.5 7 1.058054e+05 8491.286 *
##           23) FT>=1604 88 2.571086e+06 8460.068  
##             46) FT< 1781.5 55 1.359282e+06 8399.436  
##               92) FG< 3196 31 5.910899e+05 8360.065  
##                184) FG>=3174.5 7 6.830943e+04 8263.286 *
##                185) FG< 3174.5 24 4.380950e+05 8388.292  
##                  370) FT>=1694.5 8 2.961988e+04 8283.375 *
##                  371) FT< 1694.5 16 2.763850e+05 8440.750 *
##               93) FG>=3196 24 6.580670e+05 8450.292  
##                186) FT< 1733.5 17 3.321261e+05 8414.588 *
##                187) FT>=1733.5 7 2.516420e+05 8537.000 *
##             47) FT>=1781.5 33 6.726255e+05 8561.121  
##               94) FG< 3230.5 22 4.101710e+05 8518.045  
##                188) FT< 1858.5 10 1.293956e+05 8444.200 *
##                189) FT>=1858.5 12 1.808009e+05 8579.583 *
##               95) FG>=3230.5 11 1.399902e+05 8647.273 *
##      3) FG>=3306.5 308 3.968431e+07 8970.610  
##        6) FG< 3558 198 1.078816e+07 8785.652  
##         12) FT< 1816.5 123 4.962513e+06 8679.959  
##           24) FG< 3391.5 54 2.499979e+06 8577.019  
##             48) FT< 1619.5 22 1.210473e+06 8473.364  
##               96) FT< 1538.5 10 6.043476e+05 8404.200 *
##               97) FT>=1538.5 12 5.184260e+05 8531.000 *
##             49) FT>=1619.5 32 8.906225e+05 8648.281  
##               98) FG< 3336 12 1.868689e+05 8555.583 *
##               99) FG>=3336 20 5.387698e+05 8703.900  
##                198) FG< 3354 8 2.363800e+05 8663.500 *
##                199) FG>=3354 12 2.806277e+05 8730.833 *
##           25) FG>=3391.5 69 1.442475e+06 8760.522  
##             50) FT< 1641.5 32 7.335035e+05 8688.125  
##              100) FT>=1611 10 1.260449e+05 8637.100 *
##              101) FT< 1611 22 5.695888e+05 8711.318  
##                202) FT< 1508 8 1.029795e+05 8641.750 *
##                203) FT>=1508 14 4.057669e+05 8751.071 *
##             51) FT>=1641.5 37 3.961943e+05 8823.135  
##              102) FG< 3456 10 8.864200e+04 8747.000 *
##              103) FG>=3456 27 2.281180e+05 8851.333  
##                206) FG< 3529 20 1.844442e+05 8829.700  
##                  412) FT< 1755.5 12 1.295362e+05 8803.750 *
##                  413) FT>=1755.5 8 3.470588e+04 8868.625 *
##                207) FG>=3529 7 7.570857e+03 8913.143 *
##         13) FT>=1816.5 75 2.198253e+06 8958.987  
##           26) FG< 3410.5 25 3.855800e+05 8819.400  
##             52) FT< 1889.5 8 5.535350e+04 8743.250 *
##             53) FT>=1889.5 17 2.620051e+05 8855.235 *
##           27) FG>=3410.5 50 1.082007e+06 9028.780  
##             54) FT< 1988 41 5.277844e+05 8992.122  
##              108) FG< 3491.5 24 1.349566e+05 8932.875  
##                216) FT< 1948 17 1.038609e+05 8918.941 *
##                217) FT>=1948 7 1.977943e+04 8966.714 *
##              109) FG>=3491.5 17 1.896491e+05 9075.765 *
##             55) FT>=1988 9 2.481316e+05 9195.778 *
##        7) FG>=3558 110 9.930243e+06 9303.536  
##         14) FG< 3751 85 3.872524e+06 9200.212  
##           28) FT< 1724.5 23 2.974637e+05 8968.435  
##             56) FG< 3628 9 4.083556e+04 8854.222 *
##             57) FG>=3628 14 6.375571e+04 9041.857 *
##           29) FT>=1724.5 62 1.881130e+06 9286.194  
##             58) FT< 1970.5 45 9.655350e+05 9222.422  
##              116) FG< 3684.5 31 3.496780e+05 9153.000  
##                232) FT< 1906.5 22 1.497306e+05 9110.136  
##                  464) FT< 1845 14 7.671550e+04 9082.500 *
##                  465) FT>=1845 8 4.361000e+04 9158.500 *
##                233) FT>=1906.5 9 6.072156e+04 9257.778 *
##              117) FG>=3684.5 14 1.356337e+05 9376.143 *
##             59) FT>=1970.5 17 2.481640e+05 9455.000 *
##         15) FG>=3751 25 2.064905e+06 9654.840  
##           30) FT< 1892.5 15 4.331873e+05 9480.667 *
##           31) FT>=1892.5 10 4.941049e+05 9916.100 *
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##                    model_id model_method  feats max.nTuningRuns
## 1 Max.cor.Y.cv.0.cp.0.rpart        rpart FG, FT               0
##   min.elapsedtime.everything min.elapsedtime.final max.R.sq.fit
## 1                      0.462                 0.017    0.9556656
##   min.RMSE.fit max.R.sq.OOB min.RMSE.OOB
## 1      122.269    0.7874969     209.1754
```

```r
if (glb_is_regression || glb_is_binomial) # For multinomials this model will be run next by default
ret_lst <- myfit_mdl(model_id="Max.cor.Y", 
                        model_method="rpart",
                     model_type=glb_model_type,
                        indep_vars_vctr=max_cor_y_x_vars,
                        rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                        fit_df=glb_fitobs_df, OOB_df=glb_OOBobs_df,
                        n_cv_folds=glb_n_cv_folds, tune_models_df=NULL)
```

```
## [1] "fitting model: Max.cor.Y.rpart"
## [1] "    indep_vars: FG, FT"
## Aggregating results
## Selecting tuning parameters
## Fitting cp = 0.0674 on full training set
```

```
## Warning in myfit_mdl(model_id = "Max.cor.Y", model_method = "rpart",
## model_type = glb_model_type, : model's bestTune found at an extreme of
## tuneGrid for parameter: cp
```

![](NBA_PTS2_files/figure-html/fit.models_0-7.png) ![](NBA_PTS2_files/figure-html/fit.models_0-8.png) 

```
## Call:
## rpart(formula = .outcome ~ ., control = list(minsplit = 20, minbucket = 7, 
##     cp = 0, maxcompete = 4, maxsurrogate = 5, usesurrogate = 2, 
##     surrogatestyle = 0, maxdepth = 30, xval = 0))
##   n= 835 
## 
##           CP nsplit rel error
## 1 0.62472303      0 1.0000000
## 2 0.12111891      1 0.3752770
## 3 0.06735895      2 0.2541581
## 
## Variable importance
## FG FT 
## 76 24 
## 
## Node number 1: 835 observations,    complexity param=0.624723
##   mean=8370.24, MSE=337203.3 
##   left son=2 (527 obs) right son=3 (308 obs)
##   Primary splits:
##       FG < 3306.5 to the left,  improve=0.6247230, (0 missing)
##       FT < 1697.5 to the left,  improve=0.3517337, (0 missing)
##   Surrogate splits:
##       FT < 1778.5 to the left,  agree=0.766, adj=0.367, (0 split)
## 
## Node number 2: 527 observations,    complexity param=0.1211189
##   mean=8019.359, MSE=125200.1 
##   left son=4 (171 obs) right son=5 (356 obs)
##   Primary splits:
##       FG < 2945   to the left,  improve=0.5168624, (0 missing)
##       FT < 1482.5 to the left,  improve=0.1553086, (0 missing)
##   Surrogate splits:
##       FT < 1241.5 to the left,  agree=0.681, adj=0.018, (0 split)
## 
## Node number 3: 308 observations
##   mean=8970.61, MSE=128845.2 
## 
## Node number 4: 171 observations
##   mean=7652.316, MSE=58182.63 
## 
## Node number 5: 356 observations
##   mean=8195.663, MSE=61596.67 
## 
## n= 835 
## 
## node), split, n, deviance, yval
##       * denotes terminal node
## 
## 1) root 835 281564800 8370.240  
##   2) FG< 3306.5 527  65980460 8019.359  
##     4) FG< 2945 171   9949229 7652.316 *
##     5) FG>=2945 356  21928410 8195.663 *
##   3) FG>=3306.5 308  39684310 8970.610 *
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##          model_id model_method  feats max.nTuningRuns
## 1 Max.cor.Y.rpart        rpart FG, FT               3
##   min.elapsedtime.everything min.elapsedtime.final max.R.sq.fit
## 1                      1.314                 0.019    0.7458419
##   min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Rsquared.fit min.RMSESD.fit
## 1     279.5447    0.5731467     296.4608        0.7695845       24.70686
##   max.RsquaredSD.fit
## 1         0.03664412
```

```r
# Used to compare vs. Interactions.High.cor.Y and/or Max.cor.Y.TmSrs
ret_lst <- myfit_mdl(model_id="Max.cor.Y", 
                        model_method=ifelse(glb_is_regression, "lm", 
                                        ifelse(glb_is_binomial, "glm", "rpart")),
                     model_type=glb_model_type,
                        indep_vars_vctr=max_cor_y_x_vars,
                        rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                        fit_df=glb_fitobs_df, OOB_df=glb_OOBobs_df,
                        n_cv_folds=glb_n_cv_folds, tune_models_df=NULL)
```

```
## [1] "fitting model: Max.cor.Y.lm"
## [1] "    indep_vars: FG, FT"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_PTS2_files/figure-html/fit.models_0-9.png) ![](NBA_PTS2_files/figure-html/fit.models_0-10.png) ![](NBA_PTS2_files/figure-html/fit.models_0-11.png) ![](NBA_PTS2_files/figure-html/fit.models_0-12.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -363.06 -104.69  -24.73   89.75  515.11 
## 
## Coefficients:
##              Estimate Std. Error t value Pr(>|t|)    
## (Intercept) 1.905e+03  5.769e+01   33.02   <2e-16 ***
## FG          1.614e+00  2.067e-02   78.07   <2e-16 ***
## FT          7.881e-01  3.004e-02   26.24   <2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 144.5 on 832 degrees of freedom
## Multiple R-squared:  0.9383,	Adjusted R-squared:  0.9382 
## F-statistic:  6328 on 2 and 832 DF,  p-value: < 2.2e-16
## 
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##       model_id model_method  feats max.nTuningRuns
## 1 Max.cor.Y.lm           lm FG, FT               1
##   min.elapsedtime.everything min.elapsedtime.final max.R.sq.fit
## 1                      0.883                 0.003     0.938319
##   min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit max.Rsquared.fit
## 1     144.6379    0.7820592     211.8347        0.9381707        0.9382212
##   min.RMSESD.fit max.RsquaredSD.fit
## 1       2.248659        0.004161622
```

```r
if (!is.null(glb_date_vars) && 
    (sum(grepl(paste(glb_date_vars, "\\.day\\.minutes\\.poly\\.", sep=""),
               names(glb_allobs_df))) > 0)) {
# ret_lst <- myfit_mdl(model_id="Max.cor.Y.TmSrs.poly1", 
#                         model_method=ifelse(glb_is_regression, "lm", 
#                                         ifelse(glb_is_binomial, "glm", "rpart")),
#                      model_type=glb_model_type,
#                         indep_vars_vctr=c(max_cor_y_x_vars, paste0(glb_date_vars, ".day.minutes")),
#                         rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
#                         fit_df=glb_fitobs_df, OOB_df=glb_OOBobs_df,
#                         n_cv_folds=glb_n_cv_folds, tune_models_df=NULL)
# 
ret_lst <- myfit_mdl(model_id="Max.cor.Y.TmSrs.poly", 
                        model_method=ifelse(glb_is_regression, "lm", 
                                        ifelse(glb_is_binomial, "glm", "rpart")),
                     model_type=glb_model_type,
                        indep_vars_vctr=c(max_cor_y_x_vars, 
            grep(paste(glb_date_vars, "\\.day\\.minutes\\.poly\\.", sep=""),
                        names(glb_allobs_df), value=TRUE)),
                        rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                        fit_df=glb_fitobs_df, OOB_df=glb_OOBobs_df,
                        n_cv_folds=glb_n_cv_folds, tune_models_df=NULL)
}

# Interactions.High.cor.Y
if (length(int_feats <- setdiff(unique(glb_feats_df$cor.high.X), NA)) > 0) {
    # lm & glm handle interaction terms; rpart & rf do not
    if (glb_is_regression || glb_is_binomial) {
        indep_vars_vctr <- 
            c(max_cor_y_x_vars, paste(max_cor_y_x_vars[1], int_feats, sep=":"))            
    } else { indep_vars_vctr <- union(max_cor_y_x_vars, int_feats) }
    
    ret_lst <- myfit_mdl(model_id="Interact.High.cor.Y", 
                            model_method=ifelse(glb_is_regression, "lm", 
                                        ifelse(glb_is_binomial, "glm", "rpart")),
                         model_type=glb_model_type,
                            indep_vars_vctr,
                            glb_rsp_var, glb_rsp_var_out,
                            fit_df=glb_fitobs_df, OOB_df=glb_OOBobs_df,
                            n_cv_folds=glb_n_cv_folds, tune_models_df=NULL)                        
}    
```

```
## [1] "fitting model: Interact.High.cor.Y.lm"
## [1] "    indep_vars: FG, FT, FG:FG, FG:X2P, FG:FT, FG:X3PA"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_PTS2_files/figure-html/fit.models_0-13.png) ![](NBA_PTS2_files/figure-html/fit.models_0-14.png) ![](NBA_PTS2_files/figure-html/fit.models_0-15.png) ![](NBA_PTS2_files/figure-html/fit.models_0-16.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -73.832 -13.792  -1.604  12.438  93.712 
## 
## Coefficients:
##               Estimate Std. Error t value Pr(>|t|)    
## (Intercept)  5.498e+01  8.332e+01   0.660     0.51    
## FG           2.259e+00  4.273e-02  52.858   <2e-16 ***
## FT           5.731e-01  4.747e-02  12.074   <2e-16 ***
## `FG:X2P`    -8.455e-05  7.170e-06 -11.793   <2e-16 ***
## `FG:FT`      1.294e-04  1.442e-05   8.972   <2e-16 ***
## `FG:X3PA`    8.597e-05  2.928e-06  29.365   <2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 21.31 on 829 degrees of freedom
## Multiple R-squared:  0.9987,	Adjusted R-squared:  0.9987 
## F-statistic: 1.238e+05 on 5 and 829 DF,  p-value: < 2.2e-16
## 
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##                 model_id model_method
## 1 Interact.High.cor.Y.lm           lm
##                                   feats max.nTuningRuns
## 1 FG, FT, FG:FG, FG:X2P, FG:FT, FG:X3PA               1
##   min.elapsedtime.everything min.elapsedtime.final max.R.sq.fit
## 1                      0.863                 0.004    0.9986629
##   min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit max.Rsquared.fit
## 1     21.38581    0.9973044     23.55896        0.9986549        0.9986403
##   min.RMSESD.fit max.RsquaredSD.fit
## 1       1.195527       0.0001740573
```

```r
# Low.cor.X
# if (glb_is_classification && glb_is_binomial)
#     indep_vars_vctr <- subset(glb_feats_df, is.na(cor.high.X) & 
#                                             is.ConditionalX.y & 
#                                             (exclude.as.feat != 1))[, "id"] else
indep_vars_vctr <- subset(glb_feats_df, is.na(cor.high.X) & !myNearZV & 
                              (exclude.as.feat != 1))[, "id"]  
myadjust_interaction_feats <- function(vars_vctr) {
    for (feat in subset(glb_feats_df, !is.na(interaction.feat))$id)
        if (feat %in% vars_vctr)
            vars_vctr <- union(setdiff(vars_vctr, feat), 
                paste0(glb_feats_df[glb_feats_df$id == feat, "interaction.feat"], ":", feat))
    return(vars_vctr)
}
indep_vars_vctr <- myadjust_interaction_feats(indep_vars_vctr)
ret_lst <- myfit_mdl(model_id="Low.cor.X", 
                        model_method=ifelse(glb_is_regression, "lm", 
                                        ifelse(glb_is_binomial, "glm", "rpart")),
                        indep_vars_vctr=indep_vars_vctr,
                        model_type=glb_model_type,                     
                        glb_rsp_var, glb_rsp_var_out,
                        fit_df=glb_fitobs_df, OOB_df=glb_OOBobs_df,
                        n_cv_folds=glb_n_cv_folds, tune_models_df=NULL)
```

```
## [1] "fitting model: Low.cor.X.lm"
## [1] "    indep_vars: FG, FT, ORB, STL, TOV, BLK, DRB, .rnorm, X3PA"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_PTS2_files/figure-html/fit.models_0-17.png) ![](NBA_PTS2_files/figure-html/fit.models_0-18.png) ![](NBA_PTS2_files/figure-html/fit.models_0-19.png) ![](NBA_PTS2_files/figure-html/fit.models_0-20.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -64.216 -11.392  -0.525  10.377  90.869 
## 
## Coefficients:
##               Estimate Std. Error t value Pr(>|t|)    
## (Intercept) -35.007035  18.937235  -1.849   0.0649 .  
## FG            2.035443   0.003759 541.533   <2e-16 ***
## FT            1.002987   0.004169 240.589   <2e-16 ***
## ORB          -0.057778   0.006462  -8.941   <2e-16 ***
## STL          -0.028845   0.009211  -3.132   0.0018 ** 
## TOV          -0.006419   0.006219  -1.032   0.3023    
## BLK           0.015493   0.008979   1.725   0.0848 .  
## DRB          -0.012504   0.006290  -1.988   0.0471 *  
## .rnorm       -0.333099   0.684220  -0.487   0.6265    
## X3PA          0.380123   0.002244 169.363   <2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 19.2 on 825 degrees of freedom
## Multiple R-squared:  0.9989,	Adjusted R-squared:  0.9989 
## F-statistic: 8.479e+04 on 9 and 825 DF,  p-value: < 2.2e-16
## 
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##       model_id model_method                                         feats
## 1 Low.cor.X.lm           lm FG, FT, ORB, STL, TOV, BLK, DRB, .rnorm, X3PA
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      0.894                 0.005
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit
## 1    0.9989201     19.30509    0.9962007     27.96915        0.9989083
##   max.Rsquared.fit min.RMSESD.fit max.RsquaredSD.fit
## 1        0.9988886       1.762944       0.0002826896
```

```r
rm(ret_lst)

glb_chunks_df <- myadd_chunk(glb_chunks_df, "fit.models", major.inc=FALSE)
```

```
##         label step_major step_minor    bgn    end elapsed
## 10 fit.models          7          0 21.412 39.867  18.456
## 11 fit.models          7          1 39.868     NA      NA
```


```r
fit.models_1_chunk_df <- myadd_chunk(NULL, "fit.models_1_bgn")
```

```
##              label step_major step_minor    bgn end elapsed
## 1 fit.models_1_bgn          1          0 43.418  NA      NA
```

```r
# Options:
#   1. rpart & rf manual tuning
#   2. rf without pca (default: with pca)

#stop(here); sav_models_lst <- glb_models_lst; sav_models_df <- glb_models_df
#glb_models_lst <- sav_models_lst; glb_models_df <- sav_models_df

# All X that is not user excluded
# if (glb_is_classification && glb_is_binomial) {
#     model_id_pfx <- "Conditional.X"
# # indep_vars_vctr <- setdiff(names(glb_fitobs_df), union(glb_rsp_var, glb_exclude_vars_as_features))
#     indep_vars_vctr <- subset(glb_feats_df, is.ConditionalX.y & 
#                                             (exclude.as.feat != 1))[, "id"]
# } else {
    model_id_pfx <- "All.X"
    indep_vars_vctr <- subset(glb_feats_df, !myNearZV &
                                            (exclude.as.feat != 1))[, "id"]
# }

indep_vars_vctr <- myadjust_interaction_feats(indep_vars_vctr)

for (method in glb_models_method_vctr) {
    fit.models_1_chunk_df <- myadd_chunk(fit.models_1_chunk_df, 
                                paste0("fit.models_1_", method), major.inc=TRUE)
    if (method %in% c("rpart", "rf")) {
        # rpart:    fubar's the tree
        # rf:       skip the scenario w/ .rnorm for speed
        indep_vars_vctr <- setdiff(indep_vars_vctr, c(".rnorm"))
        model_id <- paste0(model_id_pfx, ".no.rnorm")
    } else model_id <- model_id_pfx
    
    ret_lst <- myfit_mdl(model_id=model_id, model_method=method,
                            indep_vars_vctr=indep_vars_vctr,
                            model_type=glb_model_type,
                            rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                            fit_df=glb_fitobs_df, OOB_df=glb_OOBobs_df,
                n_cv_folds=glb_n_cv_folds, tune_models_df=glb_tune_models_df)
    
    # If All.X.glm is less accurate than Low.Cor.X.glm
    #   check NA coefficients & filter appropriate terms in indep_vars_vctr
#     if (method == "glm") {
#         orig_glm <- glb_models_lst[[paste0(model_id, ".", model_method)]]$finalModel
#         orig_glm <- glb_models_lst[["All.X.glm"]]$finalModel; print(summary(orig_glm))
#           vif_orig_glm <- vif(orig_glm); print(vif_orig_glm)
#           print(vif_orig_glm[!is.na(vif_orig_glm) & (vif_orig_glm == Inf)])
#           print(which.max(vif_orig_glm))
#           print(sort(vif_orig_glm[vif_orig_glm >= 1.0e+03], decreasing=TRUE))
#           glb_fitobs_df[c(1143, 3637, 3953, 4105), c("UniqueID", "Popular", "H.P.quandary", "Headline")]
#           glb_feats_df[glb_feats_df$id %in% grep("[HSA]\\.nchrs.log", glb_feats_df$id, value=TRUE) | glb_feats_df$cor.high.X %in%    grep("[HSA]\\.nchrs.log", glb_feats_df$id, value=TRUE), ]
#           glb_feats_df[glb_feats_df$id %in% grep("[HSA]\\.npnct14.log", glb_feats_df$id, value=TRUE) | glb_feats_df$cor.high.X %in%    grep("[HSA]\\.npnct14.log", glb_feats_df$id, value=TRUE), ]
#           glb_feats_df[glb_feats_df$id %in% grep("[HSA]\\.T.scen", glb_feats_df$id, value=TRUE) | glb_feats_df$cor.high.X %in%         grep("[HSA]\\.T.scen", glb_feats_df$id, value=TRUE), ]
#           glb_feats_df[glb_feats_df$id %in% grep("[HSA]\\.P.first", glb_feats_df$id, value=TRUE) | glb_feats_df$cor.high.X %in%         grep("[HSA]\\.P.first", glb_feats_df$id, value=TRUE), ]
#           all.equal(glb_allobs_df$S.nuppr.log, glb_allobs_df$A.nuppr.log)
#           all.equal(glb_allobs_df$S.npnct19.log, glb_allobs_df$A.npnct19.log)
#           all.equal(glb_allobs_df$S.P.year.colon, glb_allobs_df$A.P.year.colon)
#           all.equal(glb_allobs_df$S.T.share, glb_allobs_df$A.T.share)
#           all.equal(glb_allobs_df$H.T.clip, glb_allobs_df$H.P.daily.clip.report)
#           cor(glb_allobs_df$S.T.herald, glb_allobs_df$S.T.tribun)
#           dsp_obs(Abstract.contains="[Dd]iar", cols=("Abstract"), all=TRUE)
#           dsp_obs(Abstract.contains="[Ss]hare", cols=("Abstract"), all=TRUE)
#           subset(glb_feats_df, cor.y.abs <= glb_feats_df[glb_feats_df$id == ".rnorm", "cor.y.abs"])
#         corxx_mtrx <- cor(data.matrix(glb_allobs_df[, setdiff(names(glb_allobs_df), myfind_chr_cols_df(glb_allobs_df))]), use="pairwise.complete.obs"); abs_corxx_mtrx <- abs(corxx_mtrx); diag(abs_corxx_mtrx) <- 0
#           which.max(abs_corxx_mtrx["S.T.tribun", ])
#           abs_corxx_mtrx["A.npnct08.log", "S.npnct08.log"]
#         step_glm <- step(orig_glm)
#     }
    # Since caret does not optimize rpart well
#     if (method == "rpart")
#         ret_lst <- myfit_mdl(model_id=paste0(model_id_pfx, ".cp.0"), model_method=method,
#                                 indep_vars_vctr=indep_vars_vctr,
#                                 model_type=glb_model_type,
#                                 rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
#                                 fit_df=glb_fitobs_df, OOB_df=glb_OOBobs_df,        
#             n_cv_folds=0, tune_models_df=data.frame(parameter="cp", min=0.0, max=0.0, by=0.1))
}
```

```
##              label step_major step_minor    bgn    end elapsed
## 1 fit.models_1_bgn          1          0 43.418 43.431   0.013
## 2  fit.models_1_lm          2          0 43.431     NA      NA
## [1] "fitting model: All.X.lm"
## [1] "    indep_vars: FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, .rnorm, X3P, X3PA"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_PTS2_files/figure-html/fit.models_1-1.png) ![](NBA_PTS2_files/figure-html/fit.models_1-2.png) ![](NBA_PTS2_files/figure-html/fit.models_1-3.png) 

```
## Warning in summary.lm(lcl_mdl): essentially perfect fit: summary may be
## unreliable
```

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##        Min         1Q     Median         3Q        Max 
## -1.391e-12 -3.310e-13 -9.200e-14  1.490e-13  7.043e-11 
## 
## Coefficients: (2 not defined because of singularities)
##               Estimate Std. Error    t value Pr(>|t|)    
## (Intercept)  2.014e-12  2.937e-12  6.860e-01   0.4930    
## FG           3.000e+00  4.509e-15  6.653e+14   <2e-16 ***
## X2P         -1.000e+00  4.754e-15 -2.104e+14   <2e-16 ***
## FGA         -2.950e-15  1.764e-15 -1.672e+00   0.0948 .  
## AST         -3.427e-16  7.313e-16 -4.690e-01   0.6395    
## X2PA         2.774e-15  1.850e-15  1.500e+00   0.1341    
## FT           1.000e+00  1.610e-15  6.213e+14   <2e-16 ***
## FTA         -1.398e-15  1.380e-15 -1.013e+00   0.3112    
## ORB          2.111e-15  1.160e-15  1.820e+00   0.0692 .  
## STL          1.208e-15  1.242e-15  9.730e-01   0.3311    
## TOV          4.421e-17  8.507e-16  5.200e-02   0.9586    
## BLK          8.687e-16  1.188e-15  7.310e-01   0.4648    
## DRB          6.777e-16  8.297e-16  8.170e-01   0.4143    
## .rnorm       1.451e-13  8.913e-14  1.628e+00   0.1039    
## X3P                 NA         NA         NA       NA    
## X3PA                NA         NA         NA       NA    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 2.493e-12 on 821 degrees of freedom
## Multiple R-squared:      1,	Adjusted R-squared:      1 
## F-statistic: 3.484e+30 on 13 and 821 DF,  p-value: < 2.2e-16
## 
## [1] "    calling mypredict_mdl for fit:"
```

```
## Warning in predict.lm(modelFit, newdata): prediction from a rank-deficient
## fit may be misleading
```

```
## Warning in predict.lm(modelFit, newdata): prediction from a rank-deficient
## fit may be misleading
```

```
## [1] "    calling mypredict_mdl for OOB:"
```

```
## Warning in predict.lm(modelFit, newdata): prediction from a rank-deficient
## fit may be misleading
```

```
## Warning in predict.lm(modelFit, newdata): prediction from a rank-deficient
## fit may be misleading
```

```
## Warning in summary.lm(mdl$finalModel): essentially perfect fit: summary may
## be unreliable
```

![](NBA_PTS2_files/figure-html/fit.models_1-4.png) 

```
##   model_id model_method
## 1 All.X.lm           lm
##                                                                          feats
## 1 FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, .rnorm, X3P, X3PA
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      0.907                 0.008
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit
## 1            1 2.777052e-12            1 2.019113e-12                1
##   max.Rsquared.fit min.RMSESD.fit max.RsquaredSD.fit
## 1                1   1.509542e-12                  0
##              label step_major step_minor    bgn    end elapsed
## 2  fit.models_1_lm          2          0 43.431 45.824   2.393
## 3 fit.models_1_glm          3          0 45.825     NA      NA
## [1] "fitting model: All.X.glm"
## [1] "    indep_vars: FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, .rnorm, X3P, X3PA"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_PTS2_files/figure-html/fit.models_1-5.png) ![](NBA_PTS2_files/figure-html/fit.models_1-6.png) ![](NBA_PTS2_files/figure-html/fit.models_1-7.png) 

```
## 
## Call:
## NULL
## 
## Deviance Residuals: 
##        Min          1Q      Median          3Q         Max  
## -5.457e-12  -2.728e-12  -1.819e-12  -9.095e-13   1.819e-12  
## 
## Coefficients: (2 not defined because of singularities)
##               Estimate Std. Error    t value Pr(>|t|)    
## (Intercept)  2.014e-12  2.762e-12  7.290e-01   0.4661    
## FG           3.000e+00  4.241e-15  7.073e+14   <2e-16 ***
## X2P         -1.000e+00  4.471e-15 -2.236e+14   <2e-16 ***
## FGA         -2.950e-15  1.659e-15 -1.778e+00   0.0757 .  
## AST         -3.427e-16  6.879e-16 -4.980e-01   0.6184    
## X2PA         2.774e-15  1.740e-15  1.594e+00   0.1112    
## FT           1.000e+00  1.514e-15  6.605e+14   <2e-16 ***
## FTA         -1.398e-15  1.298e-15 -1.077e+00   0.2816    
## ORB          2.111e-15  1.091e-15  1.935e+00   0.0534 .  
## STL          1.208e-15  1.168e-15  1.034e+00   0.3014    
## TOV          4.421e-17  8.002e-16  5.500e-02   0.9560    
## BLK          8.687e-16  1.117e-15  7.780e-01   0.4371    
## DRB          6.777e-16  7.804e-16  8.680e-01   0.3855    
## .rnorm       1.451e-13  8.383e-14  1.731e+00   0.0839 .  
## X3P                 NA         NA         NA       NA    
## X3PA                NA         NA         NA       NA    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for gaussian family taken to be 5.499089e-24)
## 
##     Null deviance: 2.8156e+08  on 834  degrees of freedom
## Residual deviance: 4.5148e-21  on 821  degrees of freedom
## AIC: -42335
## 
## Number of Fisher Scoring iterations: 1
## 
## [1] "    calling mypredict_mdl for fit:"
```

```
## Warning in predict.lm(object, newdata, se.fit, scale = 1, type =
## ifelse(type == : prediction from a rank-deficient fit may be misleading
```

```
## Warning in predict.lm(object, newdata, se.fit, scale = 1, type =
## ifelse(type == : prediction from a rank-deficient fit may be misleading
```

```
## [1] "    calling mypredict_mdl for OOB:"
```

```
## Warning in predict.lm(object, newdata, se.fit, scale = 1, type =
## ifelse(type == : prediction from a rank-deficient fit may be misleading
```

```
## Warning in predict.lm(object, newdata, se.fit, scale = 1, type =
## ifelse(type == : prediction from a rank-deficient fit may be misleading
```

```
##    model_id model_method
## 1 All.X.glm          glm
##                                                                          feats
## 1 FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, .rnorm, X3P, X3PA
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      0.994                 0.048
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB min.aic.fit
## 1            1 2.777052e-12            1 2.019113e-12   -42334.97
##   max.Rsquared.fit min.RMSESD.fit max.RsquaredSD.fit
## 1                1   1.509542e-12                  0
##                   label step_major step_minor    bgn   end elapsed
## 3      fit.models_1_glm          3          0 45.825 48.46   2.635
## 4 fit.models_1_bayesglm          4          0 48.460    NA      NA
## [1] "fitting model: All.X.bayesglm"
## [1] "    indep_vars: FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, .rnorm, X3P, X3PA"
```

```
## Loading required package: arm
## Loading required package: MASS
## 
## Attaching package: 'MASS'
## 
## The following object is masked from 'package:dplyr':
## 
##     select
## 
## Loading required package: Matrix
## Loading required package: lme4
## Loading required package: Rcpp
## 
## arm (Version 1.8-5, built: 2015-05-13)
## 
## Working directory is /Users/bbalaji-2012/Documents/Work/Courses/MIT/Analytics_Edge_15_071x/Recitations/Unit2_NBA
```

```
## Aggregating results
## Fitting final model on full training set
```

```
## Warning in bayesglm.fit(x = X, y = Y, weights = weights, start = start, :
## non-finite coefficients at iteration 2
```

```
## Warning: algorithm did not converge
```

```
## 
## Call:
## NULL
## 
## Deviance Residuals: 
##        Min          1Q      Median          3Q         Max  
## -1.845e-04  -3.474e-05   2.941e-06   3.638e-05   2.358e-04  
## 
## Coefficients: (2 not defined because of singularities)
##               Estimate Std. Error    t value Pr(>|t|)    
## (Intercept) -2.085e-04  6.825e-05 -3.055e+00  0.00232 ** 
## FG           1.611e+00  1.048e-07  1.537e+07  < 2e-16 ***
## X2P          3.892e-01  1.105e-07  3.523e+06  < 2e-16 ***
## FGA          6.935e-08  4.098e-08  1.692e+00  0.09099 .  
## AST          1.650e-07  1.699e-08  9.709e+00  < 2e-16 ***
## X2PA         1.617e-08  4.299e-08  3.760e-01  0.70697    
## FT           1.000e+00  3.740e-08  2.674e+07  < 2e-16 ***
## FTA          1.917e-07  3.207e-08  5.979e+00 3.34e-09 ***
## ORB         -1.947e-07  2.696e-08 -7.224e+00 1.16e-12 ***
## STL         -9.192e-08  2.885e-08 -3.186e+00  0.00150 ** 
## TOV         -3.221e-08  1.977e-08 -1.629e+00  0.10366    
## BLK         -2.366e-09  2.760e-08 -8.600e-02  0.93171    
## DRB         -8.660e-09  1.928e-08 -4.490e-01  0.65343    
## .rnorm      -9.502e-07  2.071e-06 -4.590e-01  0.64651    
## X3P                 NA         NA         NA       NA    
## X3PA                NA         NA         NA       NA    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for gaussian family taken to be 3.356414e-09)
## 
##     Null deviance: 2.8156e+08  on 834  degrees of freedom
## Residual deviance: 2.7556e-06  on 821  degrees of freedom
## AIC: -13907
## 
## Number of Fisher Scoring iterations: 2
## 
## [1] "    calling mypredict_mdl for fit:"
```

```
## Warning in predictLM(object, newdata, se.fit, scale = 1, type = ifelse(type
## == : prediction from a rank-deficient fit may be misleading
```

```
## [1] "    calling mypredict_mdl for OOB:"
```

```
## Warning in predictLM(object, newdata, se.fit, scale = 1, type = ifelse(type
## == : prediction from a rank-deficient fit may be misleading
```

![](NBA_PTS2_files/figure-html/fit.models_1-8.png) 

```
##         model_id model_method
## 1 All.X.bayesglm     bayesglm
##                                                                          feats
## 1 FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, .rnorm, X3P, X3PA
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      1.543                 0.058
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB min.aic.fit
## 1    0.4878483     522.7615    -4.431981     836.9207   -13907.34
##   max.Rsquared.fit min.RMSESD.fit max.RsquaredSD.fit
## 1        0.8978656       15.20898        0.007042963
##                   label step_major step_minor    bgn    end elapsed
## 4 fit.models_1_bayesglm          4          0 48.460 50.917   2.458
## 5    fit.models_1_rpart          5          0 50.918     NA      NA
## [1] "fitting model: All.X.no.rnorm.rpart"
## [1] "    indep_vars: FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, X3P, X3PA"
## Aggregating results
## Selecting tuning parameters
## Fitting cp = 0.0674 on full training set
```

```
## Warning in myfit_mdl(model_id = model_id, model_method = method,
## indep_vars_vctr = indep_vars_vctr, : model's bestTune found at an extreme
## of tuneGrid for parameter: cp
```

![](NBA_PTS2_files/figure-html/fit.models_1-9.png) 

```
## Call:
## rpart(formula = .outcome ~ ., control = list(minsplit = 20, minbucket = 7, 
##     cp = 0, maxcompete = 4, maxsurrogate = 5, usesurrogate = 2, 
##     surrogatestyle = 0, maxdepth = 30, xval = 0))
##   n= 835 
## 
##           CP nsplit rel error
## 1 0.62472303      0 1.0000000
## 2 0.12111891      1 0.3752770
## 3 0.06735895      2 0.2541581
## 
## Variable importance
##   FG  X2P X2PA  FGA X3PA  X3P  AST 
##   24   18   16   15   13   13    1 
## 
## Node number 1: 835 observations,    complexity param=0.624723
##   mean=8370.24, MSE=337203.3 
##   left son=2 (527 obs) right son=3 (308 obs)
##   Primary splits:
##       FG   < 3306.5 to the left,  improve=0.6247230, (0 missing)
##       X2P  < 3073   to the left,  improve=0.5440932, (0 missing)
##       FGA  < 6898.5 to the left,  improve=0.5017671, (0 missing)
##       X2PA < 6133.5 to the left,  improve=0.4870227, (0 missing)
##       AST  < 1967.5 to the left,  improve=0.4627793, (0 missing)
##   Surrogate splits:
##       X2P  < 3073   to the left,  agree=0.949, adj=0.860, (0 split)
##       X2PA < 6315.5 to the left,  agree=0.916, adj=0.773, (0 split)
##       FGA  < 7071.5 to the left,  agree=0.895, adj=0.714, (0 split)
##       X3PA < 678.5  to the right, agree=0.881, adj=0.679, (0 split)
##       X3P  < 239.5  to the right, agree=0.875, adj=0.662, (0 split)
## 
## Node number 2: 527 observations,    complexity param=0.1211189
##   mean=8019.359, MSE=125200.1 
##   left son=4 (171 obs) right son=5 (356 obs)
##   Primary splits:
##       FG  < 2945   to the left,  improve=0.5168624, (0 missing)
##       FGA < 6585.5 to the left,  improve=0.2074573, (0 missing)
##       X2P < 2679   to the left,  improve=0.1710339, (0 missing)
##       AST < 1788.5 to the left,  improve=0.1679835, (0 missing)
##       FT  < 1482.5 to the left,  improve=0.1553086, (0 missing)
##   Surrogate splits:
##       FGA  < 6456.5 to the left,  agree=0.782, adj=0.327, (0 split)
##       X2P  < 2439.5 to the left,  agree=0.776, adj=0.310, (0 split)
##       AST  < 1649.5 to the left,  agree=0.729, adj=0.164, (0 split)
##       X2PA < 4803   to the left,  agree=0.700, adj=0.076, (0 split)
##       FT   < 1241.5 to the left,  agree=0.681, adj=0.018, (0 split)
## 
## Node number 3: 308 observations
##   mean=8970.61, MSE=128845.2 
## 
## Node number 4: 171 observations
##   mean=7652.316, MSE=58182.63 
## 
## Node number 5: 356 observations
##   mean=8195.663, MSE=61596.67 
## 
## n= 835 
## 
## node), split, n, deviance, yval
##       * denotes terminal node
## 
## 1) root 835 281564800 8370.240  
##   2) FG< 3306.5 527  65980460 8019.359  
##     4) FG< 2945 171   9949229 7652.316 *
##     5) FG>=2945 356  21928410 8195.663 *
##   3) FG>=3306.5 308  39684310 8970.610 *
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##               model_id model_method
## 1 All.X.no.rnorm.rpart        rpart
##                                                                  feats
## 1 FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, X3P, X3PA
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               3                       1.07                 0.051
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Rsquared.fit
## 1    0.7458419     279.5447    0.5731467     296.4608        0.7695845
##   min.RMSESD.fit max.RsquaredSD.fit
## 1       24.70686         0.03664412
##                label step_major step_minor    bgn   end elapsed
## 5 fit.models_1_rpart          5          0 50.918 53.93   3.012
## 6    fit.models_1_rf          6          0 53.930    NA      NA
## [1] "fitting model: All.X.no.rnorm.rf"
## [1] "    indep_vars: FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, X3P, X3PA"
```

```
## Loading required package: randomForest
## randomForest 4.6-10
## Type rfNews() to see new features/changes/bug fixes.
## 
## Attaching package: 'randomForest'
## 
## The following object is masked from 'package:dplyr':
## 
##     combine
```

![](NBA_PTS2_files/figure-html/fit.models_1-10.png) 

```
## Aggregating results
## Selecting tuning parameters
## Fitting mtry = 14 on full training set
```

```
## Warning in myfit_mdl(model_id = model_id, model_method = method,
## indep_vars_vctr = indep_vars_vctr, : model's bestTune found at an extreme
## of tuneGrid for parameter: mtry
```

![](NBA_PTS2_files/figure-html/fit.models_1-11.png) ![](NBA_PTS2_files/figure-html/fit.models_1-12.png) 

```
##                 Length Class      Mode     
## call              4    -none-     call     
## type              1    -none-     character
## predicted       835    -none-     numeric  
## mse             500    -none-     numeric  
## rsq             500    -none-     numeric  
## oob.times       835    -none-     numeric  
## importance       14    -none-     numeric  
## importanceSD      0    -none-     NULL     
## localImportance   0    -none-     NULL     
## proximity         0    -none-     NULL     
## ntree             1    -none-     numeric  
## mtry              1    -none-     numeric  
## forest           11    -none-     list     
## coefs             0    -none-     NULL     
## y               835    -none-     numeric  
## test              0    -none-     NULL     
## inbag             0    -none-     NULL     
## xNames           14    -none-     character
## problemType       1    -none-     character
## tuneValue         1    data.frame list     
## obsLevels         1    -none-     logical  
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##            model_id model_method
## 1 All.X.no.rnorm.rf           rf
##                                                                  feats
## 1 FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, X3P, X3PA
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               3                     12.866                  4.84
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Rsquared.fit
## 1    0.9967355     96.64743    0.8875518     151.9081        0.9743395
##   min.RMSESD.fit max.RsquaredSD.fit
## 1       3.998567         0.00366486
```

```r
# User specified
#   Ensure at least 2 vars in each regression; else varImp crashes
# sav_models_lst <- glb_models_lst; sav_models_df <- glb_models_df; sav_featsimp_df <- glb_featsimp_df
# glb_models_lst <- sav_models_lst; glb_models_df <- sav_models_df; glm_featsimp_df <- sav_featsimp_df

    # easier to exclude features
#model_id_pfx <- "";
# indep_vars_vctr <- setdiff(names(glb_fitobs_df), 
#                         union(union(glb_rsp_var, glb_exclude_vars_as_features), 
#                                 c("<feat1_name>", "<feat2_name>")))
# method <- ""                                

    # easier to include features
# model_id <- "PTS.only"; indep_vars_vctr <- c(NULL
#    ,"PTS", "oppPTS"
# #    ,"<feat1>*<feat2>"
# #    ,"<feat1>:<feat2>"
#                                            )
# for (method in c("lm")) {
#     ret_lst <- myfit_mdl(model_id=model_id, model_method=method,
#                                 indep_vars_vctr=indep_vars_vctr,
#                                 model_type=glb_model_type,
#                                 rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
#                                 fit_df=glb_fitobs_df, OOB_df=glb_OOBobs_df,
#                     n_cv_folds=glb_n_cv_folds, tune_models_df=glb_tune_models_df)
#     csm_mdl_id <- paste0(model_id, ".", method)
#     csm_featsimp_df <- myget_feats_importance(glb_models_lst[[paste0(model_id, ".", method)]]);         print(head(csm_featsimp_df))
# }

#print(dsp_models_df <- orderBy(model_sel_frmla, glb_models_df)[, dsp_models_cols])
#csm_featsimp_df[grepl("H.npnct19.log", row.names(csm_featsimp_df)), , FALSE]
#csm_OOBobs_df <- glb_get_predictions(glb_OOBobs_df, mdl_id=csm_mdl_id, rsp_var_out=glb_rsp_var_out, prob_threshold_def=glb_models_df[glb_models_df$model_id == csm_mdl_id, "opt.prob.threshold.OOB"])
#print(sprintf("%s OOB confusion matrix & accuracy: ", csm_mdl_id)); print(t(confusionMatrix(csm_OOBobs_df[, paste0(glb_rsp_var_out, csm_mdl_id)], csm_OOBobs_df[, glb_rsp_var])$table))

#glb_models_df[, "max.Accuracy.OOB", FALSE]
#varImp(glb_models_lst[["Low.cor.X.glm"]])
#orderBy(~ -Overall, varImp(glb_models_lst[["All.X.2.glm"]])$importance)
#orderBy(~ -Overall, varImp(glb_models_lst[["All.X.3.glm"]])$importance)
#glb_feats_df[grepl("npnct28", glb_feats_df$id), ]
#print(sprintf("%s OOB confusion matrix & accuracy: ", glb_sel_mdl_id)); print(t(confusionMatrix(glb_OOBobs_df[, paste0(glb_rsp_var_out, glb_sel_mdl_id)], glb_OOBobs_df[, glb_rsp_var])$table))

    # User specified bivariate models
#     indep_vars_vctr_lst <- list()
#     for (feat in setdiff(names(glb_fitobs_df), 
#                          union(glb_rsp_var, glb_exclude_vars_as_features)))
#         indep_vars_vctr_lst[["feat"]] <- feat

    # User specified combinatorial models
#     indep_vars_vctr_lst <- list()
#     combn_mtrx <- combn(c("<feat1_name>", "<feat2_name>", "<featn_name>"), 
#                           <num_feats_to_choose>)
#     for (combn_ix in 1:ncol(combn_mtrx))
#         #print(combn_mtrx[, combn_ix])
#         indep_vars_vctr_lst[[combn_ix]] <- combn_mtrx[, combn_ix]
    
    # template for myfit_mdl
    #   rf is hard-coded in caret to recognize only Accuracy / Kappa evaluation metrics
    #       only for OOB in trainControl ?
    
#     ret_lst <- myfit_mdl_fn(model_id=paste0(model_id_pfx, ""), model_method=method,
#                             indep_vars_vctr=indep_vars_vctr,
#                             rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
#                             fit_df=glb_fitobs_df, OOB_df=glb_OOBobs_df,
#                             n_cv_folds=glb_n_cv_folds, tune_models_df=glb_tune_models_df,
#                             model_loss_mtrx=glb_model_metric_terms,
#                             model_summaryFunction=glb_model_metric_smmry,
#                             model_metric=glb_model_metric,
#                             model_metric_maximize=glb_model_metric_maximize)

# Simplify a model
# fit_df <- glb_fitobs_df; glb_mdl <- step(<complex>_mdl)

# Non-caret models
#     rpart_area_mdl <- rpart(reformulate("Area", response=glb_rsp_var), 
#                                data=glb_fitobs_df, #method="class", 
#                                control=rpart.control(cp=0.12),
#                            parms=list(loss=glb_model_metric_terms))
#     print("rpart_sel_wlm_mdl"); prp(rpart_sel_wlm_mdl)
# 

print(glb_models_df)
```

```
##                                            model_id model_method
## MFO.lm                                       MFO.lm           lm
## Max.cor.Y.cv.0.rpart           Max.cor.Y.cv.0.rpart        rpart
## Max.cor.Y.cv.0.cp.0.rpart Max.cor.Y.cv.0.cp.0.rpart        rpart
## Max.cor.Y.rpart                     Max.cor.Y.rpart        rpart
## Max.cor.Y.lm                           Max.cor.Y.lm           lm
## Interact.High.cor.Y.lm       Interact.High.cor.Y.lm           lm
## Low.cor.X.lm                           Low.cor.X.lm           lm
## All.X.lm                                   All.X.lm           lm
## All.X.glm                                 All.X.glm          glm
## All.X.bayesglm                       All.X.bayesglm     bayesglm
## All.X.no.rnorm.rpart           All.X.no.rnorm.rpart        rpart
## All.X.no.rnorm.rf                 All.X.no.rnorm.rf           rf
##                                                                                                  feats
## MFO.lm                                                                                          .rnorm
## Max.cor.Y.cv.0.rpart                                                                            FG, FT
## Max.cor.Y.cv.0.cp.0.rpart                                                                       FG, FT
## Max.cor.Y.rpart                                                                                 FG, FT
## Max.cor.Y.lm                                                                                    FG, FT
## Interact.High.cor.Y.lm                                           FG, FT, FG:FG, FG:X2P, FG:FT, FG:X3PA
## Low.cor.X.lm                                             FG, FT, ORB, STL, TOV, BLK, DRB, .rnorm, X3PA
## All.X.lm                  FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, .rnorm, X3P, X3PA
## All.X.glm                 FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, .rnorm, X3P, X3PA
## All.X.bayesglm            FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, .rnorm, X3P, X3PA
## All.X.no.rnorm.rpart              FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, X3P, X3PA
## All.X.no.rnorm.rf                 FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, X3P, X3PA
##                           max.nTuningRuns min.elapsedtime.everything
## MFO.lm                                  0                      0.682
## Max.cor.Y.cv.0.rpart                    0                      0.621
## Max.cor.Y.cv.0.cp.0.rpart               0                      0.462
## Max.cor.Y.rpart                         3                      1.314
## Max.cor.Y.lm                            1                      0.883
## Interact.High.cor.Y.lm                  1                      0.863
## Low.cor.X.lm                            1                      0.894
## All.X.lm                                1                      0.907
## All.X.glm                               1                      0.994
## All.X.bayesglm                          1                      1.543
## All.X.no.rnorm.rpart                    3                      1.070
## All.X.no.rnorm.rf                       3                     12.866
##                           min.elapsedtime.final max.R.sq.fit min.RMSE.fit
## MFO.lm                                    0.003 9.268854e-05 5.806652e+02
## Max.cor.Y.cv.0.rpart                      0.019 0.000000e+00 5.806921e+02
## Max.cor.Y.cv.0.cp.0.rpart                 0.017 9.556656e-01 1.222690e+02
## Max.cor.Y.rpart                           0.019 7.458419e-01 2.795447e+02
## Max.cor.Y.lm                              0.003 9.383190e-01 1.446379e+02
## Interact.High.cor.Y.lm                    0.004 9.986629e-01 2.138581e+01
## Low.cor.X.lm                              0.005 9.989201e-01 1.930509e+01
## All.X.lm                                  0.008 1.000000e+00 2.777052e-12
## All.X.glm                                 0.048 1.000000e+00 2.777052e-12
## All.X.bayesglm                            0.058 4.878483e-01 5.227615e+02
## All.X.no.rnorm.rpart                      0.051 7.458419e-01 2.795447e+02
## All.X.no.rnorm.rf                         4.840 9.967355e-01 9.664743e+01
##                            max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit
## MFO.lm                     0.0003645831 4.536790e+02      -0.00110768
## Max.cor.Y.cv.0.rpart       0.0000000000 4.537617e+02               NA
## Max.cor.Y.cv.0.cp.0.rpart  0.7874968789 2.091754e+02               NA
## Max.cor.Y.rpart            0.5731466783 2.964608e+02               NA
## Max.cor.Y.lm               0.7820591878 2.118347e+02       0.93817073
## Interact.High.cor.Y.lm     0.9973043928 2.355896e+01       0.99865487
## Low.cor.X.lm               0.9962007077 2.796915e+01       0.99890827
## All.X.lm                   1.0000000000 2.019113e-12       1.00000000
## All.X.glm                  1.0000000000 2.019113e-12               NA
## All.X.bayesglm            -4.4319811910 8.369207e+02               NA
## All.X.no.rnorm.rpart       0.5731466783 2.964608e+02               NA
## All.X.no.rnorm.rf          0.8875517657 1.519081e+02               NA
##                           max.Rsquared.fit min.RMSESD.fit
## MFO.lm                                  NA             NA
## Max.cor.Y.cv.0.rpart                    NA             NA
## Max.cor.Y.cv.0.cp.0.rpart               NA             NA
## Max.cor.Y.rpart                  0.7695845   2.470686e+01
## Max.cor.Y.lm                     0.9382212   2.248659e+00
## Interact.High.cor.Y.lm           0.9986403   1.195527e+00
## Low.cor.X.lm                     0.9988886   1.762944e+00
## All.X.lm                         1.0000000   1.509542e-12
## All.X.glm                        1.0000000   1.509542e-12
## All.X.bayesglm                   0.8978656   1.520898e+01
## All.X.no.rnorm.rpart             0.7695845   2.470686e+01
## All.X.no.rnorm.rf                0.9743395   3.998567e+00
##                           max.RsquaredSD.fit min.aic.fit
## MFO.lm                                    NA          NA
## Max.cor.Y.cv.0.rpart                      NA          NA
## Max.cor.Y.cv.0.cp.0.rpart                 NA          NA
## Max.cor.Y.rpart                 0.0366441201          NA
## Max.cor.Y.lm                    0.0041616217          NA
## Interact.High.cor.Y.lm          0.0001740573          NA
## Low.cor.X.lm                    0.0002826896          NA
## All.X.lm                        0.0000000000          NA
## All.X.glm                       0.0000000000   -42334.97
## All.X.bayesglm                  0.0070429632   -13907.34
## All.X.no.rnorm.rpart            0.0366441201          NA
## All.X.no.rnorm.rf               0.0036648598          NA
```

```r
rm(ret_lst)
fit.models_1_chunk_df <- myadd_chunk(fit.models_1_chunk_df, "fit.models_1_end", 
                                     major.inc=TRUE)
```

```
##              label step_major step_minor    bgn    end elapsed
## 6  fit.models_1_rf          6          0 53.930 68.796  14.866
## 7 fit.models_1_end          7          0 68.796     NA      NA
```

```r
glb_chunks_df <- myadd_chunk(glb_chunks_df, "fit.models", major.inc=FALSE)
```

```
##         label step_major step_minor    bgn    end elapsed
## 11 fit.models          7          1 39.868 68.801  28.934
## 12 fit.models          7          2 68.802     NA      NA
```


```r
if (!is.null(glb_model_metric_smmry)) {
    stats_df <- glb_models_df[, "model_id", FALSE]

    stats_mdl_df <- data.frame()
    for (model_id in stats_df$model_id) {
        stats_mdl_df <- rbind(stats_mdl_df, 
            mypredict_mdl(glb_models_lst[[model_id]], glb_fitobs_df, glb_rsp_var, 
                          glb_rsp_var_out, model_id, "fit",
        						glb_model_metric_smmry, glb_model_metric, 
        						glb_model_metric_maximize, ret_type="stats"))
    }
    stats_df <- merge(stats_df, stats_mdl_df, all.x=TRUE)
    
    stats_mdl_df <- data.frame()
    for (model_id in stats_df$model_id) {
        stats_mdl_df <- rbind(stats_mdl_df, 
            mypredict_mdl(glb_models_lst[[model_id]], glb_OOBobs_df, glb_rsp_var, 
                          glb_rsp_var_out, model_id, "OOB",
            					glb_model_metric_smmry, glb_model_metric, 
        						glb_model_metric_maximize, ret_type="stats"))
    }
    stats_df <- merge(stats_df, stats_mdl_df, all.x=TRUE)
    
    print("Merging following data into glb_models_df:")
    print(stats_mrg_df <- stats_df[, c(1, grep(glb_model_metric, names(stats_df)))])
    print(tmp_models_df <- orderBy(~model_id, glb_models_df[, c("model_id",
                                    grep(glb_model_metric, names(stats_df), value=TRUE))]))

    tmp2_models_df <- glb_models_df[, c("model_id", setdiff(names(glb_models_df),
                                    grep(glb_model_metric, names(stats_df), value=TRUE)))]
    tmp3_models_df <- merge(tmp2_models_df, stats_mrg_df, all.x=TRUE, sort=FALSE)
    print(tmp3_models_df)
    print(names(tmp3_models_df))
    print(glb_models_df <- subset(tmp3_models_df, select=-model_id.1))
}

plt_models_df <- glb_models_df[, -grep("SD|Upper|Lower", names(glb_models_df))]
for (var in grep("^min.", names(plt_models_df), value=TRUE)) {
    plt_models_df[, sub("min.", "inv.", var)] <- 
        #ifelse(all(is.na(tmp <- plt_models_df[, var])), NA, 1.0 / tmp)
        1.0 / plt_models_df[, var]
    plt_models_df <- plt_models_df[ , -grep(var, names(plt_models_df))]
}
print(plt_models_df)
```

```
##                                            model_id model_method
## MFO.lm                                       MFO.lm           lm
## Max.cor.Y.cv.0.rpart           Max.cor.Y.cv.0.rpart        rpart
## Max.cor.Y.cv.0.cp.0.rpart Max.cor.Y.cv.0.cp.0.rpart        rpart
## Max.cor.Y.rpart                     Max.cor.Y.rpart        rpart
## Max.cor.Y.lm                           Max.cor.Y.lm           lm
## Interact.High.cor.Y.lm       Interact.High.cor.Y.lm           lm
## Low.cor.X.lm                           Low.cor.X.lm           lm
## All.X.lm                                   All.X.lm           lm
## All.X.glm                                 All.X.glm          glm
## All.X.bayesglm                       All.X.bayesglm     bayesglm
## All.X.no.rnorm.rpart           All.X.no.rnorm.rpart        rpart
## All.X.no.rnorm.rf                 All.X.no.rnorm.rf           rf
##                                                                                                  feats
## MFO.lm                                                                                          .rnorm
## Max.cor.Y.cv.0.rpart                                                                            FG, FT
## Max.cor.Y.cv.0.cp.0.rpart                                                                       FG, FT
## Max.cor.Y.rpart                                                                                 FG, FT
## Max.cor.Y.lm                                                                                    FG, FT
## Interact.High.cor.Y.lm                                           FG, FT, FG:FG, FG:X2P, FG:FT, FG:X3PA
## Low.cor.X.lm                                             FG, FT, ORB, STL, TOV, BLK, DRB, .rnorm, X3PA
## All.X.lm                  FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, .rnorm, X3P, X3PA
## All.X.glm                 FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, .rnorm, X3P, X3PA
## All.X.bayesglm            FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, .rnorm, X3P, X3PA
## All.X.no.rnorm.rpart              FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, X3P, X3PA
## All.X.no.rnorm.rf                 FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, X3P, X3PA
##                           max.nTuningRuns max.R.sq.fit  max.R.sq.OOB
## MFO.lm                                  0 9.268854e-05  0.0003645831
## Max.cor.Y.cv.0.rpart                    0 0.000000e+00  0.0000000000
## Max.cor.Y.cv.0.cp.0.rpart               0 9.556656e-01  0.7874968789
## Max.cor.Y.rpart                         3 7.458419e-01  0.5731466783
## Max.cor.Y.lm                            1 9.383190e-01  0.7820591878
## Interact.High.cor.Y.lm                  1 9.986629e-01  0.9973043928
## Low.cor.X.lm                            1 9.989201e-01  0.9962007077
## All.X.lm                                1 1.000000e+00  1.0000000000
## All.X.glm                               1 1.000000e+00  1.0000000000
## All.X.bayesglm                          1 4.878483e-01 -4.4319811910
## All.X.no.rnorm.rpart                    3 7.458419e-01  0.5731466783
## All.X.no.rnorm.rf                       3 9.967355e-01  0.8875517657
##                           max.Adj.R.sq.fit max.Rsquared.fit
## MFO.lm                         -0.00110768               NA
## Max.cor.Y.cv.0.rpart                    NA               NA
## Max.cor.Y.cv.0.cp.0.rpart               NA               NA
## Max.cor.Y.rpart                         NA        0.7695845
## Max.cor.Y.lm                    0.93817073        0.9382212
## Interact.High.cor.Y.lm          0.99865487        0.9986403
## Low.cor.X.lm                    0.99890827        0.9988886
## All.X.lm                        1.00000000        1.0000000
## All.X.glm                               NA        1.0000000
## All.X.bayesglm                          NA        0.8978656
## All.X.no.rnorm.rpart                    NA        0.7695845
## All.X.no.rnorm.rf                       NA        0.9743395
##                           inv.elapsedtime.everything inv.elapsedtime.final
## MFO.lm                                    1.46627566           333.3333333
## Max.cor.Y.cv.0.rpart                      1.61030596            52.6315789
## Max.cor.Y.cv.0.cp.0.rpart                 2.16450216            58.8235294
## Max.cor.Y.rpart                           0.76103501            52.6315789
## Max.cor.Y.lm                              1.13250283           333.3333333
## Interact.High.cor.Y.lm                    1.15874855           250.0000000
## Low.cor.X.lm                              1.11856823           200.0000000
## All.X.lm                                  1.10253583           125.0000000
## All.X.glm                                 1.00603622            20.8333333
## All.X.bayesglm                            0.64808814            17.2413793
## All.X.no.rnorm.rpart                      0.93457944            19.6078431
## All.X.no.rnorm.rf                         0.07772423             0.2066116
##                           inv.RMSE.fit inv.RMSE.OOB   inv.aic.fit
## MFO.lm                    1.722163e-03 2.204202e-03            NA
## Max.cor.Y.cv.0.rpart      1.722083e-03 2.203800e-03            NA
## Max.cor.Y.cv.0.cp.0.rpart 8.178691e-03 4.780678e-03            NA
## Max.cor.Y.rpart           3.577245e-03 3.373128e-03            NA
## Max.cor.Y.lm              6.913816e-03 4.720661e-03            NA
## Interact.High.cor.Y.lm    4.675997e-02 4.244669e-02            NA
## Low.cor.X.lm              5.179980e-02 3.575368e-02            NA
## All.X.lm                  3.600940e+11 4.952669e+11            NA
## All.X.glm                 3.600940e+11 4.952669e+11 -2.362113e-05
## All.X.bayesglm            1.912918e-03 1.194856e-03 -7.190448e-05
## All.X.no.rnorm.rpart      3.577245e-03 3.373128e-03            NA
## All.X.no.rnorm.rf         1.034689e-02 6.582926e-03            NA
```

```r
print(myplot_radar(radar_inp_df=plt_models_df))
```

```
## Warning in RColorBrewer::brewer.pal(n, pal): n too large, allowed maximum for palette Set1 is 9
## Returning the palette you asked for with that many colors
```

```
## Warning: The shape palette can deal with a maximum of 6 discrete values
## because more than 6 becomes difficult to discriminate; you have
## 12. Consider specifying shapes manually. if you must have them.
```

```
## Warning in loop_apply(n, do.ply): Removed 7 rows containing missing values
## (geom_path).
```

```
## Warning in loop_apply(n, do.ply): Removed 68 rows containing missing values
## (geom_point).
```

```
## Warning in loop_apply(n, do.ply): Removed 20 rows containing missing values
## (geom_text).
```

```
## Warning in RColorBrewer::brewer.pal(n, pal): n too large, allowed maximum for palette Set1 is 9
## Returning the palette you asked for with that many colors
```

```
## Warning: The shape palette can deal with a maximum of 6 discrete values
## because more than 6 becomes difficult to discriminate; you have
## 12. Consider specifying shapes manually. if you must have them.
```

![](NBA_PTS2_files/figure-html/fit.models_2-1.png) 

```r
# print(myplot_radar(radar_inp_df=subset(plt_models_df, 
#         !(model_id %in% grep("random|MFO", plt_models_df$model_id, value=TRUE)))))

# Compute CI for <metric>SD
glb_models_df <- mutate(glb_models_df, 
                max.df = ifelse(max.nTuningRuns > 1, max.nTuningRuns - 1, NA),
                min.sd2ci.scaler = ifelse(is.na(max.df), NA, qt(0.975, max.df)))
for (var in grep("SD", names(glb_models_df), value=TRUE)) {
    # Does CI alredy exist ?
    var_components <- unlist(strsplit(var, "SD"))
    varActul <- paste0(var_components[1],          var_components[2])
    varUpper <- paste0(var_components[1], "Upper", var_components[2])
    varLower <- paste0(var_components[1], "Lower", var_components[2])
    if (varUpper %in% names(glb_models_df)) {
        warning(varUpper, " already exists in glb_models_df")
        # Assuming Lower also exists
        next
    }    
    print(sprintf("var:%s", var))
    # CI is dependent on sample size in t distribution; df=n-1
    glb_models_df[, varUpper] <- glb_models_df[, varActul] + 
        glb_models_df[, "min.sd2ci.scaler"] * glb_models_df[, var]
    glb_models_df[, varLower] <- glb_models_df[, varActul] - 
        glb_models_df[, "min.sd2ci.scaler"] * glb_models_df[, var]
}
```

```
## [1] "var:min.RMSESD.fit"
## [1] "var:max.RsquaredSD.fit"
```

```r
# Plot metrics with CI
plt_models_df <- glb_models_df[, "model_id", FALSE]
pltCI_models_df <- glb_models_df[, "model_id", FALSE]
for (var in grep("Upper", names(glb_models_df), value=TRUE)) {
    var_components <- unlist(strsplit(var, "Upper"))
    col_name <- unlist(paste(var_components, collapse=""))
    plt_models_df[, col_name] <- glb_models_df[, col_name]
    for (name in paste0(var_components[1], c("Upper", "Lower"), var_components[2]))
        pltCI_models_df[, name] <- glb_models_df[, name]
}

build_statsCI_data <- function(plt_models_df) {
    mltd_models_df <- melt(plt_models_df, id.vars="model_id")
    mltd_models_df$data <- sapply(1:nrow(mltd_models_df), 
        function(row_ix) tail(unlist(strsplit(as.character(
            mltd_models_df[row_ix, "variable"]), "[.]")), 1))
    mltd_models_df$label <- sapply(1:nrow(mltd_models_df), 
        function(row_ix) head(unlist(strsplit(as.character(
            mltd_models_df[row_ix, "variable"]), 
            paste0(".", mltd_models_df[row_ix, "data"]))), 1))
    #print(mltd_models_df)
    
    return(mltd_models_df)
}
mltd_models_df <- build_statsCI_data(plt_models_df)

mltdCI_models_df <- melt(pltCI_models_df, id.vars="model_id")
for (row_ix in 1:nrow(mltdCI_models_df)) {
    for (type in c("Upper", "Lower")) {
        if (length(var_components <- unlist(strsplit(
                as.character(mltdCI_models_df[row_ix, "variable"]), type))) > 1) {
            #print(sprintf("row_ix:%d; type:%s; ", row_ix, type))
            mltdCI_models_df[row_ix, "label"] <- var_components[1]
            mltdCI_models_df[row_ix, "data"] <- 
                unlist(strsplit(var_components[2], "[.]"))[2]
            mltdCI_models_df[row_ix, "type"] <- type
            break
        }
    }    
}
wideCI_models_df <- reshape(subset(mltdCI_models_df, select=-variable), 
                            timevar="type", 
        idvar=setdiff(names(mltdCI_models_df), c("type", "value", "variable")), 
                            direction="wide")
#print(wideCI_models_df)
mrgdCI_models_df <- merge(wideCI_models_df, mltd_models_df, all.x=TRUE)
#print(mrgdCI_models_df)

# Merge stats back in if CIs don't exist
goback_vars <- c()
for (var in unique(mltd_models_df$label)) {
    for (type in unique(mltd_models_df$data)) {
        var_type <- paste0(var, ".", type)
        # if this data is already present, next
        if (var_type %in% unique(paste(mltd_models_df$label, mltd_models_df$data,
                                       sep=".")))
            next
        #print(sprintf("var_type:%s", var_type))
        goback_vars <- c(goback_vars, var_type)
    }
}

if (length(goback_vars) > 0) {
    mltd_goback_df <- build_statsCI_data(glb_models_df[, c("model_id", goback_vars)])
    mltd_models_df <- rbind(mltd_models_df, mltd_goback_df)
}

mltd_models_df <- merge(mltd_models_df, glb_models_df[, c("model_id", "model_method")], 
                        all.x=TRUE)

png(paste0(glb_out_pfx, "models_bar.png"), width=480*3, height=480*2)
print(gp <- myplot_bar(mltd_models_df, "model_id", "value", colorcol_name="model_method") + 
        geom_errorbar(data=mrgdCI_models_df, 
            mapping=aes(x=model_id, ymax=value.Upper, ymin=value.Lower), width=0.5) + 
          facet_grid(label ~ data, scales="free") + 
          theme(axis.text.x = element_text(angle = 90,vjust = 0.5)))
```

```
## Warning in loop_apply(n, do.ply): Removed 3 rows containing missing values
## (position_stack).
```

```r
dev.off()
```

```
## quartz_off_screen 
##                 2
```

```r
print(gp)
```

```
## Warning in loop_apply(n, do.ply): Removed 3 rows containing missing values
## (position_stack).
```

![](NBA_PTS2_files/figure-html/fit.models_2-2.png) 

```r
#stop(here")
# used for console inspection
model_evl_terms <- c(NULL)
for (metric in glb_model_evl_criteria)
    model_evl_terms <- c(model_evl_terms, 
                         ifelse(length(grep("max", metric)) > 0, "-", "+"), metric)
if (glb_is_classification && glb_is_binomial)
    model_evl_terms <- c(model_evl_terms, "-", "opt.prob.threshold.OOB")
model_sel_frmla <- as.formula(paste(c("~ ", model_evl_terms), collapse=" "))
dsp_models_cols <- c("model_id", glb_model_evl_criteria) 
if (glb_is_classification && glb_is_binomial) 
    dsp_models_cols <- c(dsp_models_cols, "opt.prob.threshold.OOB")
print(dsp_models_df <- orderBy(model_sel_frmla, glb_models_df)[, dsp_models_cols])
```

```
##                     model_id min.RMSE.OOB  max.R.sq.OOB max.Adj.R.sq.fit
## 8                   All.X.lm 2.019113e-12  1.0000000000       1.00000000
## 9                  All.X.glm 2.019113e-12  1.0000000000               NA
## 6     Interact.High.cor.Y.lm 2.355896e+01  0.9973043928       0.99865487
## 7               Low.cor.X.lm 2.796915e+01  0.9962007077       0.99890827
## 12         All.X.no.rnorm.rf 1.519081e+02  0.8875517657               NA
## 3  Max.cor.Y.cv.0.cp.0.rpart 2.091754e+02  0.7874968789               NA
## 5               Max.cor.Y.lm 2.118347e+02  0.7820591878       0.93817073
## 4            Max.cor.Y.rpart 2.964608e+02  0.5731466783               NA
## 11      All.X.no.rnorm.rpart 2.964608e+02  0.5731466783               NA
## 1                     MFO.lm 4.536790e+02  0.0003645831      -0.00110768
## 2       Max.cor.Y.cv.0.rpart 4.537617e+02  0.0000000000               NA
## 10            All.X.bayesglm 8.369207e+02 -4.4319811910               NA
```

```r
print(myplot_radar(radar_inp_df=dsp_models_df))
```

```
## Warning in RColorBrewer::brewer.pal(n, pal): n too large, allowed maximum for palette Set1 is 9
## Returning the palette you asked for with that many colors
```

```
## Warning: The shape palette can deal with a maximum of 6 discrete values
## because more than 6 becomes difficult to discriminate; you have
## 12. Consider specifying shapes manually. if you must have them.
```

```
## Warning in loop_apply(n, do.ply): Removed 6 rows containing missing values
## (geom_path).
```

```
## Warning in loop_apply(n, do.ply): Removed 25 rows containing missing values
## (geom_point).
```

```
## Warning in loop_apply(n, do.ply): Removed 7 rows containing missing values
## (geom_text).
```

```
## Warning in RColorBrewer::brewer.pal(n, pal): n too large, allowed maximum for palette Set1 is 9
## Returning the palette you asked for with that many colors
```

```
## Warning: The shape palette can deal with a maximum of 6 discrete values
## because more than 6 becomes difficult to discriminate; you have
## 12. Consider specifying shapes manually. if you must have them.
```

![](NBA_PTS2_files/figure-html/fit.models_2-3.png) 

```r
print("Metrics used for model selection:"); print(model_sel_frmla)
```

```
## [1] "Metrics used for model selection:"
```

```
## ~+min.RMSE.OOB - max.R.sq.OOB - max.Adj.R.sq.fit
```

```r
print(sprintf("Best model id: %s", dsp_models_df[1, "model_id"]))
```

```
## [1] "Best model id: All.X.lm"
```

```r
if (is.null(glb_sel_mdl_id)) { 
    glb_sel_mdl_id <- dsp_models_df[1, "model_id"]
    if (glb_sel_mdl_id == "Interact.High.cor.Y.glm") {
        warning("glb_sel_mdl_id: Interact.High.cor.Y.glm; myextract_mdl_feats does not currently support interaction terms")
        glb_sel_mdl_id <- dsp_models_df[2, "model_id"]
    }
} else 
    print(sprintf("User specified selection: %s", glb_sel_mdl_id))   
    
myprint_mdl(glb_sel_mdl <- glb_models_lst[[glb_sel_mdl_id]])
```

![](NBA_PTS2_files/figure-html/fit.models_2-4.png) ![](NBA_PTS2_files/figure-html/fit.models_2-5.png) ![](NBA_PTS2_files/figure-html/fit.models_2-6.png) ![](NBA_PTS2_files/figure-html/fit.models_2-7.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##        Min         1Q     Median         3Q        Max 
## -1.391e-12 -3.310e-13 -9.200e-14  1.490e-13  7.043e-11 
## 
## Coefficients: (2 not defined because of singularities)
##               Estimate Std. Error    t value Pr(>|t|)    
## (Intercept)  2.014e-12  2.937e-12  6.860e-01   0.4930    
## FG           3.000e+00  4.509e-15  6.653e+14   <2e-16 ***
## X2P         -1.000e+00  4.754e-15 -2.104e+14   <2e-16 ***
## FGA         -2.950e-15  1.764e-15 -1.672e+00   0.0948 .  
## AST         -3.427e-16  7.313e-16 -4.690e-01   0.6395    
## X2PA         2.774e-15  1.850e-15  1.500e+00   0.1341    
## FT           1.000e+00  1.610e-15  6.213e+14   <2e-16 ***
## FTA         -1.398e-15  1.380e-15 -1.013e+00   0.3112    
## ORB          2.111e-15  1.160e-15  1.820e+00   0.0692 .  
## STL          1.208e-15  1.242e-15  9.730e-01   0.3311    
## TOV          4.421e-17  8.507e-16  5.200e-02   0.9586    
## BLK          8.687e-16  1.188e-15  7.310e-01   0.4648    
## DRB          6.777e-16  8.297e-16  8.170e-01   0.4143    
## .rnorm       1.451e-13  8.913e-14  1.628e+00   0.1039    
## X3P                 NA         NA         NA       NA    
## X3PA                NA         NA         NA       NA    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 2.493e-12 on 821 degrees of freedom
## Multiple R-squared:      1,	Adjusted R-squared:      1 
## F-statistic: 3.484e+30 on 13 and 821 DF,  p-value: < 2.2e-16
```

```
## [1] TRUE
```

```r
# From here to save(), this should all be in one function
#   these are executed in the same seq twice more:
#       fit.data.training & predict.data.new chunks
glb_get_predictions <- function(df, mdl_id, rsp_var_out, prob_threshold_def=NULL) {
    mdl <- glb_models_lst[[mdl_id]]
    rsp_var_out <- paste0(rsp_var_out, mdl_id)

    if (glb_is_regression) {
        df[, rsp_var_out] <- predict(mdl, newdata=df, type="raw")
        print(myplot_scatter(df, glb_rsp_var, rsp_var_out, smooth=TRUE))
        df[, paste0(rsp_var_out, ".err")] <- 
            abs(df[, rsp_var_out] - df[, glb_rsp_var])
        print(head(orderBy(reformulate(c("-", paste0(rsp_var_out, ".err"))), 
                           df)))                             
    }

    if (glb_is_classification && glb_is_binomial) {
        prob_threshold <- glb_models_df[glb_models_df$model_id == mdl_id, 
                                        "opt.prob.threshold.OOB"]
        if (is.null(prob_threshold) || is.na(prob_threshold)) {
            warning("Using default probability threshold: ", prob_threshold_def)
            if (is.null(prob_threshold <- prob_threshold_def))
                stop("Default probability threshold is NULL")
        }
        
        df[, paste0(rsp_var_out, ".prob")] <- 
            predict(mdl, newdata=df, type="prob")[, 2]
        df[, rsp_var_out] <- 
        		factor(levels(df[, glb_rsp_var])[
    				(df[, paste0(rsp_var_out, ".prob")] >=
    					prob_threshold) * 1 + 1], levels(df[, glb_rsp_var]))
    
        # prediction stats already reported by myfit_mdl ???
    }    
    
    if (glb_is_classification && !glb_is_binomial) {
        df[, rsp_var_out] <- predict(mdl, newdata=df, type="raw")
        df[, paste0(rsp_var_out, ".prob")] <- 
            predict(mdl, newdata=df, type="prob")
    }

    return(df)
}    
glb_OOBobs_df <- glb_get_predictions(df=glb_OOBobs_df, mdl_id=glb_sel_mdl_id, 
                                     rsp_var_out=glb_rsp_var_out)
```

```
## Warning in predict.lm(modelFit, newdata): prediction from a rank-deficient
## fit may be misleading
```

![](NBA_PTS2_files/figure-html/fit.models_2-8.png) 

```
##     SeasonEnd                   Team Playoffs  W  PTS oppPTS   FG  FGA
## 846      2013   Los Angeles Clippers        1 56 8289   7760 3160 6608
## 850      2013        Milwaukee Bucks        1 38 8108   8231 3128 7197
## 851      2013 Minnesota Timberwolves        0 31 7851   8045 2943 6702
## 852      2013    New Orleans Hornets        0 27 7714   8031 2955 6589
## 856      2013     Philadelphia 76ers        0 34 7640   7914 3059 6895
## 857      2013           Phoenix Suns        0 25 7805   8335 3061 6917
##      X2P X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV .rownames
## 846 2533 4856 627 1752 1342 1888  938 2475 1958 784 461 1197        11
## 850 2527 5527 601 1670 1251 1700 1068 2537 1876 685 550 1156        15
## 851 2493 5227 450 1475 1515 2042  973 2473 1836 700 387 1214        16
## 852 2420 5115 535 1474 1269 1636  988 2426 1721 520 440 1193        17
## 856 2541 5457 518 1438 1004 1377  896 2493 1867 609 384 1070        21
## 857 2581 5462 480 1455 1203 1618  959 2454 1855 659 434 1278        22
##     .src     .rnorm PTS.diff              Team.fctr PTS.predict.All.X.lm
## 846 Test  0.3200711      529   Los Angeles Clippers                 8289
## 850 Test  1.7399997     -123        Milwaukee Bucks                 8108
## 851 Test -0.2641936     -194 Minnesota Timberwolves                 7851
## 852 Test -0.4927864     -317    New Orleans Hornets                 7714
## 856 Test -0.4211144     -274     Philadelphia 76ers                 7640
## 857 Test  0.2165301     -530           Phoenix Suns                 7805
##     PTS.predict.All.X.lm.err
## 846             3.637979e-12
## 850             2.728484e-12
## 851             2.728484e-12
## 852             2.728484e-12
## 856             2.728484e-12
## 857             2.728484e-12
```

```r
predct_accurate_var_name <- paste0(glb_rsp_var_out, glb_sel_mdl_id, ".accurate")
glb_OOBobs_df[, predct_accurate_var_name] <-
                    (glb_OOBobs_df[, glb_rsp_var] == 
                     glb_OOBobs_df[, paste0(glb_rsp_var_out, glb_sel_mdl_id)])

#stop(here"); #sav_models_lst <- glb_models_lst; sav_models_df <- glb_models_df
glb_featsimp_df <- 
    myget_feats_importance(mdl=glb_sel_mdl, featsimp_df=NULL)
```

```
## Warning in summary.lm(object): essentially perfect fit: summary may be
## unreliable
```

```r
glb_featsimp_df[, paste0(glb_sel_mdl_id, ".importance")] <- glb_featsimp_df$importance
print(glb_featsimp_df)
```

```
##          importance All.X.lm.importance
## FG     1.000000e+02        1.000000e+02
## FT     9.337967e+01        9.337967e+01
## X2P    3.161786e+01        3.161786e+01
## ORB    2.657247e-13        2.657247e-13
## FGA    2.435779e-13        2.435779e-13
## .rnorm 2.368766e-13        2.368766e-13
## X2PA   2.175897e-13        2.175897e-13
## FTA    1.445086e-13        1.445086e-13
## STL    1.383737e-13        1.383737e-13
## DRB    1.149504e-13        1.149504e-13
## BLK    1.021147e-13        1.021147e-13
## AST    6.262793e-14        6.262793e-14
## TOV    0.000000e+00        0.000000e+00
```

```r
# Used again in fit.data.training & predict.data.new chunks
glb_analytics_diag_plots <- function(obs_df, mdl_id, prob_threshold=NULL) {
    featsimp_df <- glb_featsimp_df
    featsimp_df$feat <- gsub("`(.*?)`", "\\1", row.names(featsimp_df))    
    featsimp_df$feat.interact <- gsub("(.*?):(.*)", "\\2", featsimp_df$feat)
    featsimp_df$feat <- gsub("(.*?):(.*)", "\\1", featsimp_df$feat)    
    featsimp_df$feat.interact <- ifelse(featsimp_df$feat.interact == featsimp_df$feat, 
                                        NA, featsimp_df$feat.interact)
    featsimp_df$feat <- gsub("(.*?)\\.fctr(.*)", "\\1\\.fctr", featsimp_df$feat)
    featsimp_df$feat.interact <- gsub("(.*?)\\.fctr(.*)", "\\1\\.fctr", featsimp_df$feat.interact) 
    featsimp_df <- orderBy(~ -importance.max, summaryBy(importance ~ feat + feat.interact, 
                                                        data=featsimp_df, FUN=max))    
    #rex_str=":(.*)"; txt_vctr=tail(featsimp_df$feat); ret_lst <- regexec(rex_str, txt_vctr); ret_lst <- regmatches(txt_vctr, ret_lst); ret_vctr <- sapply(1:length(ret_lst), function(pos_ix) ifelse(length(ret_lst[[pos_ix]]) > 0, ret_lst[[pos_ix]], "")); print(ret_vctr <- ret_vctr[ret_vctr != ""])    
    if (nrow(featsimp_df) > 5) {
        warning("Limiting important feature scatter plots to 5 out of ", nrow(featsimp_df))
        featsimp_df <- head(featsimp_df, 5)
    }
    
#     if (!all(is.na(featsimp_df$feat.interact)))
#         stop("not implemented yet")
    rsp_var_out <- paste0(glb_rsp_var_out, mdl_id)
    for (var in featsimp_df$feat) {
        plot_df <- melt(obs_df, id.vars=var, 
                        measure.vars=c(glb_rsp_var, rsp_var_out))

#         if (var == "<feat_name>") print(myplot_scatter(plot_df, var, "value", 
#                                              facet_colcol_name="variable") + 
#                       geom_vline(xintercept=<divider_val>, linetype="dotted")) else     
            print(myplot_scatter(plot_df, var, "value", colorcol_name="variable",
                                 facet_colcol_name="variable", jitter=TRUE) + 
                      guides(color=FALSE))
    }
    
    if (glb_is_regression) {
        if (nrow(featsimp_df) == 0)
            warning("No important features in glb_fin_mdl") else
            print(myplot_prediction_regression(df=obs_df, 
                        feat_x=ifelse(nrow(featsimp_df) > 1, featsimp_df$feat[2],
                                      ".rownames"), 
                                               feat_y=featsimp_df$feat[1],
                        rsp_var=glb_rsp_var, rsp_var_out=rsp_var_out,
                        id_vars=glb_id_var)
    #               + facet_wrap(reformulate(featsimp_df$feat[2])) # if [1 or 2] is a factor
    #               + geom_point(aes_string(color="<col_name>.fctr")) #  to color the plot
                  )
    }    
    
    if (glb_is_classification) {
        if (nrow(featsimp_df) == 0)
            warning("No features in selected model are statistically important")
        else print(myplot_prediction_classification(df=obs_df, 
                feat_x=ifelse(nrow(featsimp_df) > 1, featsimp_df$feat[2], 
                              ".rownames"),
                                               feat_y=featsimp_df$feat[1],
                     rsp_var=glb_rsp_var, 
                     rsp_var_out=rsp_var_out, 
                     id_vars=glb_id_var,
                    prob_threshold=prob_threshold)
#               + geom_hline(yintercept=<divider_val>, linetype = "dotted")
                )
    }    
}
if (glb_is_classification && glb_is_binomial)
    glb_analytics_diag_plots(obs_df=glb_OOBobs_df, mdl_id=glb_sel_mdl_id, 
            prob_threshold=glb_models_df[glb_models_df$model_id == glb_sel_mdl_id, 
                                         "opt.prob.threshold.OOB"]) else
    glb_analytics_diag_plots(obs_df=glb_OOBobs_df, mdl_id=glb_sel_mdl_id)                  
```

```
## Warning in glb_analytics_diag_plots(obs_df = glb_OOBobs_df, mdl_id =
## glb_sel_mdl_id): Limiting important feature scatter plots to 5 out of 13
```

![](NBA_PTS2_files/figure-html/fit.models_2-9.png) ![](NBA_PTS2_files/figure-html/fit.models_2-10.png) ![](NBA_PTS2_files/figure-html/fit.models_2-11.png) ![](NBA_PTS2_files/figure-html/fit.models_2-12.png) ![](NBA_PTS2_files/figure-html/fit.models_2-13.png) 

```
##     SeasonEnd                   Team Playoffs  W  PTS oppPTS   FG  FGA
## 846      2013   Los Angeles Clippers        1 56 8289   7760 3160 6608
## 850      2013        Milwaukee Bucks        1 38 8108   8231 3128 7197
## 851      2013 Minnesota Timberwolves        0 31 7851   8045 2943 6702
## 852      2013    New Orleans Hornets        0 27 7714   8031 2955 6589
## 856      2013     Philadelphia 76ers        0 34 7640   7914 3059 6895
##      X2P X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV .rownames
## 846 2533 4856 627 1752 1342 1888  938 2475 1958 784 461 1197        11
## 850 2527 5527 601 1670 1251 1700 1068 2537 1876 685 550 1156        15
## 851 2493 5227 450 1475 1515 2042  973 2473 1836 700 387 1214        16
## 852 2420 5115 535 1474 1269 1636  988 2426 1721 520 440 1193        17
## 856 2541 5457 518 1438 1004 1377  896 2493 1867 609 384 1070        21
##     .src     .rnorm PTS.diff              Team.fctr PTS.predict.All.X.lm
## 846 Test  0.3200711      529   Los Angeles Clippers                 8289
## 850 Test  1.7399997     -123        Milwaukee Bucks                 8108
## 851 Test -0.2641936     -194 Minnesota Timberwolves                 7851
## 852 Test -0.4927864     -317    New Orleans Hornets                 7714
## 856 Test -0.4211144     -274     Philadelphia 76ers                 7640
##     PTS.predict.All.X.lm.err PTS.predict.All.X.lm.accurate .label
## 846             3.637979e-12                         FALSE     11
## 850             2.728484e-12                         FALSE     15
## 851             2.728484e-12                         FALSE     16
## 852             2.728484e-12                         FALSE     17
## 856             2.728484e-12                         FALSE     21
```

![](NBA_PTS2_files/figure-html/fit.models_2-14.png) 

```r
# gather predictions from models better than MFO.*
#mdl_id <- "Conditional.X.rf"
#mdl_id <- "Conditional.X.cp.0.rpart"
#mdl_id <- "Conditional.X.rpart"
# glb_OOBobs_df <- glb_get_predictions(df=glb_OOBobs_df, mdl_id,
#                                      glb_rsp_var_out)
# print(t(confusionMatrix(glb_OOBobs_df[, paste0(glb_rsp_var_out, mdl_id)], 
#                         glb_OOBobs_df[, glb_rsp_var])$table))
# FN_OOB_ids <- c(4721, 4020, 693, 92)
# print(glb_OOBobs_df[glb_OOBobs_df$UniqueID %in% FN_OOB_ids, 
#                     grep(glb_rsp_var, names(glb_OOBobs_df), value=TRUE)])
# print(glb_OOBobs_df[glb_OOBobs_df$UniqueID %in% FN_OOB_ids, 
#                     glb_feats_df$id[1:5]])
# print(glb_OOBobs_df[glb_OOBobs_df$UniqueID %in% FN_OOB_ids, 
#                     glb_txt_vars])
write.csv(glb_OOBobs_df[, c(glb_id_var, 
                grep(glb_rsp_var, names(glb_OOBobs_df), fixed=TRUE, value=TRUE))], 
    paste0(gsub(".", "_", paste0(glb_out_pfx, glb_sel_mdl_id), fixed=TRUE), 
           "_OOBobs.csv"), row.names=FALSE)

# print(glb_allobs_df[glb_allobs_df$UniqueID %in% FN_OOB_ids, 
#                     glb_txt_vars])
# dsp_tbl(Headline.contains="[Ee]bola")
# sum(sel_obs(Headline.contains="[Ee]bola"))
# ftable(xtabs(Popular ~ NewsDesk.fctr, data=glb_allobs_df[sel_obs(Headline.contains="[Ee]bola") ,]))
# xtabs(NewsDesk ~ Popular, #Popular ~ NewsDesk.fctr, 
#       data=glb_allobs_df[sel_obs(Headline.contains="[Ee]bola") ,],
#       exclude=NULL)
# print(mycreate_xtab_df(df=glb_allobs_df[sel_obs(Headline.contains="[Ee]bola") ,], c("Popular", "NewsDesk", "SectionName", "SubsectionName")))
# print(mycreate_tbl_df(df=glb_allobs_df[sel_obs(Headline.contains="[Ee]bola") ,], c("Popular", "NewsDesk", "SectionName", "SubsectionName")))
# print(mycreate_tbl_df(df=glb_allobs_df[sel_obs(Headline.contains="[Ee]bola") ,], c("Popular")))
# print(mycreate_tbl_df(df=glb_allobs_df[sel_obs(Headline.contains="[Ee]bola") ,], 
#                       tbl_col_names=c("Popular", "NewsDesk")))

# write.csv(glb_chunks_df, paste0(glb_out_pfx, tail(glb_chunks_df, 1)$label, "_",
#                                 tail(glb_chunks_df, 1)$step_minor,  "_chunks1.csv"),
#           row.names=FALSE)

glb_chunks_df <- myadd_chunk(glb_chunks_df, "fit.models", major.inc=FALSE)
```

```
##         label step_major step_minor    bgn    end elapsed
## 12 fit.models          7          2 68.802 79.218  10.416
## 13 fit.models          7          3 79.218     NA      NA
```


```r
print(setdiff(names(glb_trnobs_df), names(glb_allobs_df)))
```

```
## character(0)
```

```r
print(setdiff(names(glb_fitobs_df), names(glb_allobs_df)))
```

```
## character(0)
```

```r
print(setdiff(names(glb_OOBobs_df), names(glb_allobs_df)))
```

```
## [1] "PTS.predict.All.X.lm"          "PTS.predict.All.X.lm.err"     
## [3] "PTS.predict.All.X.lm.accurate"
```

```r
for (col in setdiff(names(glb_OOBobs_df), names(glb_allobs_df)))
    # Merge or cbind ?
    glb_allobs_df[glb_allobs_df$.lcn == "OOB", col] <- glb_OOBobs_df[, col]
    
print(setdiff(names(glb_newobs_df), names(glb_allobs_df)))
```

```
## character(0)
```

```r
if (glb_save_envir)
    save(glb_feats_df, 
         glb_allobs_df, #glb_trnobs_df, glb_fitobs_df, glb_OOBobs_df, glb_newobs_df,
         glb_models_df, dsp_models_df, glb_models_lst, glb_sel_mdl, glb_sel_mdl_id,
         glb_model_type,
        file=paste0(glb_out_pfx, "selmdl_dsk.RData"))
#load(paste0(glb_out_pfx, "selmdl_dsk.RData"))

rm(ret_lst)
```

```
## Warning in rm(ret_lst): object 'ret_lst' not found
```

```r
replay.petrisim(pn=glb_analytics_pn, 
    replay.trans=(glb_analytics_avl_objs <- c(glb_analytics_avl_objs, 
        "model.selected")), flip_coord=TRUE)
```

```
## time	trans	 "bgn " "fit.data.training.all " "predict.data.new " "end " 
## 0.0000 	multiple enabled transitions:  data.training.all data.new model.selected 	firing:  data.training.all 
## 1.0000 	 1 	 2 1 0 0 
## 1.0000 	multiple enabled transitions:  data.training.all data.new model.selected model.final data.training.all.prediction 	firing:  data.new 
## 2.0000 	 2 	 1 1 1 0 
## 2.0000 	multiple enabled transitions:  data.training.all data.new model.selected model.final data.training.all.prediction data.new.prediction 	firing:  model.selected 
## 3.0000 	 3 	 0 2 1 0
```

![](NBA_PTS2_files/figure-html/fit.models_3-1.png) 

```r
glb_chunks_df <- myadd_chunk(glb_chunks_df, "fit.data.training", major.inc=TRUE)
```

```
##                label step_major step_minor    bgn    end elapsed
## 13        fit.models          7          3 79.218 83.194   3.977
## 14 fit.data.training          8          0 83.195     NA      NA
```

## Step `8.0: fit data training`

```r
#load(paste0(glb_inp_pfx, "dsk.RData"))

# To create specific models
# glb_fin_mdl_id <- NULL; glb_fin_mdl <- NULL; 
# glb_sel_mdl_id <- "Conditional.X.cp.0.rpart"; 
# glb_sel_mdl <- glb_models_lst[[glb_sel_mdl_id]]; print(glb_sel_mdl)
    
if (!is.null(glb_fin_mdl_id) && (glb_fin_mdl_id %in% names(glb_models_lst))) {
    warning("Final model same as user selected model")
    glb_fin_mdl <- glb_sel_mdl
} else {    
#     print(mdl_feats_df <- myextract_mdl_feats(sel_mdl=glb_sel_mdl, 
#                                               entity_df=glb_fitobs_df))
    
    if ((model_method <- glb_sel_mdl$method) == "custom")
        # get actual method from the model_id
        model_method <- tail(unlist(strsplit(glb_sel_mdl_id, "[.]")), 1)
        
    tune_finmdl_df <- NULL
    if (nrow(glb_sel_mdl$bestTune) > 0) {
        for (param in names(glb_sel_mdl$bestTune)) {
            #print(sprintf("param: %s", param))
            if (glb_sel_mdl$bestTune[1, param] != "none")
                tune_finmdl_df <- rbind(tune_finmdl_df, 
                    data.frame(parameter=param, 
                               min=glb_sel_mdl$bestTune[1, param], 
                               max=glb_sel_mdl$bestTune[1, param], 
                               by=1)) # by val does not matter
        }
    } 
    
    # Sync with parameters in mydsutils.R
    require(gdata)
    ret_lst <- myfit_mdl(model_id="Final", model_method=model_method,
        indep_vars_vctr=trim(unlist(strsplit(glb_models_df[glb_models_df$model_id == glb_sel_mdl_id,
                                                    "feats"], "[,]"))), 
                         model_type=glb_model_type,
                            rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out, 
                            fit_df=glb_trnobs_df, OOB_df=NULL,
                            n_cv_folds=glb_n_cv_folds, tune_models_df=tune_finmdl_df,
                         # Automate from here
                         #  Issues if glb_sel_mdl$method == "rf" b/c trainControl is "oob"; not "cv"
                            model_loss_mtrx=glb_model_metric_terms,
                            model_summaryFunction=glb_sel_mdl$control$summaryFunction,
                            model_metric=glb_sel_mdl$metric,
                            model_metric_maximize=glb_sel_mdl$maximize)
    glb_fin_mdl <- glb_models_lst[[length(glb_models_lst)]] 
    glb_fin_mdl_id <- glb_models_df[length(glb_models_lst), "model_id"]
}
```

```
## Loading required package: gdata
## gdata: read.xls support for 'XLS' (Excel 97-2004) files ENABLED.
## 
## gdata: read.xls support for 'XLSX' (Excel 2007+) files ENABLED.
## 
## Attaching package: 'gdata'
## 
## The following object is masked from 'package:randomForest':
## 
##     combine
## 
## The following objects are masked from 'package:dplyr':
## 
##     combine, first, last
## 
## The following object is masked from 'package:stats':
## 
##     nobs
## 
## The following object is masked from 'package:utils':
## 
##     object.size
```

```
## [1] "fitting model: Final.lm"
## [1] "    indep_vars: FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, .rnorm, X3P, X3PA"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_PTS2_files/figure-html/fit.data.training_0-1.png) ![](NBA_PTS2_files/figure-html/fit.data.training_0-2.png) ![](NBA_PTS2_files/figure-html/fit.data.training_0-3.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##        Min         1Q     Median         3Q        Max 
## -1.391e-12 -3.310e-13 -9.200e-14  1.490e-13  7.043e-11 
## 
## Coefficients: (2 not defined because of singularities)
##               Estimate Std. Error    t value Pr(>|t|)    
## (Intercept)  2.014e-12  2.937e-12  6.860e-01   0.4930    
## FG           3.000e+00  4.509e-15  6.653e+14   <2e-16 ***
## X2P         -1.000e+00  4.754e-15 -2.104e+14   <2e-16 ***
## FGA         -2.950e-15  1.764e-15 -1.672e+00   0.0948 .  
## AST         -3.427e-16  7.313e-16 -4.690e-01   0.6395    
## X2PA         2.774e-15  1.850e-15  1.500e+00   0.1341    
## FT           1.000e+00  1.610e-15  6.213e+14   <2e-16 ***
## FTA         -1.398e-15  1.380e-15 -1.013e+00   0.3112    
## ORB          2.111e-15  1.160e-15  1.820e+00   0.0692 .  
## STL          1.208e-15  1.242e-15  9.730e-01   0.3311    
## TOV          4.421e-17  8.507e-16  5.200e-02   0.9586    
## BLK          8.687e-16  1.188e-15  7.310e-01   0.4648    
## DRB          6.777e-16  8.297e-16  8.170e-01   0.4143    
## .rnorm       1.451e-13  8.913e-14  1.628e+00   0.1039    
## X3P                 NA         NA         NA       NA    
## X3PA                NA         NA         NA       NA    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 2.493e-12 on 821 degrees of freedom
## Multiple R-squared:      1,	Adjusted R-squared:      1 
## F-statistic: 3.484e+30 on 13 and 821 DF,  p-value: < 2.2e-16
## 
## [1] "    calling mypredict_mdl for fit:"
```

```
## Warning in predict.lm(modelFit, newdata): prediction from a rank-deficient
## fit may be misleading
```

```
## Warning in summary.lm(mdl$finalModel): essentially perfect fit: summary may
## be unreliable
```

![](NBA_PTS2_files/figure-html/fit.data.training_0-4.png) 

```
##   model_id model_method
## 1 Final.lm           lm
##                                                                          feats
## 1 FG, X2P, FGA, AST, X2PA, FT, FTA, ORB, STL, TOV, BLK, DRB, .rnorm, X3P, X3PA
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      0.937                 0.007
##   max.R.sq.fit min.RMSE.fit max.Adj.R.sq.fit max.Rsquared.fit
## 1            1 2.777052e-12                1                1
##   min.RMSESD.fit max.RsquaredSD.fit
## 1   1.509542e-12                  0
```

```r
rm(ret_lst)
glb_chunks_df <- myadd_chunk(glb_chunks_df, "fit.data.training", major.inc=FALSE)
```

```
##                label step_major step_minor    bgn    end elapsed
## 14 fit.data.training          8          0 83.195 87.348   4.153
## 15 fit.data.training          8          1 87.348     NA      NA
```


```r
glb_trnobs_df <- glb_get_predictions(df=glb_trnobs_df, mdl_id=glb_fin_mdl_id, 
                                     rsp_var_out=glb_rsp_var_out,
    prob_threshold_def=ifelse(glb_is_classification && glb_is_binomial, 
        glb_models_df[glb_models_df$model_id == glb_sel_mdl_id, "opt.prob.threshold.OOB"], NULL))
```

```
## Warning in predict.lm(modelFit, newdata): prediction from a rank-deficient
## fit may be misleading
```

![](NBA_PTS2_files/figure-html/fit.data.training_1-1.png) 

```
##     SeasonEnd               Team Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 1        1980      Atlanta Hawks        1 50 8573   8334 3261 7027 3248
## 107      1984 Philadelphia 76ers        1 52 8838   8658 3384 6833 3355
## 131      1985 Philadelphia 76ers        1 58 9261   8925 3443 6992 3384
## 153      1986 Philadelphia 76ers        1 54 9051   8858 3435 7058 3384
## 183      1987 Washington Bullets        1 42 8690   8802 3356 7397 3313
## 207      1989      Atlanta Hawks        1 52 9102   8699 3412 7230 3302
##     X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV .rownames  .src
## 1   6952  13   75 2038 2645 1369 2406 1913 782 539 1495         1 Train
## 107 6726  29  107 2041 2706 1181 2382 2032 807 653 1628       107 Train
## 131 6768  59  224 2316 2883 1301 2364 1999 817 534 1575       131 Train
## 153 6834  51  224 2130 2810 1326 2378 2017 862 490 1595       153 Train
## 183 7179  43  218 1935 2531 1305 2315 1750 755 685 1301       183 Train
## 207 6833 110  397 2168 2709 1372 2316 1990 817 474 1310       207 Train
##         .rnorm PTS.diff          Team.fctr PTS.predict.Final.lm
## 1    1.6611541      239      Atlanta Hawks                 8573
## 107 -0.1784143      180 Philadelphia 76ers                 8838
## 131  0.2982758      336 Philadelphia 76ers                 9261
## 153 -0.3798679      193 Philadelphia 76ers                 9051
## 183  0.5534608     -112 Washington Bullets                 8690
## 207 -0.2245829      403      Atlanta Hawks                 9102
##     PTS.predict.Final.lm.err
## 1               5.456968e-12
## 107             5.456968e-12
## 131             5.456968e-12
## 153             5.456968e-12
## 183             5.456968e-12
## 207             5.456968e-12
```

```r
sav_featsimp_df <- glb_featsimp_df
#glb_feats_df <- sav_feats_df
# glb_feats_df <- mymerge_feats_importance(feats_df=glb_feats_df, sel_mdl=glb_fin_mdl, 
#                                                entity_df=glb_trnobs_df)
glb_featsimp_df <- myget_feats_importance(mdl=glb_fin_mdl, featsimp_df=glb_featsimp_df)
```

```
## Warning in summary.lm(object): essentially perfect fit: summary may be
## unreliable
```

```r
glb_featsimp_df[, paste0(glb_fin_mdl_id, ".importance")] <- glb_featsimp_df$importance
print(glb_featsimp_df)
```

```
##        All.X.lm.importance   importance Final.lm.importance
## FG            1.000000e+02 1.000000e+02        1.000000e+02
## FT            9.337967e+01 9.337967e+01        9.337967e+01
## X2P           3.161786e+01 3.161786e+01        3.161786e+01
## ORB           2.657247e-13 2.657247e-13        2.657247e-13
## FGA           2.435779e-13 2.435779e-13        2.435779e-13
## .rnorm        2.368766e-13 2.368766e-13        2.368766e-13
## X2PA          2.175897e-13 2.175897e-13        2.175897e-13
## FTA           1.445086e-13 1.445086e-13        1.445086e-13
## STL           1.383737e-13 1.383737e-13        1.383737e-13
## DRB           1.149504e-13 1.149504e-13        1.149504e-13
## BLK           1.021147e-13 1.021147e-13        1.021147e-13
## AST           6.262793e-14 6.262793e-14        6.262793e-14
## TOV           0.000000e+00 0.000000e+00        0.000000e+00
```

```r
if (glb_is_classification && glb_is_binomial)
    glb_analytics_diag_plots(obs_df=glb_trnobs_df, mdl_id=glb_fin_mdl_id, 
            prob_threshold=glb_models_df[glb_models_df$model_id == glb_sel_mdl_id, 
                                         "opt.prob.threshold.OOB"]) else
    glb_analytics_diag_plots(obs_df=glb_trnobs_df, mdl_id=glb_fin_mdl_id)                  
```

```
## Warning in glb_analytics_diag_plots(obs_df = glb_trnobs_df, mdl_id =
## glb_fin_mdl_id): Limiting important feature scatter plots to 5 out of 13
```

![](NBA_PTS2_files/figure-html/fit.data.training_1-2.png) ![](NBA_PTS2_files/figure-html/fit.data.training_1-3.png) ![](NBA_PTS2_files/figure-html/fit.data.training_1-4.png) ![](NBA_PTS2_files/figure-html/fit.data.training_1-5.png) ![](NBA_PTS2_files/figure-html/fit.data.training_1-6.png) 

```
##     SeasonEnd               Team Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 1        1980      Atlanta Hawks        1 50 8573   8334 3261 7027 3248
## 107      1984 Philadelphia 76ers        1 52 8838   8658 3384 6833 3355
## 131      1985 Philadelphia 76ers        1 58 9261   8925 3443 6992 3384
## 153      1986 Philadelphia 76ers        1 54 9051   8858 3435 7058 3384
## 183      1987 Washington Bullets        1 42 8690   8802 3356 7397 3313
##     X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV .rownames  .src
## 1   6952  13   75 2038 2645 1369 2406 1913 782 539 1495         1 Train
## 107 6726  29  107 2041 2706 1181 2382 2032 807 653 1628       107 Train
## 131 6768  59  224 2316 2883 1301 2364 1999 817 534 1575       131 Train
## 153 6834  51  224 2130 2810 1326 2378 2017 862 490 1595       153 Train
## 183 7179  43  218 1935 2531 1305 2315 1750 755 685 1301       183 Train
##         .rnorm PTS.diff          Team.fctr PTS.predict.Final.lm
## 1    1.6611541      239      Atlanta Hawks                 8573
## 107 -0.1784143      180 Philadelphia 76ers                 8838
## 131  0.2982758      336 Philadelphia 76ers                 9261
## 153 -0.3798679      193 Philadelphia 76ers                 9051
## 183  0.5534608     -112 Washington Bullets                 8690
##     PTS.predict.Final.lm.err .label
## 1               5.456968e-12      1
## 107             5.456968e-12    107
## 131             5.456968e-12    131
## 153             5.456968e-12    153
## 183             5.456968e-12    183
```

![](NBA_PTS2_files/figure-html/fit.data.training_1-7.png) 

```r
dsp_feats_vctr <- c(NULL)
for(var in grep(".importance", names(glb_feats_df), fixed=TRUE, value=TRUE))
    dsp_feats_vctr <- union(dsp_feats_vctr, 
                            glb_feats_df[!is.na(glb_feats_df[, var]), "id"])

# print(glb_trnobs_df[glb_trnobs_df$UniqueID %in% FN_OOB_ids, 
#                     grep(glb_rsp_var, names(glb_trnobs_df), value=TRUE)])

print(setdiff(names(glb_trnobs_df), names(glb_allobs_df)))
```

```
## [1] "PTS.predict.Final.lm"     "PTS.predict.Final.lm.err"
```

```r
for (col in setdiff(names(glb_trnobs_df), names(glb_allobs_df)))
    # Merge or cbind ?
    glb_allobs_df[glb_allobs_df$.src == "Train", col] <- glb_trnobs_df[, col]

print(setdiff(names(glb_fitobs_df), names(glb_allobs_df)))
```

```
## character(0)
```

```r
print(setdiff(names(glb_OOBobs_df), names(glb_allobs_df)))
```

```
## character(0)
```

```r
for (col in setdiff(names(glb_OOBobs_df), names(glb_allobs_df)))
    # Merge or cbind ?
    glb_allobs_df[glb_allobs_df$.lcn == "OOB", col] <- glb_OOBobs_df[, col]
    
print(setdiff(names(glb_newobs_df), names(glb_allobs_df)))
```

```
## character(0)
```

```r
if (glb_save_envir)
    save(glb_feats_df, glb_allobs_df, 
         #glb_trnobs_df, glb_fitobs_df, glb_OOBobs_df, glb_newobs_df,
         glb_models_df, dsp_models_df, glb_models_lst, glb_model_type,
         glb_sel_mdl, glb_sel_mdl_id,
         glb_fin_mdl, glb_fin_mdl_id,
        file=paste0(glb_out_pfx, "dsk.RData"))

replay.petrisim(pn=glb_analytics_pn, 
    replay.trans=(glb_analytics_avl_objs <- c(glb_analytics_avl_objs, 
        "data.training.all.prediction","model.final")), flip_coord=TRUE)
```

```
## time	trans	 "bgn " "fit.data.training.all " "predict.data.new " "end " 
## 0.0000 	multiple enabled transitions:  data.training.all data.new model.selected 	firing:  data.training.all 
## 1.0000 	 1 	 2 1 0 0 
## 1.0000 	multiple enabled transitions:  data.training.all data.new model.selected model.final data.training.all.prediction 	firing:  data.new 
## 2.0000 	 2 	 1 1 1 0 
## 2.0000 	multiple enabled transitions:  data.training.all data.new model.selected model.final data.training.all.prediction data.new.prediction 	firing:  model.selected 
## 3.0000 	 3 	 0 2 1 0 
## 3.0000 	multiple enabled transitions:  model.final data.training.all.prediction data.new.prediction 	firing:  data.training.all.prediction 
## 4.0000 	 5 	 0 1 1 1 
## 4.0000 	multiple enabled transitions:  model.final data.training.all.prediction data.new.prediction 	firing:  model.final 
## 5.0000 	 4 	 0 0 2 1
```

![](NBA_PTS2_files/figure-html/fit.data.training_1-8.png) 

```r
glb_chunks_df <- myadd_chunk(glb_chunks_df, "predict.data.new", major.inc=TRUE)
```

```
##                label step_major step_minor    bgn    end elapsed
## 15 fit.data.training          8          1 87.348 92.539   5.191
## 16  predict.data.new          9          0 92.539     NA      NA
```

## Step `9.0: predict data new`

```r
# Compute final model predictions
glb_newobs_df <- glb_get_predictions(glb_newobs_df, mdl_id=glb_fin_mdl_id, 
                                     rsp_var_out=glb_rsp_var_out,
    prob_threshold_def=ifelse(glb_is_classification && glb_is_binomial, 
        glb_models_df[glb_models_df$model_id == glb_sel_mdl_id, 
                      "opt.prob.threshold.OOB"], NULL))
```

```
## Warning in predict.lm(modelFit, newdata): prediction from a rank-deficient
## fit may be misleading
```

![](NBA_PTS2_files/figure-html/predict.data.new-1.png) 

```
##     SeasonEnd                   Team Playoffs  W  PTS oppPTS   FG  FGA
## 846      2013   Los Angeles Clippers        1 56 8289   7760 3160 6608
## 850      2013        Milwaukee Bucks        1 38 8108   8231 3128 7197
## 851      2013 Minnesota Timberwolves        0 31 7851   8045 2943 6702
## 852      2013    New Orleans Hornets        0 27 7714   8031 2955 6589
## 856      2013     Philadelphia 76ers        0 34 7640   7914 3059 6895
## 857      2013           Phoenix Suns        0 25 7805   8335 3061 6917
##      X2P X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV .rownames
## 846 2533 4856 627 1752 1342 1888  938 2475 1958 784 461 1197        11
## 850 2527 5527 601 1670 1251 1700 1068 2537 1876 685 550 1156        15
## 851 2493 5227 450 1475 1515 2042  973 2473 1836 700 387 1214        16
## 852 2420 5115 535 1474 1269 1636  988 2426 1721 520 440 1193        17
## 856 2541 5457 518 1438 1004 1377  896 2493 1867 609 384 1070        21
## 857 2581 5462 480 1455 1203 1618  959 2454 1855 659 434 1278        22
##     .src     .rnorm PTS.diff              Team.fctr PTS.predict.Final.lm
## 846 Test  0.3200711      529   Los Angeles Clippers                 8289
## 850 Test  1.7399997     -123        Milwaukee Bucks                 8108
## 851 Test -0.2641936     -194 Minnesota Timberwolves                 7851
## 852 Test -0.4927864     -317    New Orleans Hornets                 7714
## 856 Test -0.4211144     -274     Philadelphia 76ers                 7640
## 857 Test  0.2165301     -530           Phoenix Suns                 7805
##     PTS.predict.Final.lm.err
## 846             3.637979e-12
## 850             2.728484e-12
## 851             2.728484e-12
## 852             2.728484e-12
## 856             2.728484e-12
## 857             2.728484e-12
```

```r
if (glb_is_classification && glb_is_binomial)
    glb_analytics_diag_plots(obs_df=glb_newobs_df, mdl_id=glb_fin_mdl_id, 
            prob_threshold=glb_models_df[glb_models_df$model_id == glb_sel_mdl_id, 
                                         "opt.prob.threshold.OOB"]) else
    glb_analytics_diag_plots(obs_df=glb_newobs_df, mdl_id=glb_fin_mdl_id)                  
```

```
## Warning in glb_analytics_diag_plots(obs_df = glb_newobs_df, mdl_id =
## glb_fin_mdl_id): Limiting important feature scatter plots to 5 out of 13
```

![](NBA_PTS2_files/figure-html/predict.data.new-2.png) ![](NBA_PTS2_files/figure-html/predict.data.new-3.png) ![](NBA_PTS2_files/figure-html/predict.data.new-4.png) ![](NBA_PTS2_files/figure-html/predict.data.new-5.png) ![](NBA_PTS2_files/figure-html/predict.data.new-6.png) 

```
##     SeasonEnd                   Team Playoffs  W  PTS oppPTS   FG  FGA
## 846      2013   Los Angeles Clippers        1 56 8289   7760 3160 6608
## 850      2013        Milwaukee Bucks        1 38 8108   8231 3128 7197
## 851      2013 Minnesota Timberwolves        0 31 7851   8045 2943 6702
## 852      2013    New Orleans Hornets        0 27 7714   8031 2955 6589
## 856      2013     Philadelphia 76ers        0 34 7640   7914 3059 6895
##      X2P X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV .rownames
## 846 2533 4856 627 1752 1342 1888  938 2475 1958 784 461 1197        11
## 850 2527 5527 601 1670 1251 1700 1068 2537 1876 685 550 1156        15
## 851 2493 5227 450 1475 1515 2042  973 2473 1836 700 387 1214        16
## 852 2420 5115 535 1474 1269 1636  988 2426 1721 520 440 1193        17
## 856 2541 5457 518 1438 1004 1377  896 2493 1867 609 384 1070        21
##     .src     .rnorm PTS.diff              Team.fctr PTS.predict.Final.lm
## 846 Test  0.3200711      529   Los Angeles Clippers                 8289
## 850 Test  1.7399997     -123        Milwaukee Bucks                 8108
## 851 Test -0.2641936     -194 Minnesota Timberwolves                 7851
## 852 Test -0.4927864     -317    New Orleans Hornets                 7714
## 856 Test -0.4211144     -274     Philadelphia 76ers                 7640
##     PTS.predict.Final.lm.err .label
## 846             3.637979e-12     11
## 850             2.728484e-12     15
## 851             2.728484e-12     16
## 852             2.728484e-12     17
## 856             2.728484e-12     21
```

![](NBA_PTS2_files/figure-html/predict.data.new-7.png) 

```r
if (glb_is_classification && glb_is_binomial) {
    submit_df <- glb_newobs_df[, c(glb_id_var, 
                                   paste0(glb_rsp_var_out, glb_fin_mdl_id, ".prob"))]
    names(submit_df)[2] <- "Probability1"
} else submit_df <- glb_newobs_df[, c(glb_id_var, 
                                   paste0(glb_rsp_var_out, glb_fin_mdl_id))]
write.csv(submit_df, 
    paste0(gsub(".", "_", paste0(glb_out_pfx, glb_fin_mdl_id), fixed=TRUE), 
           "_submit.csv"), row.names=FALSE)

# print(orderBy(~ -max.auc.OOB, glb_models_df[, c("model_id", 
#             "max.auc.OOB", "max.Accuracy.OOB")]))
if (glb_is_classification && glb_is_binomial)
    print(glb_models_df[glb_models_df$model_id == glb_sel_mdl_id, 
                        "opt.prob.threshold.OOB"])
print(sprintf("glb_sel_mdl_id: %s", glb_sel_mdl_id))
```

```
## [1] "glb_sel_mdl_id: All.X.lm"
```

```r
print(sprintf("glb_fin_mdl_id: %s", glb_fin_mdl_id))
```

```
## [1] "glb_fin_mdl_id: Final.lm"
```

```r
print(dim(glb_fitobs_df))
```

```
## [1] 835  25
```

```r
print(dsp_models_df)
```

```
##                     model_id min.RMSE.OOB  max.R.sq.OOB max.Adj.R.sq.fit
## 8                   All.X.lm 2.019113e-12  1.0000000000       1.00000000
## 9                  All.X.glm 2.019113e-12  1.0000000000               NA
## 6     Interact.High.cor.Y.lm 2.355896e+01  0.9973043928       0.99865487
## 7               Low.cor.X.lm 2.796915e+01  0.9962007077       0.99890827
## 12         All.X.no.rnorm.rf 1.519081e+02  0.8875517657               NA
## 3  Max.cor.Y.cv.0.cp.0.rpart 2.091754e+02  0.7874968789               NA
## 5               Max.cor.Y.lm 2.118347e+02  0.7820591878       0.93817073
## 4            Max.cor.Y.rpart 2.964608e+02  0.5731466783               NA
## 11      All.X.no.rnorm.rpart 2.964608e+02  0.5731466783               NA
## 1                     MFO.lm 4.536790e+02  0.0003645831      -0.00110768
## 2       Max.cor.Y.cv.0.rpart 4.537617e+02  0.0000000000               NA
## 10            All.X.bayesglm 8.369207e+02 -4.4319811910               NA
```

```r
if (glb_is_regression) {
    print(sprintf("%s OOB RMSE: %0.4f", glb_sel_mdl_id,
                  glb_models_df[glb_models_df$model_id == glb_sel_mdl_id, "min.RMSE.OOB"]))

    if (!is.null(glb_category_vars)) {
        stop("not implemented yet")
        tmp_OOBobs_df <- glb_OOBobs_df[, c(glb_category_vars, predct_accurate_var_name)]
        names(tmp_OOBobs_df)[length(names(tmp_OOBobs_df))] <- "accurate.OOB"
        aOOB_ctgry_df <- mycreate_xtab_df(tmp_OOBobs_df, names(tmp_OOBobs_df)) 
        aOOB_ctgry_df[is.na(aOOB_ctgry_df)] <- 0
        aOOB_ctgry_df <- mutate(aOOB_ctgry_df, 
                                .n.OOB = accurate.OOB.FALSE + accurate.OOB.TRUE,
                                max.accuracy.OOB = accurate.OOB.TRUE / .n.OOB)
        #intersect(names(glb_ctgry_df), names(aOOB_ctgry_df))
        glb_ctgry_df <- merge(glb_ctgry_df, aOOB_ctgry_df, all=TRUE)
        print(orderBy(~-accurate.OOB.FALSE, glb_ctgry_df))
    }
    
    if ((glb_rsp_var %in% names(glb_newobs_df)) &&
        !(any(is.na(glb_newobs_df[, glb_rsp_var])))) {
            pred_stats_df <- 
                mypredict_mdl(mdl=glb_models_lst[[glb_fin_mdl_id]], 
                              df=glb_newobs_df, 
                              rsp_var=glb_rsp_var, 
                              rsp_var_out=glb_rsp_var_out, 
                              model_id_method=glb_fin_mdl_id, 
                              label="new",
						      model_summaryFunction=glb_sel_mdl$control$summaryFunction, 
						      model_metric=glb_sel_mdl$metric,
						      model_metric_maximize=glb_sel_mdl$maximize,
						      ret_type="stats")        
            print(sprintf("%s prediction stats for glb_newobs_df:", glb_fin_mdl_id))
            print(pred_stats_df)
    }    
}    
```

```
## [1] "All.X.lm OOB RMSE: 0.0000"
```

```
## Warning in predict.lm(modelFit, newdata): prediction from a rank-deficient
## fit may be misleading
```

```
## [1] "Final.lm prediction stats for glb_newobs_df:"
##   model_id max.R.sq.new min.RMSE.new
## 1 Final.lm            1 2.019113e-12
```

```r
if (glb_is_classification) {
    print(sprintf("%s OOB confusion matrix & accuracy: ", glb_sel_mdl_id))
    print(t(confusionMatrix(glb_OOBobs_df[, paste0(glb_rsp_var_out, glb_sel_mdl_id)], 
                            glb_OOBobs_df[, glb_rsp_var])$table))

    if (!is.null(glb_category_vars)) {
        tmp_OOBobs_df <- glb_OOBobs_df[, c(glb_category_vars, predct_accurate_var_name)]
        names(tmp_OOBobs_df)[length(names(tmp_OOBobs_df))] <- "accurate.OOB"
        aOOB_ctgry_df <- mycreate_xtab_df(tmp_OOBobs_df, names(tmp_OOBobs_df)) 
        aOOB_ctgry_df[is.na(aOOB_ctgry_df)] <- 0
        aOOB_ctgry_df <- mutate(aOOB_ctgry_df, 
                                .n.OOB = accurate.OOB.FALSE + accurate.OOB.TRUE,
                                max.accuracy.OOB = accurate.OOB.TRUE / .n.OOB)
        #intersect(names(glb_ctgry_df), names(aOOB_ctgry_df))
        glb_ctgry_df <- merge(glb_ctgry_df, aOOB_ctgry_df, all=TRUE)
        print(orderBy(~-accurate.OOB.FALSE, glb_ctgry_df))
    }
    
    if ((glb_rsp_var %in% names(glb_newobs_df)) &&
        !(any(is.na(glb_newobs_df[, glb_rsp_var])))) {
        print(sprintf("%s new confusion matrix & accuracy: ", glb_fin_mdl_id))
        print(t(confusionMatrix(glb_newobs_df[, paste0(glb_rsp_var_out, glb_fin_mdl_id)], 
                                glb_newobs_df[, glb_rsp_var])$table))
    }    

}    

dsp_myCategory_conf_mtrx <- function(myCategory) {
    print(sprintf("%s OOB::myCategory=%s confusion matrix & accuracy: ", 
                  glb_sel_mdl_id, myCategory))
    print(t(confusionMatrix(
        glb_OOBobs_df[glb_OOBobs_df$myCategory == myCategory, 
                      paste0(glb_rsp_var_out, glb_sel_mdl_id)], 
        glb_OOBobs_df[glb_OOBobs_df$myCategory == myCategory, glb_rsp_var])$table))
    print(sum(glb_OOBobs_df[glb_OOBobs_df$myCategory == myCategory, 
                            predct_accurate_var_name]) / 
         nrow(glb_OOBobs_df[glb_OOBobs_df$myCategory == myCategory, ]))
    err_ids <- glb_OOBobs_df[(glb_OOBobs_df$myCategory == myCategory) & 
                             (!glb_OOBobs_df[, predct_accurate_var_name]), glb_id_var]

    OOB_FNerr_df <- glb_OOBobs_df[(glb_OOBobs_df$UniqueID %in% err_ids) & 
                               (glb_OOBobs_df$Popular == 1), 
                        c(
                            ".clusterid", 
                            "Popular", "Headline", "Snippet", "Abstract")]
    print(sprintf("%s OOB::myCategory=%s FN errors: %d", glb_sel_mdl_id, myCategory,
                  nrow(OOB_FNerr_df)))
    print(OOB_FNerr_df)

    OOB_FPerr_df <- glb_OOBobs_df[(glb_OOBobs_df$UniqueID %in% err_ids) & 
                               (glb_OOBobs_df$Popular == 0), 
                        c(
                            ".clusterid", 
                            "Popular", "Headline", "Snippet", "Abstract")]
    print(sprintf("%s OOB::myCategory=%s FP errors: %d", glb_sel_mdl_id, myCategory,
                  nrow(OOB_FPerr_df)))
    print(OOB_FPerr_df)
}
#dsp_myCategory_conf_mtrx(myCategory="OpEd#Opinion#")
#dsp_myCategory_conf_mtrx(myCategory="Business#Business Day#Dealbook")
#dsp_myCategory_conf_mtrx(myCategory="##")

# if (glb_is_classification) {
#     print("FN_OOB_ids:")
#     print(glb_OOBobs_df[glb_OOBobs_df$UniqueID %in% FN_OOB_ids, 
#                         grep(glb_rsp_var, names(glb_OOBobs_df), value=TRUE)])
#     print(glb_OOBobs_df[glb_OOBobs_df$UniqueID %in% FN_OOB_ids, 
#                         glb_txt_vars])
#     print(dsp_vctr <- colSums(glb_OOBobs_df[glb_OOBobs_df$UniqueID %in% FN_OOB_ids, 
#                         setdiff(grep("[HSA].", names(glb_OOBobs_df), value=TRUE),
#                                 union(myfind_chr_cols_df(glb_OOBobs_df),
#                     grep(".fctr", names(glb_OOBobs_df), fixed=TRUE, value=TRUE)))]))
# }

dsp_hdlpfx_results <- function(hdlpfx) {
    print(hdlpfx)
    print(glb_OOBobs_df[glb_OOBobs_df$Headline.pfx %in% c(hdlpfx), 
                        grep(glb_rsp_var, names(glb_OOBobs_df), value=TRUE)])
    print(glb_newobs_df[glb_newobs_df$Headline.pfx %in% c(hdlpfx), 
                        grep(glb_rsp_var, names(glb_newobs_df), value=TRUE)])
    print(dsp_vctr <- colSums(glb_newobs_df[glb_newobs_df$Headline.pfx %in% c(hdlpfx), 
                        setdiff(grep("[HSA]\\.", names(glb_newobs_df), value=TRUE),
                                union(myfind_chr_cols_df(glb_newobs_df),
                    grep(".fctr", names(glb_newobs_df), fixed=TRUE, value=TRUE)))]))
    print(dsp_vctr <- dsp_vctr[dsp_vctr != 0])
    print(glb_newobs_df[glb_newobs_df$Headline.pfx %in% c(hdlpfx), 
                        union(names(dsp_vctr), myfind_chr_cols_df(glb_newobs_df))])
}
#dsp_hdlpfx_results(hdlpfx="Ask Well::")

# print("myMisc::|OpEd|blank|blank|1:")
# print(glb_OOBobs_df[glb_OOBobs_df$UniqueID %in% c(6446), 
#                     grep(glb_rsp_var, names(glb_OOBobs_df), value=TRUE)])

# print(glb_OOBobs_df[glb_OOBobs_df$UniqueID %in% FN_OOB_ids, 
#                     c("WordCount", "WordCount.log", "myMultimedia",
#                       "NewsDesk", "SectionName", "SubsectionName")])
# print(mycreate_sqlxtab_df(glb_allobs_df[sel_obs(Headline.contains="[Vv]ideo"), ], 
#                           c(glb_rsp_var, "myMultimedia")))
# dsp_chisq.test(Headline.contains="[Vi]deo")
# print(glb_allobs_df[sel_obs(Headline.contains="[Vv]ideo"), 
#                           c(glb_rsp_var, "Popular", "myMultimedia", "Headline")])
# print(glb_allobs_df[sel_obs(Headline.contains="[Ee]bola", Popular=1), 
#                           c(glb_rsp_var, "Popular", "myMultimedia", "Headline",
#                             "NewsDesk", "SectionName", "SubsectionName")])
# print(subset(glb_feats_df, !is.na(importance))[,
#     c("is.ConditionalX.y", 
#       grep("importance", names(glb_feats_df), fixed=TRUE, value=TRUE))])
# print(subset(glb_feats_df, is.ConditionalX.y & is.na(importance))[,
#     c("is.ConditionalX.y", 
#       grep("importance", names(glb_feats_df), fixed=TRUE, value=TRUE))])
# print(subset(glb_feats_df, !is.na(importance))[,
#     c("zeroVar", "nzv", "myNearZV", 
#       grep("importance", names(glb_feats_df), fixed=TRUE, value=TRUE))])
# print(subset(glb_feats_df, is.na(importance))[,
#     c("zeroVar", "nzv", "myNearZV", 
#       grep("importance", names(glb_feats_df), fixed=TRUE, value=TRUE))])
print(orderBy(as.formula(paste0("~ -", glb_sel_mdl_id, ".importance")), glb_featsimp_df))
```

```
##        All.X.lm.importance   importance Final.lm.importance
## FG            1.000000e+02 1.000000e+02        1.000000e+02
## FT            9.337967e+01 9.337967e+01        9.337967e+01
## X2P           3.161786e+01 3.161786e+01        3.161786e+01
## ORB           2.657247e-13 2.657247e-13        2.657247e-13
## FGA           2.435779e-13 2.435779e-13        2.435779e-13
## .rnorm        2.368766e-13 2.368766e-13        2.368766e-13
## X2PA          2.175897e-13 2.175897e-13        2.175897e-13
## FTA           1.445086e-13 1.445086e-13        1.445086e-13
## STL           1.383737e-13 1.383737e-13        1.383737e-13
## DRB           1.149504e-13 1.149504e-13        1.149504e-13
## BLK           1.021147e-13 1.021147e-13        1.021147e-13
## AST           6.262793e-14 6.262793e-14        6.262793e-14
## TOV           0.000000e+00 0.000000e+00        0.000000e+00
```

```r
# players_df <- data.frame(id=c("Chavez", "Giambi", "Menechino", "Myers", "Pena"),
#                          OBP=c(0.338, 0.391, 0.369, 0.313, 0.361),
#                          SLG=c(0.540, 0.450, 0.374, 0.447, 0.500),
#                         cost=c(1400000, 1065000, 295000, 800000, 300000))
# players_df$RS.predict <- predict(glb_models_lst[[csm_mdl_id]], players_df)
# print(orderBy(~ -RS.predict, players_df))

if (length(diff <- setdiff(names(glb_trnobs_df), names(glb_allobs_df))) > 0)   
    print(diff)
for (col in setdiff(names(glb_trnobs_df), names(glb_allobs_df)))
    # Merge or cbind ?
    glb_allobs_df[glb_allobs_df$.src == "Train", col] <- glb_trnobs_df[, col]

if (length(diff <- setdiff(names(glb_fitobs_df), names(glb_allobs_df))) > 0)   
    print(diff)
if (length(diff <- setdiff(names(glb_OOBobs_df), names(glb_allobs_df))) > 0)   
    print(diff)

for (col in setdiff(names(glb_OOBobs_df), names(glb_allobs_df)))
    # Merge or cbind ?
    glb_allobs_df[glb_allobs_df$.lcn == "OOB", col] <- glb_OOBobs_df[, col]

if (length(diff <- setdiff(names(glb_newobs_df), names(glb_allobs_df))) > 0)   
    print(diff)

if (glb_save_envir)
    save(glb_feats_df, glb_allobs_df, 
         #glb_trnobs_df, glb_fitobs_df, glb_OOBobs_df, glb_newobs_df,
         glb_models_df, dsp_models_df, glb_models_lst, glb_model_type,
         glb_sel_mdl, glb_sel_mdl_id,
         glb_fin_mdl, glb_fin_mdl_id,
        file=paste0(glb_out_pfx, "prdnew_dsk.RData"))

rm(submit_df, tmp_OOBobs_df)
```

```
## Warning in rm(submit_df, tmp_OOBobs_df): object 'tmp_OOBobs_df' not found
```

```r
# tmp_replay_lst <- replay.petrisim(pn=glb_analytics_pn, 
#     replay.trans=(glb_analytics_avl_objs <- c(glb_analytics_avl_objs, 
#         "data.new.prediction")), flip_coord=TRUE)
# print(ggplot.petrinet(tmp_replay_lst[["pn"]]) + coord_flip())

glb_chunks_df <- myadd_chunk(glb_chunks_df, "display.session.info", major.inc=TRUE)
```

```
##                   label step_major step_minor    bgn    end elapsed
## 16     predict.data.new          9          0 92.539 97.692   5.153
## 17 display.session.info         10          0 97.693     NA      NA
```

Null Hypothesis ($\sf{H_{0}}$): mpg is not impacted by am_fctr.  
The variance by am_fctr appears to be independent. 
#```{r q1, cache=FALSE}
# print(t.test(subset(cars_df, am_fctr == "automatic")$mpg, 
#              subset(cars_df, am_fctr == "manual")$mpg, 
#              var.equal=FALSE)$conf)
#```
We reject the null hypothesis i.e. we have evidence to conclude that am_fctr impacts mpg (95% confidence). Manual transmission is better for miles per gallon versus automatic transmission.


```
##                      label step_major step_minor    bgn    end elapsed
## 11              fit.models          7          1 39.868 68.801  28.934
## 10              fit.models          7          0 21.412 39.867  18.456
## 12              fit.models          7          2 68.802 79.218  10.416
## 2             inspect.data          2          0  9.477 16.557   7.080
## 15       fit.data.training          8          1 87.348 92.539   5.191
## 16        predict.data.new          9          0 92.539 97.692   5.153
## 14       fit.data.training          8          0 83.195 87.348   4.153
## 13              fit.models          7          3 79.218 83.194   3.977
## 3               scrub.data          2          1 16.557 18.741   2.184
## 6         extract.features          3          0 18.852 20.168   1.316
## 8          select.features          5          0 20.441 21.119   0.678
## 1              import.data          1          0  9.041  9.476   0.435
## 9  partition.data.training          6          0 21.119 21.412   0.293
## 7             cluster.data          4          0 20.168 20.441   0.273
## 5      manage.missing.data          2          3 18.778 18.852   0.074
## 4           transform.data          2          2 18.741 18.778   0.037
##    duration
## 11   28.933
## 10   18.455
## 12   10.416
## 2     7.080
## 15    5.191
## 16    5.153
## 14    4.153
## 13    3.976
## 3     2.184
## 6     1.316
## 8     0.678
## 1     0.435
## 9     0.293
## 7     0.273
## 5     0.074
## 4     0.037
## [1] "Total Elapsed Time: 97.692 secs"
```

![](NBA_PTS2_files/figure-html/display.session.info-1.png) 

```
## R version 3.2.0 (2015-04-16)
## Platform: x86_64-apple-darwin13.4.0 (64-bit)
## Running under: OS X 10.10.3 (Yosemite)
## 
## locale:
## [1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8
## 
## attached base packages:
## [1] grid      parallel  stats     graphics  grDevices utils     datasets 
## [8] methods   base     
## 
## other attached packages:
##  [1] gdata_2.16.1        randomForest_4.6-10 arm_1.8-5          
##  [4] lme4_1.1-7          Rcpp_0.11.6         Matrix_1.2-1       
##  [7] MASS_7.3-40         rpart.plot_1.5.2    rpart_4.1-9        
## [10] reshape2_1.4.1      dplyr_0.4.1         plyr_1.8.2         
## [13] doMC_1.3.3          iterators_1.0.7     foreach_1.4.2      
## [16] doBy_4.5-13         survival_2.38-1     caret_6.0-47       
## [19] ggplot2_1.0.1       lattice_0.20-31    
## 
## loaded via a namespace (and not attached):
##  [1] compiler_3.2.0      RColorBrewer_1.1-2  formatR_1.2        
##  [4] nloptr_1.0.4        tools_3.2.0         digest_0.6.8       
##  [7] evaluate_0.7        nlme_3.1-120        gtable_0.1.2       
## [10] mgcv_1.8-6          DBI_0.3.1           yaml_2.1.13        
## [13] brglm_0.5-9         SparseM_1.6         proto_0.3-10       
## [16] coda_0.17-1         BradleyTerry2_1.0-6 stringr_1.0.0      
## [19] knitr_1.10.5        gtools_3.5.0        nnet_7.3-9         
## [22] rmarkdown_0.6.1     minqa_1.2.4         car_2.0-25         
## [25] magrittr_1.5        scales_0.2.4        codetools_0.2-11   
## [28] htmltools_0.2.6     splines_3.2.0       abind_1.4-3        
## [31] assertthat_0.1      pbkrtest_0.4-2      colorspace_1.2-6   
## [34] labeling_0.3        quantreg_5.11       stringi_0.4-1      
## [37] lazyeval_0.1.10     munsell_0.4.2
```
