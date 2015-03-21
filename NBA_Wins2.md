# NBA:  Basketball-Reference.com: W classification:: Wins2
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
    <glb_sel_mdl_id>: OOB_RMSE=<0.4f>; <feat1>=<imp>; <feat2>=<imp>

First run:
    All.X.lm:       OOB_RMSE=2.8780; new_RMSE=2.8779; oppPTS=100.00; Playoffs=24.27; PTS=11.73
    
-Playoffs features:
    All.X.lm:       OOB_RMSE=3.1678; new_RMSE=3.1677; oppPTS=100.00; PTS=7.42; BLK=4.87; .rnorm=0.00
    PTS.only.lm:    OOB_RMSE=3.0993; new_RMSE=3.0993; oppPTS=100.00; PTS=0.00
PTS.interact.lm:    OOB_RMSE=3.0888; new_RMSE=3.0888; oppPTS=100.00; PTS=81.56; oppPTS:PTS=0.00    

-Playoffs +PTS.diff features:
    Interact.High.cor.Y.lm:    
        OOB_RMSE=3.0558; new_RMSE=3.0558; PTS.diff=100.00; DRB=27.25
    
Classification results:
First run:
    <glb_sel_mdl_id>: OOB_conf_mtrx=[NY, YN]; <feat1>=<imp>; <feat2>=<imp>

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
glb_out_pfx <- "Wins2_"
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

glb_rsp_var_raw <- "W"

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
glb_exclude_vars_as_features <- c("Team.fctr", "Playoffs") 
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

![](NBA_Wins2_files/figure-html/set_global_options-1.png) 

```r
glb_analytics_avl_objs <- NULL

glb_chunks_df <- myadd_chunk(NULL, "import.data")
```

```
##         label step_major step_minor    bgn end elapsed
## 1 import.data          1          0 11.436  NA      NA
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
##          label step_major step_minor    bgn    end elapsed
## 1  import.data          1          0 11.436 11.902   0.466
## 2 inspect.data          2          0 11.902     NA      NA
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

![](NBA_Wins2_files/figure-html/inspect.data-1.png) 

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
## [1] "feat: SeasonEnd"
```

![](NBA_Wins2_files/figure-html/inspect.data-2.png) 

```
## [1] "feat: PTS"
```

![](NBA_Wins2_files/figure-html/inspect.data-3.png) 

```
## [1] "feat: oppPTS"
```

![](NBA_Wins2_files/figure-html/inspect.data-4.png) 

```
## [1] "feat: FG"
```

![](NBA_Wins2_files/figure-html/inspect.data-5.png) 

```
## [1] "feat: FGA"
```

![](NBA_Wins2_files/figure-html/inspect.data-6.png) 

```
## [1] "feat: X2P"
```

![](NBA_Wins2_files/figure-html/inspect.data-7.png) 

```
## [1] "feat: X2PA"
```

![](NBA_Wins2_files/figure-html/inspect.data-8.png) 

```
## [1] "feat: X3P"
```

![](NBA_Wins2_files/figure-html/inspect.data-9.png) 

```
## [1] "feat: X3PA"
```

![](NBA_Wins2_files/figure-html/inspect.data-10.png) 

```
## [1] "feat: FT"
```

![](NBA_Wins2_files/figure-html/inspect.data-11.png) 

```
## [1] "feat: FTA"
```

![](NBA_Wins2_files/figure-html/inspect.data-12.png) 

```
## [1] "feat: ORB"
```

![](NBA_Wins2_files/figure-html/inspect.data-13.png) 

```
## [1] "feat: DRB"
```

![](NBA_Wins2_files/figure-html/inspect.data-14.png) 

```
## [1] "feat: AST"
```

![](NBA_Wins2_files/figure-html/inspect.data-15.png) 

```
## [1] "feat: STL"
```

![](NBA_Wins2_files/figure-html/inspect.data-16.png) 

```
## [1] "feat: BLK"
```

![](NBA_Wins2_files/figure-html/inspect.data-17.png) 

```
## [1] "feat: TOV"
```

![](NBA_Wins2_files/figure-html/inspect.data-18.png) 

```
## [1] "feat: .rnorm"
```

![](NBA_Wins2_files/figure-html/inspect.data-19.png) 

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
## 2 inspect.data          2          0 11.902 20.193   8.291
## 3   scrub.data          2          1 20.193     NA      NA
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
## 3     scrub.data          2          1 20.193 22.958   2.765
## 4 transform.data          2          2 22.959     NA      NA
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
## 4      transform.data          2          2 22.959 22.991   0.032
## 5 manage.missing.data          2          3 22.991     NA      NA
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
## 5 manage.missing.data          2          3 22.991 23.073   0.082
## 6    extract.features          3          0 23.073     NA      NA
```

```r
extract.features_chunk_df <- myadd_chunk(NULL, "extract.features_bgn")
```

```
##                  label step_major step_minor    bgn end elapsed
## 1 extract.features_bgn          1          0 23.079  NA      NA
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
## 1                extract.features_bgn          1          0 23.079 23.087
## 2 extract.features_factorize.str.vars          2          0 23.088     NA
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
## 2 extract.features_factorize.str.vars          2          0 23.088 23.105
## 3                extract.features_end          3          0 23.105     NA
##   elapsed
## 2   0.017
## 3      NA
```

```r
myplt_chunk(extract.features_chunk_df)
```

```
##                                 label step_major step_minor    bgn    end
## 2 extract.features_factorize.str.vars          2          0 23.088 23.105
## 1                extract.features_bgn          1          0 23.079 23.087
##   elapsed duration
## 2   0.017    0.017
## 1   0.008    0.008
## [1] "Total Elapsed Time: 23.105 secs"
```

![](NBA_Wins2_files/figure-html/extract.features-1.png) 

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

![](NBA_Wins2_files/figure-html/extract.features-2.png) 

```r
glb_chunks_df <- myadd_chunk(glb_chunks_df, "cluster.data", major.inc=TRUE)
```

```
##              label step_major step_minor    bgn    end elapsed
## 6 extract.features          3          0 23.073 24.182   1.109
## 7     cluster.data          4          0 24.182     NA      NA
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
## 7    cluster.data          4          0 24.182 24.451   0.269
## 8 select.features          5          0 24.451     NA      NA
```

## Step `5.0: select features`

```r
print(glb_feats_df <- myselect_features(entity_df=glb_trnobs_df, 
                       exclude_vars_as_features=glb_exclude_vars_as_features, 
                       rsp_var=glb_rsp_var))
```

```
##                  id        cor.y exclude.as.feat   cor.y.abs
## PTS.diff   PTS.diff  0.970743263               0 0.970743263
## Playoffs   Playoffs  0.798675595               1 0.798675595
## DRB             DRB  0.470897497               0 0.470897497
## oppPTS       oppPTS -0.331572940               0 0.331572940
## AST             AST  0.320051771               0 0.320051771
## PTS             PTS  0.298825614               0 0.298825614
## TOV             TOV -0.243185881               0 0.243185881
## FT               FT  0.204906000               0 0.204906000
## BLK             BLK  0.203921004               0 0.203921004
## FG               FG  0.190396422               0 0.190396422
## Team.fctr Team.fctr -0.181789508               1 0.181789508
## FTA             FTA  0.161887310               0 0.161887310
## X3P             X3P  0.119044564               0 0.119044564
## STL             STL  0.116194398               0 0.116194398
## ORB             ORB -0.095736760               0 0.095736760
## X2PA           X2PA -0.087036534               0 0.087036534
## X3PA           X3PA  0.083286248               0 0.083286248
## FGA             FGA -0.071445659               0 0.071445659
## X2P             X2P  0.069278983               0 0.069278983
## .rnorm       .rnorm  0.006191449               0 0.006191449
## SeasonEnd SeasonEnd  0.000000000               0 0.000000000
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
## [1] "cor(W, X3P)=0.1190"
## [1] "cor(W, X3PA)=0.0833"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified X3PA as highly correlated with X3P
```

```
## [1] "cor(X2P, X2PA)=0.9653"
## [1] "cor(W, X2P)=0.0693"
## [1] "cor(W, X2PA)=-0.0870"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified X2P as highly correlated with X2PA
```

```
## [1] "cor(FT, FTA)=0.9505"
## [1] "cor(W, FT)=0.2049"
## [1] "cor(W, FTA)=0.1619"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified FTA as highly correlated with FT
```

```
## [1] "cor(FG, PTS)=0.9420"
## [1] "cor(W, FG)=0.1904"
## [1] "cor(W, PTS)=0.2988"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified FG as highly correlated with PTS
```

```
## [1] "cor(X2PA, X3P)=-0.9207"
## [1] "cor(W, X2PA)=-0.0870"
## [1] "cor(W, X3P)=0.1190"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified X2PA as highly correlated with X3P
```

```
## [1] "cor(FGA, oppPTS)=0.8309"
## [1] "cor(W, FGA)=-0.0714"
## [1] "cor(W, oppPTS)=-0.3316"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified FGA as highly correlated with oppPTS
```

```
## [1] "cor(oppPTS, PTS)=0.7891"
## [1] "cor(W, oppPTS)=-0.3316"
## [1] "cor(W, PTS)=0.2988"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified PTS as highly correlated with oppPTS
```

```
##           id        cor.y exclude.as.feat   cor.y.abs cor.high.X freqRatio
## 13  PTS.diff  0.970743263               0 0.970743263       <NA>  1.200000
## 11  Playoffs  0.798675595               1 0.798675595       <NA>  1.352113
## 4        DRB  0.470897497               0 0.470897497       <NA>  1.428571
## 2        AST  0.320051771               0 0.320051771       <NA>  1.000000
## 12       PTS  0.298825614               0 0.298825614     oppPTS  1.000000
## 7         FT  0.204906000               0 0.204906000       <NA>  1.166667
## 3        BLK  0.203921004               0 0.203921004       <NA>  1.125000
## 5         FG  0.190396422               0 0.190396422        PTS  1.000000
## 8        FTA  0.161887310               0 0.161887310         FT  1.000000
## 20       X3P  0.119044564               0 0.119044564       <NA>  1.500000
## 15       STL  0.116194398               0 0.116194398       <NA>  1.000000
## 21      X3PA  0.083286248               0 0.083286248        X3P  1.000000
## 18       X2P  0.069278983               0 0.069278983       X2PA  1.200000
## 1     .rnorm  0.006191449               0 0.006191449       <NA>  1.000000
## 14 SeasonEnd  0.000000000               0 0.000000000       <NA>  1.000000
## 6        FGA -0.071445659               0 0.071445659     oppPTS  1.000000
## 19      X2PA -0.087036534               0 0.087036534        X3P  1.000000
## 10       ORB -0.095736760               0 0.095736760       <NA>  1.000000
## 16 Team.fctr -0.181789508               1 0.181789508       <NA>  1.000000
## 17       TOV -0.243185881               0 0.243185881       <NA>  1.166667
## 9     oppPTS -0.331572940               0 0.331572940       <NA>  1.000000
##    percentUnique zeroVar   nzv myNearZV is.cor.y.abs.low
## 13     75.089820   FALSE FALSE    FALSE            FALSE
## 11      0.239521   FALSE FALSE    FALSE            FALSE
## 4      49.700599   FALSE FALSE    FALSE            FALSE
## 2      62.874251   FALSE FALSE    FALSE            FALSE
## 12     81.197605   FALSE FALSE    FALSE            FALSE
## 7      57.964072   FALSE FALSE    FALSE            FALSE
## 3      36.526946   FALSE FALSE    FALSE            FALSE
## 5      70.898204   FALSE FALSE    FALSE            FALSE
## 8      63.473054   FALSE FALSE    FALSE            FALSE
## 20     57.724551   FALSE FALSE    FALSE            FALSE
## 15     38.562874   FALSE FALSE    FALSE            FALSE
## 21     77.844311   FALSE FALSE    FALSE            FALSE
## 18     72.934132   FALSE FALSE    FALSE            FALSE
## 1     100.000000   FALSE FALSE    FALSE            FALSE
## 14      3.712575   FALSE FALSE    FALSE             TRUE
## 6      76.287425   FALSE FALSE    FALSE            FALSE
## 19     85.748503   FALSE FALSE    FALSE            FALSE
## 10     51.736527   FALSE FALSE    FALSE            FALSE
## 16      4.431138   FALSE FALSE    FALSE            FALSE
## 17     53.772455   FALSE FALSE    FALSE            FALSE
## 9      83.353293   FALSE FALSE    FALSE            FALSE
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
## Warning in loop_apply(n, do.ply): Removed 18 rows containing missing values
## (geom_point).
```

```
## Warning in loop_apply(n, do.ply): Removed 18 rows containing missing values
## (geom_point).
```

```
## Warning in loop_apply(n, do.ply): Removed 18 rows containing missing values
## (geom_point).
```

![](NBA_Wins2_files/figure-html/select.features-1.png) 

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
## 8         select.features          5          0 24.451 25.144   0.693
## 9 partition.data.training          6          0 25.144     NA      NA
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
## [1] "Newdata contains non-NA data for W; setting OOB to Newdata"
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
##   id exclude.as.feat rsp_var
## W  W            TRUE    TRUE
```

```r
glb_feats_df <- myrbind_df(glb_feats_df, add_feats_df)
if (glb_id_var != ".rownames")
    print(subset(glb_feats_df, rsp_var_raw | rsp_var | id_var)) else
    print(subset(glb_feats_df, rsp_var_raw | rsp_var))    
```

```
##   id cor.y exclude.as.feat cor.y.abs cor.high.X freqRatio percentUnique
## W  W    NA            TRUE        NA       <NA>        NA            NA
##   zeroVar nzv myNearZV is.cor.y.abs.low interaction.feat rsp_var_raw
## W      NA  NA       NA               NA               NA          NA
##   rsp_var
## W    TRUE
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
## 9  partition.data.training          6          0 25.144 25.441   0.297
## 10              fit.models          7          0 25.441     NA      NA
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

![](NBA_Wins2_files/figure-html/fit.models_0-1.png) ![](NBA_Wins2_files/figure-html/fit.models_0-2.png) ![](NBA_Wins2_files/figure-html/fit.models_0-3.png) ![](NBA_Wins2_files/figure-html/fit.models_0-4.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -30.056  -9.910   0.935   9.531  31.110 
## 
## Coefficients:
##             Estimate Std. Error t value Pr(>|t|)    
## (Intercept) 40.99783    0.44134  92.894   <2e-16 ***
## .rnorm       0.08103    0.45344   0.179    0.858    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 12.75 on 833 degrees of freedom
## Multiple R-squared:  3.833e-05,	Adjusted R-squared:  -0.001162 
## F-statistic: 0.03193 on 1 and 833 DF,  p-value: 0.8582
## 
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##   model_id model_method  feats max.nTuningRuns min.elapsedtime.everything
## 1   MFO.lm           lm .rnorm               0                       0.45
##   min.elapsedtime.final max.R.sq.fit min.RMSE.fit max.R.sq.OOB
## 1                 0.003 3.833404e-05     12.73295 0.0002542561
##   min.RMSE.OOB max.Adj.R.sq.fit
## 1     12.84499       -0.0011621
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
## [1] "    indep_vars: PTS.diff, DRB"
```

```
## Loading required package: rpart
```

```
## Fitting cp = 0.662 on full training set
```

```
## Loading required package: rpart.plot
```

![](NBA_Wins2_files/figure-html/fit.models_0-5.png) 

```
## Call:
## rpart(formula = .outcome ~ ., control = list(minsplit = 20, minbucket = 7, 
##     cp = 0, maxcompete = 4, maxsurrogate = 5, usesurrogate = 2, 
##     surrogatestyle = 0, maxdepth = 30, xval = 0))
##   n= 835 
## 
##          CP nsplit rel error
## 1 0.6623927      0         1
## 
## Node number 1: 835 observations
##   mean=41, MSE=162.1341 
## 
## n= 835 
## 
## node), split, n, deviance, yval
##       * denotes terminal node
## 
## 1) root 835 135382 41 *
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##               model_id model_method         feats max.nTuningRuns
## 1 Max.cor.Y.cv.0.rpart        rpart PTS.diff, DRB               0
##   min.elapsedtime.everything min.elapsedtime.final max.R.sq.fit
## 1                      0.603                 0.019            0
##   min.RMSE.fit max.R.sq.OOB min.RMSE.OOB
## 1     12.73319            0     12.84662
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
## [1] "    indep_vars: PTS.diff, DRB"
## Fitting cp = 0 on full training set
```

![](NBA_Wins2_files/figure-html/fit.models_0-6.png) 

