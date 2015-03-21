# NBA:  Basketball-Reference.com: Playoffs classification:: Playoffs2
bdanalytics  

**  **    
**Date: (Tue) Jun 16, 2015**    

# Introduction:  

Data: 
Source: 
    Training:   https://courses.edx.org/asset-v1:MITx+15.071x_2a+2T2015+type@asset+block/NBA_train.csv  
    New:        https://courses.edx.org/asset-v1:MITx+15.071x_2a+2T2015+type@asset+block/NBA_test.csv  
Time period: 



# Synopsis:

Based on analysis utilizing <> techniques, <conclusion heading>:  

Low.cor.X.glm: [1, 0]; W 100.00; AST 25.45; STL 23.41; FT 20.34; X3PA 17.24; X2PA 16.24
        W.glm: [1, 0]; W 100.00

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
glb_out_pfx <- "Playoffs2_"
glb_save_envir <- FALSE # or TRUE

glb_is_separate_newent_dataset <- TRUE    # or TRUE
    glb_split_entity_newent_datasets <- TRUE   # or FALSE
    glb_split_newdata_method <- "sample"          # "condition" or "sample" or "copy"
    glb_split_newdata_condition <- NULL # or "is.na(<var>)"; "<var> <condition_operator> <value>"
    glb_split_newdata_size_ratio <- 0.3               # > 0 & < 1
    glb_split_sample.seed <- 123               # or any integer

glb_max_fitent_obs <- NULL # or any integer                         
glb_is_regression <- FALSE; glb_is_classification <- !glb_is_regression; 
    glb_is_binomial <- TRUE # or TRUE or FALSE

glb_rsp_var_raw <- "Playoffs"

# for classification, the response variable has to be a factor
glb_rsp_var <- "Playoffs.fctr"

# if the response factor is based on numbers/logicals e.g (0/1 OR TRUE/FALSE vs. "A"/"B"), 
#   or contains spaces (e.g. "Not in Labor Force")
#   caret predict(..., type="prob") crashes
glb_map_rsp_raw_to_var <- function(raw) {
    relevel(factor(ifelse(raw == 1, "Y", "N")), as.factor(c("Y", "N")), ref="N")
    #as.factor(paste0("B", raw))
    #as.factor(gsub(" ", "\\.", raw))    
}
glb_map_rsp_raw_to_var(c(1, 1, 0, 0, 0))
```

```
## [1] Y Y N N N
## Levels: N Y
```

```r
glb_map_rsp_var_to_raw <- function(var) {
    as.numeric(var) - 1
    #as.numeric(var)
    #gsub("\\.", " ", levels(var)[as.numeric(var)])
    #c(" <=50K", " >50K")[as.numeric(var)]
    #c(FALSE, TRUE)[as.numeric(var)]
}
glb_map_rsp_var_to_raw(glb_map_rsp_raw_to_var(c(1, 1, 0, 0, 0)))
```

```
## [1] 1 1 0 0 0
```

```r
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

# User-specified exclusions  
glb_exclude_vars_as_features <- c("Team.fctr") 
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

![](NBA_Playoffs2_files/figure-html/set_global_options-1.png) 

```r
glb_analytics_avl_objs <- NULL

glb_chunks_df <- myadd_chunk(NULL, "import.data")
```

```
##         label step_major step_minor    bgn end elapsed
## 1 import.data          1          0 10.377  NA      NA
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
## 1  import.data          1          0 10.377 10.802   0.425
## 2 inspect.data          2          0 10.803     NA      NA
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
## Loading required package: reshape2
```

![](NBA_Playoffs2_files/figure-html/inspect.data-1.png) 

```
##       Playoffs.0 Playoffs.1
## Test          14         14
## Train        355        480
##       Playoffs.0 Playoffs.1
## Test   0.5000000  0.5000000
## Train  0.4251497  0.5748503
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
```

```
## Loading required package: sqldf
## Loading required package: gsubfn
## Loading required package: proto
## Loading required package: RSQLite
## Loading required package: DBI
## Loading required package: tcltk
```

```
##   Playoffs Playoffs.fctr  .n
## 1        1             Y 494
## 2        0             N 369
```

![](NBA_Playoffs2_files/figure-html/inspect.data-2.png) 

```
##       Playoffs.fctr.N Playoffs.fctr.Y
## Test               14              14
## Train             355             480
##       Playoffs.fctr.N Playoffs.fctr.Y
## Test        0.5000000       0.5000000
## Train       0.4251497       0.5748503
```

```r
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

![](NBA_Playoffs2_files/figure-html/inspect.data-3.png) 

```
## [1] "feat: W"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-4.png) 

```
## [1] "feat: PTS"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-5.png) 

```
## [1] "feat: oppPTS"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-6.png) 

```
## [1] "feat: FG"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-7.png) 

```
## [1] "feat: FGA"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-8.png) 

```
## [1] "feat: X2P"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-9.png) 

```
## [1] "feat: X2PA"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-10.png) 

```
## [1] "feat: X3P"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-11.png) 

```
## [1] "feat: X3PA"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-12.png) 

```
## [1] "feat: FT"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-13.png) 

```
## [1] "feat: FTA"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-14.png) 

```
## [1] "feat: ORB"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-15.png) 

```
## [1] "feat: DRB"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-16.png) 

```
## [1] "feat: AST"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-17.png) 

```
## [1] "feat: STL"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-18.png) 

```
## [1] "feat: BLK"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-19.png) 

```
## [1] "feat: TOV"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-20.png) 

```
## [1] "feat: .rnorm"
```

![](NBA_Playoffs2_files/figure-html/inspect.data-21.png) 

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
## 2 inspect.data          2          0 10.803 20.805  10.002
## 3   scrub.data          2          1 20.805     NA      NA
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
## 3     scrub.data          2          1 20.805 23.074   2.269
## 4 transform.data          2          2 23.074     NA      NA
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
```

### Step `2.2: transform data`

```r
glb_chunks_df <- myadd_chunk(glb_chunks_df, "manage.missing.data", major.inc=FALSE)
```

```
##                 label step_major step_minor    bgn    end elapsed
## 4      transform.data          2          2 23.074 23.109   0.035
## 5 manage.missing.data          2          3 23.109     NA      NA
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
## 5 manage.missing.data          2          3 23.109 23.202   0.093
## 6    extract.features          3          0 23.202     NA      NA
```

```r
extract.features_chunk_df <- myadd_chunk(NULL, "extract.features_bgn")
```

```
##                  label step_major step_minor   bgn end elapsed
## 1 extract.features_bgn          1          0 23.21  NA      NA
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
##                                 label step_major step_minor   bgn   end
## 1                extract.features_bgn          1          0 23.21 23.22
## 2 extract.features_factorize.str.vars          2          0 23.22    NA
##   elapsed
## 1    0.01
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
## 2 extract.features_factorize.str.vars          2          0 23.220 23.243
## 3                extract.features_end          3          0 23.244     NA
##   elapsed
## 2   0.023
## 3      NA
```

```r
myplt_chunk(extract.features_chunk_df)
```

```
##                                 label step_major step_minor   bgn    end
## 2 extract.features_factorize.str.vars          2          0 23.22 23.243
## 1                extract.features_bgn          1          0 23.21 23.220
##   elapsed duration
## 2   0.023    0.023
## 1   0.010    0.010
## [1] "Total Elapsed Time: 23.243 secs"
```

![](NBA_Playoffs2_files/figure-html/extract.features-1.png) 

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

![](NBA_Playoffs2_files/figure-html/extract.features-2.png) 

```r
glb_chunks_df <- myadd_chunk(glb_chunks_df, "cluster.data", major.inc=TRUE)
```

```
##              label step_major step_minor    bgn    end elapsed
## 6 extract.features          3          0 23.202 24.669   1.467
## 7     cluster.data          4          0 24.669     NA      NA
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
## 7    cluster.data          4          0 24.669 24.983   0.314
## 8 select.features          5          0 24.983     NA      NA
```

## Step `5.0: select features`

```r
print(glb_feats_df <- myselect_features(entity_df=glb_trnobs_df, 
                       exclude_vars_as_features=glb_exclude_vars_as_features, 
                       rsp_var=glb_rsp_var))
```

```
##                  id        cor.y exclude.as.feat   cor.y.abs
## Playoffs   Playoffs  1.000000000               1 1.000000000
## W                 W  0.798675595               0 0.798675595
## DRB             DRB  0.344749300               0 0.344749300
## AST             AST  0.314705084               0 0.314705084
## PTS             PTS  0.270601328               0 0.270601328
## oppPTS       oppPTS -0.232760995               0 0.232760995
## FT               FT  0.221799063               0 0.221799063
## FG               FG  0.187552799               0 0.187552799
## BLK             BLK  0.187056116               0 0.187056116
## FTA             FTA  0.179413465               0 0.179413465
## STL             STL  0.173822947               0 0.173822947
## Team.fctr Team.fctr -0.172716430               1 0.172716430
## TOV             TOV -0.168601605               0 0.168601605
## X2P             X2P  0.108033881               0 0.108033881
## SeasonEnd SeasonEnd -0.059127866               0 0.059127866
## ORB             ORB -0.041702843               0 0.041702843
## .rnorm       .rnorm  0.030300464               0 0.030300464
## X3P             X3P  0.028382516               0 0.028382516
## FGA             FGA -0.011985459               0 0.011985459
## X2PA           X2PA -0.009931886               0 0.009931886
## X3PA           X3PA  0.006570622               0 0.006570622
```

```r
# sav_feats_df <- glb_feats_df; glb_feats_df <- sav_feats_df
print(glb_feats_df <- orderBy(~-cor.y, 
          myfind_cor_features(feats_df=glb_feats_df, obs_df=glb_trnobs_df, 
                              rsp_var=glb_rsp_var)))
```

```
## [1] "cor(FT, FTA)=0.9505"
## [1] "cor(Playoffs.fctr, FT)=0.2218"
## [1] "cor(Playoffs.fctr, FTA)=0.1794"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified FTA as highly correlated with FT
```

```
## [1] "cor(FG, X2P)=0.9429"
## [1] "cor(Playoffs.fctr, FG)=0.1876"
## [1] "cor(Playoffs.fctr, X2P)=0.1080"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified X2P as highly correlated with FG
```

```
## [1] "cor(FG, PTS)=0.9420"
## [1] "cor(Playoffs.fctr, FG)=0.1876"
## [1] "cor(Playoffs.fctr, PTS)=0.2706"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified FG as highly correlated with PTS
```

```
## [1] "cor(oppPTS, PTS)=0.7891"
## [1] "cor(Playoffs.fctr, oppPTS)=-0.2328"
## [1] "cor(Playoffs.fctr, PTS)=0.2706"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified oppPTS as highly correlated with PTS
```

```
## [1] "cor(AST, PTS)=0.7599"
## [1] "cor(Playoffs.fctr, AST)=0.3147"
## [1] "cor(Playoffs.fctr, PTS)=0.2706"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified PTS as highly correlated with AST
```

```
## [1] "cor(SeasonEnd, TOV)=-0.7239"
## [1] "cor(Playoffs.fctr, SeasonEnd)=-0.0591"
## [1] "cor(Playoffs.fctr, TOV)=-0.1686"
```

```
## Warning in myfind_cor_features(feats_df = glb_feats_df, obs_df =
## glb_trnobs_df, : Identified SeasonEnd as highly correlated with TOV
```

```
##           id        cor.y exclude.as.feat   cor.y.abs cor.high.X freqRatio
## 11  Playoffs  1.000000000               1 1.000000000       <NA>  1.352113
## 17         W  0.798675595               0 0.798675595       <NA>  1.000000
## 4        DRB  0.344749300               0 0.344749300       <NA>  1.428571
## 2        AST  0.314705084               0 0.314705084       <NA>  1.000000
## 12       PTS  0.270601328               0 0.270601328        AST  1.000000
## 7         FT  0.221799063               0 0.221799063       <NA>  1.166667
## 5         FG  0.187552799               0 0.187552799        PTS  1.000000
## 3        BLK  0.187056116               0 0.187056116       <NA>  1.125000
## 8        FTA  0.179413465               0 0.179413465         FT  1.000000
## 14       STL  0.173822947               0 0.173822947       <NA>  1.000000
## 18       X2P  0.108033881               0 0.108033881         FG  1.200000
## 1     .rnorm  0.030300464               0 0.030300464       <NA>  1.000000
## 20       X3P  0.028382516               0 0.028382516       <NA>  1.500000
## 21      X3PA  0.006570622               0 0.006570622       <NA>  1.000000
## 19      X2PA -0.009931886               0 0.009931886       <NA>  1.000000
## 6        FGA -0.011985459               0 0.011985459       <NA>  1.000000
## 10       ORB -0.041702843               0 0.041702843       <NA>  1.000000
## 13 SeasonEnd -0.059127866               0 0.059127866        TOV  1.000000
## 16       TOV -0.168601605               0 0.168601605       <NA>  1.166667
## 15 Team.fctr -0.172716430               1 0.172716430       <NA>  1.000000
## 9     oppPTS -0.232760995               0 0.232760995        PTS  1.000000
##    percentUnique zeroVar   nzv myNearZV is.cor.y.abs.low
## 11      0.239521   FALSE FALSE    FALSE            FALSE
## 17      7.065868   FALSE FALSE    FALSE            FALSE
## 4      49.700599   FALSE FALSE    FALSE            FALSE
## 2      62.874251   FALSE FALSE    FALSE            FALSE
## 12     81.197605   FALSE FALSE    FALSE            FALSE
## 7      57.964072   FALSE FALSE    FALSE            FALSE
## 5      70.898204   FALSE FALSE    FALSE            FALSE
## 3      36.526946   FALSE FALSE    FALSE            FALSE
## 8      63.473054   FALSE FALSE    FALSE            FALSE
## 14     38.562874   FALSE FALSE    FALSE            FALSE
## 18     72.934132   FALSE FALSE    FALSE            FALSE
## 1     100.000000   FALSE FALSE    FALSE            FALSE
## 20     57.724551   FALSE FALSE    FALSE             TRUE
## 21     77.844311   FALSE FALSE    FALSE             TRUE
## 19     85.748503   FALSE FALSE    FALSE             TRUE
## 6      76.287425   FALSE FALSE    FALSE             TRUE
## 10     51.736527   FALSE FALSE    FALSE            FALSE
## 13      3.712575   FALSE FALSE    FALSE            FALSE
## 16     53.772455   FALSE FALSE    FALSE            FALSE
## 15      4.431138   FALSE FALSE    FALSE            FALSE
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

![](NBA_Playoffs2_files/figure-html/select.features-1.png) 

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
## 8         select.features          5          0 24.983 25.736   0.754
## 9 partition.data.training          6          0 25.737     NA      NA
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
## [1] "Newdata contains non-NA data for Playoffs.fctr; setting OOB to Newdata"
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
##                          id exclude.as.feat rsp_var
## Playoffs.fctr Playoffs.fctr            TRUE    TRUE
```

```r
glb_feats_df <- myrbind_df(glb_feats_df, add_feats_df)
if (glb_id_var != ".rownames")
    print(subset(glb_feats_df, rsp_var_raw | rsp_var | id_var)) else
    print(subset(glb_feats_df, rsp_var_raw | rsp_var))    