```
## Call:
## rpart(formula = .outcome ~ ., control = list(minsplit = 20, minbucket = 7, 
##     cp = 0, maxcompete = 4, maxsurrogate = 5, usesurrogate = 2, 
##     surrogatestyle = 0, maxdepth = 30, xval = 0))
##   n= 835 
## 
##              CP nsplit  rel error
## 1  6.623927e-01      0 1.00000000
## 2  1.073213e-01      1 0.33760727
## 3  1.065599e-01      2 0.23028599
## 4  1.703383e-02      3 0.12372610
## 5  1.569972e-02      4 0.10669227
## 6  1.387508e-02      5 0.09099255
## 7  1.284477e-02      6 0.07711747
## 8  3.102830e-03      7 0.06427270
## 9  1.911398e-03      8 0.06116987
## 10 1.874313e-03      9 0.05925847
## 11 1.816205e-03     10 0.05738416
## 12 1.806570e-03     11 0.05556795
## 13 1.537916e-03     12 0.05376138
## 14 6.495652e-04     13 0.05222347
## 15 5.506328e-04     14 0.05157390
## 16 5.103544e-04     15 0.05102327
## 17 5.062708e-04     16 0.05051291
## 18 4.905921e-04     17 0.05000664
## 19 4.673565e-04     19 0.04902546
## 20 4.504875e-04     20 0.04855810
## 21 4.332054e-04     21 0.04810762
## 22 4.309939e-04     22 0.04767441
## 23 3.697523e-04     23 0.04724342
## 24 3.579615e-04     25 0.04650391
## 25 3.016490e-04     26 0.04614595
## 26 2.836418e-04     28 0.04554265
## 27 2.687710e-04     29 0.04525901
## 28 2.603248e-04     30 0.04499024
## 29 2.459988e-04     34 0.04394894
## 30 2.443703e-04     35 0.04370294
## 31 2.350252e-04     37 0.04321420
## 32 2.197229e-04     38 0.04297918
## 33 2.167317e-04     39 0.04275945
## 34 2.123464e-04     40 0.04254272
## 35 2.007520e-04     43 0.04190568
## 36 1.739257e-04     45 0.04150418
## 37 1.736716e-04     46 0.04133025
## 38 1.650639e-04     47 0.04115658
## 39 1.645247e-04     48 0.04099152
## 40 1.625911e-04     49 0.04082699
## 41 1.609864e-04     50 0.04066440
## 42 1.348997e-04     51 0.04050341
## 43 1.188876e-04     52 0.04036851
## 44 1.180149e-04     53 0.04024963
## 45 1.142798e-04     55 0.04001360
## 46 1.094386e-04     56 0.03989932
## 47 9.835073e-05     57 0.03978988
## 48 8.889782e-05     58 0.03969153
## 49 8.341044e-05     59 0.03960263
## 50 7.755832e-05     60 0.03951922
## 51 7.565611e-05     61 0.03944166
## 52 7.506639e-05     62 0.03936601
## 53 6.753783e-05     63 0.03929094
## 54 6.660433e-05     64 0.03922340
## 55 5.518541e-05     66 0.03909019
## 56 5.503109e-05     67 0.03903501
## 57 5.042522e-05     69 0.03892494
## 58 4.316789e-05     70 0.03887452
## 59 3.510214e-05     71 0.03883135
## 60 2.997770e-05     72 0.03879625
## 61 0.000000e+00     73 0.03876627
## 
## Variable importance
## PTS.diff      DRB 
##       81       19 
## 
## Node number 1: 835 observations,    complexity param=0.6623927
##   mean=41, MSE=162.1341 
##   left son=2 (373 obs) right son=3 (462 obs)
##   Primary splits:
##       PTS.diff < -32.5  to the left,  improve=0.6623927, (0 missing)
##       DRB      < 2377.5 to the left,  improve=0.1549703, (0 missing)
##   Surrogate splits:
##       DRB < 2377.5 to the left,  agree=0.668, adj=0.257, (0 split)
## 
## Node number 2: 373 observations,    complexity param=0.1065599
##   mean=29.46649, MSE=60.08802 
##   left son=4 (159 obs) right son=5 (214 obs)
##   Primary splits:
##       PTS.diff < -359.5 to the left,  improve=0.64366220, (0 missing)
##       DRB      < 2221.5 to the left,  improve=0.09562385, (0 missing)
##   Surrogate splits:
##       DRB < 2364.5 to the left,  agree=0.646, adj=0.17, (0 split)
## 
## Node number 3: 462 observations,    complexity param=0.1073213
##   mean=50.31169, MSE=50.418 
##   left son=6 (275 obs) right son=7 (187 obs)
##   Primary splits:
##       PTS.diff < 316    to the left,  improve=0.62376240, (0 missing)
##       DRB      < 2582.5 to the left,  improve=0.09916884, (0 missing)
##   Surrogate splits:
##       DRB < 2584   to the left,  agree=0.688, adj=0.23, (0 split)
## 
## Node number 4: 159 observations,    complexity param=0.01387508
##   mean=22.25157, MSE=23.09394 
##   left son=8 (67 obs) right son=9 (92 obs)
##   Primary splits:
##       PTS.diff < -572   to the left,  improve=0.5115656, (0 missing)
##       DRB      < 2198   to the left,  improve=0.1124918, (0 missing)
##   Surrogate splits:
##       DRB < 2215.5 to the left,  agree=0.686, adj=0.254, (0 split)
## 
## Node number 5: 214 observations,    complexity param=0.01569972
##   mean=34.8271, MSE=20.1617 
##   left son=10 (77 obs) right son=11 (137 obs)
##   Primary splits:
##       PTS.diff < -232.5 to the left,  improve=0.4926199, (0 missing)
##       DRB      < 2192   to the left,  improve=0.0265797, (0 missing)
##   Surrogate splits:
##       DRB < 2192   to the left,  agree=0.654, adj=0.039, (0 split)
## 
## Node number 6: 275 observations,    complexity param=0.01703383
##   mean=45.68727, MSE=18.17129 
##   left son=12 (159 obs) right son=13 (116 obs)
##   Primary splits:
##       PTS.diff < 160    to the left,  improve=0.46148200, (0 missing)
##       DRB      < 2581   to the left,  improve=0.05347047, (0 missing)
##   Surrogate splits:
##       DRB < 2568.5 to the left,  agree=0.618, adj=0.095, (0 split)
## 
## Node number 7: 187 observations,    complexity param=0.01284477
##   mean=57.1123, MSE=20.14247 
##   left son=14 (83 obs) right son=15 (104 obs)
##   Primary splits:
##       PTS.diff < 449.5  to the left,  improve=0.46167140, (0 missing)
##       DRB      < 2398.5 to the left,  improve=0.04033874, (0 missing)
##   Surrogate splits:
##       DRB < 2387   to the left,  agree=0.578, adj=0.048, (0 split)
## 
## Node number 8: 67 observations,    complexity param=0.001874313
##   mean=18.22388, MSE=12.26331 
##   left son=16 (40 obs) right son=17 (27 obs)
##   Primary splits:
##       PTS.diff < -670.5 to the left,  improve=0.3088308, (0 missing)
##       DRB      < 2179   to the left,  improve=0.1065289, (0 missing)
## 
## Node number 9: 92 observations,    complexity param=0.001816205
##   mean=25.18478, MSE=10.56368 
##   left son=18 (44 obs) right son=19 (48 obs)
##   Primary splits:
##       PTS.diff < -458.5 to the left,  improve=0.25300120, (0 missing)
##       DRB      < 2223.5 to the left,  improve=0.03677405, (0 missing)
##   Surrogate splits:
##       DRB < 2370.5 to the left,  agree=0.565, adj=0.091, (0 split)
## 
## Node number 10: 77 observations,    complexity param=0.0006495652
##   mean=30.62338, MSE=10.23478 
##   left son=20 (54 obs) right son=21 (23 obs)
##   Primary splits:
##       PTS.diff < -258.5 to the left,  improve=0.11158720, (0 missing)
##       DRB      < 2295   to the left,  improve=0.04230683, (0 missing)
##   Surrogate splits:
##       DRB < 2558.5 to the left,  agree=0.727, adj=0.087, (0 split)
## 
## Node number 11: 137 observations,    complexity param=0.001911398
##   mean=37.18978, MSE=10.22676 
##   left son=22 (61 obs) right son=23 (76 obs)
##   Primary splits:
##       PTS.diff < -125   to the left,  improve=0.18469430, (0 missing)
##       DRB      < 2543.5 to the right, improve=0.01643089, (0 missing)
##   Surrogate splits:
##       DRB < 2543.5 to the right, agree=0.599, adj=0.098, (0 split)
## 
## Node number 12: 159 observations,    complexity param=0.001537916
##   mean=43.21384, MSE=8.771884 
##   left son=24 (97 obs) right son=25 (62 obs)
##   Primary splits:
##       PTS.diff < 87.5   to the left,  improve=0.14928060, (0 missing)
##       DRB      < 2395.5 to the left,  improve=0.02616903, (0 missing)
##   Surrogate splits:
##       DRB < 2505.5 to the left,  agree=0.648, adj=0.097, (0 split)
## 
## Node number 13: 116 observations,    complexity param=0.00180657
##   mean=49.07759, MSE=11.17501 
##   left son=26 (67 obs) right son=27 (49 obs)
##   Primary splits:
##       PTS.diff < 250.5  to the left,  improve=0.18867300, (0 missing)
##       DRB      < 2579.5 to the left,  improve=0.06031685, (0 missing)
##   Surrogate splits:
##       DRB < 2300.5 to the right, agree=0.612, adj=0.082, (0 split)
## 
## Node number 14: 83 observations,    complexity param=0.0004332054
##   mean=53.6988, MSE=7.583974 
##   left son=28 (8 obs) right son=29 (75 obs)
##   Primary splits:
##       DRB      < 2358   to the left,  improve=0.09317080, (0 missing)
##       PTS.diff < 345    to the left,  improve=0.08817094, (0 missing)
## 
## Node number 15: 104 observations,    complexity param=0.00310283
##   mean=59.83654, MSE=13.44443 
##   left son=30 (91 obs) right son=31 (13 obs)
##   Primary splits:
##       PTS.diff < 696    to the left,  improve=0.30042980, (0 missing)
##       DRB      < 2597   to the right, improve=0.01673677, (0 missing)
##   Surrogate splits:
##       DRB < 2737   to the left,  agree=0.885, adj=0.077, (0 split)
## 
## Node number 16: 40 observations,    complexity param=0.0004504875
##   mean=16.625, MSE=7.984375 
##   left son=32 (9 obs) right son=33 (31 obs)
##   Primary splits:
##       PTS.diff < -829.5 to the left,  improve=0.1909602, (0 missing)
##       DRB      < 2358.5 to the left,  improve=0.1878669, (0 missing)
## 
## Node number 17: 27 observations,    complexity param=0.0005062708
##   mean=20.59259, MSE=9.20439 
##   left son=34 (20 obs) right son=35 (7 obs)
##   Primary splits:
##       DRB      < 2351   to the left,  improve=0.27579410, (0 missing)
##       PTS.diff < -632.5 to the left,  improve=0.02933362, (0 missing)
## 
## Node number 18: 44 observations,    complexity param=0.0001736716
##   mean=23.47727, MSE=9.249483 
##   left son=36 (25 obs) right son=37 (19 obs)
##   Primary splits:
##       PTS.diff < -498.5 to the left,  improve=0.05777229, (0 missing)
##       DRB      < 2346   to the left,  improve=0.02591213, (0 missing)
##   Surrogate splits:
##       DRB < 2414.5 to the right, agree=0.614, adj=0.105, (0 split)
## 
## Node number 19: 48 observations,    complexity param=0.000268771
##   mean=26.75, MSE=6.645833 
##   left son=38 (41 obs) right son=39 (7 obs)
##   Primary splits:
##       DRB      < 2500   to the left,  improve=0.11406510, (0 missing)
##       PTS.diff < -421.5 to the left,  improve=0.07594637, (0 missing)
## 
## Node number 20: 54 observations,    complexity param=0.0002603248
##   mean=29.92593, MSE=9.994513 
##   left son=40 (7 obs) right son=41 (47 obs)
##   Primary splits:
##       PTS.diff < -275   to the right, improve=0.05527359, (0 missing)
##       DRB      < 2282.5 to the left,  improve=0.02719884, (0 missing)
## 
## Node number 21: 23 observations,    complexity param=7.565611e-05
##   mean=32.26087, MSE=6.975425 
##   left son=42 (10 obs) right son=43 (13 obs)
##   Primary splits:
##       DRB      < 2441   to the right, improve=0.06384198, (0 missing)
##       PTS.diff < -252.5 to the left,  improve=0.04426378, (0 missing)
##   Surrogate splits:
##       PTS.diff < -236   to the right, agree=0.652, adj=0.2, (0 split)
## 
## Node number 22: 61 observations,    complexity param=0.0004905921
##   mean=35.65574, MSE=8.979844 
##   left son=44 (12 obs) right son=45 (49 obs)
##   Primary splits:
##       DRB      < 2333.5 to the left,  improve=0.11712930, (0 missing)
##       PTS.diff < -217.5 to the left,  improve=0.02757184, (0 missing)
## 
## Node number 23: 76 observations,    complexity param=0.0004673565
##   mean=38.42105, MSE=7.822715 
##   left son=46 (69 obs) right son=47 (7 obs)
##   Primary splits:
##       DRB      < 2272.5 to the right, improve=0.10642360, (0 missing)
##       PTS.diff < -80.5  to the left,  improve=0.07905675, (0 missing)
## 
## Node number 24: 97 observations,    complexity param=0.0003697523
##   mean=42.29897, MSE=6.333298 
##   left son=48 (48 obs) right son=49 (49 obs)
##   Primary splits:
##       DRB      < 2418.5 to the left,  improve=0.07025790, (0 missing)
##       PTS.diff < 70     to the left,  improve=0.03392774, (0 missing)
##   Surrogate splits:
##       PTS.diff < 4      to the right, agree=0.588, adj=0.167, (0 split)
## 
## Node number 25: 62 observations,    complexity param=0.000301649
##   mean=44.64516, MSE=9.228928 
##   left son=50 (12 obs) right son=51 (50 obs)
##   Primary splits:
##       DRB      < 2548   to the right, improve=0.07038449, (0 missing)
##       PTS.diff < 140    to the left,  improve=0.02736625, (0 missing)
##   Surrogate splits:
##       PTS.diff < 154.5  to the right, agree=0.823, adj=0.083, (0 split)
## 
## Node number 26: 67 observations,    complexity param=0.0001739257
##   mean=47.83582, MSE=8.107374 
##   left son=52 (60 obs) right son=53 (7 obs)
##   Primary splits:
##       DRB      < 2579.5 to the left,  improve=0.04334807, (0 missing)
##       PTS.diff < 235.5  to the right, improve=0.01378308, (0 missing)
## 
## Node number 27: 49 observations,    complexity param=0.0005506328
##   mean=50.77551, MSE=10.37818 
##   left son=54 (22 obs) right son=55 (27 obs)
##   Primary splits:
##       DRB      < 2455   to the left,  improve=0.14659050, (0 missing)
##       PTS.diff < 267.5  to the right, improve=0.04334227, (0 missing)
##   Surrogate splits:
##       PTS.diff < 271.5  to the left,  agree=0.612, adj=0.136, (0 split)
## 
## Node number 28: 8 observations
##   mean=51.125, MSE=9.109375 
## 
## Node number 29: 75 observations,    complexity param=0.0002459988
##   mean=53.97333, MSE=6.639289 
##   left son=58 (21 obs) right son=59 (54 obs)
##   Primary splits:
##       PTS.diff < 345    to the left,  improve=0.06688228, (0 missing)
##       DRB      < 2640.5 to the right, improve=0.02717955, (0 missing)
##   Surrogate splits:
##       DRB < 2404.5 to the left,  agree=0.76, adj=0.143, (0 split)
## 
## Node number 30: 91 observations,    complexity param=0.0003579615
##   mean=59.07692, MSE=8.906171 
##   left son=60 (56 obs) right son=61 (35 obs)
##   Primary splits:
##       PTS.diff < 582.5  to the left,  improve=0.05979499, (0 missing)
##       DRB      < 2663.5 to the right, improve=0.03002995, (0 missing)
##   Surrogate splits:
##       DRB < 2277   to the right, agree=0.626, adj=0.029, (0 split)
## 
## Node number 31: 13 observations
##   mean=65.15385, MSE=12.89941 
## 
## Node number 32: 9 observations
##   mean=14.33333, MSE=7.555556 
## 
## Node number 33: 31 observations,    complexity param=0.0001609864
##   mean=17.29032, MSE=6.141519 
##   left son=66 (17 obs) right son=67 (14 obs)
##   Primary splits:
##       DRB      < 2358.5 to the left,  improve=0.11447550, (0 missing)
##       PTS.diff < -740.5 to the left,  improve=0.04735753, (0 missing)
##   Surrogate splits:
##       PTS.diff < -744   to the left,  agree=0.645, adj=0.214, (0 split)
## 
## Node number 34: 20 observations,    complexity param=6.753783e-05
##   mean=19.65, MSE=8.7275 
##   left son=68 (13 obs) right son=69 (7 obs)
##   Primary splits:
##       PTS.diff < -600   to the left,  improve=0.05238274, (0 missing)
##       DRB      < 2237.5 to the right, improve=0.02722441, (0 missing)
##   Surrogate splits:
##       DRB < 2278.5 to the left,  agree=0.7, adj=0.143, (0 split)
## 
## Node number 35: 7 observations
##   mean=23.28571, MSE=0.7755102 
## 
## Node number 36: 25 observations,    complexity param=5.518541e-05
##   mean=22.84, MSE=9.4944 
##   left son=72 (9 obs) right son=73 (16 obs)
##   Primary splits:
##       PTS.diff < -519.5 to the right, improve=0.03147586, (0 missing)
##       DRB      < 2304.5 to the right, improve=0.03130868, (0 missing)
##   Surrogate splits:
##       DRB < 2561.5 to the right, agree=0.72, adj=0.222, (0 split)
## 
## Node number 37: 19 observations
##   mean=24.31579, MSE=7.689751 
## 
## Node number 38: 41 observations,    complexity param=0.0002197229
##   mean=26.39024, MSE=5.555027 
##   left son=76 (19 obs) right son=77 (22 obs)
##   Primary splits:
##       PTS.diff < -411   to the left,  improve=0.1306069, (0 missing)
##       DRB      < 2435.5 to the right, improve=0.1226056, (0 missing)
##   Surrogate splits:
##       DRB < 2375.5 to the left,  agree=0.61, adj=0.158, (0 split)
## 
## Node number 39: 7 observations
##   mean=28.85714, MSE=7.836735 
## 
## Node number 40: 7 observations
##   mean=28, MSE=2.857143 
## 
## Node number 41: 47 observations,    complexity param=0.0002603248
##   mean=30.21277, MSE=10.42282 
##   left son=82 (38 obs) right son=83 (9 obs)
##   Primary splits:
##       PTS.diff < -293.5 to the left,  improve=0.05565563, (0 missing)
##       DRB      < 2384   to the right, improve=0.02647364, (0 missing)
## 
## Node number 42: 10 observations
##   mean=31.5, MSE=2.65 
## 
## Node number 43: 13 observations
##   mean=32.84615, MSE=9.514793 
## 
## Node number 44: 12 observations
##   mean=33.58333, MSE=4.076389 
## 
## Node number 45: 49 observations,    complexity param=0.0004905921
##   mean=36.16327, MSE=8.871304 
##   left son=90 (38 obs) right son=91 (11 obs)
##   Primary splits:
##       DRB      < 2392.5 to the right, improve=0.15798410, (0 missing)
##       PTS.diff < -151   to the left,  improve=0.01881491, (0 missing)
## 
## Node number 46: 69 observations,    complexity param=0.0004309939
##   mean=38.13043, MSE=7.359798 
##   left son=92 (36 obs) right son=93 (33 obs)
##   Primary splits:
##       PTS.diff < -80.5  to the left,  improve=0.1148992, (0 missing)
##       DRB      < 2431.5 to the left,  improve=0.0224669, (0 missing)
##   Surrogate splits:
##       DRB < 2405   to the left,  agree=0.667, adj=0.303, (0 split)
## 
## Node number 47: 7 observations
##   mean=41.28571, MSE=3.346939 
## 
## Node number 48: 48 observations,    complexity param=0.0002836418
##   mean=41.625, MSE=6.484375 
##   left son=96 (40 obs) right son=97 (8 obs)
##   Primary splits:
##       PTS.diff < -15    to the right, improve=0.12337350, (0 missing)
##       DRB      < 2375   to the right, improve=0.03000446, (0 missing)
## 
## Node number 49: 49 observations,    complexity param=0.0003697523
##   mean=42.95918, MSE=5.304456 
##   left son=98 (21 obs) right son=99 (28 obs)
##   Primary splits:
##       PTS.diff < 1.5    to the left,  improve=0.2191230, (0 missing)
##       DRB      < 2449.5 to the right, improve=0.1003487, (0 missing)
##   Surrogate splits:
##       DRB < 2470.5 to the right, agree=0.612, adj=0.095, (0 split)
## 
## Node number 50: 12 observations
##   mean=43, MSE=5 
## 
## Node number 51: 50 observations,    complexity param=0.000301649
##   mean=45.04, MSE=9.4384 
##   left son=102 (42 obs) right son=103 (8 obs)
##   Primary splits:
##       PTS.diff < 140    to the left,  improve=0.08773127, (0 missing)
##       DRB      < 2390   to the left,  improve=0.02113823, (0 missing)
## 
## Node number 52: 60 observations,    complexity param=0.0001650639
##   mean=47.63333, MSE=7.298889 
##   left son=104 (19 obs) right son=105 (41 obs)
##   Primary splits:
##       DRB      < 2474   to the right, improve=0.05102759, (0 missing)
##       PTS.diff < 200.5  to the right, improve=0.02762978, (0 missing)
## 
## Node number 53: 7 observations
##   mean=49.57143, MSE=11.67347 
## 
## Node number 54: 22 observations,    complexity param=0.0002350252
##   mean=49.40909, MSE=8.605372 
##   left son=108 (14 obs) right son=109 (8 obs)
##   Primary splits:
##       PTS.diff < 275    to the right, improve=0.1680672, (0 missing)
##       DRB      < 2294   to the right, improve=0.0923821, (0 missing)
## 
## Node number 55: 27 observations,    complexity param=0.0005103544
##   mean=51.88889, MSE=9.061728 
##   left son=110 (16 obs) right son=111 (11 obs)
##   Primary splits:
##       DRB      < 2516.5 to the right, improve=0.28239570, (0 missing)
##       PTS.diff < 286.5  to the left,  improve=0.01117785, (0 missing)
##   Surrogate splits:
##       PTS.diff < 259.5  to the right, agree=0.667, adj=0.182, (0 split)
## 
## Node number 58: 21 observations,    complexity param=0.0001188876
##   mean=52.90476, MSE=7.038549 
##   left son=116 (14 obs) right son=117 (7 obs)
##   Primary splits:
##       PTS.diff < 321.5  to the right, improve=0.10889180, (0 missing)
##       DRB      < 2461.5 to the right, improve=0.08521263, (0 missing)
## 
## Node number 59: 54 observations,    complexity param=0.0002123464
##   mean=54.38889, MSE=5.867284 
##   left son=118 (45 obs) right son=119 (9 obs)
##   Primary splits:
##       PTS.diff < 356    to the right, improve=0.08847975, (0 missing)
##       DRB      < 2639   to the right, improve=0.04639663, (0 missing)
## 
## Node number 60: 56 observations,    complexity param=0.0002443703
##   mean=58.5, MSE=8.892857 
##   left son=120 (7 obs) right son=121 (49 obs)
##   Primary splits:
##       PTS.diff < 557.5  to the right, improve=0.03614458, (0 missing)
##       DRB      < 2491.5 to the right, improve=0.03114963, (0 missing)
## 
## Node number 61: 35 observations,    complexity param=0.000200752
##   mean=60, MSE=7.542857 
##   left son=122 (12 obs) right son=123 (23 obs)
##   Primary splits:
##       DRB      < 2601.5 to the right, improve=0.09414800, (0 missing)
##       PTS.diff < 629.5  to the right, improve=0.01668786, (0 missing)
##   Surrogate splits:
##       PTS.diff < 586    to the left,  agree=0.686, adj=0.083, (0 split)
## 
## Node number 66: 17 observations
##   mean=16.52941, MSE=7.425606 
## 
## Node number 67: 14 observations
##   mean=18.21429, MSE=3.02551 
## 
## Node number 68: 13 observations
##   mean=19.15385, MSE=5.207101 
## 
## Node number 69: 7 observations
##   mean=20.57143, MSE=13.95918 
## 
## Node number 72: 9 observations
##   mean=22.11111, MSE=1.654321 
## 
## Node number 73: 16 observations
##   mean=23.25, MSE=13.4375 
## 
## Node number 76: 19 observations
##   mean=25.47368, MSE=4.880886 
## 
## Node number 77: 22 observations,    complexity param=2.99777e-05
##   mean=27.18182, MSE=4.785124 
##   left son=154 (14 obs) right son=155 (8 obs)
##   Primary splits:
##       DRB      < 2340   to the right, improve=0.03855169, (0 missing)
##       PTS.diff < -383   to the right, improve=0.02538860, (0 missing)
##   Surrogate splits:
##       PTS.diff < -378   to the right, agree=0.682, adj=0.125, (0 split)
## 
## Node number 82: 38 observations,    complexity param=0.0002603248
##   mean=29.84211, MSE=9.238227 
##   left son=164 (10 obs) right son=165 (28 obs)
##   Primary splits:
##       DRB      < 2275   to the left,  improve=0.06963483, (0 missing)
##       PTS.diff < -350.5 to the right, improve=0.02518326, (0 missing)
##   Surrogate splits:
##       PTS.diff < -296   to the right, agree=0.763, adj=0.1, (0 split)
## 
## Node number 83: 9 observations
##   mean=31.77778, MSE=12.39506 
## 
## Node number 90: 38 observations,    complexity param=0.0001645247
##   mean=35.52632, MSE=8.196676 
##   left son=180 (20 obs) right son=181 (18 obs)
##   Primary splits:
##       PTS.diff < -164.5 to the left,  improve=0.07151065, (0 missing)
##       DRB      < 2474   to the left,  improve=0.04964297, (0 missing)
##   Surrogate splits:
##       DRB < 2435   to the right, agree=0.632, adj=0.222, (0 split)
## 
## Node number 91: 11 observations
##   mean=38.36364, MSE=4.958678 
## 
## Node number 92: 36 observations,    complexity param=0.0001348997
##   mean=37.25, MSE=6.131944 
##   left son=184 (22 obs) right son=185 (14 obs)
##   Primary splits:
##       PTS.diff < -111.5 to the right, improve=0.08273154, (0 missing)
##       DRB      < 2335   to the right, improve=0.01634510, (0 missing)
##   Surrogate splits:
##       DRB < 2286.5 to the right, agree=0.667, adj=0.143, (0 split)
## 
## Node number 93: 33 observations,    complexity param=8.341044e-05
##   mean=39.09091, MSE=6.931129 
##   left son=186 (25 obs) right son=187 (8 obs)
##   Primary splits:
##       PTS.diff < -70    to the right, improve=0.04937003, (0 missing)
##       DRB      < 2431.5 to the left,  improve=0.02868857, (0 missing)
## 
## Node number 96: 40 observations,    complexity param=5.503109e-05
##   mean=41.225, MSE=5.624375 
##   left son=192 (19 obs) right son=193 (21 obs)
##   Primary splits:
##       PTS.diff < 41     to the left,  improve=0.030513280, (0 missing)
##       DRB      < 2369   to the right, improve=0.009797385, (0 missing)
##   Surrogate splits:
##       DRB < 2357.5 to the right, agree=0.6, adj=0.158, (0 split)
## 
## Node number 97: 8 observations
##   mean=43.625, MSE=5.984375 
## 
## Node number 98: 21 observations,    complexity param=7.755832e-05
##   mean=41.71429, MSE=3.918367 
##   left son=196 (14 obs) right son=197 (7 obs)
##   Primary splits:
##       PTS.diff < -24    to the right, improve=0.12760420, (0 missing)
##       DRB      < 2511.5 to the right, improve=0.08012821, (0 missing)
##   Surrogate splits:
##       DRB < 2559   to the left,  agree=0.714, adj=0.143, (0 split)
## 
## Node number 99: 28 observations,    complexity param=0.0001625911
##   mean=43.89286, MSE=4.309949 
##   left son=198 (21 obs) right son=199 (7 obs)
##   Primary splits:
##       DRB      < 2446.5 to the right, improve=0.18240110, (0 missing)
##       PTS.diff < 67.5   to the left,  improve=0.06658775, (0 missing)
##   Surrogate splits:
##       PTS.diff < 12.5   to the right, agree=0.821, adj=0.286, (0 split)
## 
## Node number 102: 42 observations,    complexity param=0.0001142798
##   mean=44.64286, MSE=7.610544 
##   left son=204 (35 obs) right son=205 (7 obs)
##   Primary splits:
##       DRB      < 2517   to the left,  improve=0.04840223, (0 missing)
##       PTS.diff < 128    to the right, improve=0.03650279, (0 missing)
## 
## Node number 103: 8 observations
##   mean=47.125, MSE=13.85938 
## 
## Node number 104: 19 observations
##   mean=46.73684, MSE=6.404432 
## 
## Node number 105: 41 observations,    complexity param=0.0001180149
##   mean=48.04878, MSE=7.168352 
##   left son=210 (19 obs) right son=211 (22 obs)
##   Primary splits:
##       PTS.diff < 200.5  to the right, improve=0.04747384, (0 missing)
##       DRB      < 2355.5 to the left,  improve=0.04491725, (0 missing)
##   Surrogate splits:
##       DRB < 2283   to the left,  agree=0.634, adj=0.211, (0 split)
## 
## Node number 108: 14 observations
##   mean=48.5, MSE=7.25 
## 
## Node number 109: 8 observations
##   mean=51, MSE=7 
## 
## Node number 110: 16 observations
##   mean=50.5625, MSE=8.246094 
## 
## Node number 111: 11 observations
##   mean=53.81818, MSE=3.966942 
## 
## Node number 116: 14 observations
##   mean=52.28571, MSE=6.061224 
## 
## Node number 117: 7 observations
##   mean=54.14286, MSE=6.693878 
## 
## Node number 118: 45 observations,    complexity param=0.0002123464
##   mean=54.06667, MSE=5.662222 
##   left son=236 (17 obs) right son=237 (28 obs)
##   Primary splits:
##       DRB      < 2576.5 to the right, improve=0.09657270, (0 missing)
##       PTS.diff < 429.5  to the left,  improve=0.08992793, (0 missing)
##   Surrogate splits:
##       PTS.diff < 447.5  to the right, agree=0.644, adj=0.059, (0 split)
## 
## Node number 119: 9 observations
##   mean=56, MSE=3.777778 
## 
## Node number 120: 7 observations
##   mean=57, MSE=8.857143 
## 
## Node number 121: 49 observations,    complexity param=0.0002443703
##   mean=58.71429, MSE=8.530612 
##   left son=242 (42 obs) right son=243 (7 obs)
##   Primary splits:
##       PTS.diff < 542.5  to the left,  improve=0.11523130, (0 missing)
##       DRB      < 2491.5 to the right, improve=0.07762948, (0 missing)
## 
## Node number 122: 12 observations
##   mean=58.83333, MSE=5.138889 
## 
## Node number 123: 23 observations,    complexity param=0.000200752
##   mean=60.6087, MSE=7.716446 
##   left son=246 (13 obs) right son=247 (10 obs)
##   Primary splits:
##       DRB      < 2502   to the left,  improve=0.16622510, (0 missing)
##       PTS.diff < 629.5  to the right, improve=0.05006784, (0 missing)
##   Surrogate splits:
##       PTS.diff < 629.5  to the right, agree=0.609, adj=0.1, (0 split)
## 
## Node number 154: 14 observations
##   mean=26.85714, MSE=3.979592 
## 
## Node number 155: 8 observations
##   mean=27.75, MSE=5.6875 
## 
## Node number 164: 10 observations
##   mean=28.5, MSE=10.85 
## 
## Node number 165: 28 observations,    complexity param=0.0002603248
##   mean=30.32143, MSE=7.789541 
##   left son=330 (20 obs) right son=331 (8 obs)
##   Primary splits:
##       DRB      < 2384   to the right, improve=0.27249060, (0 missing)
##       PTS.diff < -339   to the right, improve=0.05589215, (0 missing)
## 
## Node number 180: 20 observations,    complexity param=8.889782e-05
##   mean=34.8, MSE=6.06 
##   left son=360 (13 obs) right son=361 (7 obs)
##   Primary splits:
##       DRB      < 2519   to the left,  improve=0.09930004, (0 missing)
##       PTS.diff < -203   to the left,  improve=0.05012376, (0 missing)
##   Surrogate splits:
##       PTS.diff < -175.5 to the left,  agree=0.7, adj=0.143, (0 split)
## 
## Node number 181: 18 observations
##   mean=36.33333, MSE=9.333333 
## 
## Node number 184: 22 observations,    complexity param=4.316789e-05
##   mean=36.68182, MSE=5.671488 
##   left son=368 (8 obs) right son=369 (14 obs)
##   Primary splits:
##       PTS.diff < -89.5  to the right, improve=0.046838410, (0 missing)
##       DRB      < 2371.5 to the left,  improve=0.009484778, (0 missing)
## 
## Node number 185: 14 observations
##   mean=38.14286, MSE=5.55102 
## 
## Node number 186: 25 observations,    complexity param=5.042522e-05
##   mean=38.76, MSE=5.4624 
##   left son=372 (15 obs) right son=373 (10 obs)
##   Primary splits:
##       DRB      < 2458.5 to the left,  improve=0.04999024, (0 missing)
##       PTS.diff < -54.5  to the left,  improve=0.02559362, (0 missing)
##   Surrogate splits:
##       PTS.diff < -39.5  to the left,  agree=0.68, adj=0.2, (0 split)
## 
## Node number 187: 8 observations
##   mean=40.125, MSE=10.10938 
## 
## Node number 192: 19 observations
##   mean=40.78947, MSE=8.481994 
## 
## Node number 193: 21 observations,    complexity param=5.503109e-05
##   mean=41.61905, MSE=2.712018 
##   left son=386 (12 obs) right son=387 (9 obs)
##   Primary splits:
##       PTS.diff < 63     to the right, improve=0.14109530, (0 missing)
##       DRB      < 2372   to the right, improve=0.02048495, (0 missing)
##   Surrogate splits:
##       DRB < 2382   to the right, agree=0.619, adj=0.111, (0 split)
## 
## Node number 196: 14 observations
##   mean=41.21429, MSE=4.311224 
## 
## Node number 197: 7 observations
##   mean=42.71429, MSE=1.632653 
## 
## Node number 198: 21 observations,    complexity param=0.0001094386
##   mean=43.38095, MSE=3.854875 
##   left son=396 (10 obs) right son=397 (11 obs)
##   Primary splits:
##       PTS.diff < 72.5   to the left,  improve=0.1830214, (0 missing)
##       DRB      < 2493.5 to the right, improve=0.0342246, (0 missing)
##   Surrogate splits:
##       DRB < 2484.5 to the right, agree=0.667, adj=0.3, (0 split)
## 
## Node number 199: 7 observations
##   mean=45.42857, MSE=2.530612 
## 
## Node number 204: 35 observations,    complexity param=6.660433e-05
##   mean=44.37143, MSE=7.433469 
##   left son=408 (11 obs) right son=409 (24 obs)
##   Primary splits:
##       PTS.diff < 106    to the left,  improve=0.02558420, (0 missing)
##       DRB      < 2327.5 to the right, improve=0.02126071, (0 missing)
##   Surrogate splits:
##       DRB < 2189   to the left,  agree=0.714, adj=0.091, (0 split)
## 
## Node number 205: 7 observations
##   mean=46, MSE=6.285714 
## 
## Node number 210: 19 observations
##   mean=47.42105, MSE=6.34903 
## 
## Node number 211: 22 observations,    complexity param=0.0001180149
##   mean=48.59091, MSE=7.241736 
##   left son=422 (10 obs) right son=423 (12 obs)
##   Primary splits:
##       PTS.diff < 179    to the left,  improve=0.1129910, (0 missing)
##       DRB      < 2386.5 to the right, improve=0.0175844, (0 missing)
##   Surrogate splits:
##       DRB < 2418   to the right, agree=0.682, adj=0.3, (0 split)
## 
## Node number 236: 17 observations
##   mean=53.11765, MSE=5.633218 
## 
## Node number 237: 28 observations,    complexity param=0.0002123464
##   mean=54.64286, MSE=4.80102 
##   left son=474 (20 obs) right son=475 (8 obs)
##   Primary splits:
##       PTS.diff < 430    to the left,  improve=0.24997340, (0 missing)
##       DRB      < 2530   to the left,  improve=0.07970244, (0 missing)
## 
## Node number 242: 42 observations,    complexity param=0.0002167317
##   mean=58.30952, MSE=7.975624 
##   left son=484 (26 obs) right son=485 (16 obs)
##   Primary splits:
##       DRB      < 2491.5 to the right, improve=0.08759302, (0 missing)
##       PTS.diff < 476.5  to the right, improve=0.02121113, (0 missing)
##   Surrogate splits:
##       PTS.diff < 462    to the right, agree=0.667, adj=0.125, (0 split)
## 
## Node number 243: 7 observations
##   mean=61.14286, MSE=4.979592 
## 
## Node number 246: 13 observations
##   mean=59.61538, MSE=4.544379 
## 
## Node number 247: 10 observations
##   mean=61.9, MSE=8.89 
## 
## Node number 330: 20 observations,    complexity param=7.506639e-05
##   mean=29.4, MSE=5.34 
##   left son=660 (7 obs) right son=661 (13 obs)
##   Primary splits:
##       DRB      < 2434.5 to the left,  improve=0.09515578, (0 missing)
##       PTS.diff < -323   to the right, improve=0.01613368, (0 missing)
##   Surrogate splits:
##       PTS.diff < -326.5 to the right, agree=0.75, adj=0.286, (0 split)
## 
## Node number 331: 8 observations
##   mean=32.625, MSE=6.484375 
## 
## Node number 360: 13 observations
##   mean=34.23077, MSE=4.792899 
## 
## Node number 361: 7 observations
##   mean=35.85714, MSE=6.693878 
## 
## Node number 368: 8 observations
##   mean=36, MSE=7.5 
## 
## Node number 369: 14 observations
##   mean=37.07143, MSE=4.209184 
## 
## Node number 372: 15 observations
##   mean=38.33333, MSE=3.555556 
## 
## Node number 373: 10 observations
##   mean=39.4, MSE=7.64 
## 
## Node number 386: 12 observations
##   mean=41.08333, MSE=3.076389 
## 
## Node number 387: 9 observations
##   mean=42.33333, MSE=1.333333 
## 
## Node number 396: 10 observations
##   mean=42.5, MSE=1.85 
## 
## Node number 397: 11 observations
##   mean=44.18182, MSE=4.330579 
## 
## Node number 408: 11 observations
##   mean=43.72727, MSE=2.561983 
## 
## Node number 409: 24 observations,    complexity param=6.660433e-05
##   mean=44.66667, MSE=9.388889 
##   left son=818 (9 obs) right son=819 (15 obs)
##   Primary splits:
##       PTS.diff < 128    to the right, improve=0.0504931, (0 missing)
##       DRB      < 2454   to the right, improve=0.0156250, (0 missing)
##   Surrogate splits:
##       DRB < 2217.5 to the left,  agree=0.708, adj=0.222, (0 split)
## 
## Node number 422: 10 observations
##   mean=47.6, MSE=5.44 
## 
## Node number 423: 12 observations
##   mean=49.41667, MSE=7.243056 
## 
## Node number 474: 20 observations,    complexity param=3.510214e-05
##   mean=53.95, MSE=3.9475 
##   left son=948 (7 obs) right son=949 (13 obs)
##   Primary splits:
##       PTS.diff < 406.5  to the right, improve=0.06019250, (0 missing)
##       DRB      < 2465   to the right, improve=0.03124108, (0 missing)
## 
## Node number 475: 8 observations
##   mean=56.375, MSE=2.734375 
## 
## Node number 484: 26 observations,    complexity param=9.835073e-05
##   mean=57.65385, MSE=8.610947 
##   left son=968 (11 obs) right son=969 (15 obs)
##   Primary splits:
##       PTS.diff < 488    to the left,  improve=0.05947223, (0 missing)
##       DRB      < 2559   to the left,  improve=0.02628414, (0 missing)
##   Surrogate splits:
##       DRB < 2642   to the right, agree=0.654, adj=0.182, (0 split)
## 
## Node number 485: 16 observations
##   mean=59.375, MSE=5.109375 
## 
## Node number 660: 7 observations
##   mean=28.42857, MSE=3.387755 
## 
## Node number 661: 13 observations
##   mean=29.92308, MSE=5.609467 
## 
## Node number 818: 9 observations
##   mean=43.77778, MSE=11.50617 
## 
## Node number 819: 15 observations
##   mean=45.2, MSE=7.36 
## 
## Node number 948: 7 observations
##   mean=53.28571, MSE=4.204082 
## 
## Node number 949: 13 observations
##   mean=54.30769, MSE=3.443787 
## 
## Node number 968: 11 observations
##   mean=56.81818, MSE=10.69421 
## 
## Node number 969: 15 observations
##   mean=58.26667, MSE=6.195556 
## 
## n= 835 
## 
## node), split, n, deviance, yval
##       * denotes terminal node
## 
##   1) root 835 1.353820e+05 41.00000  
##     2) PTS.diff< -32.5 373 2.241283e+04 29.46649  
##       4) PTS.diff< -359.5 159 3.671937e+03 22.25157  
##         8) PTS.diff< -572 67 8.216418e+02 18.22388  
##          16) PTS.diff< -670.5 40 3.193750e+02 16.62500  
##            32) PTS.diff< -829.5 9 6.800000e+01 14.33333 *
##            33) PTS.diff>=-829.5 31 1.903871e+02 17.29032  
##              66) DRB< 2358.5 17 1.262353e+02 16.52941 *
##              67) DRB>=2358.5 14 4.235714e+01 18.21429 *
##          17) PTS.diff>=-670.5 27 2.485185e+02 20.59259  
##            34) DRB< 2351 20 1.745500e+02 19.65000  
##              68) PTS.diff< -600 13 6.769231e+01 19.15385 *
##              69) PTS.diff>=-600 7 9.771429e+01 20.57143 *
##            35) DRB>=2351 7 5.428571e+00 23.28571 *
##         9) PTS.diff>=-572 92 9.718587e+02 25.18478  
##          18) PTS.diff< -458.5 44 4.069773e+02 23.47727  
##            36) PTS.diff< -498.5 25 2.373600e+02 22.84000  
##              72) PTS.diff>=-519.5 9 1.488889e+01 22.11111 *
##              73) PTS.diff< -519.5 16 2.150000e+02 23.25000 *
##            37) PTS.diff>=-498.5 19 1.461053e+02 24.31579 *
##          19) PTS.diff>=-458.5 48 3.190000e+02 26.75000  
##            38) DRB< 2500 41 2.277561e+02 26.39024  
##              76) PTS.diff< -411 19 9.273684e+01 25.47368 *
##              77) PTS.diff>=-411 22 1.052727e+02 27.18182  
##               154) DRB>=2340 14 5.571429e+01 26.85714 *
##               155) DRB< 2340 8 4.550000e+01 27.75000 *
##            39) DRB>=2500 7 5.485714e+01 28.85714 *
##       5) PTS.diff>=-359.5 214 4.314603e+03 34.82710  
##        10) PTS.diff< -232.5 77 7.880779e+02 30.62338  
##          20) PTS.diff< -258.5 54 5.397037e+02 29.92593  
##            40) PTS.diff>=-275 7 2.000000e+01 28.00000 *
##            41) PTS.diff< -275 47 4.898723e+02 30.21277  
##              82) PTS.diff< -293.5 38 3.510526e+02 29.84211  
##               164) DRB< 2275 10 1.085000e+02 28.50000 *
##               165) DRB>=2275 28 2.181071e+02 30.32143  
##                 330) DRB>=2384 20 1.068000e+02 29.40000  
##                   660) DRB< 2434.5 7 2.371429e+01 28.42857 *
##                   661) DRB>=2434.5 13 7.292308e+01 29.92308 *
##                 331) DRB< 2384 8 5.187500e+01 32.62500 *
##              83) PTS.diff>=-293.5 9 1.115556e+02 31.77778 *
##          21) PTS.diff>=-258.5 23 1.604348e+02 32.26087  
##            42) DRB>=2441 10 2.650000e+01 31.50000 *
##            43) DRB< 2441 13 1.236923e+02 32.84615 *
##        11) PTS.diff>=-232.5 137 1.401066e+03 37.18978  
##          22) PTS.diff< -125 61 5.477705e+02 35.65574  
##            44) DRB< 2333.5 12 4.891667e+01 33.58333 *
##            45) DRB>=2333.5 49 4.346939e+02 36.16327  
##              90) DRB>=2392.5 38 3.114737e+02 35.52632  
##               180) PTS.diff< -164.5 20 1.212000e+02 34.80000  
##                 360) DRB< 2519 13 6.230769e+01 34.23077 *
##                 361) DRB>=2519 7 4.685714e+01 35.85714 *
##               181) PTS.diff>=-164.5 18 1.680000e+02 36.33333 *
##              91) DRB< 2392.5 11 5.454545e+01 38.36364 *
##          23) PTS.diff>=-125 76 5.945263e+02 38.42105  
##            46) DRB>=2272.5 69 5.078261e+02 38.13043  
##              92) PTS.diff< -80.5 36 2.207500e+02 37.25000  
##               184) PTS.diff>=-111.5 22 1.247727e+02 36.68182  
##                 368) PTS.diff>=-89.5 8 6.000000e+01 36.00000 *
##                 369) PTS.diff< -89.5 14 5.892857e+01 37.07143 *
##               185) PTS.diff< -111.5 14 7.771429e+01 38.14286 *
##              93) PTS.diff>=-80.5 33 2.287273e+02 39.09091  
##               186) PTS.diff>=-70 25 1.365600e+02 38.76000  
##                 372) DRB< 2458.5 15 5.333333e+01 38.33333 *
##                 373) DRB>=2458.5 10 7.640000e+01 39.40000 *
##               187) PTS.diff< -70 8 8.087500e+01 40.12500 *
##            47) DRB< 2272.5 7 2.342857e+01 41.28571 *
##     3) PTS.diff>=-32.5 462 2.329312e+04 50.31169  
##       6) PTS.diff< 316 275 4.997105e+03 45.68727  
##        12) PTS.diff< 160 159 1.394730e+03 43.21384  
##          24) PTS.diff< 87.5 97 6.143299e+02 42.29897  
##            48) DRB< 2418.5 48 3.112500e+02 41.62500  
##              96) PTS.diff>=-15 40 2.249750e+02 41.22500  
##               192) PTS.diff< 41 19 1.611579e+02 40.78947 *
##               193) PTS.diff>=41 21 5.695238e+01 41.61905  
##                 386) PTS.diff>=63 12 3.691667e+01 41.08333 *
##                 387) PTS.diff< 63 9 1.200000e+01 42.33333 *
##              97) PTS.diff< -15 8 4.787500e+01 43.62500 *
##            49) DRB>=2418.5 49 2.599184e+02 42.95918  
##              98) PTS.diff< 1.5 21 8.228571e+01 41.71429  
##               196) PTS.diff>=-24 14 6.035714e+01 41.21429 *
##               197) PTS.diff< -24 7 1.142857e+01 42.71429 *
##              99) PTS.diff>=1.5 28 1.206786e+02 43.89286  
##               198) DRB>=2446.5 21 8.095238e+01 43.38095  
##                 396) PTS.diff< 72.5 10 1.850000e+01 42.50000 *
##                 397) PTS.diff>=72.5 11 4.763636e+01 44.18182 *
##               199) DRB< 2446.5 7 1.771429e+01 45.42857 *
##          25) PTS.diff>=87.5 62 5.721935e+02 44.64516  
##            50) DRB>=2548 12 6.000000e+01 43.00000 *
##            51) DRB< 2548 50 4.719200e+02 45.04000  
##             102) PTS.diff< 140 42 3.196429e+02 44.64286  
##               204) DRB< 2517 35 2.601714e+02 44.37143  
##                 408) PTS.diff< 106 11 2.818182e+01 43.72727 *
##                 409) PTS.diff>=106 24 2.253333e+02 44.66667  
##                   818) PTS.diff>=128 9 1.035556e+02 43.77778 *
##                   819) PTS.diff< 128 15 1.104000e+02 45.20000 *
##               205) DRB>=2517 7 4.400000e+01 46.00000 *
##             103) PTS.diff>=140 8 1.108750e+02 47.12500 *
##        13) PTS.diff>=160 116 1.296302e+03 49.07759  
##          26) PTS.diff< 250.5 67 5.431940e+02 47.83582  
##            52) DRB< 2579.5 60 4.379333e+02 47.63333  
##             104) DRB>=2474 19 1.216842e+02 46.73684 *
##             105) DRB< 2474 41 2.939024e+02 48.04878  
##               210) PTS.diff>=200.5 19 1.206316e+02 47.42105 *
##               211) PTS.diff< 200.5 22 1.593182e+02 48.59091  
##                 422) PTS.diff< 179 10 5.440000e+01 47.60000 *
##                 423) PTS.diff>=179 12 8.691667e+01 49.41667 *
##            53) DRB>=2579.5 7 8.171429e+01 49.57143 *
##          27) PTS.diff>=250.5 49 5.085306e+02 50.77551  
##            54) DRB< 2455 22 1.893182e+02 49.40909  
##             108) PTS.diff>=275 14 1.015000e+02 48.50000 *
##             109) PTS.diff< 275 8 5.600000e+01 51.00000 *
##            55) DRB>=2455 27 2.446667e+02 51.88889  
##             110) DRB>=2516.5 16 1.319375e+02 50.56250 *
##             111) DRB< 2516.5 11 4.363636e+01 53.81818 *
##       7) PTS.diff>=316 187 3.766642e+03 57.11230  
##        14) PTS.diff< 449.5 83 6.294699e+02 53.69880  
##          28) DRB< 2358 8 7.287500e+01 51.12500 *
##          29) DRB>=2358 75 4.979467e+02 53.97333  
##            58) PTS.diff< 345 21 1.478095e+02 52.90476  
##             116) PTS.diff>=321.5 14 8.485714e+01 52.28571 *
##             117) PTS.diff< 321.5 7 4.685714e+01 54.14286 *
##            59) PTS.diff>=345 54 3.168333e+02 54.38889  
##             118) PTS.diff>=356 45 2.548000e+02 54.06667  
##               236) DRB>=2576.5 17 9.576471e+01 53.11765 *
##               237) DRB< 2576.5 28 1.344286e+02 54.64286  
##                 474) PTS.diff< 430 20 7.895000e+01 53.95000  
##                   948) PTS.diff>=406.5 7 2.942857e+01 53.28571 *
##                   949) PTS.diff< 406.5 13 4.476923e+01 54.30769 *
##                 475) PTS.diff>=430 8 2.187500e+01 56.37500 *
##             119) PTS.diff< 356 9 3.400000e+01 56.00000 *
##        15) PTS.diff>=449.5 104 1.398221e+03 59.83654  
##          30) PTS.diff< 696 91 8.104615e+02 59.07692  
##            60) PTS.diff< 582.5 56 4.980000e+02 58.50000  
##             120) PTS.diff>=557.5 7 6.200000e+01 57.00000 *
##             121) PTS.diff< 557.5 49 4.180000e+02 58.71429  
##               242) PTS.diff< 542.5 42 3.349762e+02 58.30952  
##                 484) DRB>=2491.5 26 2.238846e+02 57.65385  
##                   968) PTS.diff< 488 11 1.176364e+02 56.81818 *
##                   969) PTS.diff>=488 15 9.293333e+01 58.26667 *
##                 485) DRB< 2491.5 16 8.175000e+01 59.37500 *
##               243) PTS.diff>=542.5 7 3.485714e+01 61.14286 *
##            61) PTS.diff>=582.5 35 2.640000e+02 60.00000  
##             122) DRB>=2601.5 12 6.166667e+01 58.83333 *
##             123) DRB< 2601.5 23 1.774783e+02 60.60870  
##               246) DRB< 2502 13 5.907692e+01 59.61538 *
##               247) DRB>=2502 10 8.890000e+01 61.90000 *
##          31) PTS.diff>=696 13 1.676923e+02 65.15385 *
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##                    model_id model_method         feats max.nTuningRuns
## 1 Max.cor.Y.cv.0.cp.0.rpart        rpart PTS.diff, DRB               0
##   min.elapsedtime.everything min.elapsedtime.final max.R.sq.fit
## 1                      0.469                 0.018    0.9612337
##   min.RMSE.fit max.R.sq.OOB min.RMSE.OOB
## 1     2.507057    0.9260059     3.494519
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
## [1] "    indep_vars: PTS.diff, DRB"
```

```
## Warning in nominalTrainWorkflow(x = x, y = y, wts = weights, info =
## trainInfo, : There were missing values in resampled performance measures.
```

```
## Aggregating results
## Selecting tuning parameters
## Fitting cp = 0.107 on full training set
```

```
## Warning in myfit_mdl(model_id = "Max.cor.Y", model_method = "rpart",
## model_type = glb_model_type, : model's bestTune found at an extreme of
## tuneGrid for parameter: cp
```

![](NBA_Wins2_files/figure-html/fit.models_0-7.png) ![](NBA_Wins2_files/figure-html/fit.models_0-8.png) 

```
## Call:
## rpart(formula = .outcome ~ ., control = list(minsplit = 20, minbucket = 7, 
##     cp = 0, maxcompete = 4, maxsurrogate = 5, usesurrogate = 2, 
##     surrogatestyle = 0, maxdepth = 30, xval = 0))
##   n= 835 
## 
##          CP nsplit rel error
## 1 0.6623927      0 1.0000000
## 2 0.1073213      1 0.3376073
## 3 0.1065599      2 0.2302860
## 
## Variable importance
## PTS.diff      DRB 
##       80       20 
## 
## Node number 1: 835 observations,    complexity param=0.6623927
##   mean=41, MSE=162.1341 
##   left son=2 (373 obs) right son=3 (462 obs)
##   Primary splits:
##       PTS.diff < -32.5  to the left,  improve=0.6623927, (0 missing)
##       DRB      < 2377.5 to the left,  improve=0.1549703, (0 missing)
##   Surrogate splits:
##       DRB < 2377.5 to the left,  agree=0.668, adj=0.257, (0 split)
## 
## Node number 2: 373 observations
##   mean=29.46649, MSE=60.08802 
## 
## Node number 3: 462 observations,    complexity param=0.1073213
##   mean=50.31169, MSE=50.418 
##   left son=6 (275 obs) right son=7 (187 obs)
##   Primary splits:
##       PTS.diff < 316    to the left,  improve=0.62376240, (0 missing)
##       DRB      < 2582.5 to the left,  improve=0.09916884, (0 missing)
##   Surrogate splits:
##       DRB < 2584   to the left,  agree=0.688, adj=0.23, (0 split)
## 
## Node number 6: 275 observations
##   mean=45.68727, MSE=18.17129 
## 
## Node number 7: 187 observations
##   mean=57.1123, MSE=20.14247 
## 
## n= 835 
## 
## node), split, n, deviance, yval
##       * denotes terminal node
## 
## 1) root 835 135382.000 41.00000  
##   2) PTS.diff< -32.5 373  22412.830 29.46649 *
##   3) PTS.diff>=-32.5 462  23293.120 50.31169  
##     6) PTS.diff< 316 275   4997.105 45.68727 *
##     7) PTS.diff>=316 187   3766.642 57.11230 *
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##          model_id model_method         feats max.nTuningRuns
## 1 Max.cor.Y.rpart        rpart PTS.diff, DRB               3
##   min.elapsedtime.everything min.elapsedtime.final max.R.sq.fit
## 1                      1.032                 0.018     0.769714
##   min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Rsquared.fit min.RMSESD.fit
## 1     5.970313    0.8688472     4.652407        0.7716525       1.571127
##   max.RsquaredSD.fit
## 1           0.119624
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
## [1] "    indep_vars: PTS.diff, DRB"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_Wins2_files/figure-html/fit.models_0-9.png) ![](NBA_Wins2_files/figure-html/fit.models_0-10.png) ![](NBA_Wins2_files/figure-html/fit.models_0-11.png) ![](NBA_Wins2_files/figure-html/fit.models_0-12.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -9.3744 -2.0916 -0.1428  2.0214 10.5207 
## 
## Coefficients:
##              Estimate Std. Error t value Pr(>|t|)    
## (Intercept) 3.579e+01  2.224e+00  16.095   <2e-16 ***
## PTS.diff    3.224e-02  3.151e-04 102.335   <2e-16 ***
## DRB         2.145e-03  9.151e-04   2.344   0.0193 *  
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3.053 on 832 degrees of freedom
## Multiple R-squared:  0.9427,	Adjusted R-squared:  0.9426 
## F-statistic:  6847 on 2 and 832 DF,  p-value: < 2.2e-16
## 
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##       model_id model_method         feats max.nTuningRuns
## 1 Max.cor.Y.lm           lm PTS.diff, DRB               1
##   min.elapsedtime.everything min.elapsedtime.final max.R.sq.fit
## 1                      0.932                 0.003    0.9427209
##   min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit max.Rsquared.fit
## 1     3.067261    0.9401179     3.143675        0.9425832        0.9432758
##   min.RMSESD.fit max.RsquaredSD.fit
## 1      0.1709692        0.005666001
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
## [1] "    indep_vars: PTS.diff, DRB, PTS.diff:oppPTS, PTS.diff:PTS, PTS.diff:FT, PTS.diff:X3P, PTS.diff:X2PA"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_Wins2_files/figure-html/fit.models_0-13.png) ![](NBA_Wins2_files/figure-html/fit.models_0-14.png) ![](NBA_Wins2_files/figure-html/fit.models_0-15.png) ![](NBA_Wins2_files/figure-html/fit.models_0-16.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -9.3505 -2.0452 -0.1462  2.0161  9.1299 
## 
## Coefficients:
##                     Estimate Std. Error t value Pr(>|t|)    
## (Intercept)        3.608e+01  2.228e+00  16.190  < 2e-16 ***
## PTS.diff           4.806e-02  6.703e-03   7.169 1.67e-12 ***
## DRB                2.007e-03  9.163e-04   2.190   0.0288 *  
## `PTS.diff:oppPTS` -2.819e-07  8.685e-07  -0.325   0.7456    
## `PTS.diff:PTS`     1.669e-06  1.180e-06   1.414   0.1578    
## `PTS.diff:FT`     -4.279e-06  2.589e-06  -1.653   0.0988 .  
## `PTS.diff:X3P`    -1.031e-05  5.895e-06  -1.748   0.0808 .  
## `PTS.diff:X2PA`   -2.865e-06  1.771e-06  -1.617   0.1062    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3.049 on 827 degrees of freedom
## Multiple R-squared:  0.9432,	Adjusted R-squared:  0.9427 
## F-statistic:  1962 on 7 and 827 DF,  p-value: < 2.2e-16
## 
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##                 model_id model_method
## 1 Interact.High.cor.Y.lm           lm
##                                                                                    feats
## 1 PTS.diff, DRB, PTS.diff:oppPTS, PTS.diff:PTS, PTS.diff:FT, PTS.diff:X3P, PTS.diff:X2PA
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      0.886                 0.005
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit
## 1    0.9431984     3.083847     0.943418     3.055823        0.9427176
##   max.Rsquared.fit min.RMSESD.fit max.RsquaredSD.fit
## 1        0.9425587      0.1530238         0.00491491
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
## [1] "    indep_vars: PTS.diff, DRB, AST, FT, BLK, X3P, STL, .rnorm, SeasonEnd, ORB, TOV, oppPTS"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_Wins2_files/figure-html/fit.models_0-17.png) ![](NBA_Wins2_files/figure-html/fit.models_0-18.png) ![](NBA_Wins2_files/figure-html/fit.models_0-19.png) ![](NBA_Wins2_files/figure-html/fit.models_0-20.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -9.0100 -1.9832 -0.1585  1.9600 10.5287 
## 
## Coefficients:
##               Estimate Std. Error t value Pr(>|t|)    
## (Intercept)  1.163e+02  5.961e+01   1.951  0.05137 .  
## PTS.diff     3.142e-02  6.122e-04  51.327  < 2e-16 ***
## DRB          1.985e-03  1.181e-03   1.681  0.09316 .  
## AST          1.113e-03  9.103e-04   1.223  0.22172    
## FT           2.762e-04  8.614e-04   0.321  0.74851    
## BLK          3.958e-03  1.438e-03   2.753  0.00603 ** 
## X3P          1.060e-03  1.119e-03   0.948  0.34364    
## STL          1.546e-04  1.600e-03   0.097  0.92304    
## .rnorm       1.807e-01  1.087e-01   1.663  0.09667 .  
## SeasonEnd   -3.960e-02  2.956e-02  -1.340  0.18071    
## ORB         -6.175e-04  1.101e-03  -0.561  0.57503    
## TOV         -1.682e-03  1.216e-03  -1.384  0.16684    
## oppPTS      -3.503e-04  4.010e-04  -0.874  0.38258    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3.044 on 822 degrees of freedom
## Multiple R-squared:  0.9437,	Adjusted R-squared:  0.9429 
## F-statistic:  1149 on 12 and 822 DF,  p-value: < 2.2e-16
## 
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##       model_id model_method
## 1 Low.cor.X.lm           lm
##                                                                        feats
## 1 PTS.diff, DRB, AST, FT, BLK, X3P, STL, .rnorm, SeasonEnd, ORB, TOV, oppPTS
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      0.915                 0.006
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit
## 1    0.9437381     3.119017    0.9412438     3.113979        0.9429167
##   max.Rsquared.fit min.RMSESD.fit max.RsquaredSD.fit
## 1        0.9414636      0.2222216        0.007492174
```