```

```
##                          id cor.y exclude.as.feat cor.y.abs cor.high.X
## 11                 Playoffs     1            TRUE         1       <NA>
## Playoffs.fctr Playoffs.fctr    NA            TRUE        NA       <NA>
##               freqRatio percentUnique zeroVar   nzv myNearZV
## 11             1.352113      0.239521   FALSE FALSE    FALSE
## Playoffs.fctr        NA            NA      NA    NA       NA
##               is.cor.y.abs.low interaction.feat rsp_var_raw rsp_var
## 11                       FALSE               NA        TRUE      NA
## Playoffs.fctr               NA               NA          NA    TRUE
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
##                      label step_major step_minor    bgn   end elapsed
## 9  partition.data.training          6          0 25.737 26.08   0.343
## 10              fit.models          7          0 26.080    NA      NA
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
## [1] "fitting model: MFO.myMFO_classfr"
## [1] "    indep_vars: .rnorm"
## Fitting parameter = none on full training set
## [1] "in MFO.Classifier$fit"
## [1] "unique.vals:"
## [1] N Y
## Levels: N Y
## [1] "unique.prob:"
## y
##         Y         N 
## 0.5748503 0.4251497 
## [1] "MFO.val:"
## [1] "Y"
##             Length Class      Mode     
## unique.vals 2      factor     numeric  
## unique.prob 2      -none-     numeric  
## MFO.val     1      -none-     character
## x.names     1      -none-     character
## xNames      1      -none-     character
## problemType 1      -none-     character
## tuneValue   1      data.frame list     
## obsLevels   2      -none-     character
## [1] "    calling mypredict_mdl for fit:"
```

```
## Loading required package: ROCR
## Loading required package: gplots
## 
## Attaching package: 'gplots'
## 
## The following object is masked from 'package:stats':
## 
##     lowess
```

```
## [1] "in MFO.Classifier$prob"
##           N         Y
## 1 0.5748503 0.4251497
## 2 0.5748503 0.4251497
## 3 0.5748503 0.4251497
## 4 0.5748503 0.4251497
## 5 0.5748503 0.4251497
## 6 0.5748503 0.4251497
## [1] "Classifier Probability Threshold: 0.5000 to maximize f.score.fit"
##   Playoffs.fctr Playoffs.fctr.predict.MFO.myMFO_classfr.N
## 1             N                                       355
## 2             Y                                       480
##          Prediction
## Reference   N   Y
##         N 355   0
##         Y 480   0
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   4.251497e-01   0.000000e+00   3.913345e-01   4.594951e-01   5.748503e-01 
## AccuracyPValue  McnemarPValue 
##   1.000000e+00  5.814170e-106 
## [1] "    calling mypredict_mdl for OOB:"
## [1] "in MFO.Classifier$prob"
##           N         Y
## 1 0.5748503 0.4251497
## 2 0.5748503 0.4251497
## 3 0.5748503 0.4251497
## 4 0.5748503 0.4251497
## 5 0.5748503 0.4251497
## 6 0.5748503 0.4251497
## [1] "Classifier Probability Threshold: 0.5000 to maximize f.score.OOB"
##   Playoffs.fctr Playoffs.fctr.predict.MFO.myMFO_classfr.N
## 1             N                                        14
## 2             Y                                        14
##          Prediction
## Reference  N  Y
##         N 14  0
##         Y 14  0
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   0.5000000000   0.0000000000   0.3064709615   0.6935290385   0.5000000000 
## AccuracyPValue  McnemarPValue 
##   0.5747229904   0.0005120045 
##            model_id  model_method  feats max.nTuningRuns
## 1 MFO.myMFO_classfr myMFO_classfr .rnorm               0
##   min.elapsedtime.everything min.elapsedtime.final max.auc.fit
## 1                      0.305                 0.002         0.5
##   opt.prob.threshold.fit max.f.score.fit max.Accuracy.fit
## 1                    0.5               0        0.4251497
##   max.AccuracyLower.fit max.AccuracyUpper.fit max.Kappa.fit max.auc.OOB
## 1             0.3913345             0.4594951             0         0.5
##   opt.prob.threshold.OOB max.f.score.OOB max.Accuracy.OOB
## 1                    0.5               0              0.5
##   max.AccuracyLower.OOB max.AccuracyUpper.OOB max.Kappa.OOB
## 1              0.306471              0.693529             0
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
```

```
## [1] "fitting model: Random.myrandom_classfr"
## [1] "    indep_vars: .rnorm"
## Fitting parameter = none on full training set
##             Length Class      Mode     
## unique.vals 2      factor     numeric  
## unique.prob 2      table      numeric  
## xNames      1      -none-     character
## problemType 1      -none-     character
## tuneValue   1      data.frame list     
## obsLevels   2      -none-     character
## [1] "    calling mypredict_mdl for fit:"
## [1] "in Random.Classifier$prob"
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-1.png) 

```
##    threshold   f.score
## 1        0.0 0.7300380
## 2        0.1 0.7300380
## 3        0.2 0.7300380
## 4        0.3 0.7300380
## 5        0.4 0.7300380
## 6        0.5 0.5422315
## 7        0.6 0.0000000
## 8        0.7 0.0000000
## 9        0.8 0.0000000
## 10       0.9 0.0000000
## 11       1.0 0.0000000
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-2.png) 

```
## [1] "Classifier Probability Threshold: 0.4000 to maximize f.score.fit"
##   Playoffs.fctr Playoffs.fctr.predict.Random.myrandom_classfr.Y
## 1             N                                             355
## 2             Y                                             480
##          Prediction
## Reference   N   Y
##         N   0 355
##         Y   0 480
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   5.748503e-01   0.000000e+00   5.405049e-01   6.086655e-01   5.748503e-01 
## AccuracyPValue  McnemarPValue 
##   5.146549e-01   9.402461e-79 
## [1] "    calling mypredict_mdl for OOB:"
## [1] "in Random.Classifier$prob"
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-3.png) 

```
##    threshold   f.score
## 1        0.0 0.6666667
## 2        0.1 0.6666667
## 3        0.2 0.6666667
## 4        0.3 0.6666667
## 5        0.4 0.6666667
## 6        0.5 0.4666667
## 7        0.6 0.0000000
## 8        0.7 0.0000000
## 9        0.8 0.0000000
## 10       0.9 0.0000000
## 11       1.0 0.0000000
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-4.png) 

```
## [1] "Classifier Probability Threshold: 0.4000 to maximize f.score.OOB"
##   Playoffs.fctr Playoffs.fctr.predict.Random.myrandom_classfr.Y
## 1             N                                              14
## 2             Y                                              14
##          Prediction
## Reference  N  Y
##         N  0 14
##         Y  0 14
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   0.5000000000   0.0000000000   0.3064709615   0.6935290385   0.5000000000 
## AccuracyPValue  McnemarPValue 
##   0.5747229904   0.0005120045 
##                  model_id     model_method  feats max.nTuningRuns
## 1 Random.myrandom_classfr myrandom_classfr .rnorm               0
##   min.elapsedtime.everything min.elapsedtime.final max.auc.fit
## 1                      0.229                 0.002   0.4623826
##   opt.prob.threshold.fit max.f.score.fit max.Accuracy.fit
## 1                    0.4        0.730038        0.5748503
##   max.AccuracyLower.fit max.AccuracyUpper.fit max.Kappa.fit max.auc.OOB
## 1             0.5405049             0.6086655             0   0.4285714
##   opt.prob.threshold.OOB max.f.score.OOB max.Accuracy.OOB
## 1                    0.4       0.6666667              0.5
##   max.AccuracyLower.OOB max.AccuracyUpper.OOB max.Kappa.OOB
## 1              0.306471              0.693529             0
```

```r
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
## [1] "    indep_vars: W, DRB"
```

```
## Loading required package: rpart
```

```
## Fitting cp = 0.811 on full training set
```

```
## Loading required package: rpart.plot
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-5.png) 

```
## Call:
## rpart(formula = .outcome ~ ., control = list(minsplit = 20, minbucket = 7, 
##     cp = 0, maxcompete = 4, maxsurrogate = 5, usesurrogate = 2, 
##     surrogatestyle = 0, maxdepth = 30, xval = 0))
##   n= 835 
## 
##          CP nsplit rel error
## 1 0.8112676      0         1
## 
## Node number 1: 835 observations
##   predicted class=Y  expected loss=0.4251497  P(node) =1
##     class counts:   355   480
##    probabilities: 0.425 0.575 
## 
## n= 835 
## 
## node), split, n, loss, yval, (yprob)
##       * denotes terminal node
## 
## 1) root 835 355 Y (0.4251497 0.5748503) *
## [1] "    calling mypredict_mdl for fit:"
## [1] "Classifier Probability Threshold: 0.5000 to maximize f.score.fit"
##   Playoffs.fctr Playoffs.fctr.predict.Max.cor.Y.cv.0.rpart.Y
## 1             N                                          355
## 2             Y                                          480
##          Prediction
## Reference   N   Y
##         N   0 355
##         Y   0 480
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   5.748503e-01   0.000000e+00   5.405049e-01   6.086655e-01   5.748503e-01 
## AccuracyPValue  McnemarPValue 
##   5.146549e-01   9.402461e-79 
## [1] "    calling mypredict_mdl for OOB:"
## [1] "Classifier Probability Threshold: 0.5000 to maximize f.score.OOB"
##   Playoffs.fctr Playoffs.fctr.predict.Max.cor.Y.cv.0.rpart.Y
## 1             N                                           14
## 2             Y                                           14
##          Prediction
## Reference  N  Y
##         N  0 14
##         Y  0 14
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   0.5000000000   0.0000000000   0.3064709615   0.6935290385   0.5000000000 
## AccuracyPValue  McnemarPValue 
##   0.5747229904   0.0005120045 
##               model_id model_method  feats max.nTuningRuns
## 1 Max.cor.Y.cv.0.rpart        rpart W, DRB               0
##   min.elapsedtime.everything min.elapsedtime.final max.auc.fit
## 1                      0.635                 0.019         0.5
##   opt.prob.threshold.fit max.f.score.fit max.Accuracy.fit
## 1                    0.5        0.730038        0.5748503
##   max.AccuracyLower.fit max.AccuracyUpper.fit max.Kappa.fit max.auc.OOB
## 1             0.5405049             0.6086655             0         0.5
##   opt.prob.threshold.OOB max.f.score.OOB max.Accuracy.OOB
## 1                    0.5       0.6666667              0.5
##   max.AccuracyLower.OOB max.AccuracyUpper.OOB max.Kappa.OOB
## 1              0.306471              0.693529             0
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
## [1] "    indep_vars: W, DRB"
## Fitting cp = 0 on full training set
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-6.png) 

```
## Call:
## rpart(formula = .outcome ~ ., control = list(minsplit = 20, minbucket = 7, 
##     cp = 0, maxcompete = 4, maxsurrogate = 5, usesurrogate = 2, 
##     surrogatestyle = 0, maxdepth = 30, xval = 0))
##   n= 835 
## 
##            CP nsplit rel error
## 1 0.811267606      0 1.0000000
## 2 0.004929577      1 0.1887324
## 3 0.002816901      5 0.1690141
## 4 0.000000000      9 0.1577465
## 
## Variable importance
##   W DRB 
##  84  16 
## 
## Node number 1: 835 observations,    complexity param=0.8112676
##   predicted class=Y  expected loss=0.4251497  P(node) =1
##     class counts:   355   480
##    probabilities: 0.425 0.575 
##   left son=2 (328 obs) right son=3 (507 obs)
##   Primary splits:
##       W   < 38.5   to the left,  improve=285.29670, (0 missing)
##       DRB < 2386   to the left,  improve= 34.12326, (0 missing)
##   Surrogate splits:
##       DRB < 2372.5 to the left,  agree=0.674, adj=0.171, (0 split)
## 
## Node number 2: 328 observations,    complexity param=0.002816901
##   predicted class=N  expected loss=0.06097561  P(node) =0.3928144
##     class counts:   308    20
##    probabilities: 0.939 0.061 
##   left son=4 (257 obs) right son=5 (71 obs)
##   Primary splits:
##       W   < 34.5   to the left,  improve=6.7188650, (0 missing)
##       DRB < 2454.5 to the right, improve=0.4746166, (0 missing)
##   Surrogate splits:
##       DRB < 2629.5 to the left,  agree=0.79, adj=0.028, (0 split)
## 
## Node number 3: 507 observations,    complexity param=0.004929577
##   predicted class=Y  expected loss=0.09270217  P(node) =0.6071856
##     class counts:    47   460
##    probabilities: 0.093 0.907 
##   left son=6 (116 obs) right son=7 (391 obs)
##   Primary splits:
##       W   < 42.5   to the left,  improve=16.59687, (0 missing)
##       DRB < 2584   to the left,  improve= 1.75519, (0 missing)
## 
## Node number 4: 257 observations
##   predicted class=N  expected loss=0.007782101  P(node) =0.3077844
##     class counts:   255     2
##    probabilities: 0.992 0.008 
## 
## Node number 5: 71 observations,    complexity param=0.002816901
##   predicted class=N  expected loss=0.2535211  P(node) =0.08502994
##     class counts:    53    18
##    probabilities: 0.746 0.254 
##   left son=10 (20 obs) right son=11 (51 obs)
##   Primary splits:
##       DRB < 2454.5 to the right, improve=2.306573, (0 missing)
##       W   < 37.5   to the left,  improve=1.728001, (0 missing)
## 
## Node number 6: 116 observations,    complexity param=0.004929577
##   predicted class=Y  expected loss=0.3275862  P(node) =0.1389222
##     class counts:    38    78
##    probabilities: 0.328 0.672 
##   left son=12 (42 obs) right son=13 (74 obs)
##   Primary splits:
##       W   < 40.5   to the left,  improve=2.050681, (0 missing)
##       DRB < 2280   to the left,  improve=1.624203, (0 missing)
## 
## Node number 7: 391 observations
##   predicted class=Y  expected loss=0.0230179  P(node) =0.4682635
##     class counts:     9   382
##    probabilities: 0.023 0.977 
## 
## Node number 10: 20 observations
##   predicted class=N  expected loss=0.05  P(node) =0.0239521
##     class counts:    19     1
##    probabilities: 0.950 0.050 
## 
## Node number 11: 51 observations,    complexity param=0.002816901
##   predicted class=N  expected loss=0.3333333  P(node) =0.06107784
##     class counts:    34    17
##    probabilities: 0.667 0.333 
##   left son=22 (40 obs) right son=23 (11 obs)
##   Primary splits:
##       W   < 37.5   to the left,  improve=2.575758, (0 missing)
##       DRB < 2405   to the left,  improve=1.468286, (0 missing)
## 
## Node number 12: 42 observations,    complexity param=0.004929577
##   predicted class=Y  expected loss=0.452381  P(node) =0.0502994
##     class counts:    19    23
##    probabilities: 0.452 0.548 
##   left son=24 (12 obs) right son=25 (30 obs)
##   Primary splits:
##       DRB < 2462   to the right, improve=1.5428570, (0 missing)
##       W   < 39.5   to the left,  improve=0.1731602, (0 missing)
## 
## Node number 13: 74 observations,    complexity param=0.004929577
##   predicted class=Y  expected loss=0.2567568  P(node) =0.08862275
##     class counts:    19    55
##    probabilities: 0.257 0.743 
##   left son=26 (9 obs) right son=27 (65 obs)
##   Primary splits:
##       DRB < 2280.5 to the left,  improve=3.4432430, (0 missing)
##       W   < 41.5   to the left,  improve=0.2432432, (0 missing)
## 
## Node number 22: 40 observations
##   predicted class=N  expected loss=0.25  P(node) =0.04790419
##     class counts:    30    10
##    probabilities: 0.750 0.250 
## 
## Node number 23: 11 observations
##   predicted class=Y  expected loss=0.3636364  P(node) =0.01317365
##     class counts:     4     7
##    probabilities: 0.364 0.636 
## 
## Node number 24: 12 observations
##   predicted class=N  expected loss=0.3333333  P(node) =0.01437126
##     class counts:     8     4
##    probabilities: 0.667 0.333 
## 
## Node number 25: 30 observations,    complexity param=0.002816901
##   predicted class=Y  expected loss=0.3666667  P(node) =0.03592814
##     class counts:    11    19
##    probabilities: 0.367 0.633 
##   left son=50 (19 obs) right son=51 (11 obs)
##   Primary splits:
##       DRB < 2391   to the left,  improve=2.6414670, (0 missing)
##       W   < 39.5   to the left,  improve=0.9333333, (0 missing)
## 
## Node number 26: 9 observations
##   predicted class=N  expected loss=0.3333333  P(node) =0.01077844
##     class counts:     6     3
##    probabilities: 0.667 0.333 
## 
## Node number 27: 65 observations
##   predicted class=Y  expected loss=0.2  P(node) =0.07784431
##     class counts:    13    52
##    probabilities: 0.200 0.800 
## 
## Node number 50: 19 observations
##   predicted class=N  expected loss=0.4736842  P(node) =0.02275449
##     class counts:    10     9
##    probabilities: 0.526 0.474 
## 
## Node number 51: 11 observations
##   predicted class=Y  expected loss=0.09090909  P(node) =0.01317365
##     class counts:     1    10
##    probabilities: 0.091 0.909 
## 
## n= 835 
## 
## node), split, n, loss, yval, (yprob)
##       * denotes terminal node
## 
##  1) root 835 355 Y (0.425149701 0.574850299)  
##    2) W< 38.5 328  20 N (0.939024390 0.060975610)  
##      4) W< 34.5 257   2 N (0.992217899 0.007782101) *
##      5) W>=34.5 71  18 N (0.746478873 0.253521127)  
##       10) DRB>=2454.5 20   1 N (0.950000000 0.050000000) *
##       11) DRB< 2454.5 51  17 N (0.666666667 0.333333333)  
##         22) W< 37.5 40  10 N (0.750000000 0.250000000) *
##         23) W>=37.5 11   4 Y (0.363636364 0.636363636) *
##    3) W>=38.5 507  47 Y (0.092702170 0.907297830)  
##      6) W< 42.5 116  38 Y (0.327586207 0.672413793)  
##       12) W< 40.5 42  19 Y (0.452380952 0.547619048)  
##         24) DRB>=2462 12   4 N (0.666666667 0.333333333) *
##         25) DRB< 2462 30  11 Y (0.366666667 0.633333333)  
##           50) DRB< 2391 19   9 N (0.526315789 0.473684211) *
##           51) DRB>=2391 11   1 Y (0.090909091 0.909090909) *
##       13) W>=40.5 74  19 Y (0.256756757 0.743243243)  
##         26) DRB< 2280.5 9   3 N (0.666666667 0.333333333) *
##         27) DRB>=2280.5 65  13 Y (0.200000000 0.800000000) *
##      7) W>=42.5 391   9 Y (0.023017903 0.976982097) *
## [1] "    calling mypredict_mdl for fit:"
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-7.png) 

```
##    threshold   f.score
## 1        0.0 0.7300380
## 2        0.1 0.9190751
## 3        0.2 0.9190751
## 4        0.3 0.9358717
## 5        0.4 0.9416581
## 6        0.5 0.9415449
## 7        0.6 0.9415449
## 8        0.7 0.9376980
## 9        0.8 0.8888889
## 10       0.9 0.8888889
## 11       1.0 0.0000000
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-8.png) 

```
## [1] "Classifier Probability Threshold: 0.4000 to maximize f.score.fit"
##   Playoffs.fctr Playoffs.fctr.predict.Max.cor.Y.cv.0.cp.0.rpart.N
## 1             N                                               318
## 2             Y                                                20
##   Playoffs.fctr.predict.Max.cor.Y.cv.0.cp.0.rpart.Y
## 1                                                37
## 2                                               460
##          Prediction
## Reference   N   Y
##         N 318  37
##         Y  20 460
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.317365e-01   8.594670e-01   9.124581e-01   9.478914e-01   5.748503e-01 
## AccuracyPValue  McnemarPValue 
##  7.665483e-120   3.406920e-02 
## [1] "    calling mypredict_mdl for OOB:"
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-9.png) 

```
##    threshold   f.score
## 1        0.0 0.6666667
## 2        0.1 0.8965517
## 3        0.2 0.8965517
## 4        0.3 0.8965517
## 5        0.4 0.8965517
## 6        0.5 0.8965517
## 7        0.6 0.8965517
## 8        0.7 0.8965517
## 9        0.8 0.9285714
## 10       0.9 0.9285714
## 11       1.0 0.0000000
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-10.png) 

```
## [1] "Classifier Probability Threshold: 0.9000 to maximize f.score.OOB"
##   Playoffs.fctr Playoffs.fctr.predict.Max.cor.Y.cv.0.cp.0.rpart.N
## 1             N                                                13
## 2             Y                                                 1
##   Playoffs.fctr.predict.Max.cor.Y.cv.0.cp.0.rpart.Y
## 1                                                 1
## 2                                                13
##          Prediction
## Reference  N  Y
##         N 13  1
##         Y  1 13
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.285714e-01   8.571429e-01   7.649652e-01   9.912295e-01   5.000000e-01 
## AccuracyPValue  McnemarPValue 
##   1.516193e-06   1.000000e+00 
##                    model_id model_method  feats max.nTuningRuns
## 1 Max.cor.Y.cv.0.cp.0.rpart        rpart W, DRB               0
##   min.elapsedtime.everything min.elapsedtime.final max.auc.fit
## 1                      0.459                 0.016   0.9727201
##   opt.prob.threshold.fit max.f.score.fit max.Accuracy.fit
## 1                    0.4       0.9416581        0.9317365
##   max.AccuracyLower.fit max.AccuracyUpper.fit max.Kappa.fit max.auc.OOB
## 1             0.9124581             0.9478914      0.859467   0.9566327
##   opt.prob.threshold.OOB max.f.score.OOB max.Accuracy.OOB
## 1                    0.9       0.9285714        0.9285714
##   max.AccuracyLower.OOB max.AccuracyUpper.OOB max.Kappa.OOB
## 1             0.7649652             0.9912295     0.8571429
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
## [1] "    indep_vars: W, DRB"
## Aggregating results
## Selecting tuning parameters
## Fitting cp = 0.00493 on full training set
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-11.png) ![](NBA_Playoffs2_files/figure-html/fit.models_0-12.png) 

```
## Call:
## rpart(formula = .outcome ~ ., control = list(minsplit = 20, minbucket = 7, 
##     cp = 0, maxcompete = 4, maxsurrogate = 5, usesurrogate = 2, 
##     surrogatestyle = 0, maxdepth = 30, xval = 0))
##   n= 835 
## 
##            CP nsplit rel error
## 1 0.811267606      0 1.0000000
## 2 0.004929577      1 0.1887324
## 
## Variable importance
##   W DRB 
##  85  15 
## 
## Node number 1: 835 observations,    complexity param=0.8112676
##   predicted class=Y  expected loss=0.4251497  P(node) =1
##     class counts:   355   480
##    probabilities: 0.425 0.575 
##   left son=2 (328 obs) right son=3 (507 obs)
##   Primary splits:
##       W   < 38.5   to the left,  improve=285.29670, (0 missing)
##       DRB < 2386   to the left,  improve= 34.12326, (0 missing)
##   Surrogate splits:
##       DRB < 2372.5 to the left,  agree=0.674, adj=0.171, (0 split)
## 
## Node number 2: 328 observations
##   predicted class=N  expected loss=0.06097561  P(node) =0.3928144
##     class counts:   308    20
##    probabilities: 0.939 0.061 
## 
## Node number 3: 507 observations
##   predicted class=Y  expected loss=0.09270217  P(node) =0.6071856
##     class counts:    47   460
##    probabilities: 0.093 0.907 
## 
## n= 835 
## 
## node), split, n, loss, yval, (yprob)
##       * denotes terminal node
## 
## 1) root 835 355 Y (0.42514970 0.57485030)  
##   2) W< 38.5 328  20 N (0.93902439 0.06097561) *
##   3) W>=38.5 507  47 Y (0.09270217 0.90729783) *
## [1] "    calling mypredict_mdl for fit:"
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-13.png) 

```
##    threshold   f.score
## 1        0.0 0.7300380
## 2        0.1 0.9321175
## 3        0.2 0.9321175
## 4        0.3 0.9321175
## 5        0.4 0.9321175
## 6        0.5 0.9321175
## 7        0.6 0.9321175
## 8        0.7 0.9321175
## 9        0.8 0.9321175
## 10       0.9 0.9321175
## 11       1.0 0.0000000
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-14.png) 

```
## [1] "Classifier Probability Threshold: 0.9000 to maximize f.score.fit"
##   Playoffs.fctr Playoffs.fctr.predict.Max.cor.Y.rpart.N
## 1             N                                     308
## 2             Y                                      20
##   Playoffs.fctr.predict.Max.cor.Y.rpart.Y
## 1                                      47
## 2                                     460
##          Prediction
## Reference   N   Y
##         N 308  47
##         Y  20 460
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.197605e-01   8.342002e-01   8.992163e-01   9.372770e-01   5.748503e-01 
## AccuracyPValue  McnemarPValue 
##  3.265448e-110   1.491123e-03 
## [1] "    calling mypredict_mdl for OOB:"
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-15.png) 

```
##    threshold   f.score
## 1        0.0 0.6666667
## 2        0.1 0.8965517
## 3        0.2 0.8965517
## 4        0.3 0.8965517
## 5        0.4 0.8965517
## 6        0.5 0.8965517
## 7        0.6 0.8965517
## 8        0.7 0.8965517
## 9        0.8 0.8965517
## 10       0.9 0.8965517
## 11       1.0 0.0000000
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-16.png) 

```
## [1] "Classifier Probability Threshold: 0.9000 to maximize f.score.OOB"
##   Playoffs.fctr Playoffs.fctr.predict.Max.cor.Y.rpart.N
## 1             N                                      12
## 2             Y                                       1
##   Playoffs.fctr.predict.Max.cor.Y.rpart.Y
## 1                                       2
## 2                                      13
##          Prediction
## Reference  N  Y
##         N 12  2
##         Y  1 13
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   8.928571e-01   7.857143e-01   7.177356e-01   9.773349e-01   5.000000e-01 
## AccuracyPValue  McnemarPValue 
##   1.372024e-05   1.000000e+00 
##          model_id model_method  feats max.nTuningRuns
## 1 Max.cor.Y.rpart        rpart W, DRB               3
##   min.elapsedtime.everything min.elapsedtime.final max.auc.fit
## 1                      1.004                 0.019   0.9129695
##   opt.prob.threshold.fit max.f.score.fit max.Accuracy.fit
## 1                    0.9       0.9321175        0.9053875
##   max.AccuracyLower.fit max.AccuracyUpper.fit max.Kappa.fit max.auc.OOB
## 1             0.8992163              0.937277     0.8048875   0.8928571
##   opt.prob.threshold.OOB max.f.score.OOB max.Accuracy.OOB
## 1                    0.9       0.8965517        0.8928571
##   max.AccuracyLower.OOB max.AccuracyUpper.OOB max.Kappa.OOB
## 1             0.7177356             0.9773349     0.7857143
##   max.AccuracySD.fit max.KappaSD.fit
## 1         0.03060035      0.06433844
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
## [1] "fitting model: Max.cor.Y.glm"
## [1] "    indep_vars: W, DRB"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-17.png) ![](NBA_Playoffs2_files/figure-html/fit.models_0-18.png) ![](NBA_Playoffs2_files/figure-html/fit.models_0-19.png) ![](NBA_Playoffs2_files/figure-html/fit.models_0-20.png) 

```
## 
## Call:
## NULL
## 
## Deviance Residuals: 
##      Min        1Q    Median        3Q       Max  
## -2.88252  -0.08966   0.01378   0.18339   2.92215  
## 
## Coefficients:
##               Estimate Std. Error z value Pr(>|z|)    
## (Intercept) -1.706e+01  3.417e+00  -4.993 5.93e-07 ***
## W            4.738e-01  4.085e-02  11.598  < 2e-16 ***
## DRB         -6.183e-04  1.324e-03  -0.467     0.64    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for binomial family taken to be 1)
## 
##     Null deviance: 1138.77  on 834  degrees of freedom
## Residual deviance:  308.68  on 832  degrees of freedom
## AIC: 314.68
## 
## Number of Fisher Scoring iterations: 8
## 
## [1] "    calling mypredict_mdl for fit:"
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-21.png) 

```
##    threshold   f.score
## 1        0.0 0.7300380
## 2        0.1 0.9035917
## 3        0.2 0.9208211
## 4        0.3 0.9321357
## 5        0.4 0.9321175
## 6        0.5 0.9318182
## 7        0.6 0.9320794
## 8        0.7 0.9102703
## 9        0.8 0.8856172
## 10       0.9 0.8537736
## 11       1.0 0.0000000
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-22.png) 

```
## [1] "Classifier Probability Threshold: 0.3000 to maximize f.score.fit"
##   Playoffs.fctr Playoffs.fctr.predict.Max.cor.Y.glm.N
## 1             N                                   300
## 2             Y                                    13
##   Playoffs.fctr.predict.Max.cor.Y.glm.Y
## 1                                    55
## 2                                   467
##          Prediction
## Reference   N   Y
##         N 300  55
##         Y  13 467
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.185629e-01   8.307853e-01   8.978991e-01   9.362084e-01   5.748503e-01 
## AccuracyPValue  McnemarPValue 
##  2.733522e-109   6.627244e-07 
## [1] "    calling mypredict_mdl for OOB:"
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-23.png) 

```
##    threshold   f.score
## 1        0.0 0.6666667
## 2        0.1 0.9333333
## 3        0.2 0.9333333
## 4        0.3 0.9333333
## 5        0.4 0.8965517
## 6        0.5 0.8965517
## 7        0.6 0.8965517
## 8        0.7 0.9285714
## 9        0.8 0.9285714
## 10       0.9 0.9230769
## 11       1.0 0.0000000
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-24.png) 

```
## [1] "Classifier Probability Threshold: 0.3000 to maximize f.score.OOB"
##   Playoffs.fctr Playoffs.fctr.predict.Max.cor.Y.glm.N
## 1             N                                    12
## 2             Y                                    NA
##   Playoffs.fctr.predict.Max.cor.Y.glm.Y
## 1                                     2
## 2                                    14
##          Prediction
## Reference  N  Y
##         N 12  2
##         Y  0 14
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.285714e-01   8.571429e-01   7.649652e-01   9.912295e-01   5.000000e-01 
## AccuracyPValue  McnemarPValue 
##   1.516193e-06   4.795001e-01 
##        model_id model_method  feats max.nTuningRuns
## 1 Max.cor.Y.glm          glm W, DRB               1
##   min.elapsedtime.everything min.elapsedtime.final max.auc.fit
## 1                      0.914                 0.022   0.9778638
##   opt.prob.threshold.fit max.f.score.fit max.Accuracy.fit
## 1                    0.3       0.9321357        0.9173779
##   max.AccuracyLower.fit max.AccuracyUpper.fit max.Kappa.fit max.auc.OOB
## 1             0.8978991             0.9362084     0.8299249   0.9897959
##   opt.prob.threshold.OOB max.f.score.OOB max.Accuracy.OOB
## 1                    0.3       0.9333333        0.9285714
##   max.AccuracyLower.OOB max.AccuracyUpper.OOB max.Kappa.OOB min.aic.fit
## 1             0.7649652             0.9912295     0.8571429    314.6768
##   max.AccuracySD.fit max.KappaSD.fit
## 1         0.02848851      0.05891038
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
## [1] "fitting model: Interact.High.cor.Y.glm"
## [1] "    indep_vars: W, DRB, W:AST, W:PTS, W:FT, W:FG, W:TOV"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-25.png) ![](NBA_Playoffs2_files/figure-html/fit.models_0-26.png) ![](NBA_Playoffs2_files/figure-html/fit.models_0-27.png) ![](NBA_Playoffs2_files/figure-html/fit.models_0-28.png) 

```
## 
## Call:
## NULL
## 
## Deviance Residuals: 
##      Min        1Q    Median        3Q       Max  
## -2.55528  -0.06660   0.00817   0.14413   2.87976  
## 
## Coefficients:
##               Estimate Std. Error z value Pr(>|z|)    
## (Intercept) -2.026e+01  3.856e+00  -5.254 1.49e-07 ***
## W            4.484e-01  9.067e-02   4.946 7.58e-07 ***
## DRB          2.964e-04  1.429e-03   0.207  0.83563    
## `W:AST`      1.087e-04  3.424e-05   3.176  0.00150 ** 
## `W:PTS`     -4.034e-05  3.182e-05  -1.268  0.20490    
## `W:FT`       1.051e-04  3.820e-05   2.751  0.00594 ** 
## `W:FG`       8.670e-06  6.062e-05   0.143  0.88627    
## `W:TOV`     -1.532e-05  3.867e-05  -0.396  0.69198    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for binomial family taken to be 1)
## 
##     Null deviance: 1138.8  on 834  degrees of freedom
## Residual deviance:  281.8  on 827  degrees of freedom
## AIC: 297.8
## 
## Number of Fisher Scoring iterations: 8
## 
## [1] "    calling mypredict_mdl for fit:"
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-29.png) 

```
##    threshold   f.score
## 1        0.0 0.7300380
## 2        0.1 0.9127517
## 3        0.2 0.9303238
## 4        0.3 0.9313433
## 5        0.4 0.9344097
## 6        0.5 0.9396111
## 7        0.6 0.9347368
## 8        0.7 0.9179266
## 9        0.8 0.9026549
## 10       0.9 0.8517200
## 11       1.0 0.0000000
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-30.png) 

```
## [1] "Classifier Probability Threshold: 0.5000 to maximize f.score.fit"
##   Playoffs.fctr Playoffs.fctr.predict.Interact.High.cor.Y.glm.N
## 1             N                                             317
## 2             Y                                              21
##   Playoffs.fctr.predict.Interact.High.cor.Y.glm.Y
## 1                                              38
## 2                                             459
##          Prediction
## Reference   N   Y
##         N 317  38
##         Y  21 459
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.293413e-01   8.545361e-01   9.097989e-01   9.457796e-01   5.748503e-01 
## AccuracyPValue  McnemarPValue 
##  7.437607e-118   3.724917e-02 
## [1] "    calling mypredict_mdl for OOB:"
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-31.png) 

```
##    threshold   f.score
## 1        0.0 0.6666667
## 2        0.1 0.9333333
## 3        0.2 0.8965517
## 4        0.3 0.8965517
## 5        0.4 0.8965517
## 6        0.5 0.9285714
## 7        0.6 0.9285714
## 8        0.7 0.9285714
## 9        0.8 0.9230769
## 10       0.9 0.8800000
## 11       1.0 0.0000000
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-32.png) 