```r
rm(ret_lst)

glb_chunks_df <- myadd_chunk(glb_chunks_df, "fit.models", major.inc=FALSE)
```

```
##         label step_major step_minor    bgn    end elapsed
## 10 fit.models          7          0 25.441 43.976  18.535
## 11 fit.models          7          1 43.977     NA      NA
```


```r
fit.models_1_chunk_df <- myadd_chunk(NULL, "fit.models_1_bgn")
```

```
##              label step_major step_minor    bgn end elapsed
## 1 fit.models_1_bgn          1          0 48.244  NA      NA
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
## 1 fit.models_1_bgn          1          0 48.244 48.259   0.015
## 2  fit.models_1_lm          2          0 48.260     NA      NA
## [1] "fitting model: All.X.lm"
## [1] "    indep_vars: PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, .rnorm, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_Wins2_files/figure-html/fit.models_1-1.png) ![](NBA_Wins2_files/figure-html/fit.models_1-2.png) ![](NBA_Wins2_files/figure-html/fit.models_1-3.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -8.7862 -1.9322 -0.1506  1.9776 10.6742 
## 
## Coefficients: (4 not defined because of singularities)
##               Estimate Std. Error t value Pr(>|t|)    
## (Intercept)  1.709e+02  6.247e+01   2.736  0.00635 ** 
## PTS.diff     3.066e-02  7.107e-04  43.140  < 2e-16 ***
## DRB          3.908e-03  1.538e-03   2.541  0.01124 *  
## AST          8.808e-04  9.125e-04   0.965  0.33470    
## PTS         -8.300e-03  6.018e-03  -1.379  0.16818    
## FT           9.264e-03  6.395e-03   1.449  0.14782    
## BLK          3.865e-03  1.446e-03   2.673  0.00767 ** 
## FG           1.983e-02  1.231e-02   1.611  0.10754    
## FTA         -1.133e-03  1.694e-03  -0.669  0.50359    
## X3P                 NA         NA      NA       NA    
## STL          2.173e-03  1.986e-03   1.094  0.27417    
## X3PA         3.916e-03  2.345e-03   1.669  0.09541 .  
## X2P                 NA         NA      NA       NA    
## .rnorm       2.000e-01  1.086e-01   1.841  0.06595 .  
## SeasonEnd   -6.528e-02  3.083e-02  -2.117  0.03456 *  
## FGA         -3.152e-03  1.295e-03  -2.435  0.01512 *  
## X2PA                NA         NA      NA       NA    
## ORB          2.030e-03  1.733e-03   1.171  0.24182    
## TOV         -3.645e-03  1.568e-03  -2.325  0.02031 *  
## oppPTS              NA         NA      NA       NA    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3.034 on 819 degrees of freedom
## Multiple R-squared:  0.9443,	Adjusted R-squared:  0.9433 
## F-statistic: 925.6 on 15 and 819 DF,  p-value: < 2.2e-16
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

![](NBA_Wins2_files/figure-html/fit.models_1-4.png) 

```
##   model_id model_method
## 1 All.X.lm           lm
##                                                                                                            feats
## 1 PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, .rnorm, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      0.919                 0.009
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit
## 1    0.9442961     3.115887    0.9382505     3.192316        0.9432758
##   max.Rsquared.fit min.RMSESD.fit max.RsquaredSD.fit
## 1        0.9415448      0.2207614        0.007401005
##              label step_major step_minor   bgn    end elapsed
## 2  fit.models_1_lm          2          0 48.26 51.169    2.91
## 3 fit.models_1_glm          3          0 51.17     NA      NA
## [1] "fitting model: All.X.glm"
## [1] "    indep_vars: PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, .rnorm, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_Wins2_files/figure-html/fit.models_1-5.png) ![](NBA_Wins2_files/figure-html/fit.models_1-6.png) ![](NBA_Wins2_files/figure-html/fit.models_1-7.png) 