```
## [1] "Classifier Probability Threshold: 0.1000 to maximize f.score.OOB"
##   Playoffs.fctr Playoffs.fctr.predict.Interact.High.cor.Y.glm.N
## 1             N                                              12
## 2             Y                                              NA
##   Playoffs.fctr.predict.Interact.High.cor.Y.glm.Y
## 1                                               2
## 2                                              14
##          Prediction
## Reference  N  Y
##         N 12  2
##         Y  0 14
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.285714e-01   8.571429e-01   7.649652e-01   9.912295e-01   5.000000e-01 
## AccuracyPValue  McnemarPValue 
##   1.516193e-06   4.795001e-01 
##                  model_id model_method
## 1 Interact.High.cor.Y.glm          glm
##                                     feats max.nTuningRuns
## 1 W, DRB, W:AST, W:PTS, W:FT, W:FG, W:TOV               1
##   min.elapsedtime.everything min.elapsedtime.final max.auc.fit
## 1                      1.179                 0.033   0.9818075
##   opt.prob.threshold.fit max.f.score.fit max.Accuracy.fit
## 1                    0.5       0.9396111        0.9221741
##   max.AccuracyLower.fit max.AccuracyUpper.fit max.Kappa.fit max.auc.OOB
## 1             0.9097989             0.9457796     0.8400543   0.9846939
##   opt.prob.threshold.OOB max.f.score.OOB max.Accuracy.OOB
## 1                    0.1       0.9333333        0.9285714
##   max.AccuracyLower.OOB max.AccuracyUpper.OOB max.Kappa.OOB min.aic.fit
## 1             0.7649652             0.9912295     0.8571429    297.8003
##   max.AccuracySD.fit max.KappaSD.fit
## 1         0.02095379      0.04392501
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
## [1] "fitting model: Low.cor.X.glm"
## [1] "    indep_vars: W, DRB, AST, FT, BLK, STL, .rnorm, X3P, X3PA, X2PA, FGA, ORB, TOV"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-33.png) ![](NBA_Playoffs2_files/figure-html/fit.models_0-34.png) ![](NBA_Playoffs2_files/figure-html/fit.models_0-35.png) 

```
## 
## Call:
## NULL
## 
## Deviance Residuals: 
##      Min        1Q    Median        3Q       Max  
## -2.67406  -0.06962   0.00809   0.14806   2.97710  
## 
## Coefficients: (1 not defined because of singularities)
##               Estimate Std. Error z value Pr(>|z|)    
## (Intercept) -2.349e+01  5.439e+00  -4.318 1.57e-05 ***
## W            4.765e-01  4.746e-02  10.039  < 2e-16 ***
## DRB          1.368e-03  1.967e-03   0.696  0.48669    
## AST          3.415e-03  1.301e-03   2.625  0.00867 ** 
## FT           2.207e-03  1.000e-03   2.207  0.02734 *  
## BLK          1.270e-03  2.032e-03   0.625  0.53212    
## STL          5.769e-03  2.776e-03   2.078  0.03770 *  
## .rnorm       2.020e-01  1.544e-01   1.308  0.19080    
## X3P         -9.499e-03  8.863e-03  -1.072  0.28385    
## X3PA         1.773e-03  3.654e-03   0.485  0.62747    
## X2PA        -1.287e-03  8.678e-04  -1.483  0.13805    
## FGA                 NA         NA      NA       NA    
## ORB         -1.185e-03  1.947e-03  -0.609  0.54282    
## TOV         -1.998e-03  1.849e-03  -1.080  0.27992    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for binomial family taken to be 1)
## 
##     Null deviance: 1138.77  on 834  degrees of freedom
## Residual deviance:  275.27  on 822  degrees of freedom
## AIC: 301.27
## 
## Number of Fisher Scoring iterations: 8
## 
## [1] "    calling mypredict_mdl for fit:"
```

```
## Warning in predict.lm(object, newdata, se.fit, scale = 1, type =
## ifelse(type == : prediction from a rank-deficient fit may be misleading
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-36.png) ![](NBA_Playoffs2_files/figure-html/fit.models_0-37.png) 

```
##    threshold   f.score
## 1        0.0 0.7300380
## 2        0.1 0.9127517
## 3        0.2 0.9266862
## 4        0.3 0.9304175
## 5        0.4 0.9338759
## 6        0.5 0.9411765
## 7        0.6 0.9412998
## 8        0.7 0.9202586
## 9        0.8 0.9042316
## 10       0.9 0.8714953
## 11       1.0 0.0000000
```

```
## [1] "Classifier Probability Threshold: 0.6000 to maximize f.score.fit"
##   Playoffs.fctr Playoffs.fctr.predict.Low.cor.X.glm.N
## 1             N                                   330
## 2             Y                                    31
##   Playoffs.fctr.predict.Low.cor.X.glm.Y
## 1                                    25
## 2                                   449
##          Prediction
## Reference   N   Y
##         N 330  25
##         Y  31 449
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.329341e-01   8.630947e-01   9.137899e-01   9.489450e-01   5.748503e-01 
## AccuracyPValue  McnemarPValue 
##  7.568269e-121   5.040359e-01 
## [1] "    calling mypredict_mdl for OOB:"
```

```
## Warning in predict.lm(object, newdata, se.fit, scale = 1, type =
## ifelse(type == : prediction from a rank-deficient fit may be misleading
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-38.png) ![](NBA_Playoffs2_files/figure-html/fit.models_0-39.png) 

```
##    threshold   f.score
## 1        0.0 0.6666667
## 2        0.1 0.9333333
## 3        0.2 0.9333333
## 4        0.3 0.8965517
## 5        0.4 0.8965517
## 6        0.5 0.8965517
## 7        0.6 0.8965517
## 8        0.7 0.9285714
## 9        0.8 0.9285714
## 10       0.9 0.8800000
## 11       1.0 0.0000000
```

![](NBA_Playoffs2_files/figure-html/fit.models_0-40.png) 

```
## [1] "Classifier Probability Threshold: 0.2000 to maximize f.score.OOB"
##   Playoffs.fctr Playoffs.fctr.predict.Low.cor.X.glm.N
## 1             N                                    12
## 2             Y                                    NA
##   Playoffs.fctr.predict.Low.cor.X.glm.Y
## 1                                     2
## 2                                    14
##          Prediction
## Reference  N  Y
##         N 12  2
##         Y  0 14
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.285714e-01   8.571429e-01   7.649652e-01   9.912295e-01   5.000000e-01 
## AccuracyPValue  McnemarPValue 
##   1.516193e-06   4.795001e-01 
##        model_id model_method
## 1 Low.cor.X.glm          glm
##                                                               feats
## 1 W, DRB, AST, FT, BLK, STL, .rnorm, X3P, X3PA, X2PA, FGA, ORB, TOV
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      1.008                 0.053
##   max.auc.fit opt.prob.threshold.fit max.f.score.fit max.Accuracy.fit
## 1   0.9827054                    0.6       0.9412998        0.9233602
##   max.AccuracyLower.fit max.AccuracyUpper.fit max.Kappa.fit max.auc.OOB
## 1             0.9137899              0.948945     0.8426415   0.9846939
##   opt.prob.threshold.OOB max.f.score.OOB max.Accuracy.OOB
## 1                    0.2       0.9333333        0.9285714
##   max.AccuracyLower.OOB max.AccuracyUpper.OOB max.Kappa.OOB min.aic.fit
## 1             0.7649652             0.9912295     0.8571429    301.2685
##   max.AccuracySD.fit max.KappaSD.fit
## 1         0.02391429      0.04992758
```

```r
rm(ret_lst)

glb_chunks_df <- myadd_chunk(glb_chunks_df, "fit.models", major.inc=FALSE)
```

```
##         label step_major step_minor   bgn    end elapsed
## 10 fit.models          7          0 26.08 49.399  23.319
## 11 fit.models          7          1 49.40     NA      NA
```


```r
fit.models_1_chunk_df <- myadd_chunk(NULL, "fit.models_1_bgn")
```

```
##              label step_major step_minor    bgn end elapsed
## 1 fit.models_1_bgn          1          0 53.058  NA      NA
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
## 1 fit.models_1_bgn          1          0 53.058 53.072   0.014
## 2 fit.models_1_glm          2          0 53.073     NA      NA
## [1] "fitting model: All.X.glm"
## [1] "    indep_vars: W, DRB, AST, PTS, FT, FG, BLK, FTA, STL, X2P, .rnorm, X3P, X3PA, X2PA, FGA, ORB, SeasonEnd, TOV, oppPTS"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-1.png) ![](NBA_Playoffs2_files/figure-html/fit.models_1-2.png) ![](NBA_Playoffs2_files/figure-html/fit.models_1-3.png) 

```
## 
## Call:
## NULL
## 
## Deviance Residuals: 
##      Min        1Q    Median        3Q       Max  
## -2.53981  -0.06407   0.00768   0.13929   3.00413  
## 
## Coefficients: (3 not defined because of singularities)
##               Estimate Std. Error z value Pr(>|z|)    
## (Intercept) 76.6077099 94.1498423   0.814  0.41583    
## W            0.4874659  0.0686597   7.100 1.25e-12 ***
## DRB          0.0013320  0.0023542   0.566  0.57153    
## AST          0.0039410  0.0014844   2.655  0.00793 ** 
## PTS         -0.0105494  0.0094248  -1.119  0.26300    
## FT           0.0180695  0.0103434   1.747  0.08064 .  
## FG           0.0189566  0.0194098   0.977  0.32874    
## BLK          0.0014321  0.0021015   0.681  0.49560    
## FTA         -0.0040937  0.0027653  -1.480  0.13876    
## STL          0.0053928  0.0030949   1.742  0.08142 .  
## X2P                 NA         NA      NA       NA    
## .rnorm       0.2251992  0.1581554   1.424  0.15447    
## X3P                 NA         NA      NA       NA    
## X3PA         0.0034918  0.0040019   0.873  0.38292    
## X2PA        -0.0003084  0.0018774  -0.164  0.86952    
## FGA                 NA         NA      NA       NA    
## ORB         -0.0015066  0.0024775  -0.608  0.54310    
## SeasonEnd   -0.0496359  0.0464344  -1.069  0.28509    
## TOV         -0.0013896  0.0024207  -0.574  0.56593    
## oppPTS      -0.0003193  0.0019623  -0.163  0.87073    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for binomial family taken to be 1)
## 
##     Null deviance: 1138.77  on 834  degrees of freedom
## Residual deviance:  269.42  on 818  degrees of freedom
## AIC: 303.42
## 
## Number of Fisher Scoring iterations: 8
## 
## [1] "    calling mypredict_mdl for fit:"
```

```
## Warning in predict.lm(object, newdata, se.fit, scale = 1, type =
## ifelse(type == : prediction from a rank-deficient fit may be misleading
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-4.png) ![](NBA_Playoffs2_files/figure-html/fit.models_1-5.png) 

```
##    threshold   f.score
## 1        0.0 0.7300380
## 2        0.1 0.9110048
## 3        0.2 0.9262537
## 4        0.3 0.9321357
## 5        0.4 0.9349593
## 6        0.5 0.9388601
## 7        0.6 0.9390756
## 8        0.7 0.9277238
## 9        0.8 0.9133333
## 10       0.9 0.8780488
## 11       1.0 0.0000000
```

```
## [1] "Classifier Probability Threshold: 0.6000 to maximize f.score.fit"
##   Playoffs.fctr Playoffs.fctr.predict.All.X.glm.N
## 1             N                               330
## 2             Y                                33
##   Playoffs.fctr.predict.All.X.glm.Y
## 1                                25
## 2                               447
##          Prediction
## Reference   N   Y
##         N 330  25
##         Y  33 447
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.305389e-01   8.583090e-01   9.111278e-01   9.468362e-01   5.748503e-01 
## AccuracyPValue  McnemarPValue 
##  7.620361e-119   3.580197e-01 
## [1] "    calling mypredict_mdl for OOB:"
```

```
## Warning in predict.lm(object, newdata, se.fit, scale = 1, type =
## ifelse(type == : prediction from a rank-deficient fit may be misleading
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-6.png) ![](NBA_Playoffs2_files/figure-html/fit.models_1-7.png) 

```
##    threshold   f.score
## 1        0.0 0.6666667
## 2        0.1 0.9333333
## 3        0.2 0.8965517
## 4        0.3 0.8965517
## 5        0.4 0.8965517
## 6        0.5 0.8965517
## 7        0.6 0.8965517
## 8        0.7 0.9285714
## 9        0.8 0.8461538
## 10       0.9 0.8333333
## 11       1.0 0.0000000
```

```
## [1] "Classifier Probability Threshold: 0.1000 to maximize f.score.OOB"
##   Playoffs.fctr Playoffs.fctr.predict.All.X.glm.N
## 1             N                                12
## 2             Y                                NA
##   Playoffs.fctr.predict.All.X.glm.Y
## 1                                 2
## 2                                14
##          Prediction
## Reference  N  Y
##         N 12  2
##         Y  0 14
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.285714e-01   8.571429e-01   7.649652e-01   9.912295e-01   5.000000e-01 
## AccuracyPValue  McnemarPValue 
##   1.516193e-06   4.795001e-01 
##    model_id model_method
## 1 All.X.glm          glm
##                                                                                                     feats
## 1 W, DRB, AST, PTS, FT, FG, BLK, FTA, STL, X2P, .rnorm, X3P, X3PA, X2PA, FGA, ORB, SeasonEnd, TOV, oppPTS
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      1.006                 0.073
##   max.auc.fit opt.prob.threshold.fit max.f.score.fit max.Accuracy.fit
## 1   0.9839613                    0.6       0.9390756        0.9125904
##   max.AccuracyLower.fit max.AccuracyUpper.fit max.Kappa.fit max.auc.OOB
## 1             0.9111278             0.9468362     0.8210586   0.9795918
##   opt.prob.threshold.OOB max.f.score.OOB max.Accuracy.OOB
## 1                    0.1       0.9333333        0.9285714
##   max.AccuracyLower.OOB max.AccuracyUpper.OOB max.Kappa.OOB min.aic.fit
## 1             0.7649652             0.9912295     0.8571429    303.4227
##   max.AccuracySD.fit max.KappaSD.fit
## 1            0.01686      0.03578204
##                   label step_major step_minor    bgn   end elapsed
## 2      fit.models_1_glm          2          0 53.073 57.25   4.177
## 3 fit.models_1_bayesglm          3          0 57.250    NA      NA
## [1] "fitting model: All.X.bayesglm"
## [1] "    indep_vars: W, DRB, AST, PTS, FT, FG, BLK, FTA, STL, X2P, .rnorm, X3P, X3PA, X2PA, FGA, ORB, SeasonEnd, TOV, oppPTS"
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

![](NBA_Playoffs2_files/figure-html/fit.models_1-8.png) 

```
## Aggregating results
## Fitting final model on full training set
## 
## Call:
## NULL
## 
## Deviance Residuals: 
##      Min        1Q    Median        3Q       Max  
## -2.42502  -0.06836   0.00910   0.15041   2.99093  
## 
## Coefficients:
##               Estimate Std. Error z value Pr(>|z|)    
## (Intercept)  4.418e+01  8.079e+01   0.547  0.58448    
## W            4.628e-01  6.018e-02   7.690 1.47e-14 ***
## DRB          1.167e-03  1.962e-03   0.595  0.55189    
## AST          3.570e-03  1.368e-03   2.609  0.00909 ** 
## PTS         -1.889e-04  1.529e-03  -0.124  0.90168    
## FT           5.514e-03  2.843e-03   1.939  0.05246 .  
## FG          -1.012e-03  3.189e-03  -0.317  0.75106    
## BLK          1.279e-03  2.004e-03   0.638  0.52334    
## FTA         -1.931e-03  2.075e-03  -0.930  0.35221    
## STL          5.307e-03  2.604e-03   2.038  0.04154 *  
## X2P         -4.399e-05  2.194e-03  -0.020  0.98400    
## .rnorm       2.105e-01  1.528e-01   1.378  0.16833    
## X3P         -1.773e-03  4.073e-03  -0.435  0.66332    
## X3PA         3.808e-04  1.555e-03   0.245  0.80657    
## X2PA        -1.115e-04  1.125e-03  -0.099  0.92105    
## FGA          1.793e-04  1.574e-03   0.114  0.90931    
## ORB         -1.764e-03  2.028e-03  -0.870  0.38429    
## SeasonEnd   -3.328e-02  3.979e-02  -0.836  0.40290    
## TOV         -1.376e-03  2.024e-03  -0.680  0.49672    
## oppPTS      -7.295e-04  1.598e-03  -0.456  0.64809    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for binomial family taken to be 1)
## 
##     Null deviance: 1138.77  on 834  degrees of freedom
## Residual deviance:  271.01  on 815  degrees of freedom
## AIC: 311.01
## 
## Number of Fisher Scoring iterations: 14
## 
## [1] "    calling mypredict_mdl for fit:"
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-9.png) 

```
##    threshold   f.score
## 1        0.0 0.7300380
## 2        0.1 0.9110048
## 3        0.2 0.9263984
## 4        0.3 0.9314796
## 5        0.4 0.9361702
## 6        0.5 0.9399586
## 7        0.6 0.9390756
## 8        0.7 0.9245690
## 9        0.8 0.9072626
## 10       0.9 0.8711944
## 11       1.0 0.0000000
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-10.png) 

```
## [1] "Classifier Probability Threshold: 0.5000 to maximize f.score.fit"
##   Playoffs.fctr Playoffs.fctr.predict.All.X.bayesglm.N
## 1             N                                    323
## 2             Y                                     26
##   Playoffs.fctr.predict.All.X.bayesglm.Y
## 1                                     32
## 2                                    454
##          Prediction
## Reference   N   Y
##         N 323  32
##         Y  26 454
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.305389e-01   8.575798e-01   9.111278e-01   9.468362e-01   5.748503e-01 
## AccuracyPValue  McnemarPValue 
##  7.620361e-119   5.114818e-01 
## [1] "    calling mypredict_mdl for OOB:"
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-11.png) 

```
##    threshold   f.score
## 1        0.0 0.6666667
## 2        0.1 0.9333333
## 3        0.2 0.9333333
## 4        0.3 0.8965517
## 5        0.4 0.8965517
## 6        0.5 0.8965517
## 7        0.6 0.8965517
## 8        0.7 0.9285714
## 9        0.8 0.9285714
## 10       0.9 0.8333333
## 11       1.0 0.0000000
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-12.png) 

```
## [1] "Classifier Probability Threshold: 0.2000 to maximize f.score.OOB"
##   Playoffs.fctr Playoffs.fctr.predict.All.X.bayesglm.N
## 1             N                                     12
## 2             Y                                     NA
##   Playoffs.fctr.predict.All.X.bayesglm.Y
## 1                                      2
## 2                                     14
##          Prediction
## Reference  N  Y
##         N 12  2
##         Y  0 14
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.285714e-01   8.571429e-01   7.649652e-01   9.912295e-01   5.000000e-01 
## AccuracyPValue  McnemarPValue 
##   1.516193e-06   4.795001e-01 
##         model_id model_method
## 1 All.X.bayesglm     bayesglm
##                                                                                                     feats
## 1 W, DRB, AST, PTS, FT, FG, BLK, FTA, STL, X2P, .rnorm, X3P, X3PA, X2PA, FGA, ORB, SeasonEnd, TOV, oppPTS
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               1                      1.797                 0.117
##   max.auc.fit opt.prob.threshold.fit max.f.score.fit max.Accuracy.fit
## 1   0.9836092                    0.5       0.9399586        0.9173779
##   max.AccuracyLower.fit max.AccuracyUpper.fit max.Kappa.fit max.auc.OOB
## 1             0.9111278             0.9468362     0.8306003   0.9795918
##   opt.prob.threshold.OOB max.f.score.OOB max.Accuracy.OOB
## 1                    0.2       0.9333333        0.9285714
##   max.AccuracyLower.OOB max.AccuracyUpper.OOB max.Kappa.OOB min.aic.fit
## 1             0.7649652             0.9912295     0.8571429    311.0097
##   max.AccuracySD.fit max.KappaSD.fit
## 1         0.01859518      0.03915204
##                   label step_major step_minor    bgn    end elapsed
## 3 fit.models_1_bayesglm          3          0 57.250 61.638   4.388
## 4    fit.models_1_rpart          4          0 61.639     NA      NA
## [1] "fitting model: All.X.no.rnorm.rpart"
## [1] "    indep_vars: W, DRB, AST, PTS, FT, FG, BLK, FTA, STL, X2P, X3P, X3PA, X2PA, FGA, ORB, SeasonEnd, TOV, oppPTS"
## Aggregating results
## Selecting tuning parameters
## Fitting cp = 0.00563 on full training set
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-13.png) ![](NBA_Playoffs2_files/figure-html/fit.models_1-14.png) 

```
## Call:
## rpart(formula = .outcome ~ ., control = list(minsplit = 20, minbucket = 7, 
##     cp = 0, maxcompete = 4, maxsurrogate = 5, usesurrogate = 2, 
##     surrogatestyle = 0, maxdepth = 30, xval = 0))
##   n= 835 
## 
##            CP nsplit rel error
## 1 0.811267606      0 1.0000000
## 2 0.005633803      1 0.1887324
## 
## Variable importance
##      W    DRB    AST oppPTS    PTS    BLK 
##     66     11      7      6      5      4 
## 
## Node number 1: 835 observations,    complexity param=0.8112676
##   predicted class=Y  expected loss=0.4251497  P(node) =1
##     class counts:   355   480
##    probabilities: 0.425 0.575 
##   left son=2 (328 obs) right son=3 (507 obs)
##   Primary splits:
##       W      < 38.5   to the left,  improve=285.29670, (0 missing)
##       DRB    < 2386   to the left,  improve= 34.12326, (0 missing)
##       AST    < 1991.5 to the left,  improve= 33.29244, (0 missing)
##       oppPTS < 7961   to the right, improve= 29.65890, (0 missing)
##       PTS    < 8686   to the left,  improve= 20.67583, (0 missing)
##   Surrogate splits:
##       DRB    < 2372.5 to the left,  agree=0.674, adj=0.171, (0 split)
##       AST    < 1749.5 to the left,  agree=0.649, adj=0.107, (0 split)
##       oppPTS < 8820   to the right, agree=0.641, adj=0.085, (0 split)
##       PTS    < 7726.5 to the left,  agree=0.637, adj=0.076, (0 split)
##       BLK    < 310.5  to the left,  agree=0.634, adj=0.067, (0 split)
## 
## Node number 2: 328 observations
##   predicted class=N  expected loss=0.06097561  P(node) =0.3928144
##     class counts:   308    20
##    probabilities: 0.939 0.061 
## 
## Node number 3: 507 observations
##   predicted class=Y  expected loss=0.09270217  P(node) =0.6071856
##     class counts:    47   460
##    probabilities: 0.093 0.907 
## 
## n= 835 
## 
## node), split, n, loss, yval, (yprob)
##       * denotes terminal node
## 
## 1) root 835 355 Y (0.42514970 0.57485030)  
##   2) W< 38.5 328  20 N (0.93902439 0.06097561) *
##   3) W>=38.5 507  47 Y (0.09270217 0.90729783) *
## [1] "    calling mypredict_mdl for fit:"
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-15.png) 

```
##    threshold   f.score
## 1        0.0 0.7300380
## 2        0.1 0.9321175
## 3        0.2 0.9321175
## 4        0.3 0.9321175
## 5        0.4 0.9321175
## 6        0.5 0.9321175
## 7        0.6 0.9321175
## 8        0.7 0.9321175
## 9        0.8 0.9321175
## 10       0.9 0.9321175
## 11       1.0 0.0000000
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-16.png) 

```
## [1] "Classifier Probability Threshold: 0.9000 to maximize f.score.fit"
##   Playoffs.fctr Playoffs.fctr.predict.All.X.no.rnorm.rpart.N
## 1             N                                          308
## 2             Y                                           20
##   Playoffs.fctr.predict.All.X.no.rnorm.rpart.Y
## 1                                           47
## 2                                          460
##          Prediction
## Reference   N   Y
##         N 308  47
##         Y  20 460
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.197605e-01   8.342002e-01   8.992163e-01   9.372770e-01   5.748503e-01 
## AccuracyPValue  McnemarPValue 
##  3.265448e-110   1.491123e-03 
## [1] "    calling mypredict_mdl for OOB:"
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-17.png) 

```
##    threshold   f.score
## 1        0.0 0.6666667
## 2        0.1 0.8965517
## 3        0.2 0.8965517
## 4        0.3 0.8965517
## 5        0.4 0.8965517
## 6        0.5 0.8965517
## 7        0.6 0.8965517
## 8        0.7 0.8965517
## 9        0.8 0.8965517
## 10       0.9 0.8965517
## 11       1.0 0.0000000
```

```
## [1] "Classifier Probability Threshold: 0.9000 to maximize f.score.OOB"
##   Playoffs.fctr Playoffs.fctr.predict.All.X.no.rnorm.rpart.N
## 1             N                                           12
## 2             Y                                            1
##   Playoffs.fctr.predict.All.X.no.rnorm.rpart.Y
## 1                                            2
## 2                                           13
##          Prediction
## Reference  N  Y
##         N 12  2
##         Y  1 13
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   8.928571e-01   7.857143e-01   7.177356e-01   9.773349e-01   5.000000e-01 
## AccuracyPValue  McnemarPValue 
##   1.372024e-05   1.000000e+00 
##               model_id model_method
## 1 All.X.no.rnorm.rpart        rpart
##                                                                                             feats
## 1 W, DRB, AST, PTS, FT, FG, BLK, FTA, STL, X2P, X3P, X3PA, X2PA, FGA, ORB, SeasonEnd, TOV, oppPTS
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               3                       1.09                  0.06
##   max.auc.fit opt.prob.threshold.fit max.f.score.fit max.Accuracy.fit
## 1   0.9129695                    0.9       0.9321175        0.8981976
##   max.AccuracyLower.fit max.AccuracyUpper.fit max.Kappa.fit max.auc.OOB
## 1             0.8992163              0.937277     0.7908403   0.8928571
##   opt.prob.threshold.OOB max.f.score.OOB max.Accuracy.OOB
## 1                    0.9       0.8965517        0.8928571
##   max.AccuracyLower.OOB max.AccuracyUpper.OOB max.Kappa.OOB
## 1             0.7177356             0.9773349     0.7857143
##   max.AccuracySD.fit max.KappaSD.fit
## 1         0.00840949      0.01827986
##                label step_major step_minor    bgn    end elapsed
## 4 fit.models_1_rpart          4          0 61.639 66.101   4.462
## 5    fit.models_1_rf          5          0 66.102     NA      NA
## [1] "fitting model: All.X.no.rnorm.rf"
## [1] "    indep_vars: W, DRB, AST, PTS, FT, FG, BLK, FTA, STL, X2P, X3P, X3PA, X2PA, FGA, ORB, SeasonEnd, TOV, oppPTS"
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

![](NBA_Playoffs2_files/figure-html/fit.models_1-18.png) 

```
## Aggregating results
## Selecting tuning parameters
## Fitting mtry = 10 on full training set
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-19.png) ![](NBA_Playoffs2_files/figure-html/fit.models_1-20.png) 

```
##                 Length Class      Mode     
## call               4   -none-     call     
## type               1   -none-     character
## predicted        835   factor     numeric  
## err.rate        1500   -none-     numeric  
## confusion          6   -none-     numeric  
## votes           1670   matrix     numeric  
## oob.times        835   -none-     numeric  
## classes            2   -none-     character
## importance        18   -none-     numeric  
## importanceSD       0   -none-     NULL     
## localImportance    0   -none-     NULL     
## proximity          0   -none-     NULL     
## ntree              1   -none-     numeric  
## mtry               1   -none-     numeric  
## forest            14   -none-     list     
## y                835   factor     numeric  
## test               0   -none-     NULL     
## inbag              0   -none-     NULL     
## xNames            18   -none-     character
## problemType        1   -none-     character
## tuneValue          1   data.frame list     
## obsLevels          2   -none-     character
## [1] "    calling mypredict_mdl for fit:"
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-21.png) 

```
##    threshold   f.score
## 1        0.0 0.7300380
## 2        0.1 0.9356725
## 3        0.2 0.9628887
## 4        0.3 0.9886715
## 5        0.4 1.0000000
## 6        0.5 1.0000000
## 7        0.6 0.9989572
## 8        0.7 0.9915966
## 9        0.8 0.9765458
## 10       0.9 0.9309577
## 11       1.0 0.2545455
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-22.png) 

```
## [1] "Classifier Probability Threshold: 0.5000 to maximize f.score.fit"
##   Playoffs.fctr Playoffs.fctr.predict.All.X.no.rnorm.rf.N
## 1             N                                       355
## 2             Y                                        NA
##   Playoffs.fctr.predict.All.X.no.rnorm.rf.Y
## 1                                        NA
## 2                                       480
##          Prediction
## Reference   N   Y
##         N 355   0
##         Y   0 480
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   1.000000e+00   1.000000e+00   9.955919e-01   1.000000e+00   5.748503e-01 
## AccuracyPValue  McnemarPValue 
##  1.691322e-201            NaN 
## [1] "    calling mypredict_mdl for OOB:"
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-23.png) 

```
##    threshold   f.score
## 1        0.0 0.6666667
## 2        0.1 0.9032258
## 3        0.2 0.9333333
## 4        0.3 0.9333333
## 5        0.4 0.8965517
## 6        0.5 0.8965517
## 7        0.6 0.9285714
## 8        0.7 0.8888889
## 9        0.8 0.8000000
## 10       0.9 0.7272727
## 11       1.0 0.0000000
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-24.png) 

```
## [1] "Classifier Probability Threshold: 0.3000 to maximize f.score.OOB"
##   Playoffs.fctr Playoffs.fctr.predict.All.X.no.rnorm.rf.N
## 1             N                                        12
## 2             Y                                        NA
##   Playoffs.fctr.predict.All.X.no.rnorm.rf.Y
## 1                                         2
## 2                                        14
##          Prediction
## Reference  N  Y
##         N 12  2
##         Y  0 14
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.285714e-01   8.571429e-01   7.649652e-01   9.912295e-01   5.000000e-01 
## AccuracyPValue  McnemarPValue 
##   1.516193e-06   4.795001e-01 
##            model_id model_method
## 1 All.X.no.rnorm.rf           rf
##                                                                                             feats
## 1 W, DRB, AST, PTS, FT, FG, BLK, FTA, STL, X2P, X3P, X3PA, X2PA, FGA, ORB, SeasonEnd, TOV, oppPTS
##   max.nTuningRuns min.elapsedtime.everything min.elapsedtime.final
## 1               3                      7.341                 1.381
##   max.auc.fit opt.prob.threshold.fit max.f.score.fit max.Accuracy.fit
## 1           1                    0.5               1        0.9197717
##   max.AccuracyLower.fit max.AccuracyUpper.fit max.Kappa.fit max.auc.OOB
## 1             0.9955919                     1     0.8343263   0.9642857
##   opt.prob.threshold.OOB max.f.score.OOB max.Accuracy.OOB
## 1                    0.3       0.9333333        0.9285714
##   max.AccuracyLower.OOB max.AccuracyUpper.OOB max.Kappa.OOB
## 1             0.7649652             0.9912295     0.8571429
##   max.AccuracySD.fit max.KappaSD.fit
## 1          0.0197341      0.04207761
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
model_id <- "W.only"; indep_vars_vctr <- c(NULL
   ,"W", ".rnorm"
#    ,"<feat1>*<feat2>"
#    ,"<feat1>:<feat2>"
                                           )
for (method in c("glm")) {
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
## [1] "fitting model: W.only.glm"
## [1] "    indep_vars: W, .rnorm"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-25.png) ![](NBA_Playoffs2_files/figure-html/fit.models_1-26.png) ![](NBA_Playoffs2_files/figure-html/fit.models_1-27.png) ![](NBA_Playoffs2_files/figure-html/fit.models_1-28.png) 

```
## 
## Call:
## NULL
## 
## Deviance Residuals: 
##      Min        1Q    Median        3Q       Max  
## -2.85595  -0.08894   0.01313   0.19013   2.93072  
## 
## Coefficients:
##             Estimate Std. Error z value Pr(>|z|)    
## (Intercept) -18.5144     1.6261 -11.385   <2e-16 ***
## W             0.4727     0.0407  11.614   <2e-16 ***
## .rnorm        0.1124     0.1451   0.775    0.439    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for binomial family taken to be 1)
## 
##     Null deviance: 1138.8  on 834  degrees of freedom
## Residual deviance:  308.3  on 832  degrees of freedom
## AIC: 314.3
## 
## Number of Fisher Scoring iterations: 8
## 
## [1] "    calling mypredict_mdl for fit:"
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-29.png) 

```
##    threshold   f.score
## 1        0.0 0.7300380
## 2        0.1 0.9035917
## 3        0.2 0.9199219
## 4        0.3 0.9321357
## 5        0.4 0.9333333
## 6        0.5 0.9320988
## 7        0.6 0.9275971
## 8        0.7 0.9161290
## 9        0.8 0.8951522
## 10       0.9 0.8524203
## 11       1.0 0.0000000
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-30.png) 