```
## 
## Call:
## NULL
## 
## Deviance Residuals: 
##     Min       1Q   Median       3Q      Max  
## -8.7862  -1.9322  -0.1506   1.9776  10.6742  
## 
## Coefficients: (4 not defined because of singularities)
##               Estimate Std. Error t value Pr(>|t|)    
## (Intercept)  1.709e+02  6.247e+01   2.736  0.00635 ** 
## PTS.diff     3.066e-02  7.107e-04  43.140  < 2e-16 ***
## DRB          3.908e-03  1.538e-03   2.541  0.01124 *  
## AST          8.808e-04  9.125e-04   0.965  0.33470    
## PTS         -8.300e-03  6.018e-03  -1.379  0.16818    
## FT           9.264e-03  6.395e-03   1.449  0.14782    
## BLK          3.865e-03  1.446e-03   2.673  0.00767 ** 
## FG           1.983e-02  1.231e-02   1.611  0.10754    
## FTA         -1.133e-03  1.694e-03  -0.669  0.50359    
## X3P                 NA         NA      NA       NA    
## STL          2.173e-03  1.986e-03   1.094  0.27417    
## X3PA         3.916e-03  2.345e-03   1.669  0.09541 .  
## X2P                 NA         NA      NA       NA    
## .rnorm       2.000e-01  1.086e-01   1.841  0.06595 .  
## SeasonEnd   -6.528e-02  3.083e-02  -2.117  0.03456 *  
## FGA         -3.152e-03  1.295e-03  -2.435  0.01512 *  
## X2PA                NA         NA      NA       NA    
## ORB          2.030e-03  1.733e-03   1.171  0.24182    
## TOV         -3.645e-03  1.568e-03  -2.325  0.02031 *  
## oppPTS              NA         NA      NA       NA    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for gaussian family taken to be 9.207951)
## 
##     Null deviance: 135382.0  on 834  degrees of freedom
## Residual deviance:   7541.3  on 819  degrees of freedom
## AIC: 4241.2
## 
## Number of Fisher Scoring iterations: 2
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
##                                                                                                            feats
## 1 PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, .rnorm, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      0.924                 0.059
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB min.aic.fit
## 1    0.9442961     3.115887    0.9382505     3.192316    4241.228
##   max.Rsquared.fit min.RMSESD.fit max.RsquaredSD.fit
## 1        0.9415448      0.2207614        0.007401005
##                   label step_major step_minor    bgn    end elapsed
## 3      fit.models_1_glm          3          0 51.170 54.008   2.838
## 4 fit.models_1_bayesglm          4          0 54.008     NA      NA
## [1] "fitting model: All.X.bayesglm"
## [1] "    indep_vars: PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, .rnorm, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS"
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
## 
## Call:
## NULL
## 
## Deviance Residuals: 
##     Min       1Q   Median       3Q      Max  
## -8.7862  -1.9322  -0.1506   1.9776  10.6743  
## 
## Coefficients:
##               Estimate Std. Error t value Pr(>|t|)   
## (Intercept)  1.709e+02  6.262e+01   2.729  0.00648 **
## PTS.diff     1.953e-02  2.972e+01   0.001  0.99948   
## DRB          3.908e-03  1.542e-03   2.535  0.01144 * 
## AST          8.808e-04  9.147e-04   0.963  0.33588   
## PTS          8.666e-03  3.222e+01   0.000  0.99979   
## FT           3.426e-03  2.115e+01   0.000  0.99987   
## BLK          3.865e-03  1.450e-03   2.666  0.00782 **
## FG           3.814e-03  4.789e+01   0.000  0.99994   
## FTA         -1.133e-03  1.698e-03  -0.668  0.50463   
## X3P         -1.495e-03  4.074e+01   0.000  0.99997   
## STL          2.173e-03  1.991e-03   1.092  0.27535   
## X3PA         1.560e-03  2.856e+01   0.000  0.99996   
## X2P          4.343e-03  3.124e+01   0.000  0.99989   
## .rnorm       2.000e-01  1.089e-01   1.837  0.06662 . 
## SeasonEnd   -6.528e-02  3.091e-02  -2.112  0.03501 * 
## FGA         -7.962e-04  2.856e+01   0.000  0.99998   
## X2PA        -2.356e-03  2.856e+01   0.000  0.99993   
## ORB          2.030e-03  1.737e-03   1.168  0.24297   
## TOV         -3.645e-03  1.571e-03  -2.319  0.02062 * 
## oppPTS      -1.113e-02  2.972e+01   0.000  0.99970   
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for gaussian family taken to be 9.253144)
## 
##     Null deviance: 135382.0  on 834  degrees of freedom
## Residual deviance:   7541.3  on 815  degrees of freedom
## AIC: 4249.2
## 
## Number of Fisher Scoring iterations: 14
## 
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##         model_id model_method
## 1 All.X.bayesglm     bayesglm
##                                                                                                            feats
## 1 PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, .rnorm, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      4.557                 0.113
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB min.aic.fit
## 1    0.9442961     3.115887    0.9382505     3.192316    4249.228
##   max.Rsquared.fit min.RMSESD.fit max.RsquaredSD.fit
## 1        0.9415448      0.2207615        0.007401005
##                   label step_major step_minor    bgn    end elapsed
## 4 fit.models_1_bayesglm          4          0 54.008 59.504   5.496
## 5    fit.models_1_rpart          5          0 59.505     NA      NA
## [1] "fitting model: All.X.no.rnorm.rpart"
## [1] "    indep_vars: PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS"
```