```
## [1] "Classifier Probability Threshold: 0.4000 to maximize f.score.fit"
##   Playoffs.fctr Playoffs.fctr.predict.W.only.glm.N
## 1             N                                307
## 2             Y                                 18
##   Playoffs.fctr.predict.W.only.glm.Y
## 1                                 48
## 2                                462
##          Prediction
## Reference   N   Y
##         N 307  48
##         Y  18 462
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.209581e-01   8.364931e-01   9.005347e-01   9.383443e-01   5.748503e-01 
## AccuracyPValue  McnemarPValue 
##  3.838559e-111   3.574541e-04 
## [1] "    calling mypredict_mdl for OOB:"
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-31.png) 

```
##    threshold   f.score
## 1        0.0 0.6666667
## 2        0.1 0.9333333
## 3        0.2 0.9333333
## 4        0.3 0.9333333
## 5        0.4 0.9333333
## 6        0.5 0.8965517
## 7        0.6 0.8965517
## 8        0.7 0.8965517
## 9        0.8 0.9285714
## 10       0.9 0.9629630
## 11       1.0 0.0000000
```

![](NBA_Playoffs2_files/figure-html/fit.models_1-32.png) 

```
## [1] "Classifier Probability Threshold: 0.9000 to maximize f.score.OOB"
##   Playoffs.fctr Playoffs.fctr.predict.W.only.glm.N
## 1             N                                 14
## 2             Y                                  1
##   Playoffs.fctr.predict.W.only.glm.Y
## 1                                 NA
## 2                                 13
##          Prediction
## Reference  N  Y
##         N 14  0
##         Y  1 13
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.642857e-01   9.285714e-01   8.165224e-01   9.990962e-01   5.000000e-01 
## AccuracyPValue  McnemarPValue 
##   1.080334e-07   1.000000e+00 
##     model_id model_method     feats max.nTuningRuns
## 1 W.only.glm          glm W, .rnorm               1
##   min.elapsedtime.everything min.elapsedtime.final max.auc.fit
## 1                       1.21                  0.02    0.978257
##   opt.prob.threshold.fit max.f.score.fit max.Accuracy.fit
## 1                    0.4       0.9333333        0.9149798
##   max.AccuracyLower.fit max.AccuracyUpper.fit max.Kappa.fit max.auc.OOB
## 1             0.9005347             0.9383443     0.8248854   0.9897959
##   opt.prob.threshold.OOB max.f.score.OOB max.Accuracy.OOB
## 1                    0.9        0.962963        0.9642857
##   max.AccuracyLower.OOB max.AccuracyUpper.OOB max.Kappa.OOB min.aic.fit
## 1             0.8165224             0.9990962     0.9285714    314.2956
##   max.AccuracySD.fit max.KappaSD.fit
## 1         0.03489742      0.07191793
##        importance
## W             100
## .rnorm          0
```

```r
# model_id <- "W.only.no.Playoffs.fctr"; indep_vars_vctr <- c(NULL
#    ,"W"
# #    ,"<feat1>*<feat2>"
# #    ,"<feat1>:<feat2>"
#                                            )
# for (method in c("lm")) {
#     ret_lst <- myfit_mdl(model_id=model_id, model_method=method,
#                                 indep_vars_vctr=indep_vars_vctr,
#                                 model_type=glb_model_type,
#                                 rsp_var="Playoffs", rsp_var_out="Playoffs.predict.",
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
##                                            model_id     model_method
## MFO.myMFO_classfr                 MFO.myMFO_classfr    myMFO_classfr
## Random.myrandom_classfr     Random.myrandom_classfr myrandom_classfr
## Max.cor.Y.cv.0.rpart           Max.cor.Y.cv.0.rpart            rpart
## Max.cor.Y.cv.0.cp.0.rpart Max.cor.Y.cv.0.cp.0.rpart            rpart
## Max.cor.Y.rpart                     Max.cor.Y.rpart            rpart
## Max.cor.Y.glm                         Max.cor.Y.glm              glm
## Interact.High.cor.Y.glm     Interact.High.cor.Y.glm              glm
## Low.cor.X.glm                         Low.cor.X.glm              glm
## All.X.glm                                 All.X.glm              glm
## All.X.bayesglm                       All.X.bayesglm         bayesglm
## All.X.no.rnorm.rpart           All.X.no.rnorm.rpart            rpart
## All.X.no.rnorm.rf                 All.X.no.rnorm.rf               rf
## W.only.glm                               W.only.glm              glm
##                                                                                                                             feats
## MFO.myMFO_classfr                                                                                                          .rnorm
## Random.myrandom_classfr                                                                                                    .rnorm
## Max.cor.Y.cv.0.rpart                                                                                                       W, DRB
## Max.cor.Y.cv.0.cp.0.rpart                                                                                                  W, DRB
## Max.cor.Y.rpart                                                                                                            W, DRB
## Max.cor.Y.glm                                                                                                              W, DRB
## Interact.High.cor.Y.glm                                                                   W, DRB, W:AST, W:PTS, W:FT, W:FG, W:TOV
## Low.cor.X.glm                                                   W, DRB, AST, FT, BLK, STL, .rnorm, X3P, X3PA, X2PA, FGA, ORB, TOV
## All.X.glm                 W, DRB, AST, PTS, FT, FG, BLK, FTA, STL, X2P, .rnorm, X3P, X3PA, X2PA, FGA, ORB, SeasonEnd, TOV, oppPTS
## All.X.bayesglm            W, DRB, AST, PTS, FT, FG, BLK, FTA, STL, X2P, .rnorm, X3P, X3PA, X2PA, FGA, ORB, SeasonEnd, TOV, oppPTS
## All.X.no.rnorm.rpart              W, DRB, AST, PTS, FT, FG, BLK, FTA, STL, X2P, X3P, X3PA, X2PA, FGA, ORB, SeasonEnd, TOV, oppPTS
## All.X.no.rnorm.rf                 W, DRB, AST, PTS, FT, FG, BLK, FTA, STL, X2P, X3P, X3PA, X2PA, FGA, ORB, SeasonEnd, TOV, oppPTS
## W.only.glm                                                                                                              W, .rnorm
##                           max.nTuningRuns min.elapsedtime.everything
## MFO.myMFO_classfr                       0                      0.305
## Random.myrandom_classfr                 0                      0.229
## Max.cor.Y.cv.0.rpart                    0                      0.635
## Max.cor.Y.cv.0.cp.0.rpart               0                      0.459
## Max.cor.Y.rpart                         3                      1.004
## Max.cor.Y.glm                           1                      0.914
## Interact.High.cor.Y.glm                 1                      1.179
## Low.cor.X.glm                           1                      1.008
## All.X.glm                               1                      1.006
## All.X.bayesglm                          1                      1.797
## All.X.no.rnorm.rpart                    3                      1.090
## All.X.no.rnorm.rf                       3                      7.341
## W.only.glm                              1                      1.210
##                           min.elapsedtime.final max.auc.fit
## MFO.myMFO_classfr                         0.002   0.5000000
## Random.myrandom_classfr                   0.002   0.4623826
## Max.cor.Y.cv.0.rpart                      0.019   0.5000000
## Max.cor.Y.cv.0.cp.0.rpart                 0.016   0.9727201
## Max.cor.Y.rpart                           0.019   0.9129695
## Max.cor.Y.glm                             0.022   0.9778638
## Interact.High.cor.Y.glm                   0.033   0.9818075
## Low.cor.X.glm                             0.053   0.9827054
## All.X.glm                                 0.073   0.9839613
## All.X.bayesglm                            0.117   0.9836092
## All.X.no.rnorm.rpart                      0.060   0.9129695
## All.X.no.rnorm.rf                         1.381   1.0000000
## W.only.glm                                0.020   0.9782570
##                           opt.prob.threshold.fit max.f.score.fit
## MFO.myMFO_classfr                            0.5       0.0000000
## Random.myrandom_classfr                      0.4       0.7300380
## Max.cor.Y.cv.0.rpart                         0.5       0.7300380
## Max.cor.Y.cv.0.cp.0.rpart                    0.4       0.9416581
## Max.cor.Y.rpart                              0.9       0.9321175
## Max.cor.Y.glm                                0.3       0.9321357
## Interact.High.cor.Y.glm                      0.5       0.9396111
## Low.cor.X.glm                                0.6       0.9412998
## All.X.glm                                    0.6       0.9390756
## All.X.bayesglm                               0.5       0.9399586
## All.X.no.rnorm.rpart                         0.9       0.9321175
## All.X.no.rnorm.rf                            0.5       1.0000000
## W.only.glm                                   0.4       0.9333333
##                           max.Accuracy.fit max.AccuracyLower.fit
## MFO.myMFO_classfr                0.4251497             0.3913345
## Random.myrandom_classfr          0.5748503             0.5405049
## Max.cor.Y.cv.0.rpart             0.5748503             0.5405049
## Max.cor.Y.cv.0.cp.0.rpart        0.9317365             0.9124581
## Max.cor.Y.rpart                  0.9053875             0.8992163
## Max.cor.Y.glm                    0.9173779             0.8978991
## Interact.High.cor.Y.glm          0.9221741             0.9097989
## Low.cor.X.glm                    0.9233602             0.9137899
## All.X.glm                        0.9125904             0.9111278
## All.X.bayesglm                   0.9173779             0.9111278
## All.X.no.rnorm.rpart             0.8981976             0.8992163
## All.X.no.rnorm.rf                0.9197717             0.9955919
## W.only.glm                       0.9149798             0.9005347
##                           max.AccuracyUpper.fit max.Kappa.fit max.auc.OOB
## MFO.myMFO_classfr                     0.4594951     0.0000000   0.5000000
## Random.myrandom_classfr               0.6086655     0.0000000   0.4285714
## Max.cor.Y.cv.0.rpart                  0.6086655     0.0000000   0.5000000
## Max.cor.Y.cv.0.cp.0.rpart             0.9478914     0.8594670   0.9566327
## Max.cor.Y.rpart                       0.9372770     0.8048875   0.8928571
## Max.cor.Y.glm                         0.9362084     0.8299249   0.9897959
## Interact.High.cor.Y.glm               0.9457796     0.8400543   0.9846939
## Low.cor.X.glm                         0.9489450     0.8426415   0.9846939
## All.X.glm                             0.9468362     0.8210586   0.9795918
## All.X.bayesglm                        0.9468362     0.8306003   0.9795918
## All.X.no.rnorm.rpart                  0.9372770     0.7908403   0.8928571
## All.X.no.rnorm.rf                     1.0000000     0.8343263   0.9642857
## W.only.glm                            0.9383443     0.8248854   0.9897959
##                           opt.prob.threshold.OOB max.f.score.OOB
## MFO.myMFO_classfr                            0.5       0.0000000
## Random.myrandom_classfr                      0.4       0.6666667
## Max.cor.Y.cv.0.rpart                         0.5       0.6666667
## Max.cor.Y.cv.0.cp.0.rpart                    0.9       0.9285714
## Max.cor.Y.rpart                              0.9       0.8965517
## Max.cor.Y.glm                                0.3       0.9333333
## Interact.High.cor.Y.glm                      0.1       0.9333333
## Low.cor.X.glm                                0.2       0.9333333
## All.X.glm                                    0.1       0.9333333
## All.X.bayesglm                               0.2       0.9333333
## All.X.no.rnorm.rpart                         0.9       0.8965517
## All.X.no.rnorm.rf                            0.3       0.9333333
## W.only.glm                                   0.9       0.9629630
##                           max.Accuracy.OOB max.AccuracyLower.OOB
## MFO.myMFO_classfr                0.5000000             0.3064710
## Random.myrandom_classfr          0.5000000             0.3064710
## Max.cor.Y.cv.0.rpart             0.5000000             0.3064710
## Max.cor.Y.cv.0.cp.0.rpart        0.9285714             0.7649652
## Max.cor.Y.rpart                  0.8928571             0.7177356
## Max.cor.Y.glm                    0.9285714             0.7649652
## Interact.High.cor.Y.glm          0.9285714             0.7649652
## Low.cor.X.glm                    0.9285714             0.7649652
## All.X.glm                        0.9285714             0.7649652
## All.X.bayesglm                   0.9285714             0.7649652
## All.X.no.rnorm.rpart             0.8928571             0.7177356
## All.X.no.rnorm.rf                0.9285714             0.7649652
## W.only.glm                       0.9642857             0.8165224
##                           max.AccuracyUpper.OOB max.Kappa.OOB
## MFO.myMFO_classfr                     0.6935290     0.0000000
## Random.myrandom_classfr               0.6935290     0.0000000
## Max.cor.Y.cv.0.rpart                  0.6935290     0.0000000
## Max.cor.Y.cv.0.cp.0.rpart             0.9912295     0.8571429
## Max.cor.Y.rpart                       0.9773349     0.7857143
## Max.cor.Y.glm                         0.9912295     0.8571429
## Interact.High.cor.Y.glm               0.9912295     0.8571429
## Low.cor.X.glm                         0.9912295     0.8571429
## All.X.glm                             0.9912295     0.8571429
## All.X.bayesglm                        0.9912295     0.8571429
## All.X.no.rnorm.rpart                  0.9773349     0.7857143
## All.X.no.rnorm.rf                     0.9912295     0.8571429
## W.only.glm                            0.9990962     0.9285714
##                           max.AccuracySD.fit max.KappaSD.fit min.aic.fit
## MFO.myMFO_classfr                         NA              NA          NA
## Random.myrandom_classfr                   NA              NA          NA
## Max.cor.Y.cv.0.rpart                      NA              NA          NA
## Max.cor.Y.cv.0.cp.0.rpart                 NA              NA          NA
## Max.cor.Y.rpart                   0.03060035      0.06433844          NA
## Max.cor.Y.glm                     0.02848851      0.05891038    314.6768
## Interact.High.cor.Y.glm           0.02095379      0.04392501    297.8003
## Low.cor.X.glm                     0.02391429      0.04992758    301.2685
## All.X.glm                         0.01686000      0.03578204    303.4227
## All.X.bayesglm                    0.01859518      0.03915204    311.0097
## All.X.no.rnorm.rpart              0.00840949      0.01827986          NA
## All.X.no.rnorm.rf                 0.01973410      0.04207761          NA
## W.only.glm                        0.03489742      0.07191793    314.2956
```

```r
rm(ret_lst)
fit.models_1_chunk_df <- myadd_chunk(fit.models_1_chunk_df, "fit.models_1_end", 
                                     major.inc=TRUE)
```

```
##              label step_major step_minor    bgn    end elapsed
## 5  fit.models_1_rf          5          0 66.102 80.874  14.772
## 6 fit.models_1_end          6          0 80.875     NA      NA
```

```r
glb_chunks_df <- myadd_chunk(glb_chunks_df, "fit.models", major.inc=FALSE)
```

```
##         label step_major step_minor    bgn    end elapsed
## 11 fit.models          7          1 49.400 80.887  31.487
## 12 fit.models          7          2 80.887     NA      NA
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
##                                            model_id     model_method
## MFO.myMFO_classfr                 MFO.myMFO_classfr    myMFO_classfr
## Random.myrandom_classfr     Random.myrandom_classfr myrandom_classfr
## Max.cor.Y.cv.0.rpart           Max.cor.Y.cv.0.rpart            rpart
## Max.cor.Y.cv.0.cp.0.rpart Max.cor.Y.cv.0.cp.0.rpart            rpart
## Max.cor.Y.rpart                     Max.cor.Y.rpart            rpart
## Max.cor.Y.glm                         Max.cor.Y.glm              glm
## Interact.High.cor.Y.glm     Interact.High.cor.Y.glm              glm
## Low.cor.X.glm                         Low.cor.X.glm              glm
## All.X.glm                                 All.X.glm              glm
## All.X.bayesglm                       All.X.bayesglm         bayesglm
## All.X.no.rnorm.rpart           All.X.no.rnorm.rpart            rpart
## All.X.no.rnorm.rf                 All.X.no.rnorm.rf               rf
## W.only.glm                               W.only.glm              glm
##                                                                                                                             feats
## MFO.myMFO_classfr                                                                                                          .rnorm
## Random.myrandom_classfr                                                                                                    .rnorm
## Max.cor.Y.cv.0.rpart                                                                                                       W, DRB
## Max.cor.Y.cv.0.cp.0.rpart                                                                                                  W, DRB
## Max.cor.Y.rpart                                                                                                            W, DRB
## Max.cor.Y.glm                                                                                                              W, DRB
## Interact.High.cor.Y.glm                                                                   W, DRB, W:AST, W:PTS, W:FT, W:FG, W:TOV
## Low.cor.X.glm                                                   W, DRB, AST, FT, BLK, STL, .rnorm, X3P, X3PA, X2PA, FGA, ORB, TOV
## All.X.glm                 W, DRB, AST, PTS, FT, FG, BLK, FTA, STL, X2P, .rnorm, X3P, X3PA, X2PA, FGA, ORB, SeasonEnd, TOV, oppPTS
## All.X.bayesglm            W, DRB, AST, PTS, FT, FG, BLK, FTA, STL, X2P, .rnorm, X3P, X3PA, X2PA, FGA, ORB, SeasonEnd, TOV, oppPTS
## All.X.no.rnorm.rpart              W, DRB, AST, PTS, FT, FG, BLK, FTA, STL, X2P, X3P, X3PA, X2PA, FGA, ORB, SeasonEnd, TOV, oppPTS
## All.X.no.rnorm.rf                 W, DRB, AST, PTS, FT, FG, BLK, FTA, STL, X2P, X3P, X3PA, X2PA, FGA, ORB, SeasonEnd, TOV, oppPTS
## W.only.glm                                                                                                              W, .rnorm
##                           max.nTuningRuns max.auc.fit
## MFO.myMFO_classfr                       0   0.5000000
## Random.myrandom_classfr                 0   0.4623826
## Max.cor.Y.cv.0.rpart                    0   0.5000000
## Max.cor.Y.cv.0.cp.0.rpart               0   0.9727201
## Max.cor.Y.rpart                         3   0.9129695
## Max.cor.Y.glm                           1   0.9778638
## Interact.High.cor.Y.glm                 1   0.9818075
## Low.cor.X.glm                           1   0.9827054
## All.X.glm                               1   0.9839613
## All.X.bayesglm                          1   0.9836092
## All.X.no.rnorm.rpart                    3   0.9129695
## All.X.no.rnorm.rf                       3   1.0000000
## W.only.glm                              1   0.9782570
##                           opt.prob.threshold.fit max.f.score.fit
## MFO.myMFO_classfr                            0.5       0.0000000
## Random.myrandom_classfr                      0.4       0.7300380
## Max.cor.Y.cv.0.rpart                         0.5       0.7300380
## Max.cor.Y.cv.0.cp.0.rpart                    0.4       0.9416581
## Max.cor.Y.rpart                              0.9       0.9321175
## Max.cor.Y.glm                                0.3       0.9321357
## Interact.High.cor.Y.glm                      0.5       0.9396111
## Low.cor.X.glm                                0.6       0.9412998
## All.X.glm                                    0.6       0.9390756
## All.X.bayesglm                               0.5       0.9399586
## All.X.no.rnorm.rpart                         0.9       0.9321175
## All.X.no.rnorm.rf                            0.5       1.0000000
## W.only.glm                                   0.4       0.9333333
##                           max.Accuracy.fit max.Kappa.fit max.auc.OOB
## MFO.myMFO_classfr                0.4251497     0.0000000   0.5000000
## Random.myrandom_classfr          0.5748503     0.0000000   0.4285714
## Max.cor.Y.cv.0.rpart             0.5748503     0.0000000   0.5000000
## Max.cor.Y.cv.0.cp.0.rpart        0.9317365     0.8594670   0.9566327
## Max.cor.Y.rpart                  0.9053875     0.8048875   0.8928571
## Max.cor.Y.glm                    0.9173779     0.8299249   0.9897959
## Interact.High.cor.Y.glm          0.9221741     0.8400543   0.9846939
## Low.cor.X.glm                    0.9233602     0.8426415   0.9846939
## All.X.glm                        0.9125904     0.8210586   0.9795918
## All.X.bayesglm                   0.9173779     0.8306003   0.9795918
## All.X.no.rnorm.rpart             0.8981976     0.7908403   0.8928571
## All.X.no.rnorm.rf                0.9197717     0.8343263   0.9642857
## W.only.glm                       0.9149798     0.8248854   0.9897959
##                           opt.prob.threshold.OOB max.f.score.OOB
## MFO.myMFO_classfr                            0.5       0.0000000
## Random.myrandom_classfr                      0.4       0.6666667
## Max.cor.Y.cv.0.rpart                         0.5       0.6666667
## Max.cor.Y.cv.0.cp.0.rpart                    0.9       0.9285714
## Max.cor.Y.rpart                              0.9       0.8965517
## Max.cor.Y.glm                                0.3       0.9333333
## Interact.High.cor.Y.glm                      0.1       0.9333333
## Low.cor.X.glm                                0.2       0.9333333
## All.X.glm                                    0.1       0.9333333
## All.X.bayesglm                               0.2       0.9333333
## All.X.no.rnorm.rpart                         0.9       0.8965517
## All.X.no.rnorm.rf                            0.3       0.9333333
## W.only.glm                                   0.9       0.9629630
##                           max.Accuracy.OOB max.Kappa.OOB
## MFO.myMFO_classfr                0.5000000     0.0000000
## Random.myrandom_classfr          0.5000000     0.0000000
## Max.cor.Y.cv.0.rpart             0.5000000     0.0000000
## Max.cor.Y.cv.0.cp.0.rpart        0.9285714     0.8571429
## Max.cor.Y.rpart                  0.8928571     0.7857143
## Max.cor.Y.glm                    0.9285714     0.8571429
## Interact.High.cor.Y.glm          0.9285714     0.8571429
## Low.cor.X.glm                    0.9285714     0.8571429
## All.X.glm                        0.9285714     0.8571429
## All.X.bayesglm                   0.9285714     0.8571429
## All.X.no.rnorm.rpart             0.8928571     0.7857143
## All.X.no.rnorm.rf                0.9285714     0.8571429
## W.only.glm                       0.9642857     0.9285714
##                           inv.elapsedtime.everything inv.elapsedtime.final
## MFO.myMFO_classfr                          3.2786885            500.000000
## Random.myrandom_classfr                    4.3668122            500.000000
## Max.cor.Y.cv.0.rpart                       1.5748031             52.631579
## Max.cor.Y.cv.0.cp.0.rpart                  2.1786492             62.500000
## Max.cor.Y.rpart                            0.9960159             52.631579
## Max.cor.Y.glm                              1.0940919             45.454545
## Interact.High.cor.Y.glm                    0.8481764             30.303030
## Low.cor.X.glm                              0.9920635             18.867925
## All.X.glm                                  0.9940358             13.698630
## All.X.bayesglm                             0.5564830              8.547009
## All.X.no.rnorm.rpart                       0.9174312             16.666667
## All.X.no.rnorm.rf                          0.1362212              0.724113
## W.only.glm                                 0.8264463             50.000000
##                           inv.aic.fit
## MFO.myMFO_classfr                  NA
## Random.myrandom_classfr            NA
## Max.cor.Y.cv.0.rpart               NA
## Max.cor.Y.cv.0.cp.0.rpart          NA
## Max.cor.Y.rpart                    NA
## Max.cor.Y.glm             0.003177864
## Interact.High.cor.Y.glm   0.003357955
## Low.cor.X.glm             0.003319298
## All.X.glm                 0.003295733
## All.X.bayesglm            0.003215333
## All.X.no.rnorm.rpart               NA
## All.X.no.rnorm.rf                  NA
## W.only.glm                0.003181718
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
## 13. Consider specifying shapes manually. if you must have them.
```

```
## Warning in loop_apply(n, do.ply): Removed 4 rows containing missing values
## (geom_path).
```

```
## Warning in loop_apply(n, do.ply): Removed 102 rows containing missing
## values (geom_point).
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
## 13. Consider specifying shapes manually. if you must have them.
```

![](NBA_Playoffs2_files/figure-html/fit.models_2-1.png) 

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
## Warning: max.AccuracyUpper.fit already exists in glb_models_df
```

```
## [1] "var:max.KappaSD.fit"
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
dev.off()
```

```
## quartz_off_screen 
##                 2
```

```r
print(gp)
```

![](NBA_Playoffs2_files/figure-html/fit.models_2-2.png) 

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
##                     model_id max.Accuracy.OOB max.auc.OOB max.Kappa.OOB
## 13                W.only.glm        0.9642857   0.9897959     0.9285714
## 6              Max.cor.Y.glm        0.9285714   0.9897959     0.8571429
## 7    Interact.High.cor.Y.glm        0.9285714   0.9846939     0.8571429
## 8              Low.cor.X.glm        0.9285714   0.9846939     0.8571429
## 9                  All.X.glm        0.9285714   0.9795918     0.8571429
## 10            All.X.bayesglm        0.9285714   0.9795918     0.8571429
## 12         All.X.no.rnorm.rf        0.9285714   0.9642857     0.8571429
## 4  Max.cor.Y.cv.0.cp.0.rpart        0.9285714   0.9566327     0.8571429
## 5            Max.cor.Y.rpart        0.8928571   0.8928571     0.7857143
## 11      All.X.no.rnorm.rpart        0.8928571   0.8928571     0.7857143
## 1          MFO.myMFO_classfr        0.5000000   0.5000000     0.0000000
## 3       Max.cor.Y.cv.0.rpart        0.5000000   0.5000000     0.0000000
## 2    Random.myrandom_classfr        0.5000000   0.4285714     0.0000000
##    min.aic.fit opt.prob.threshold.OOB
## 13    314.2956                    0.9
## 6     314.6768                    0.3
## 7     297.8003                    0.1
## 8     301.2685                    0.2
## 9     303.4227                    0.1
## 10    311.0097                    0.2
## 12          NA                    0.3
## 4           NA                    0.9
## 5           NA                    0.9
## 11          NA                    0.9
## 1           NA                    0.5
## 3           NA                    0.5
## 2           NA                    0.4
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
## 13. Consider specifying shapes manually. if you must have them.
```

```
## Warning in loop_apply(n, do.ply): Removed 44 rows containing missing values
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
## 13. Consider specifying shapes manually. if you must have them.
```

![](NBA_Playoffs2_files/figure-html/fit.models_2-3.png) 

```r
print("Metrics used for model selection:"); print(model_sel_frmla)
```

```
## [1] "Metrics used for model selection:"
```

```
## ~-max.Accuracy.OOB - max.auc.OOB - max.Kappa.OOB + min.aic.fit - 
##     opt.prob.threshold.OOB
```

```r
print(sprintf("Best model id: %s", dsp_models_df[1, "model_id"]))
```

```
## [1] "Best model id: W.only.glm"
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

![](NBA_Playoffs2_files/figure-html/fit.models_2-4.png) ![](NBA_Playoffs2_files/figure-html/fit.models_2-5.png) ![](NBA_Playoffs2_files/figure-html/fit.models_2-6.png) ![](NBA_Playoffs2_files/figure-html/fit.models_2-7.png) 

```
## 
## Call:
## NULL
## 
## Deviance Residuals: 
##      Min        1Q    Median        3Q       Max  
## -2.85595  -0.08894   0.01313   0.19013   2.93072  
## 
## Coefficients:
##             Estimate Std. Error z value Pr(>|z|)    
## (Intercept) -18.5144     1.6261 -11.385   <2e-16 ***
## W             0.4727     0.0407  11.614   <2e-16 ***
## .rnorm        0.1124     0.1451   0.775    0.439    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for binomial family taken to be 1)
## 
##     Null deviance: 1138.8  on 834  degrees of freedom
## Residual deviance:  308.3  on 832  degrees of freedom
## AIC: 314.3
## 
## Number of Fisher Scoring iterations: 8
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
##        importance W.only.glm.importance
## W             100                   100
## .rnorm          0                     0
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

![](NBA_Playoffs2_files/figure-html/fit.models_2-8.png) ![](NBA_Playoffs2_files/figure-html/fit.models_2-9.png) 

```
## [1] "Min/Max Boundaries: "
##     .rownames Playoffs.fctr Playoffs.fctr.predict.W.only.glm.prob
## 850        15             Y                          0.4115584899
## 849        14             Y                          0.9999969053
## 855        20             N                          0.0001227762
## 861        26             N                          0.0652014350
##     Playoffs.fctr.predict.W.only.glm
## 850                                N
## 849                                Y
## 855                                N
## 861                                N
##     Playoffs.fctr.predict.W.only.glm.accurate
## 850                                     FALSE
## 849                                      TRUE
## 855                                      TRUE
## 861                                      TRUE
##     Playoffs.fctr.predict.W.only.glm.error .label
## 850                             -0.4884415     15
## 849                              0.0000000     14
## 855                              0.0000000     20
## 861                              0.0000000     26
## [1] "Inaccurate: "
##     .rownames Playoffs.fctr Playoffs.fctr.predict.W.only.glm.prob
## 850        15             Y                             0.4115585
##     Playoffs.fctr.predict.W.only.glm
## 850                                N
##     Playoffs.fctr.predict.W.only.glm.accurate
## 850                                     FALSE
##     Playoffs.fctr.predict.W.only.glm.error
## 850                             -0.4884415
```

![](NBA_Playoffs2_files/figure-html/fit.models_2-10.png) 

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
## 12 fit.models          7          2 80.887 94.811  13.924
## 13 fit.models          7          3 94.811     NA      NA
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
## [1] "Playoffs.fctr.predict.W.only.glm.prob"    
## [2] "Playoffs.fctr.predict.W.only.glm"         
## [3] "Playoffs.fctr.predict.W.only.glm.accurate"
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

![](NBA_Playoffs2_files/figure-html/fit.models_3-1.png) 

```r
glb_chunks_df <- myadd_chunk(glb_chunks_df, "fit.data.training", major.inc=TRUE)
```

```
##                label step_major step_minor    bgn    end elapsed
## 13        fit.models          7          3 94.811 98.352   3.541
## 14 fit.data.training          8          0 98.352     NA      NA
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
## [1] "fitting model: Final.glm"
## [1] "    indep_vars: W, .rnorm"
## Aggregating results
## Fitting final model on full training set
```

![](NBA_Playoffs2_files/figure-html/fit.data.training_0-1.png) ![](NBA_Playoffs2_files/figure-html/fit.data.training_0-2.png) ![](NBA_Playoffs2_files/figure-html/fit.data.training_0-3.png) ![](NBA_Playoffs2_files/figure-html/fit.data.training_0-4.png) 

```
## 
## Call:
## NULL
## 
## Deviance Residuals: 
##      Min        1Q    Median        3Q       Max  
## -2.85595  -0.08894   0.01313   0.19013   2.93072  
## 
## Coefficients:
##             Estimate Std. Error z value Pr(>|z|)    
## (Intercept) -18.5144     1.6261 -11.385   <2e-16 ***
## W             0.4727     0.0407  11.614   <2e-16 ***
## .rnorm        0.1124     0.1451   0.775    0.439    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for binomial family taken to be 1)
## 
##     Null deviance: 1138.8  on 834  degrees of freedom
## Residual deviance:  308.3  on 832  degrees of freedom
## AIC: 314.3
## 
## Number of Fisher Scoring iterations: 8
## 
## [1] "    calling mypredict_mdl for fit:"
```

![](NBA_Playoffs2_files/figure-html/fit.data.training_0-5.png) 

```
##    threshold   f.score
## 1        0.0 0.7300380
## 2        0.1 0.9035917
## 3        0.2 0.9199219
## 4        0.3 0.9321357
## 5        0.4 0.9333333
## 6        0.5 0.9320988
## 7        0.6 0.9275971
## 8        0.7 0.9161290
## 9        0.8 0.8951522
## 10       0.9 0.8524203
## 11       1.0 0.0000000
```

```
## [1] "Classifier Probability Threshold: 0.4000 to maximize f.score.fit"
##   Playoffs.fctr Playoffs.fctr.predict.Final.glm.N
## 1             N                               307
## 2             Y                                18
##   Playoffs.fctr.predict.Final.glm.Y
## 1                                48
## 2                               462
##          Prediction
## Reference   N   Y
##         N 307  48
##         Y  18 462
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.209581e-01   8.364931e-01   9.005347e-01   9.383443e-01   5.748503e-01 
## AccuracyPValue  McnemarPValue 
##  3.838559e-111   3.574541e-04
```

```
## Warning in mypredict_mdl(mdl, df = fit_df, rsp_var, rsp_var_out,
## model_id_method, : Expecting 1 metric: Accuracy; recd: Accuracy, Kappa;
## retaining Accuracy only
```

![](NBA_Playoffs2_files/figure-html/fit.data.training_0-6.png) 

```
##    model_id model_method     feats max.nTuningRuns
## 1 Final.glm          glm W, .rnorm               1
##   min.elapsedtime.everything min.elapsedtime.final max.auc.fit
## 1                      1.014                 0.021    0.978257
##   opt.prob.threshold.fit max.f.score.fit max.Accuracy.fit
## 1                    0.4       0.9333333        0.9149798
##   max.AccuracyLower.fit max.AccuracyUpper.fit max.Kappa.fit min.aic.fit
## 1             0.9005347             0.9383443     0.8248854    314.2956
##   max.AccuracySD.fit max.KappaSD.fit
## 1         0.03489742      0.07191793
```

```r
rm(ret_lst)
glb_chunks_df <- myadd_chunk(glb_chunks_df, "fit.data.training", major.inc=FALSE)
```

```
##                label step_major step_minor     bgn     end elapsed
## 14 fit.data.training          8          0  98.352 104.118   5.767
## 15 fit.data.training          8          1 104.119      NA      NA
```


```r
glb_trnobs_df <- glb_get_predictions(df=glb_trnobs_df, mdl_id=glb_fin_mdl_id, 
                                     rsp_var_out=glb_rsp_var_out,
    prob_threshold_def=ifelse(glb_is_classification && glb_is_binomial, 
        glb_models_df[glb_models_df$model_id == glb_sel_mdl_id, "opt.prob.threshold.OOB"], NULL))