```
## Warning in nominalTrainWorkflow(x = x, y = y, wts = weights, info =
## trainInfo, : There were missing values in resampled performance measures.
```

![](NBA_Wins2_files/figure-html/fit.models_1-8.png) 

```
## Aggregating results
## Selecting tuning parameters
## Fitting cp = 0.107 on full training set
```

```
## Warning in myfit_mdl(model_id = model_id, model_method = method,
## indep_vars_vctr = indep_vars_vctr, : model's bestTune found at an extreme
## of tuneGrid for parameter: cp
```

![](NBA_Wins2_files/figure-html/fit.models_1-9.png) 

```
## Call:
## rpart(formula = .outcome ~ ., control = list(minsplit = 20, minbucket = 7, 
##     cp = 0, maxcompete = 4, maxsurrogate = 5, usesurrogate = 2, 
##     surrogatestyle = 0, maxdepth = 30, xval = 0))
##   n= 835 
## 
##          CP nsplit rel error
## 1 0.6623927      0 1.0000000
## 2 0.1073213      1 0.3376073
## 3 0.1065599      2 0.2302860
## 
## Variable importance
## PTS.diff      DRB      AST   oppPTS      PTS      STL      X3P 
##       56       14       10        7        6        6        1 
## 
## Node number 1: 835 observations,    complexity param=0.6623927
##   mean=41, MSE=162.1341 
##   left son=2 (373 obs) right son=3 (462 obs)
##   Primary splits:
##       PTS.diff < -32.5  to the left,  improve=0.66239270, (0 missing)
##       DRB      < 2377.5 to the left,  improve=0.15497030, (0 missing)
##       oppPTS   < 7961   to the right, improve=0.12078130, (0 missing)
##       AST      < 2012.5 to the left,  improve=0.07994841, (0 missing)
##       TOV      < 1145.5 to the right, improve=0.06589163, (0 missing)
##   Surrogate splits:
##       DRB    < 2377.5 to the left,  agree=0.668, adj=0.257, (0 split)
##       AST    < 1812   to the left,  agree=0.641, adj=0.196, (0 split)
##       oppPTS < 8026   to the right, agree=0.614, adj=0.137, (0 split)
##       PTS    < 7868   to the left,  agree=0.613, adj=0.134, (0 split)
##       STL    < 651.5  to the left,  agree=0.611, adj=0.129, (0 split)
## 
## Node number 2: 373 observations
##   mean=29.46649, MSE=60.08802 
## 
## Node number 3: 462 observations,    complexity param=0.1073213
##   mean=50.31169, MSE=50.418 
##   left son=6 (275 obs) right son=7 (187 obs)
##   Primary splits:
##       PTS.diff < 316    to the left,  improve=0.62376240, (0 missing)
##       DRB      < 2582.5 to the left,  improve=0.09916884, (0 missing)
##       BLK      < 403.5  to the left,  improve=0.04759291, (0 missing)
##       oppPTS   < 9180   to the right, improve=0.03923016, (0 missing)
##       AST      < 2362   to the left,  improve=0.02832203, (0 missing)
##   Surrogate splits:
##       DRB    < 2584   to the left,  agree=0.688, adj=0.230, (0 split)
##       oppPTS < 7557.5 to the right, agree=0.630, adj=0.086, (0 split)
##       X3P    < 578    to the left,  agree=0.626, adj=0.075, (0 split)
##       AST    < 2276   to the left,  agree=0.613, adj=0.043, (0 split)
##       X3PA   < 1667   to the left,  agree=0.613, adj=0.043, (0 split)
## 
## Node number 6: 275 observations
##   mean=45.68727, MSE=18.17129 
## 
## Node number 7: 187 observations
##   mean=57.1123, MSE=20.14247 
## 
## n= 835 
## 
## node), split, n, deviance, yval
##       * denotes terminal node
## 
## 1) root 835 135382.000 41.00000  
##   2) PTS.diff< -32.5 373  22412.830 29.46649 *
##   3) PTS.diff>=-32.5 462  23293.120 50.31169  
##     6) PTS.diff< 316 275   4997.105 45.68727 *
##     7) PTS.diff>=316 187   3766.642 57.11230 *
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##               model_id model_method
## 1 All.X.no.rnorm.rpart        rpart
##                                                                                                    feats
## 1 PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               3                      1.157                 0.063
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Rsquared.fit
## 1     0.769714     5.970313    0.8688472     4.652407        0.7716525
##   min.RMSESD.fit max.RsquaredSD.fit
## 1       1.571127           0.119624
##                label step_major step_minor    bgn    end elapsed
## 5 fit.models_1_rpart          5          0 59.505 62.683   3.178
## 6    fit.models_1_rf          6          0 62.684     NA      NA
## [1] "fitting model: All.X.no.rnorm.rf"
## [1] "    indep_vars: PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS"
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

![](NBA_Wins2_files/figure-html/fit.models_1-10.png) 

```
## Aggregating results
## Selecting tuning parameters
## Fitting mtry = 18 on full training set
```

```
## Warning in myfit_mdl(model_id = model_id, model_method = method,
## indep_vars_vctr = indep_vars_vctr, : model's bestTune found at an extreme
## of tuneGrid for parameter: mtry
```

![](NBA_Wins2_files/figure-html/fit.models_1-11.png) ![](NBA_Wins2_files/figure-html/fit.models_1-12.png) 

```
##                 Length Class      Mode     
## call              4    -none-     call     
## type              1    -none-     character
## predicted       835    -none-     numeric  
## mse             500    -none-     numeric  
## rsq             500    -none-     numeric  
## oob.times       835    -none-     numeric  
## importance       18    -none-     numeric  
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
## xNames           18    -none-     character
## problemType       1    -none-     character
## tuneValue         1    data.frame list     
## obsLevels         1    -none-     logical  
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##            model_id model_method
## 1 All.X.no.rnorm.rf           rf
##                                                                                                    feats
## 1 PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               3                     16.812                 6.345
##   max.R.sq.fit min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Rsquared.fit
## 1    0.9899911     3.134206    0.9294243     3.412877        0.9398484
##   min.RMSESD.fit max.RsquaredSD.fit
## 1      0.1565449        0.005243458
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
model_id <- "PTS.only"; indep_vars_vctr <- c(NULL
   ,"PTS", "oppPTS"
#    ,"<feat1>*<feat2>"
#    ,"<feat1>:<feat2>"
                                           )
for (method in c("lm")) {
    ret_lst <- myfit_mdl(model_id=model_id, model_method=method,
                                indep_vars_vctr=indep_vars_vctr,
                                model_type=glb_model_type,
                                rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                                fit_df=glb_fitobs_df, OOB_df=glb_OOBobs_df,
                    n_cv_folds=glb_n_cv_folds, tune_models_df=glb_tune_models_df)
    csm_mdl_id <- paste0(model_id, ".", method)
    csm_featsimp_df <- myget_feats_importance(glb_models_lst[[paste0(model_id, ".", method)]]);         print(head(csm_featsimp_df))
}
```

```
## [1] "fitting model: PTS.only.lm"
## [1] "    indep_vars: PTS, oppPTS"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_Wins2_files/figure-html/fit.models_1-13.png) ![](NBA_Wins2_files/figure-html/fit.models_1-14.png) ![](NBA_Wins2_files/figure-html/fit.models_1-15.png) ![](NBA_Wins2_files/figure-html/fit.models_1-16.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -9.7321 -2.1093 -0.0583  2.0138 10.6157 
## 
## Coefficients:
##               Estimate Std. Error t value Pr(>|t|)    
## (Intercept) 41.3048239  1.6101879   25.65   <2e-16 ***
## PTS          0.0325672  0.0002971  109.60   <2e-16 ***
## oppPTS      -0.0326036  0.0002939 -110.95   <2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3.063 on 832 degrees of freedom
## Multiple R-squared:  0.9423,	Adjusted R-squared:  0.9422 
## F-statistic:  6799 on 2 and 832 DF,  p-value: < 2.2e-16
## 
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##      model_id model_method       feats max.nTuningRuns
## 1 PTS.only.lm           lm PTS, oppPTS               1
##   min.elapsedtime.everything min.elapsedtime.final max.R.sq.fit
## 1                      0.858                 0.003     0.942345
##   min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit max.Rsquared.fit
## 1     3.075688    0.9417946     3.099349        0.9422064        0.9429442
##   min.RMSESD.fit max.RsquaredSD.fit
## 1      0.1709393        0.005805662
##        importance
## oppPTS        100
## PTS             0
```

```r
model_id <- "PTS.interact"; indep_vars_vctr <- c("oppPTS", "oppPTS*PTS")
for (method in c("lm")) {
    ret_lst <- myfit_mdl(model_id=model_id, model_method=method,
                                indep_vars_vctr=indep_vars_vctr,
                                model_type=glb_model_type,
                                rsp_var=glb_rsp_var, rsp_var_out=glb_rsp_var_out,
                                fit_df=glb_fitobs_df, OOB_df=glb_OOBobs_df,
                    n_cv_folds=glb_n_cv_folds, tune_models_df=glb_tune_models_df)
    csm_mdl_id <- paste0(model_id, ".", method)
    csm_featsimp_df <- myget_feats_importance(glb_models_lst[[paste0(model_id, ".", method)]]);         print(head(csm_featsimp_df))
}
```

```
## [1] "fitting model: PTS.interact.lm"
## [1] "    indep_vars: oppPTS, oppPTS*PTS"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_Wins2_files/figure-html/fit.models_1-17.png) ![](NBA_Wins2_files/figure-html/fit.models_1-18.png) ![](NBA_Wins2_files/figure-html/fit.models_1-19.png) ![](NBA_Wins2_files/figure-html/fit.models_1-20.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -9.6462 -2.1099 -0.0962  2.0463 10.7746 
## 
## Coefficients:
##                Estimate Std. Error t value Pr(>|t|)    
## (Intercept)   6.450e+01  1.898e+01   3.398  0.00071 ***
## oppPTS       -3.534e-02  2.246e-03 -15.730  < 2e-16 ***
## PTS           2.979e-02  2.282e-03  13.056  < 2e-16 ***
## `oppPTS:PTS`  3.256e-07  2.654e-07   1.227  0.22028    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3.062 on 831 degrees of freedom
## Multiple R-squared:  0.9424,	Adjusted R-squared:  0.9422 
## F-statistic:  4536 on 3 and 831 DF,  p-value: < 2.2e-16
## 
## [1] "    calling mypredict_mdl for fit:"
## [1] "    calling mypredict_mdl for OOB:"
##          model_id model_method              feats max.nTuningRuns
## 1 PTS.interact.lm           lm oppPTS, oppPTS*PTS               1
##   min.elapsedtime.everything min.elapsedtime.final max.R.sq.fit
## 1                      0.851                 0.003    0.9424492
##   min.RMSE.fit max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit max.Rsquared.fit
## 1     3.085891    0.9421886     3.088843        0.9422414         0.942569
##   min.RMSESD.fit max.RsquaredSD.fit
## 1      0.1746107        0.005991684
##              importance
## oppPTS        100.00000
## PTS            81.56245
## `oppPTS:PTS`    0.00000
```

```r
# model_id <- "W.only.no.<rsp_var>.fctr"; indep_vars_vctr <- c(NULL
#    ,"W"
# #    ,"<feat1>*<feat2>"
# #    ,"<feat1>:<feat2>"
#                                            )
# for (method in c("lm")) {
#     ret_lst <- myfit_mdl(model_id=model_id, model_method=method,
#                                 indep_vars_vctr=indep_vars_vctr,
#                                 model_type=glb_model_type,
#                                 rsp_var="<rsp_var>", rsp_var_out="<rsp_var>.predict.",
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
## PTS.only.lm                             PTS.only.lm           lm
## PTS.interact.lm                     PTS.interact.lm           lm
##                                                                                                                                    feats
## MFO.lm                                                                                                                            .rnorm
## Max.cor.Y.cv.0.rpart                                                                                                       PTS.diff, DRB
## Max.cor.Y.cv.0.cp.0.rpart                                                                                                  PTS.diff, DRB
## Max.cor.Y.rpart                                                                                                            PTS.diff, DRB
## Max.cor.Y.lm                                                                                                               PTS.diff, DRB
## Interact.High.cor.Y.lm                            PTS.diff, DRB, PTS.diff:oppPTS, PTS.diff:PTS, PTS.diff:FT, PTS.diff:X3P, PTS.diff:X2PA
## Low.cor.X.lm                                                  PTS.diff, DRB, AST, FT, BLK, X3P, STL, .rnorm, SeasonEnd, ORB, TOV, oppPTS
## All.X.lm                  PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, .rnorm, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS
## All.X.glm                 PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, .rnorm, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS
## All.X.bayesglm            PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, .rnorm, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS
## All.X.no.rnorm.rpart              PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS
## All.X.no.rnorm.rf                 PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS
## PTS.only.lm                                                                                                                  PTS, oppPTS
## PTS.interact.lm                                                                                                       oppPTS, oppPTS*PTS
##                           max.nTuningRuns min.elapsedtime.everything
## MFO.lm                                  0                      0.450
## Max.cor.Y.cv.0.rpart                    0                      0.603
## Max.cor.Y.cv.0.cp.0.rpart               0                      0.469
## Max.cor.Y.rpart                         3                      1.032
## Max.cor.Y.lm                            1                      0.932
## Interact.High.cor.Y.lm                  1                      0.886
## Low.cor.X.lm                            1                      0.915
## All.X.lm                                1                      0.919
## All.X.glm                               1                      0.924
## All.X.bayesglm                          1                      4.557
## All.X.no.rnorm.rpart                    3                      1.157
## All.X.no.rnorm.rf                       3                     16.812
## PTS.only.lm                             1                      0.858
## PTS.interact.lm                         1                      0.851
##                           min.elapsedtime.final max.R.sq.fit min.RMSE.fit
## MFO.lm                                    0.003 3.833404e-05    12.732946
## Max.cor.Y.cv.0.rpart                      0.019 0.000000e+00    12.733190
## Max.cor.Y.cv.0.cp.0.rpart                 0.018 9.612337e-01     2.507057
## Max.cor.Y.rpart                           0.018 7.697140e-01     5.970313
## Max.cor.Y.lm                              0.003 9.427209e-01     3.067261
## Interact.High.cor.Y.lm                    0.005 9.431984e-01     3.083847
## Low.cor.X.lm                              0.006 9.437381e-01     3.119017
## All.X.lm                                  0.009 9.442961e-01     3.115887
## All.X.glm                                 0.059 9.442961e-01     3.115887
## All.X.bayesglm                            0.113 9.442961e-01     3.115887
## All.X.no.rnorm.rpart                      0.063 7.697140e-01     5.970313
## All.X.no.rnorm.rf                         6.345 9.899911e-01     3.134206
## PTS.only.lm                               0.003 9.423450e-01     3.075688
## PTS.interact.lm                           0.003 9.424492e-01     3.085891
##                           max.R.sq.OOB min.RMSE.OOB max.Adj.R.sq.fit
## MFO.lm                    0.0002542561    12.844989       -0.0011621
## Max.cor.Y.cv.0.rpart      0.0000000000    12.846623               NA
## Max.cor.Y.cv.0.cp.0.rpart 0.9260059366     3.494519               NA
## Max.cor.Y.rpart           0.8688472184     4.652407               NA
## Max.cor.Y.lm              0.9401178663     3.143675        0.9425832
## Interact.High.cor.Y.lm    0.9434179629     3.055823        0.9427176
## Low.cor.X.lm              0.9412438212     3.113979        0.9429167
## All.X.lm                  0.9382504560     3.192316        0.9432758
## All.X.glm                 0.9382504560     3.192316               NA
## All.X.bayesglm            0.9382504551     3.192316               NA
## All.X.no.rnorm.rpart      0.8688472184     4.652407               NA
## All.X.no.rnorm.rf         0.9294242789     3.412877               NA
## PTS.only.lm               0.9417946385     3.099349        0.9422064
## PTS.interact.lm           0.9421885606     3.088843        0.9422414
##                           max.Rsquared.fit min.RMSESD.fit
## MFO.lm                                  NA             NA
## Max.cor.Y.cv.0.rpart                    NA             NA
## Max.cor.Y.cv.0.cp.0.rpart               NA             NA
## Max.cor.Y.rpart                  0.7716525      1.5711267
## Max.cor.Y.lm                     0.9432758      0.1709692
## Interact.High.cor.Y.lm           0.9425587      0.1530238
## Low.cor.X.lm                     0.9414636      0.2222216
## All.X.lm                         0.9415448      0.2207614
## All.X.glm                        0.9415448      0.2207614
## All.X.bayesglm                   0.9415448      0.2207615
## All.X.no.rnorm.rpart             0.7716525      1.5711267
## All.X.no.rnorm.rf                0.9398484      0.1565449
## PTS.only.lm                      0.9429442      0.1709393
## PTS.interact.lm                  0.9425690      0.1746107
##                           max.RsquaredSD.fit min.aic.fit
## MFO.lm                                    NA          NA
## Max.cor.Y.cv.0.rpart                      NA          NA
## Max.cor.Y.cv.0.cp.0.rpart                 NA          NA
## Max.cor.Y.rpart                  0.119623950          NA
## Max.cor.Y.lm                     0.005666001          NA
## Interact.High.cor.Y.lm           0.004914910          NA
## Low.cor.X.lm                     0.007492174          NA
## All.X.lm                         0.007401005          NA
## All.X.glm                        0.007401005    4241.228
## All.X.bayesglm                   0.007401005    4249.228
## All.X.no.rnorm.rpart             0.119623950          NA
## All.X.no.rnorm.rf                0.005243458          NA
## PTS.only.lm                      0.005805662          NA
## PTS.interact.lm                  0.005991684          NA
```

```r
rm(ret_lst)
fit.models_1_chunk_df <- myadd_chunk(fit.models_1_chunk_df, "fit.models_1_end", 
                                     major.inc=TRUE)
```

```
##              label step_major step_minor    bgn    end elapsed
## 6  fit.models_1_rf          6          0 62.684 86.424   23.74
## 7 fit.models_1_end          7          0 86.425     NA      NA
```

```r
glb_chunks_df <- myadd_chunk(glb_chunks_df, "fit.models", major.inc=FALSE)
```

```
##         label step_major step_minor    bgn    end elapsed
## 11 fit.models          7          1 43.977 86.431  42.454
## 12 fit.models          7          2 86.431     NA      NA
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
## PTS.only.lm                             PTS.only.lm           lm
## PTS.interact.lm                     PTS.interact.lm           lm
##                                                                                                                                    feats
## MFO.lm                                                                                                                            .rnorm
## Max.cor.Y.cv.0.rpart                                                                                                       PTS.diff, DRB
## Max.cor.Y.cv.0.cp.0.rpart                                                                                                  PTS.diff, DRB
## Max.cor.Y.rpart                                                                                                            PTS.diff, DRB
## Max.cor.Y.lm                                                                                                               PTS.diff, DRB
## Interact.High.cor.Y.lm                            PTS.diff, DRB, PTS.diff:oppPTS, PTS.diff:PTS, PTS.diff:FT, PTS.diff:X3P, PTS.diff:X2PA
## Low.cor.X.lm                                                  PTS.diff, DRB, AST, FT, BLK, X3P, STL, .rnorm, SeasonEnd, ORB, TOV, oppPTS
## All.X.lm                  PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, .rnorm, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS
## All.X.glm                 PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, .rnorm, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS
## All.X.bayesglm            PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, .rnorm, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS
## All.X.no.rnorm.rpart              PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS
## All.X.no.rnorm.rf                 PTS.diff, DRB, AST, PTS, FT, BLK, FG, FTA, X3P, STL, X3PA, X2P, SeasonEnd, FGA, X2PA, ORB, TOV, oppPTS
## PTS.only.lm                                                                                                                  PTS, oppPTS
## PTS.interact.lm                                                                                                       oppPTS, oppPTS*PTS
##                           max.nTuningRuns max.R.sq.fit max.R.sq.OOB
## MFO.lm                                  0 3.833404e-05 0.0002542561
## Max.cor.Y.cv.0.rpart                    0 0.000000e+00 0.0000000000
## Max.cor.Y.cv.0.cp.0.rpart               0 9.612337e-01 0.9260059366
## Max.cor.Y.rpart                         3 7.697140e-01 0.8688472184
## Max.cor.Y.lm                            1 9.427209e-01 0.9401178663
## Interact.High.cor.Y.lm                  1 9.431984e-01 0.9434179629
## Low.cor.X.lm                            1 9.437381e-01 0.9412438212
## All.X.lm                                1 9.442961e-01 0.9382504560
## All.X.glm                               1 9.442961e-01 0.9382504560
## All.X.bayesglm                          1 9.442961e-01 0.9382504551
## All.X.no.rnorm.rpart                    3 7.697140e-01 0.8688472184
## All.X.no.rnorm.rf                       3 9.899911e-01 0.9294242789
## PTS.only.lm                             1 9.423450e-01 0.9417946385
## PTS.interact.lm                         1 9.424492e-01 0.9421885606
##                           max.Adj.R.sq.fit max.Rsquared.fit
## MFO.lm                          -0.0011621               NA
## Max.cor.Y.cv.0.rpart                    NA               NA
## Max.cor.Y.cv.0.cp.0.rpart               NA               NA
## Max.cor.Y.rpart                         NA        0.7716525
## Max.cor.Y.lm                     0.9425832        0.9432758
## Interact.High.cor.Y.lm           0.9427176        0.9425587
## Low.cor.X.lm                     0.9429167        0.9414636
## All.X.lm                         0.9432758        0.9415448
## All.X.glm                               NA        0.9415448
## All.X.bayesglm                          NA        0.9415448
## All.X.no.rnorm.rpart                    NA        0.7716525
## All.X.no.rnorm.rf                       NA        0.9398484
## PTS.only.lm                      0.9422064        0.9429442
## PTS.interact.lm                  0.9422414        0.9425690
##                           inv.elapsedtime.everything inv.elapsedtime.final
## MFO.lm                                    2.22222222           333.3333333
## Max.cor.Y.cv.0.rpart                      1.65837479            52.6315789
## Max.cor.Y.cv.0.cp.0.rpart                 2.13219616            55.5555556
## Max.cor.Y.rpart                           0.96899225            55.5555556
## Max.cor.Y.lm                              1.07296137           333.3333333
## Interact.High.cor.Y.lm                    1.12866817           200.0000000
## Low.cor.X.lm                              1.09289617           166.6666667
## All.X.lm                                  1.08813928           111.1111111
## All.X.glm                                 1.08225108            16.9491525
## All.X.bayesglm                            0.21944262             8.8495575
## All.X.no.rnorm.rpart                      0.86430424            15.8730159
## All.X.no.rnorm.rf                         0.05948132             0.1576044
## PTS.only.lm                               1.16550117           333.3333333
## PTS.interact.lm                           1.17508813           333.3333333
##                           inv.RMSE.fit inv.RMSE.OOB  inv.aic.fit
## MFO.lm                      0.07853642   0.07785137           NA
## Max.cor.Y.cv.0.rpart        0.07853491   0.07784147           NA
## Max.cor.Y.cv.0.cp.0.rpart   0.39887403   0.28616242           NA
## Max.cor.Y.rpart             0.16749541   0.21494249           NA
## Max.cor.Y.lm                0.32602380   0.31809907           NA
## Interact.High.cor.Y.lm      0.32427033   0.32724404           NA
## Low.cor.X.lm                0.32061384   0.32113250           NA
## All.X.lm                    0.32093590   0.31325222           NA
## All.X.glm                   0.32093590   0.31325222 0.0002357808
## All.X.bayesglm              0.32093591   0.31325222 0.0002353368
## All.X.no.rnorm.rpart        0.16749541   0.21494249           NA
## All.X.no.rnorm.rf           0.31906005   0.29300790           NA
## PTS.only.lm                 0.32513049   0.32264842           NA
## PTS.interact.lm             0.32405557   0.32374580           NA
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
## 14. Consider specifying shapes manually. if you must have them.
```

```
## Warning in loop_apply(n, do.ply): Removed 7 rows containing missing values
## (geom_path).
```

```
## Warning in loop_apply(n, do.ply): Removed 88 rows containing missing values
## (geom_point).
```

```
## Warning in loop_apply(n, do.ply): Removed 22 rows containing missing values
## (geom_text).
```

```
## Warning in RColorBrewer::brewer.pal(n, pal): n too large, allowed maximum for palette Set1 is 9
## Returning the palette you asked for with that many colors
```

```
## Warning: The shape palette can deal with a maximum of 6 discrete values
## because more than 6 becomes difficult to discriminate; you have
## 14. Consider specifying shapes manually. if you must have them.
```

![](NBA_Wins2_files/figure-html/fit.models_2-1.png) 

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

![](NBA_Wins2_files/figure-html/fit.models_2-2.png) 

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
##                     model_id min.RMSE.OOB max.R.sq.OOB max.Adj.R.sq.fit
## 6     Interact.High.cor.Y.lm     3.055823 0.9434179629        0.9427176
## 14           PTS.interact.lm     3.088843 0.9421885606        0.9422414
## 13               PTS.only.lm     3.099349 0.9417946385        0.9422064
## 7               Low.cor.X.lm     3.113979 0.9412438212        0.9429167
## 5               Max.cor.Y.lm     3.143675 0.9401178663        0.9425832
## 10            All.X.bayesglm     3.192316 0.9382504551               NA
## 8                   All.X.lm     3.192316 0.9382504560        0.9432758
## 9                  All.X.glm     3.192316 0.9382504560               NA
## 12         All.X.no.rnorm.rf     3.412877 0.9294242789               NA
## 3  Max.cor.Y.cv.0.cp.0.rpart     3.494519 0.9260059366               NA
## 4            Max.cor.Y.rpart     4.652407 0.8688472184               NA
## 11      All.X.no.rnorm.rpart     4.652407 0.8688472184               NA
## 1                     MFO.lm    12.844989 0.0002542561       -0.0011621
## 2       Max.cor.Y.cv.0.rpart    12.846623 0.0000000000               NA
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
## 14. Consider specifying shapes manually. if you must have them.
```

```
## Warning in loop_apply(n, do.ply): Removed 6 rows containing missing values
## (geom_path).
```

```
## Warning in loop_apply(n, do.ply): Removed 33 rows containing missing values
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
## 14. Consider specifying shapes manually. if you must have them.
```

![](NBA_Wins2_files/figure-html/fit.models_2-3.png) 

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
## [1] "Best model id: Interact.High.cor.Y.lm"
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