```

```
## Warning in glb_get_predictions(df = glb_trnobs_df, mdl_id =
## glb_fin_mdl_id, : Using default probability threshold: 0.9
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
##        W.only.glm.importance importance Final.glm.importance
## W                        100        100                  100
## .rnorm                     0          0                    0
```

```r
if (glb_is_classification && glb_is_binomial)
    glb_analytics_diag_plots(obs_df=glb_trnobs_df, mdl_id=glb_fin_mdl_id, 
            prob_threshold=glb_models_df[glb_models_df$model_id == glb_sel_mdl_id, 
                                         "opt.prob.threshold.OOB"]) else
    glb_analytics_diag_plots(obs_df=glb_trnobs_df, mdl_id=glb_fin_mdl_id)                  
```

![](NBA_Playoffs2_files/figure-html/fit.data.training_1-1.png) ![](NBA_Playoffs2_files/figure-html/fit.data.training_1-2.png) 

```
## [1] "Min/Max Boundaries: "
##     .rownames Playoffs.fctr Playoffs.fctr.predict.Final.glm.prob
## 201       201             Y                         9.989186e-01
## 318       318             N                         1.788424e-06
## 397       397             Y                         9.999998e-01
## 564       564             Y                         9.158155e-01
##     Playoffs.fctr.predict.Final.glm
## 201                               Y
## 318                               N
## 397                               Y
## 564                               Y
##     Playoffs.fctr.predict.Final.glm.accurate
## 201                                     TRUE
## 318                                     TRUE
## 397                                     TRUE
## 564                                     TRUE
##     Playoffs.fctr.predict.Final.glm.error .label
## 201                                     0    201
## 318                                     0    318
## 397                                     0    397
## 564                                     0    564
## [1] "Inaccurate: "
##     .rownames Playoffs.fctr Playoffs.fctr.predict.Final.glm.prob
## 140       140             Y                           0.01364261
## 203       203             Y                           0.01672342
## 368       368             Y                           0.11007627
## 114       114             Y                           0.11388258
## 157       157             Y                           0.12520065
## 118       118             Y                           0.16587698
##     Playoffs.fctr.predict.Final.glm
## 140                               N
## 203                               N
## 368                               N
## 114                               N
## 157                               N
## 118                               N
##     Playoffs.fctr.predict.Final.glm.accurate
## 140                                    FALSE
## 203                                    FALSE
## 368                                    FALSE
## 114                                    FALSE
## 157                                    FALSE
## 118                                    FALSE
##     Playoffs.fctr.predict.Final.glm.error
## 140                            -0.8863574
## 203                            -0.8832766
## 368                            -0.7899237
## 114                            -0.7861174
## 157                            -0.7747994
## 118                            -0.7341230
##     .rownames Playoffs.fctr Playoffs.fctr.predict.Final.glm.prob
## 549       549             Y                            0.7611855
## 694       694             Y                            0.7896040
## 187       187             Y                            0.8014601
## 610       610             Y                            0.8223485
## 528       528             Y                            0.8686152
## 142       142             Y                            0.8987021
##     Playoffs.fctr.predict.Final.glm
## 549                               N
## 694                               N
## 187                               N
## 610                               N
## 528                               N
## 142                               N
##     Playoffs.fctr.predict.Final.glm.accurate
## 549                                    FALSE
## 694                                    FALSE
## 187                                    FALSE
## 610                                    FALSE
## 528                                    FALSE
## 142                                    FALSE
##     Playoffs.fctr.predict.Final.glm.error
## 549                          -0.138814545
## 694                          -0.110396012
## 187                          -0.098539925
## 610                          -0.077651519
## 528                          -0.031384848
## 142                          -0.001297934
##     .rownames Playoffs.fctr Playoffs.fctr.predict.Final.glm.prob
## 642       642             N                            0.9107223
## 519       519             N                            0.9328332
## 53         53             N                            0.9364528
## 79         79             N                            0.9368984
## 769       769             N                            0.9590288
## 724       724             N                            0.9830626
##     Playoffs.fctr.predict.Final.glm
## 642                               Y
## 519                               Y
## 53                                Y
## 79                                Y
## 769                               Y
## 724                               Y
##     Playoffs.fctr.predict.Final.glm.accurate
## 642                                    FALSE
## 519                                    FALSE
## 53                                     FALSE
## 79                                     FALSE
## 769                                    FALSE
## 724                                    FALSE
##     Playoffs.fctr.predict.Final.glm.error
## 642                            0.01072235
## 519                            0.03283320
## 53                             0.03645277
## 79                             0.03689837
## 769                            0.05902879
## 724                            0.08306264
```

![](NBA_Playoffs2_files/figure-html/fit.data.training_1-3.png) 

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
## [1] "Playoffs.fctr.predict.Final.glm.prob"
## [2] "Playoffs.fctr.predict.Final.glm"
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

![](NBA_Playoffs2_files/figure-html/fit.data.training_1-4.png) 

```r
glb_chunks_df <- myadd_chunk(glb_chunks_df, "predict.data.new", major.inc=TRUE)
```

```
##                label step_major step_minor     bgn     end elapsed
## 15 fit.data.training          8          1 104.119 107.445   3.326
## 16  predict.data.new          9          0 107.445      NA      NA
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
## Warning in glb_get_predictions(glb_newobs_df, mdl_id = glb_fin_mdl_id,
## rsp_var_out = glb_rsp_var_out, : Using default probability threshold: 0.9
```

```r
if (glb_is_classification && glb_is_binomial)
    glb_analytics_diag_plots(obs_df=glb_newobs_df, mdl_id=glb_fin_mdl_id, 
            prob_threshold=glb_models_df[glb_models_df$model_id == glb_sel_mdl_id, 
                                         "opt.prob.threshold.OOB"]) else
    glb_analytics_diag_plots(obs_df=glb_newobs_df, mdl_id=glb_fin_mdl_id)                  
```

![](NBA_Playoffs2_files/figure-html/predict.data.new-1.png) ![](NBA_Playoffs2_files/figure-html/predict.data.new-2.png) 

```
## [1] "Min/Max Boundaries: "
##     .rownames Playoffs.fctr Playoffs.fctr.predict.Final.glm.prob
## 850        15             Y                         0.4115584899
## 849        14             Y                         0.9999969053
## 855        20             N                         0.0001227762
## 861        26             N                         0.0652014350
##     Playoffs.fctr.predict.Final.glm
## 850                               N
## 849                               Y
## 855                               N
## 861                               N
##     Playoffs.fctr.predict.Final.glm.accurate
## 850                                    FALSE
## 849                                     TRUE
## 855                                     TRUE
## 861                                     TRUE
##     Playoffs.fctr.predict.Final.glm.error .label
## 850                            -0.4884415     15
## 849                             0.0000000     14
## 855                             0.0000000     20
## 861                             0.0000000     26
## [1] "Inaccurate: "
##     .rownames Playoffs.fctr Playoffs.fctr.predict.Final.glm.prob
## 850        15             Y                            0.4115585
##     Playoffs.fctr.predict.Final.glm
## 850                               N
##     Playoffs.fctr.predict.Final.glm.accurate
## 850                                    FALSE
##     Playoffs.fctr.predict.Final.glm.error
## 850                            -0.4884415
```

![](NBA_Playoffs2_files/figure-html/predict.data.new-3.png) 

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
```

```
## [1] 0.9
```

```r
print(sprintf("glb_sel_mdl_id: %s", glb_sel_mdl_id))
```

```
## [1] "glb_sel_mdl_id: W.only.glm"
```

```r
print(sprintf("glb_fin_mdl_id: %s", glb_fin_mdl_id))
```

```
## [1] "glb_fin_mdl_id: Final.glm"
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
##                     model_id max.Accuracy.OOB max.auc.OOB max.Kappa.OOB
## 13                W.only.glm        0.9642857   0.9897959     0.9285714
## 6              Max.cor.Y.glm        0.9285714   0.9897959     0.8571429
## 7    Interact.High.cor.Y.glm        0.9285714   0.9846939     0.8571429
## 8              Low.cor.X.glm        0.9285714   0.9846939     0.8571429
## 9                  All.X.glm        0.9285714   0.9795918     0.8571429
## 10            All.X.bayesglm        0.9285714   0.9795918     0.8571429
## 12         All.X.no.rnorm.rf        0.9285714   0.9642857     0.8571429
## 4  Max.cor.Y.cv.0.cp.0.rpart        0.9285714   0.9566327     0.8571429
## 5            Max.cor.Y.rpart        0.8928571   0.8928571     0.7857143
## 11      All.X.no.rnorm.rpart        0.8928571   0.8928571     0.7857143
## 1          MFO.myMFO_classfr        0.5000000   0.5000000     0.0000000
## 3       Max.cor.Y.cv.0.rpart        0.5000000   0.5000000     0.0000000
## 2    Random.myrandom_classfr        0.5000000   0.4285714     0.0000000
##    min.aic.fit opt.prob.threshold.OOB
## 13    314.2956                    0.9
## 6     314.6768                    0.3
## 7     297.8003                    0.1
## 8     301.2685                    0.2
## 9     303.4227                    0.1
## 10    311.0097                    0.2
## 12          NA                    0.3
## 4           NA                    0.9
## 5           NA                    0.9
## 11          NA                    0.9
## 1           NA                    0.5
## 3           NA                    0.5
## 2           NA                    0.4
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
```

```
## [1] "W.only.glm OOB confusion matrix & accuracy: "
##          Prediction
## Reference  N  Y
##         N 14  0
##         Y  1 13
## [1] "Final.glm new confusion matrix & accuracy: "
##          Prediction
## Reference  N  Y
##         N 14  0
##         Y  1 13
```

```r
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
##        W.only.glm.importance importance Final.glm.importance
## W                        100        100                  100
## .rnorm                     0          0                    0
```

```r
# players_df <- data.frame(id=c("Chavez", "Giambi", "Menechino", "Myers", "Pena"),
#                          OBP=c(0.338, 0.391, 0.369, 0.313, 0.361),
#                          SLG=c(0.540, 0.450, 0.374, 0.447, 0.500),
#                         cost=c(1400000, 1065000, 295000, 800000, 300000))
# players_df$RS.predict <- predict(glb_models_lst[[csm_mdl_id]], players_df)
# print(orderBy(~ -RS.predict, players_df))

print(setdiff(names(glb_trnobs_df), names(glb_allobs_df)))
```

```
## character(0)
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
## 16     predict.data.new          9          0 107.445 109.703   2.258
## 17 display.session.info         10          0 109.704      NA      NA
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
## 11              fit.models          7          1  49.400  80.887  31.487
## 10              fit.models          7          0  26.080  49.399  23.319
## 12              fit.models          7          2  80.887  94.811  13.924
## 2             inspect.data          2          0  10.803  20.805  10.002
## 14       fit.data.training          8          0  98.352 104.118   5.767
## 13              fit.models          7          3  94.811  98.352   3.541
## 15       fit.data.training          8          1 104.119 107.445   3.326
## 3               scrub.data          2          1  20.805  23.074   2.269
## 16        predict.data.new          9          0 107.445 109.703   2.258
## 6         extract.features          3          0  23.202  24.669   1.467
## 8          select.features          5          0  24.983  25.736   0.754
## 1              import.data          1          0  10.377  10.802   0.425
## 9  partition.data.training          6          0  25.737  26.080   0.343
## 7             cluster.data          4          0  24.669  24.983   0.314
## 5      manage.missing.data          2          3  23.109  23.202   0.093
## 4           transform.data          2          2  23.074  23.109   0.035
##    duration
## 11   31.487
## 10   23.319
## 12   13.924
## 2    10.002
## 14    5.766
## 13    3.541
## 15    3.326
## 3     2.269
## 16    2.258
## 6     1.467
## 8     0.753
## 1     0.425
## 9     0.343
## 7     0.314
## 5     0.093
## 4     0.035
## [1] "Total Elapsed Time: 109.703 secs"
```

![](NBA_Playoffs2_files/figure-html/display.session.info-1.png) 

```
##                   label step_major step_minor    bgn    end elapsed
## 5       fit.models_1_rf          5          0 66.102 80.874  14.772
## 4    fit.models_1_rpart          4          0 61.639 66.101   4.462
## 3 fit.models_1_bayesglm          3          0 57.250 61.638   4.388
## 2      fit.models_1_glm          2          0 53.073 57.250   4.177
## 1      fit.models_1_bgn          1          0 53.058 53.072   0.014
##   duration
## 5   14.772
## 4    4.462
## 3    4.388
## 2    4.177
## 1    0.014
## [1] "Total Elapsed Time: 80.874 secs"
```

![](NBA_Playoffs2_files/figure-html/display.session.info-2.png) 

```
## R version 3.2.0 (2015-04-16)
## Platform: x86_64-apple-darwin13.4.0 (64-bit)
## Running under: OS X 10.10.3 (Yosemite)
## 
## locale:
## [1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8
## 
## attached base packages:
##  [1] tcltk     grid      parallel  stats     graphics  grDevices utils    
##  [8] datasets  methods   base     
## 
## other attached packages:
##  [1] gdata_2.16.1        randomForest_4.6-10 arm_1.8-5          
##  [4] lme4_1.1-7          Rcpp_0.11.6         Matrix_1.2-1       
##  [7] MASS_7.3-40         rpart.plot_1.5.2    rpart_4.1-9        
## [10] ROCR_1.0-7          gplots_2.17.0       dplyr_0.4.1        
## [13] plyr_1.8.2          sqldf_0.4-10        RSQLite_1.0.0      
## [16] DBI_0.3.1           gsubfn_0.6-6        proto_0.3-10       
## [19] reshape2_1.4.1      doMC_1.3.3          iterators_1.0.7    
## [22] foreach_1.4.2       doBy_4.5-13         survival_2.38-1    
## [25] caret_6.0-47        ggplot2_1.0.1       lattice_0.20-31    
## 
## loaded via a namespace (and not attached):
##  [1] class_7.3-12        gtools_3.5.0        assertthat_0.1     
##  [4] digest_0.6.8        BradleyTerry2_1.0-6 chron_2.3-45       
##  [7] evaluate_0.7        coda_0.17-1         e1071_1.6-4        
## [10] lazyeval_0.1.10     minqa_1.2.4         SparseM_1.6        
## [13] car_2.0-25          nloptr_1.0.4        rmarkdown_0.6.1    
## [16] labeling_0.3        splines_3.2.0       stringr_1.0.0      
## [19] munsell_0.4.2       compiler_3.2.0      mgcv_1.8-6         
## [22] htmltools_0.2.6     nnet_7.3-9          codetools_0.2-11   
## [25] brglm_0.5-9         bitops_1.0-6        nlme_3.1-120       
## [28] gtable_0.1.2        magrittr_1.5        formatR_1.2        
## [31] scales_0.2.4        KernSmooth_2.23-14  stringi_0.4-1      
## [34] RColorBrewer_1.1-2  tools_3.2.0         abind_1.4-3        
## [37] pbkrtest_0.4-2      yaml_2.1.13         colorspace_1.2-6   
## [40] caTools_1.17.1      knitr_1.10.5        quantreg_5.11
```