![](NBA_Wins2_files/figure-html/fit.models_2-4.png) ![](NBA_Wins2_files/figure-html/fit.models_2-5.png) ![](NBA_Wins2_files/figure-html/fit.models_2-6.png) ![](NBA_Wins2_files/figure-html/fit.models_2-7.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -9.3505 -2.0452 -0.1462  2.0161  9.1299 
## 
## Coefficients:
##                     Estimate Std. Error t value Pr(>|t|)    
## (Intercept)        3.608e+01  2.228e+00  16.190  < 2e-16 ***
## PTS.diff           4.806e-02  6.703e-03   7.169 1.67e-12 ***
## DRB                2.007e-03  9.163e-04   2.190   0.0288 *  
## `PTS.diff:oppPTS` -2.819e-07  8.685e-07  -0.325   0.7456    
## `PTS.diff:PTS`     1.669e-06  1.180e-06   1.414   0.1578    
## `PTS.diff:FT`     -4.279e-06  2.589e-06  -1.653   0.0988 .  
## `PTS.diff:X3P`    -1.031e-05  5.895e-06  -1.748   0.0808 .  
## `PTS.diff:X2PA`   -2.865e-06  1.771e-06  -1.617   0.1062    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3.049 on 827 degrees of freedom
## Multiple R-squared:  0.9432,	Adjusted R-squared:  0.9427 
## F-statistic:  1962 on 7 and 827 DF,  p-value: < 2.2e-16
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

![](NBA_Wins2_files/figure-html/fit.models_2-8.png) 

```
##     SeasonEnd                  Team Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 854      2013 Oklahoma City Thunder        1 60 8669   7914 3126 6504 2528
## 863      2013    Washington Wizards        0 29 7644   7852 2910 6693 2365
## 845      2013       Houston Rockets        1 45 8688   8403 3124 6782 2257
## 840      2013   Cleveland Cavaliers        0 24 7913   8297 2993 6901 2446
## 838      2013     Charlotte Bobcats        0 21 7661   8418 2823 6649 2354
## 848      2013     Memphis Grizzlies        1 56 7659   7319 2964 6679 2582
##     X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV .rownames .src
## 854 4916 598 1588 1819 2196  854 2725 1753 679 624 1253        19 Test
## 863 5198 545 1495 1279 1746  887 2652 1775 598 376 1238        28 Test
## 845 4413 867 2369 1573 2087  909 2652 1902 679 359 1348        10 Test
## 840 5320 547 1581 1380 1826 1004 2359 1694 647 334 1149         5 Test
## 838 5250 469 1399 1546 2060  917 2389 1587 591 479 1153         3 Test
## 848 5572 382 1107 1349 1746 1059 2445 1715 703 436 1144        13 Test
##          .rnorm PTS.diff             Team.fctr
## 854  1.59350961      755 Oklahoma City Thunder
## 863  1.43540954     -208    Washington Wizards
## 845 -0.27976609      285       Houston Rockets
## 840 -0.02319101     -384   Cleveland Cavaliers
## 838  0.34644930     -757     Charlotte Bobcats
## 848  0.67432067      340     Memphis Grizzlies
##     W.predict.Interact.High.cor.Y.lm W.predict.Interact.High.cor.Y.lm.err
## 854                         65.90764                             5.907643
## 863                         34.61608                             5.616085
## 845                         50.48699                             5.486987
## 840                         28.47063                             4.470631
## 838                         16.66341                             4.336589
## 848                         52.24151                             3.758489
```

```r
predct_accurate_var_name <- paste0(glb_rsp_var_out, glb_sel_mdl_id, ".accurate")
glb_OOBobs_df[, predct_accurate_var_name] <-
                    (glb_OOBobs_df[, glb_rsp_var] == 
                     glb_OOBobs_df[, paste0(glb_rsp_var_out, glb_sel_mdl_id)])

#stop(here"); #sav_models_lst <- glb_models_lst; sav_models_df <- glb_models_df
glb_featsimp_df <- 
    myget_feats_importance(mdl=glb_sel_mdl, featsimp_df=NULL)
glb_featsimp_df[, paste0(glb_sel_mdl_id, ".importance")] <- glb_featsimp_df$importance
print(glb_featsimp_df)
```

```
##                   importance Interact.High.cor.Y.lm.importance
## PTS.diff           100.00000                         100.00000
## DRB                 27.25871                          27.25871
## `PTS.diff:X3P`      20.79874                          20.79874
## `PTS.diff:FT`       19.40171                          19.40171
## `PTS.diff:X2PA`     18.88785                          18.88785
## `PTS.diff:PTS`      15.91326                          15.91326
## `PTS.diff:oppPTS`    0.00000                           0.00000
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
## glb_sel_mdl_id): Limiting important feature scatter plots to 5 out of 7
```

![](NBA_Wins2_files/figure-html/fit.models_2-9.png) ![](NBA_Wins2_files/figure-html/fit.models_2-10.png) ![](NBA_Wins2_files/figure-html/fit.models_2-11.png) ![](NBA_Wins2_files/figure-html/fit.models_2-12.png) ![](NBA_Wins2_files/figure-html/fit.models_2-13.png) 

```
##     SeasonEnd                  Team Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 854      2013 Oklahoma City Thunder        1 60 8669   7914 3126 6504 2528
## 863      2013    Washington Wizards        0 29 7644   7852 2910 6693 2365
## 845      2013       Houston Rockets        1 45 8688   8403 3124 6782 2257
## 840      2013   Cleveland Cavaliers        0 24 7913   8297 2993 6901 2446
## 838      2013     Charlotte Bobcats        0 21 7661   8418 2823 6649 2354
##     X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV .rownames .src
## 854 4916 598 1588 1819 2196  854 2725 1753 679 624 1253        19 Test
## 863 5198 545 1495 1279 1746  887 2652 1775 598 376 1238        28 Test
## 845 4413 867 2369 1573 2087  909 2652 1902 679 359 1348        10 Test
## 840 5320 547 1581 1380 1826 1004 2359 1694 647 334 1149         5 Test
## 838 5250 469 1399 1546 2060  917 2389 1587 591 479 1153         3 Test
##          .rnorm PTS.diff             Team.fctr
## 854  1.59350961      755 Oklahoma City Thunder
## 863  1.43540954     -208    Washington Wizards
## 845 -0.27976609      285       Houston Rockets
## 840 -0.02319101     -384   Cleveland Cavaliers
## 838  0.34644930     -757     Charlotte Bobcats
##     W.predict.Interact.High.cor.Y.lm W.predict.Interact.High.cor.Y.lm.err
## 854                         65.90764                             5.907643
## 863                         34.61608                             5.616085
## 845                         50.48699                             5.486987
## 840                         28.47063                             4.470631
## 838                         16.66341                             4.336589
##     W.predict.Interact.High.cor.Y.lm.accurate .label
## 854                                     FALSE     19
## 863                                     FALSE     28
## 845                                     FALSE     10
## 840                                     FALSE      5
## 838                                     FALSE      3
```

![](NBA_Wins2_files/figure-html/fit.models_2-14.png) 

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
## 12 fit.models          7          2 86.431 99.704  13.273
## 13 fit.models          7          3 99.704     NA      NA
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
## [1] "W.predict.Interact.High.cor.Y.lm"         
## [2] "W.predict.Interact.High.cor.Y.lm.err"     
## [3] "W.predict.Interact.High.cor.Y.lm.accurate"
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

![](NBA_Wins2_files/figure-html/fit.models_3-1.png) 

```r
glb_chunks_df <- myadd_chunk(glb_chunks_df, "fit.data.training", major.inc=TRUE)
```

```
##                label step_major step_minor     bgn    end elapsed
## 13        fit.models          7          3  99.704 103.77   4.067
## 14 fit.data.training          8          0 103.771     NA      NA
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
## [1] "    indep_vars: PTS.diff, DRB, PTS.diff:oppPTS, PTS.diff:PTS, PTS.diff:FT, PTS.diff:X3P, PTS.diff:X2PA"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_Wins2_files/figure-html/fit.data.training_0-1.png) ![](NBA_Wins2_files/figure-html/fit.data.training_0-2.png) ![](NBA_Wins2_files/figure-html/fit.data.training_0-3.png) ![](NBA_Wins2_files/figure-html/fit.data.training_0-4.png) 

```
## 
## Call:
## lm(formula = .outcome ~ ., data = dat)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -9.3505 -2.0452 -0.1462  2.0161  9.1299 
## 
## Coefficients:
##                     Estimate Std. Error t value Pr(>|t|)    
## (Intercept)        3.608e+01  2.228e+00  16.190  < 2e-16 ***
## PTS.diff           4.806e-02  6.703e-03   7.169 1.67e-12 ***
## DRB                2.007e-03  9.163e-04   2.190   0.0288 *  
## `PTS.diff:oppPTS` -2.819e-07  8.685e-07  -0.325   0.7456    
## `PTS.diff:PTS`     1.669e-06  1.180e-06   1.414   0.1578    
## `PTS.diff:FT`     -4.279e-06  2.589e-06  -1.653   0.0988 .  
## `PTS.diff:X3P`    -1.031e-05  5.895e-06  -1.748   0.0808 .  
## `PTS.diff:X2PA`   -2.865e-06  1.771e-06  -1.617   0.1062    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3.049 on 827 degrees of freedom
## Multiple R-squared:  0.9432,	Adjusted R-squared:  0.9427 
## F-statistic:  1962 on 7 and 827 DF,  p-value: < 2.2e-16
## 
## [1] "    calling mypredict_mdl for fit:"
##   model_id model_method
## 1 Final.lm           lm
##                                                                                    feats
## 1 PTS.diff, DRB, PTS.diff:oppPTS, PTS.diff:PTS, PTS.diff:FT, PTS.diff:X3P, PTS.diff:X2PA
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      0.892                 0.005
##   max.R.sq.fit min.RMSE.fit max.Adj.R.sq.fit max.Rsquared.fit
## 1    0.9431984     3.083847        0.9427176        0.9425587
##   min.RMSESD.fit max.RsquaredSD.fit
## 1      0.1530238         0.00491491
```

```r
rm(ret_lst)
glb_chunks_df <- myadd_chunk(glb_chunks_df, "fit.data.training", major.inc=FALSE)
```

```
##                label step_major step_minor     bgn     end elapsed
## 14 fit.data.training          8          0 103.771 107.737   3.967
## 15 fit.data.training          8          1 107.738      NA      NA
```


```r
glb_trnobs_df <- glb_get_predictions(df=glb_trnobs_df, mdl_id=glb_fin_mdl_id, 
                                     rsp_var_out=glb_rsp_var_out,
    prob_threshold_def=ifelse(glb_is_classification && glb_is_binomial, 
        glb_models_df[glb_models_df$model_id == glb_sel_mdl_id, "opt.prob.threshold.OOB"], NULL))
```

![](NBA_Wins2_files/figure-html/fit.data.training_1-1.png) 

```
##     SeasonEnd                 Team Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 158      1986  Seattle SuperSonics        0 31 8564   8572 3335 7059 3256
## 318      1993     Dallas Mavericks        0 11 8141   9387 3164 7271 2881
## 148      1986 Los Angeles Clippers        0 32 8907   9475 3388 7165 3324
## 425      1997    Charlotte Hornets        1 54 8108   7955 2988 6342 2397
## 387      1995         Phoenix Suns        1 59 9073   8755 3356 6967 2772
## 459      1998      Detroit Pistons        0 37 7721   7592 2862 6373 2569
##     X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV .rownames  .src
## 158 6759  79  300 1815 2331 1145 2256 1977 745 295 1435       158 Train
## 318 6434 283  837 1530 2171 1234 2265 1683 649 355 1459       318 Train
## 148 6936  64  229 2067 2683 1159 2258 1968 694 501 1506       148 Train
## 425 4960 591 1382 1541 1984  910 2298 2021 597 349 1203       425 Train
## 387 5383 584 1584 1777 2352 1027 2403 2198 687 312 1167       387 Train
## 459 5435 293  938 1704 2288 1044 2338 1597 678 344 1198       459 Train
##         .rnorm PTS.diff            Team.fctr W.predict.Final.lm
## 158  0.8923638       -8  Seattle SuperSonics          40.350529
## 318  0.7199727    -1246     Dallas Mavericks           1.870137
## 148  0.8696714     -568 Los Angeles Clippers          23.071963
## 425 -0.4494398      153    Charlotte Hornets          45.656013
## 387 -0.3689781      318         Phoenix Suns          50.978312
## 459  0.8742224      129      Detroit Pistons          45.017996
##     W.predict.Final.lm.err
## 158               9.350529
## 318               9.129863
## 148               8.928037
## 425               8.343987
## 387               8.021688
## 459               8.017996
```

```r
sav_featsimp_df <- glb_featsimp_df
#glb_feats_df <- sav_feats_df
# glb_feats_df <- mymerge_feats_importance(feats_df=glb_feats_df, sel_mdl=glb_fin_mdl, 
#                                                entity_df=glb_trnobs_df)
glb_featsimp_df <- myget_feats_importance(mdl=glb_fin_mdl, featsimp_df=glb_featsimp_df)
glb_featsimp_df[, paste0(glb_fin_mdl_id, ".importance")] <- glb_featsimp_df$importance
print(glb_featsimp_df)
```

```
##                   Interact.High.cor.Y.lm.importance importance
## PTS.diff                                  100.00000  100.00000
## DRB                                        27.25871   27.25871
## `PTS.diff:X3P`                             20.79874   20.79874
## `PTS.diff:FT`                              19.40171   19.40171
## `PTS.diff:X2PA`                            18.88785   18.88785
## `PTS.diff:PTS`                             15.91326   15.91326
## `PTS.diff:oppPTS`                           0.00000    0.00000
##                   Final.lm.importance
## PTS.diff                    100.00000
## DRB                          27.25871
## `PTS.diff:X3P`               20.79874
## `PTS.diff:FT`                19.40171
## `PTS.diff:X2PA`              18.88785
## `PTS.diff:PTS`               15.91326
## `PTS.diff:oppPTS`             0.00000
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
## glb_fin_mdl_id): Limiting important feature scatter plots to 5 out of 7
```

![](NBA_Wins2_files/figure-html/fit.data.training_1-2.png) ![](NBA_Wins2_files/figure-html/fit.data.training_1-3.png) ![](NBA_Wins2_files/figure-html/fit.data.training_1-4.png) ![](NBA_Wins2_files/figure-html/fit.data.training_1-5.png) ![](NBA_Wins2_files/figure-html/fit.data.training_1-6.png) 

```
##     SeasonEnd                 Team Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 158      1986  Seattle SuperSonics        0 31 8564   8572 3335 7059 3256
## 318      1993     Dallas Mavericks        0 11 8141   9387 3164 7271 2881
## 148      1986 Los Angeles Clippers        0 32 8907   9475 3388 7165 3324
## 425      1997    Charlotte Hornets        1 54 8108   7955 2988 6342 2397
## 387      1995         Phoenix Suns        1 59 9073   8755 3356 6967 2772
##     X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV .rownames  .src
## 158 6759  79  300 1815 2331 1145 2256 1977 745 295 1435       158 Train
## 318 6434 283  837 1530 2171 1234 2265 1683 649 355 1459       318 Train
## 148 6936  64  229 2067 2683 1159 2258 1968 694 501 1506       148 Train
## 425 4960 591 1382 1541 1984  910 2298 2021 597 349 1203       425 Train
## 387 5383 584 1584 1777 2352 1027 2403 2198 687 312 1167       387 Train
##         .rnorm PTS.diff            Team.fctr W.predict.Final.lm
## 158  0.8923638       -8  Seattle SuperSonics          40.350529
## 318  0.7199727    -1246     Dallas Mavericks           1.870137
## 148  0.8696714     -568 Los Angeles Clippers          23.071963
## 425 -0.4494398      153    Charlotte Hornets          45.656013
## 387 -0.3689781      318         Phoenix Suns          50.978312
##     W.predict.Final.lm.err .label
## 158               9.350529    158
## 318               9.129863    318
## 148               8.928037    148
## 425               8.343987    425
## 387               8.021688    387
```

![](NBA_Wins2_files/figure-html/fit.data.training_1-7.png) 

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
## [1] "W.predict.Final.lm"     "W.predict.Final.lm.err"
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

![](NBA_Wins2_files/figure-html/fit.data.training_1-8.png) 

```r
glb_chunks_df <- myadd_chunk(glb_chunks_df, "predict.data.new", major.inc=TRUE)
```

```
##                label step_major step_minor     bgn     end elapsed
## 15 fit.data.training          8          1 107.738 112.791   5.053
## 16  predict.data.new          9          0 112.791      NA      NA
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

![](NBA_Wins2_files/figure-html/predict.data.new-1.png) 

```
##     SeasonEnd                  Team Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 854      2013 Oklahoma City Thunder        1 60 8669   7914 3126 6504 2528
## 863      2013    Washington Wizards        0 29 7644   7852 2910 6693 2365
## 845      2013       Houston Rockets        1 45 8688   8403 3124 6782 2257
## 840      2013   Cleveland Cavaliers        0 24 7913   8297 2993 6901 2446
## 838      2013     Charlotte Bobcats        0 21 7661   8418 2823 6649 2354
## 848      2013     Memphis Grizzlies        1 56 7659   7319 2964 6679 2582
##     X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV .rownames .src
## 854 4916 598 1588 1819 2196  854 2725 1753 679 624 1253        19 Test
## 863 5198 545 1495 1279 1746  887 2652 1775 598 376 1238        28 Test
## 845 4413 867 2369 1573 2087  909 2652 1902 679 359 1348        10 Test
## 840 5320 547 1581 1380 1826 1004 2359 1694 647 334 1149         5 Test
## 838 5250 469 1399 1546 2060  917 2389 1587 591 479 1153         3 Test
## 848 5572 382 1107 1349 1746 1059 2445 1715 703 436 1144        13 Test
##          .rnorm PTS.diff             Team.fctr W.predict.Final.lm
## 854  1.59350961      755 Oklahoma City Thunder           65.90764
## 863  1.43540954     -208    Washington Wizards           34.61608
## 845 -0.27976609      285       Houston Rockets           50.48699
## 840 -0.02319101     -384   Cleveland Cavaliers           28.47063
## 838  0.34644930     -757     Charlotte Bobcats           16.66341
## 848  0.67432067      340     Memphis Grizzlies           52.24151
##     W.predict.Final.lm.err
## 854               5.907643
## 863               5.616085
## 845               5.486987
## 840               4.470631
## 838               4.336589
## 848               3.758489
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
## glb_fin_mdl_id): Limiting important feature scatter plots to 5 out of 7
```

![](NBA_Wins2_files/figure-html/predict.data.new-2.png) ![](NBA_Wins2_files/figure-html/predict.data.new-3.png) ![](NBA_Wins2_files/figure-html/predict.data.new-4.png) ![](NBA_Wins2_files/figure-html/predict.data.new-5.png) ![](NBA_Wins2_files/figure-html/predict.data.new-6.png) 

```
##     SeasonEnd                  Team Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 854      2013 Oklahoma City Thunder        1 60 8669   7914 3126 6504 2528
## 863      2013    Washington Wizards        0 29 7644   7852 2910 6693 2365
## 845      2013       Houston Rockets        1 45 8688   8403 3124 6782 2257
## 840      2013   Cleveland Cavaliers        0 24 7913   8297 2993 6901 2446
## 838      2013     Charlotte Bobcats        0 21 7661   8418 2823 6649 2354
##     X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV .rownames .src
## 854 4916 598 1588 1819 2196  854 2725 1753 679 624 1253        19 Test
## 863 5198 545 1495 1279 1746  887 2652 1775 598 376 1238        28 Test
## 845 4413 867 2369 1573 2087  909 2652 1902 679 359 1348        10 Test
## 840 5320 547 1581 1380 1826 1004 2359 1694 647 334 1149         5 Test
## 838 5250 469 1399 1546 2060  917 2389 1587 591 479 1153         3 Test
##          .rnorm PTS.diff             Team.fctr W.predict.Final.lm
## 854  1.59350961      755 Oklahoma City Thunder           65.90764
## 863  1.43540954     -208    Washington Wizards           34.61608
## 845 -0.27976609      285       Houston Rockets           50.48699
## 840 -0.02319101     -384   Cleveland Cavaliers           28.47063
## 838  0.34644930     -757     Charlotte Bobcats           16.66341
##     W.predict.Final.lm.err .label
## 854               5.907643     19
## 863               5.616085     28
## 845               5.486987     10
## 840               4.470631      5
## 838               4.336589      3
```

![](NBA_Wins2_files/figure-html/predict.data.new-7.png) 

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
## [1] "glb_sel_mdl_id: Interact.High.cor.Y.lm"
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
##                     model_id min.RMSE.OOB max.R.sq.OOB max.Adj.R.sq.fit
## 6     Interact.High.cor.Y.lm     3.055823 0.9434179629        0.9427176
## 14           PTS.interact.lm     3.088843 0.9421885606        0.9422414
## 13               PTS.only.lm     3.099349 0.9417946385        0.9422064
## 7               Low.cor.X.lm     3.113979 0.9412438212        0.9429167
## 5               Max.cor.Y.lm     3.143675 0.9401178663        0.9425832
## 10            All.X.bayesglm     3.192316 0.9382504551               NA
## 8                   All.X.lm     3.192316 0.9382504560        0.9432758
## 9                  All.X.glm     3.192316 0.9382504560               NA
## 12         All.X.no.rnorm.rf     3.412877 0.9294242789               NA
## 3  Max.cor.Y.cv.0.cp.0.rpart     3.494519 0.9260059366               NA
## 4            Max.cor.Y.rpart     4.652407 0.8688472184               NA
## 11      All.X.no.rnorm.rpart     4.652407 0.8688472184               NA
## 1                     MFO.lm    12.844989 0.0002542561       -0.0011621
## 2       Max.cor.Y.cv.0.rpart    12.846623 0.0000000000               NA
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
## [1] "Interact.High.cor.Y.lm OOB RMSE: 3.0558"
## [1] "Final.lm prediction stats for glb_newobs_df:"
##   model_id max.R.sq.new min.RMSE.new
## 1 Final.lm     0.943418     3.055823
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
##                   Interact.High.cor.Y.lm.importance importance
## PTS.diff                                  100.00000  100.00000
## DRB                                        27.25871   27.25871
## `PTS.diff:X3P`                             20.79874   20.79874
## `PTS.diff:FT`                              19.40171   19.40171
## `PTS.diff:X2PA`                            18.88785   18.88785
## `PTS.diff:PTS`                             15.91326   15.91326
## `PTS.diff:oppPTS`                           0.00000    0.00000
##                   Final.lm.importance
## PTS.diff                    100.00000
## DRB                          27.25871
## `PTS.diff:X3P`               20.79874
## `PTS.diff:FT`                19.40171
## `PTS.diff:X2PA`              18.88785
## `PTS.diff:PTS`               15.91326
## `PTS.diff:oppPTS`             0.00000
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
##                   label step_major step_minor     bgn     end elapsed
## 16     predict.data.new          9          0 112.791 117.568   4.777
## 17 display.session.info         10          0 117.568      NA      NA
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
##                      label step_major step_minor     bgn     end elapsed
## 11              fit.models          7          1  43.977  86.431  42.454
## 10              fit.models          7          0  25.441  43.976  18.535
## 12              fit.models          7          2  86.431  99.704  13.273
## 2             inspect.data          2          0  11.902  20.193   8.291
## 15       fit.data.training          8          1 107.738 112.791   5.053
## 16        predict.data.new          9          0 112.791 117.568   4.777
## 13              fit.models          7          3  99.704 103.770   4.067
## 14       fit.data.training          8          0 103.771 107.737   3.967
## 3               scrub.data          2          1  20.193  22.958   2.765
## 6         extract.features          3          0  23.073  24.182   1.109
## 8          select.features          5          0  24.451  25.144   0.693
## 1              import.data          1          0  11.436  11.902   0.466
## 9  partition.data.training          6          0  25.144  25.441   0.297
## 7             cluster.data          4          0  24.182  24.451   0.269
## 5      manage.missing.data          2          3  22.991  23.073   0.082
## 4           transform.data          2          2  22.959  22.991   0.032
##    duration
## 11   42.454
## 10   18.535
## 12   13.273
## 2     8.291
## 15    5.053
## 16    4.777
## 13    4.066
## 14    3.966
## 3     2.765
## 6     1.109
## 8     0.693
## 1     0.466
## 9     0.297
## 7     0.269
## 5     0.082
## 4     0.032
## [1] "Total Elapsed Time: 117.568 secs"
```

![](NBA_Wins2_files/figure-html/display.session.info-1.png) 

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
