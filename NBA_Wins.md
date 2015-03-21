# NBA: Wins regression
bdanalytics  

**  **    
**Date: (Sat) Mar 21, 2015**    

# Introduction:  

Data: 
Source: http://www.basketball-reference.com
    Training:   https://courses.edx.org/c4x/MITx/15.071x_2/asset/NBA_train.csv  
    New:        https://courses.edx.org/c4x/MITx/15.071x_2/asset/NBA_test.csv  
Time period: 



# Synopsis:

Based on analysis utilizing <> techniques, <conclusion heading>:  

### ![](<filename>.png)

## Potential next steps include:

# Analysis: 

```r
rm(list=ls())
set.seed(12345)
options(stringsAsFactors=FALSE)
source("~/Dropbox/datascience/R/mydsutils.R")
source("~/Dropbox/datascience/R/myplot.R")
source("~/Dropbox/datascience/R/mypetrinet.R")
# Gather all package requirements here
suppressPackageStartupMessages(require(plyr))
suppressPackageStartupMessages(require(reshape2))

#require(sos); findFn("pinv", maxPages=2, sortby="MaxScore")

# Analysis control global variables
glb_is_separate_predict_dataset <- TRUE
glb_predct_var <- "W"           # or NULL
glb_predct_var_name <- paste0(glb_predct_var, ".predict")
glb_id_vars <- c("SeasonEnd", "Team")                # or NULL

glb_exclude_vars_as_features <- NULL                      
# List chrs converted into factors 
glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, 
                                      c("Team", "Team.fctr")     # or NULL
                                      )
# List feats that shd be excluded due to known causation by prediction variable
glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, 
                                      c("Playoffs")     # or NULL
                                      )

glb_is_regression <- TRUE; glb_is_classification <- FALSE

glb_mdl <- glb_sel_mdl <- NULL
glb_models_df <- data.frame()

script_df <- data.frame(chunk_label="import_data", chunk_step_major=1, chunk_step_minor=0)
print(script_df)
```

```
##   chunk_label chunk_step_major chunk_step_minor
## 1 import_data                1                0
```

## Step `1`: import data

```r
glb_entity_df <- myimport_data(
    url="https://courses.edx.org/c4x/MITx/15.071x_2/asset/NBA_train.csv", 
    comment="glb_entity_df", print_diagn=TRUE)
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
##     SeasonEnd                  Team Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 139      1986        Boston Celtics        1 67 9359   8587 3718 7312 3580
## 380      1995            Miami Heat        0 32 8293   8427 3144 6738 2708
## 602      2004        Denver Nuggets        1 43 7972   7884 2993 6763 2662
## 634      2005 Golden State Warriors        0 34 8094   8271 3029 7039 2405
## 731      2008       Milwaukee Bucks        0 26 7956   8521 3026 6745 2574
## 738      2008          Phoenix Suns        1 55 9026   8612 3392 6782 2698
##     X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV
## 139 6919 138  393 1785 2248 1054 2753 2387 641 511 1360
## 380 5556 436 1182 1569 2133 1092 2272 1779 662 298 1291
## 602 5778 331  985 1655 2159 1083 2387 1794 745 520 1278
## 634 5265 624 1774 1412 1955 1069 2436 1811 642 420 1112
## 731 5432 452 1313 1452 1980 1055 2363 1760 544 361 1210
## 738 5018 694 1764 1548 1977  720 2683 2188 532 518 1184
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
##  - attr(*, "comment")= chr "glb_entity_df"
## NULL
```

```r
if (glb_is_separate_predict_dataset) {
    glb_predct_df <- myimport_data(
        url="https://courses.edx.org/c4x/MITx/15.071x_2/asset/NBA_test.csv", 
        comment="glb_predct_df", print_diagn=TRUE)
} else {
    glb_predct_df <- glb_entity_df[sample(1:nrow(glb_entity_df), nrow(glb_entity_df) / 1000),]
    comment(glb_predct_df) <- "glb_predct_df"
    myprint_df(glb_predct_df)
    str(glb_predct_df)
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
##    SeasonEnd                  Team Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 1       2013         Atlanta Hawks        1 44 8032   7999 3084 6644 2378
## 4       2013         Chicago Bulls        1 45 7641   7615 2926 6698 2480
## 10      2013       Houston Rockets        1 45 8688   8403 3124 6782 2257
## 14      2013            Miami Heat        1 66 8436   7791 3148 6348 2431
## 19      2013 Oklahoma City Thunder        1 60 8669   7914 3126 6504 2528
## 25      2013     San Antonio Spurs        1 58 8448   7923 3210 6675 2547
##    X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV
## 1  4743 706 1901 1158 1619  758 2593 2007 664 369 1219
## 4  5433 446 1265 1343 1738 1026 2514 1886 588 417 1171
## 10 4413 867 2369 1573 2087  909 2652 1902 679 359 1348
## 14 4539 717 1809 1423 1887  676 2490 1890 710 441 1143
## 19 4916 598 1588 1819 2196  854 2725 1753 679 624 1253
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
##  - attr(*, "comment")= chr "glb_predct_df"
## NULL
```

```r
script_df <- rbind(script_df,
                   data.frame(chunk_label="cleanse_data", 
                              chunk_step_major=max(script_df$chunk_step_major)+1, 
                              chunk_step_minor=0))
print(script_df)
```

```
##    chunk_label chunk_step_major chunk_step_minor
## 1  import_data                1                0
## 2 cleanse_data                2                0
```

## Step `2`: cleanse data

```r
script_df <- rbind(script_df, 
                   data.frame(chunk_label="inspect_explore_data", 
                              chunk_step_major=max(script_df$chunk_step_major), 
                              chunk_step_minor=1))
print(script_df)
```

```
##            chunk_label chunk_step_major chunk_step_minor
## 1          import_data                1                0
## 2         cleanse_data                2                0
## 3 inspect_explore_data                2                1
```

### Step `2`.`1`: inspect/explore data

```r
#print(str(glb_entity_df))
#View(glb_entity_df)

# List info gathered for various columns
#   SeasonEnd:  is the year the season ended.
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

# Create new features that help diagnostics
#   Convert factors to dummy variables
#   Build splines   require(splines); bsBasis <- bs(training$age, df=3)

add_new_diag_feats <- function(obs_df, obs_twin_df) {
    obs_df <- mutate(obs_df,
#         <col_name>.NA=is.na(<col_name>)
        
        Team.fctr=factor(Team, 
                    as.factor(union(obs_df$Team, obs_twin_df$Team))) 

          # This doesn't work - use sapply instead
#         <col_name>.fctr_num=grep(<col_name>, levels(<col_name>.fctr)), 
#         
#         Date.my=as.Date(strptime(Date, "%m/%d/%y %H:%M")),
#         Year=year(Date.my),
#         Month=months(Date.my),
#         Weekday=weekdays(Date.my)
        
                        )

    # If levels of a factor are different across obs_df & glb_predct_df; predict.glm fails  
    # Transformations not handled by mutate
    obs_df$Team.fctr.num <- sapply(1:nrow(obs_df), 
        function(row_ix) grep(obs_df[row_ix, "Team"],
                              levels(obs_df[row_ix, "Team.fctr"])))
    
    print(summary(obs_df))
    print(sapply(names(obs_df), function(col) sum(is.na(obs_df[, col]))))
    return(obs_df)
}

glb_entity_df <- add_new_diag_feats(glb_entity_df, glb_predct_df)
```

```
##    SeasonEnd        Team              Playoffs            W       
##  Min.   :1980   Length:835         Min.   :0.0000   Min.   :11.0  
##  1st Qu.:1989   Class :character   1st Qu.:0.0000   1st Qu.:31.0  
##  Median :1996   Mode  :character   Median :1.0000   Median :42.0  
##  Mean   :1996                      Mean   :0.5749   Mean   :41.0  
##  3rd Qu.:2005                      3rd Qu.:1.0000   3rd Qu.:50.5  
##  Max.   :2011                      Max.   :1.0000   Max.   :72.0  
##                                                                   
##       PTS            oppPTS            FG            FGA      
##  Min.   : 6901   Min.   : 6909   Min.   :2565   Min.   :5972  
##  1st Qu.: 7934   1st Qu.: 7934   1st Qu.:2974   1st Qu.:6564  
##  Median : 8312   Median : 8365   Median :3150   Median :6831  
##  Mean   : 8370   Mean   : 8370   Mean   :3200   Mean   :6873  
##  3rd Qu.: 8784   3rd Qu.: 8768   3rd Qu.:3434   3rd Qu.:7157  
##  Max.   :10371   Max.   :10723   Max.   :3980   Max.   :8868  
##                                                               
##       X2P            X2PA           X3P             X3PA       
##  Min.   :1981   Min.   :4153   Min.   : 10.0   Min.   :  75.0  
##  1st Qu.:2510   1st Qu.:5269   1st Qu.:131.5   1st Qu.: 413.0  
##  Median :2718   Median :5706   Median :329.0   Median : 942.0  
##  Mean   :2881   Mean   :5956   Mean   :319.0   Mean   : 916.9  
##  3rd Qu.:3296   3rd Qu.:6754   3rd Qu.:481.5   3rd Qu.:1347.5  
##  Max.   :3954   Max.   :7873   Max.   :841.0   Max.   :2284.0  
##                                                                
##        FT            FTA            ORB              DRB      
##  Min.   :1189   Min.   :1475   Min.   : 639.0   Min.   :2044  
##  1st Qu.:1502   1st Qu.:2008   1st Qu.: 953.5   1st Qu.:2346  
##  Median :1628   Median :2176   Median :1055.0   Median :2433  
##  Mean   :1650   Mean   :2190   Mean   :1061.6   Mean   :2427  
##  3rd Qu.:1781   3rd Qu.:2352   3rd Qu.:1167.0   3rd Qu.:2516  
##  Max.   :2388   Max.   :3051   Max.   :1520.0   Max.   :2753  
##                                                               
##       AST            STL              BLK             TOV      
##  Min.   :1423   Min.   : 455.0   Min.   :204.0   Min.   : 931  
##  1st Qu.:1735   1st Qu.: 599.0   1st Qu.:359.0   1st Qu.:1192  
##  Median :1899   Median : 658.0   Median :410.0   Median :1289  
##  Mean   :1912   Mean   : 668.4   Mean   :419.8   Mean   :1303  
##  3rd Qu.:2078   3rd Qu.: 729.0   3rd Qu.:469.5   3rd Qu.:1396  
##  Max.   :2575   Max.   :1053.0   Max.   :716.0   Max.   :1873  
##                                                                
##                Team.fctr   Team.fctr.num  
##  Atlanta Hawks      : 31   Min.   : 1.00  
##  Boston Celtics     : 31   1st Qu.: 7.00  
##  Chicago Bulls      : 31   Median :15.00  
##  Cleveland Cavaliers: 31   Mean   :15.37  
##  Denver Nuggets     : 31   3rd Qu.:23.00  
##  Detroit Pistons    : 31   Max.   :37.00  
##  (Other)            :649                  
##     SeasonEnd          Team      Playoffs             W           PTS 
##             0             0             0             0             0 
##        oppPTS            FG           FGA           X2P          X2PA 
##             0             0             0             0             0 
##           X3P          X3PA            FT           FTA           ORB 
##             0             0             0             0             0 
##           DRB           AST           STL           BLK           TOV 
##             0             0             0             0             0 
##     Team.fctr Team.fctr.num 
##             0             0
```

```r
glb_predct_df <- add_new_diag_feats(glb_predct_df, glb_entity_df)
```

```
##    SeasonEnd        Team              Playoffs         W        
##  Min.   :2013   Length:28          Min.   :0.0   Min.   :20.00  
##  1st Qu.:2013   Class :character   1st Qu.:0.0   1st Qu.:29.00  
##  Median :2013   Mode  :character   Median :0.5   Median :42.00  
##  Mean   :2013                      Mean   :0.5   Mean   :40.68  
##  3rd Qu.:2013                      3rd Qu.:1.0   3rd Qu.:50.25  
##  Max.   :2013                      Max.   :1.0   Max.   :66.00  
##                                                                 
##       PTS           oppPTS           FG            FGA      
##  Min.   :7640   Min.   :7319   Min.   :2823   Min.   :6348  
##  1st Qu.:7763   1st Qu.:7898   1st Qu.:2975   1st Qu.:6643  
##  Median :8014   Median :8068   Median :3052   Median :6696  
##  Mean   :8062   Mean   :8073   Mean   :3051   Mean   :6737  
##  3rd Qu.:8294   3rd Qu.:8288   3rd Qu.:3126   3rd Qu.:6893  
##  Max.   :8704   Max.   :8619   Max.   :3339   Max.   :7197  
##                                                             
##       X2P            X2PA           X3P             X3PA     
##  Min.   :2105   Min.   :4318   Min.   :382.0   Min.   :1107  
##  1st Qu.:2375   1st Qu.:4845   1st Qu.:511.5   1st Qu.:1469  
##  Median :2474   Median :5203   Median :584.5   Median :1608  
##  Mean   :2460   Mean   :5091   Mean   :591.1   Mean   :1646  
##  3rd Qu.:2540   3rd Qu.:5336   3rd Qu.:659.2   3rd Qu.:1761  
##  Max.   :2818   Max.   :5572   Max.   :891.0   Max.   :2371  
##                                                              
##        FT            FTA            ORB              DRB      
##  Min.   :1004   Min.   :1359   Min.   : 666.0   Min.   :2359  
##  1st Qu.:1298   1st Qu.:1695   1st Qu.: 882.2   1st Qu.:2452  
##  Median :1357   Median :1786   Median : 927.5   Median :2482  
##  Mean   :1368   Mean   :1819   Mean   : 920.0   Mean   :2533  
##  3rd Qu.:1440   3rd Qu.:1906   3rd Qu.: 989.5   3rd Qu.:2622  
##  Max.   :1819   Max.   :2289   Max.   :1092.0   Max.   :2801  
##                                                               
##       AST            STL             BLK             TOV      
##  Min.   :1579   Min.   :520.0   Min.   :294.0   Min.   : 988  
##  1st Qu.:1737   1st Qu.:590.2   1st Qu.:366.5   1st Qu.:1152  
##  Median :1840   Median :653.5   Median :408.5   Median :1201  
##  Mean   :1819   Mean   :640.4   Mean   :419.4   Mean   :1191  
##  3rd Qu.:1887   3rd Qu.:686.2   3rd Qu.:448.0   3rd Qu.:1233  
##  Max.   :2058   Max.   :784.0   Max.   :624.0   Max.   :1348  
##                                                               
##                Team.fctr  Team.fctr.num  
##  Atlanta Hawks      : 1   Min.   : 1.00  
##  Brooklyn Nets      : 1   1st Qu.: 7.75  
##  Charlotte Bobcats  : 1   Median :14.50  
##  Chicago Bulls      : 1   Mean   :14.50  
##  Cleveland Cavaliers: 1   3rd Qu.:21.25  
##  Dallas Mavericks   : 1   Max.   :28.00  
##  (Other)            :22                  
##     SeasonEnd          Team      Playoffs             W           PTS 
##             0             0             0             0             0 
##        oppPTS            FG           FGA           X2P          X2PA 
##             0             0             0             0             0 
##           X3P          X3PA            FT           FTA           ORB 
##             0             0             0             0             0 
##           DRB           AST           STL           BLK           TOV 
##             0             0             0             0             0 
##     Team.fctr Team.fctr.num 
##             0             0
```

```r
#pairs(subset(glb_entity_df, select=-c(col_symbol)))

#   Histogram of predictor in glb_entity_df & glb_predct_df
# Check for glb_predct_df & glb_entity_df features range mismatches

# Other diagnostics:
# print(subset(glb_entity_df, <col1_name> == max(glb_entity_df$<col1_name>, na.rm=TRUE) & 
#                         <col2_name> <= mean(glb_entity_df$<col1_name>, na.rm=TRUE)))

# print(<col_name>_freq_glb_entity_df <- mycreate_tbl_df(glb_entity_df, "<col_name>"))
# print(which.min(table(glb_entity_df$<col_name>)))
# print(which.max(table(glb_entity_df$<col_name>)))
# print(which.max(table(glb_entity_df$<col1_name>, glb_entity_df$<col2_name>)[, 2]))
# print(sort(table(glb_entity_df$W, glb_entity_df$Playoffs)))
# print(table(is.na(glb_entity_df$<col1_name>), glb_entity_df$<col2_name>))
# print(xtabs(~ <col1_name>, glb_entity_df))
print(xtabs(~ W + Playoffs, glb_entity_df))
```

```
##     Playoffs
## W     0  1
##   11  2  0
##   12  2  0
##   13  2  0
##   14  2  0
##   15 10  0
##   16  2  0
##   17 11  0
##   18  5  0
##   19 10  0
##   20 10  0
##   21 12  0
##   22 11  0
##   23 11  0
##   24 18  0
##   25 11  0
##   26 17  0
##   27 10  0
##   28 18  0
##   29 12  0
##   30 19  1
##   31 15  1
##   32 12  0
##   33 17  0
##   34 16  0
##   35 13  3
##   36 17  4
##   37 15  4
##   38  8  7
##   39 10 10
##   40  9 13
##   41 11 26
##   42  8 29
##   43  2 18
##   44  2 27
##   45  3 22
##   46  1 15
##   47  0 28
##   48  1 14
##   49  0 17
##   50  0 32
##   51  0 12
##   52  0 20
##   53  0 17
##   54  0 18
##   55  0 24
##   56  0 16
##   57  0 23
##   58  0 13
##   59  0 14
##   60  0  8
##   61  0 10
##   62  0 13
##   63  0  7
##   64  0  3
##   65  0  3
##   66  0  2
##   67  0  4
##   69  0  1
##   72  0  1
```

```r
# print(<col1_name>_<col2_name>_xtab_glb_entity_df <- 
#   mycreate_xtab(glb_entity_df, c("<col1_name>", "<col2_name>")))
# <col1_name>_<col2_name>_xtab_glb_entity_df[is.na(<col1_name>_<col2_name>_xtab_glb_entity_df)] <- 0
# print(<col1_name>_<col2_name>_xtab_glb_entity_df <- 
#   mutate(<col1_name>_<col2_name>_xtab_glb_entity_df, 
#             <col3_name>=(<col1_name> * 1.0) / (<col1_name> + <col2_name>))) 

# print(<col2_name>_min_entity_arr <- 
#    sort(tapply(glb_entity_df$<col1_name>, glb_entity_df$<col2_name>, min, na.rm=TRUE)))
# print(<col1_name>_na_by_<col2_name>_arr <- 
#    sort(tapply(glb_entity_df$<col1_name>.NA, glb_entity_df$<col2_name>, mean, na.rm=TRUE)))


# Other plots:
# print(myplot_histogram(glb_entity_df, "<col1_name>"))
# print(myplot_box(df=glb_entity_df, ycol_names="<col1_name>"))
# print(myplot_box(df=glb_entity_df, ycol_names="<col1_name>", xcol_name="<col2_name>"))
# print(myplot_line(subset(glb_entity_df, Symbol %in% c("KO", "PG")), 
#                   "Date.my", "StockPrice", facet_row_colnames="Symbol") + 
#     geom_vline(xintercept=as.numeric(as.Date("2003-03-01"))) +
#     geom_vline(xintercept=as.numeric(as.Date("1983-01-01")))        
#         )
# print(myplot_scatter(glb_entity_df, "<col1_name>", "<col2_name>"))

script_df <- rbind(script_df, 
    data.frame(chunk_label="manage_missing_data", 
        chunk_step_major=max(script_df$chunk_step_major), 
        chunk_step_minor=script_df[nrow(script_df), "chunk_step_minor"]+1))
print(script_df)
```

```
##            chunk_label chunk_step_major chunk_step_minor
## 1          import_data                1                0
## 2         cleanse_data                2                0
## 3 inspect_explore_data                2                1
## 4  manage_missing_data                2                2
```

### Step `2`.`2`: manage missing data

```r
script_df <- rbind(script_df, 
    data.frame(chunk_label="encode_retype_data", 
        chunk_step_major=max(script_df$chunk_step_major), 
        chunk_step_minor=script_df[nrow(script_df), "chunk_step_minor"]+1))
print(script_df)
```

```
##            chunk_label chunk_step_major chunk_step_minor
## 1          import_data                1                0
## 2         cleanse_data                2                0
## 3 inspect_explore_data                2                1
## 4  manage_missing_data                2                2
## 5   encode_retype_data                2                3
```

### Step `2`.`3`: encode/retype data

```r
# map_<col_name>_df <- myimport_data(
#     url="<map_url>", 
#     comment="map_<col_name>_df", print_diagn=TRUE)
map_Team_df <- read.csv(paste0(getwd(), "/data/Team_Conf_Map.csv"), strip.white=TRUE)

glb_entity_df <- mymap_codes(df=glb_entity_df, from_col_name="Team", 
                             to_col_name="Conf", 
                            map_df=map_Team_df, map_join_col_name="Team", 
                          map_tgt_col_name="Conf")
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
##                  Team Conf _n
## 1       Atlanta Hawks East 31
## 2      Boston Celtics East 31
## 3       Chicago Bulls East 31
## 4 Cleveland Cavaliers East 31
## 5      Denver Nuggets West 31
## 6     Detroit Pistons East 31
##                      Team Conf _n
## 1           Atlanta Hawks East 31
## 13        New York Knicks East 31
## 14     Philadelphia 76ers East 31
## 16 Portland Trail Blazers West 31
## 28      Charlotte Hornets East 13
## 33      Kansas City Kings West  6
##                                 Team Conf _n
## 32               New Orleans Hornets East  7
## 33                 Kansas City Kings West  6
## 34                San Diego Clippers West  5
## 35               Vancouver Grizzlies West  5
## 36             Oklahoma City Thunder West  3
## 37 New Orleans/Oklahoma City Hornets East  2
```

```
## Warning in myplot_hbar(map_summry_df[1:min(nrow(map_summry_df), 10), ], :
## Aggregating input dataframe:SELECT Conf, SUM(_n) AS _n FROM df GROUP BY
## Conf
```

![](NBA_Wins_files/figure-html/encode_retype_data_1-1.png) 

```r
glb_predct_df <- mymap_codes(df=glb_predct_df, from_col_name="Team", 
                             to_col_name="Conf", 
                            map_df=map_Team_df, map_join_col_name="Team", 
                          map_tgt_col_name="Conf")
```

```
##                  Team Conf _n
## 1       Atlanta Hawks East  1
## 2       Brooklyn Nets East  1
## 3   Charlotte Bobcats East  1
## 4       Chicago Bulls East  1
## 5 Cleveland Cavaliers East  1
## 6    Dallas Mavericks West  1
##                     Team Conf _n
## 6       Dallas Mavericks West  1
## 9  Golden State Warriors West  1
## 12    Los Angeles Lakers West  1
## 17   New Orleans Hornets East  1
## 24      Sacramento Kings West  1
## 26       Toronto Raptors East  1
##                      Team Conf _n
## 23 Portland Trail Blazers West  1
## 24       Sacramento Kings West  1
## 25      San Antonio Spurs West  1
## 26        Toronto Raptors East  1
## 27              Utah Jazz West  1
## 28     Washington Wizards East  1
```

```
## Warning in myplot_hbar(map_summry_df[1:min(nrow(map_summry_df), 10), ], :
## Aggregating input dataframe:SELECT Conf, SUM(_n) AS _n FROM df GROUP BY
## Conf
```

![](NBA_Wins_files/figure-html/encode_retype_data_1-2.png) 

```r
glb_entity_df$Conf.fctr <- factor(glb_entity_df$Conf, 
                    as.factor(union(glb_entity_df$Conf, glb_predct_df$Conf)))
glb_predct_df$Conf.fctr <- factor(glb_predct_df$Conf, 
                    as.factor(union(glb_entity_df$Conf, glb_predct_df$Conf)))

script_df <- rbind(script_df, 
                   data.frame(chunk_label="extract_features", 
                              chunk_step_major=max(script_df$chunk_step_major)+1, 
                              chunk_step_minor=0))
print(script_df)
```

```
##            chunk_label chunk_step_major chunk_step_minor
## 1          import_data                1                0
## 2         cleanse_data                2                0
## 3 inspect_explore_data                2                1
## 4  manage_missing_data                2                2
## 5   encode_retype_data                2                3
## 6     extract_features                3                0
```

## Step `3`: extract features

```r
# Create new features that help prediction
glb_entity_df <- mutate(glb_entity_df,
    PTS.diff=PTS - oppPTS
                    )

glb_predct_df <- mutate(glb_predct_df,
    PTS.diff=PTS - oppPTS
                    )

print(summary(glb_entity_df))
```

```
##      Team             SeasonEnd       Playoffs            W       
##  Length:835         Min.   :1980   Min.   :0.0000   Min.   :11.0  
##  Class :character   1st Qu.:1989   1st Qu.:0.0000   1st Qu.:31.0  
##  Mode  :character   Median :1996   Median :1.0000   Median :42.0  
##                     Mean   :1996   Mean   :0.5749   Mean   :41.0  
##                     3rd Qu.:2005   3rd Qu.:1.0000   3rd Qu.:50.5  
##                     Max.   :2011   Max.   :1.0000   Max.   :72.0  
##                                                                   
##       PTS            oppPTS            FG            FGA      
##  Min.   : 6901   Min.   : 6909   Min.   :2565   Min.   :5972  
##  1st Qu.: 7934   1st Qu.: 7934   1st Qu.:2974   1st Qu.:6564  
##  Median : 8312   Median : 8365   Median :3150   Median :6831  
##  Mean   : 8370   Mean   : 8370   Mean   :3200   Mean   :6873  
##  3rd Qu.: 8784   3rd Qu.: 8768   3rd Qu.:3434   3rd Qu.:7157  
##  Max.   :10371   Max.   :10723   Max.   :3980   Max.   :8868  
##                                                               
##       X2P            X2PA           X3P             X3PA       
##  Min.   :1981   Min.   :4153   Min.   : 10.0   Min.   :  75.0  
##  1st Qu.:2510   1st Qu.:5269   1st Qu.:131.5   1st Qu.: 413.0  
##  Median :2718   Median :5706   Median :329.0   Median : 942.0  
##  Mean   :2881   Mean   :5956   Mean   :319.0   Mean   : 916.9  
##  3rd Qu.:3296   3rd Qu.:6754   3rd Qu.:481.5   3rd Qu.:1347.5  
##  Max.   :3954   Max.   :7873   Max.   :841.0   Max.   :2284.0  
##                                                                
##        FT            FTA            ORB              DRB      
##  Min.   :1189   Min.   :1475   Min.   : 639.0   Min.   :2044  
##  1st Qu.:1502   1st Qu.:2008   1st Qu.: 953.5   1st Qu.:2346  
##  Median :1628   Median :2176   Median :1055.0   Median :2433  
##  Mean   :1650   Mean   :2190   Mean   :1061.6   Mean   :2427  
##  3rd Qu.:1781   3rd Qu.:2352   3rd Qu.:1167.0   3rd Qu.:2516  
##  Max.   :2388   Max.   :3051   Max.   :1520.0   Max.   :2753  
##                                                               
##       AST            STL              BLK             TOV      
##  Min.   :1423   Min.   : 455.0   Min.   :204.0   Min.   : 931  
##  1st Qu.:1735   1st Qu.: 599.0   1st Qu.:359.0   1st Qu.:1192  
##  Median :1899   Median : 658.0   Median :410.0   Median :1289  
##  Mean   :1912   Mean   : 668.4   Mean   :419.8   Mean   :1303  
##  3rd Qu.:2078   3rd Qu.: 729.0   3rd Qu.:469.5   3rd Qu.:1396  
##  Max.   :2575   Max.   :1053.0   Max.   :716.0   Max.   :1873  
##                                                                
##                Team.fctr   Team.fctr.num       Conf           Conf.fctr 
##  Atlanta Hawks      : 31   Min.   : 1.00   Length:835         East:428  
##  Boston Celtics     : 31   1st Qu.: 7.00   Class :character   West:407  
##  Chicago Bulls      : 31   Median :15.00   Mode  :character             
##  Cleveland Cavaliers: 31   Mean   :15.37                                
##  Denver Nuggets     : 31   3rd Qu.:23.00                                
##  Detroit Pistons    : 31   Max.   :37.00                                
##  (Other)            :649                                                
##     PTS.diff      
##  Min.   :-1246.0  
##  1st Qu.: -268.0  
##  Median :   21.0  
##  Mean   :    0.0  
##  3rd Qu.:  287.5  
##  Max.   : 1004.0  
## 
```

```r
print(summary(glb_predct_df))
```

```
##      Team             SeasonEnd       Playoffs         W        
##  Length:28          Min.   :2013   Min.   :0.0   Min.   :20.00  
##  Class :character   1st Qu.:2013   1st Qu.:0.0   1st Qu.:29.00  
##  Mode  :character   Median :2013   Median :0.5   Median :42.00  
##                     Mean   :2013   Mean   :0.5   Mean   :40.68  
##                     3rd Qu.:2013   3rd Qu.:1.0   3rd Qu.:50.25  
##                     Max.   :2013   Max.   :1.0   Max.   :66.00  
##                                                                 
##       PTS           oppPTS           FG            FGA      
##  Min.   :7640   Min.   :7319   Min.   :2823   Min.   :6348  
##  1st Qu.:7763   1st Qu.:7898   1st Qu.:2975   1st Qu.:6643  
##  Median :8014   Median :8068   Median :3052   Median :6696  
##  Mean   :8062   Mean   :8073   Mean   :3051   Mean   :6737  
##  3rd Qu.:8294   3rd Qu.:8288   3rd Qu.:3126   3rd Qu.:6893  
##  Max.   :8704   Max.   :8619   Max.   :3339   Max.   :7197  
##                                                             
##       X2P            X2PA           X3P             X3PA     
##  Min.   :2105   Min.   :4318   Min.   :382.0   Min.   :1107  
##  1st Qu.:2375   1st Qu.:4845   1st Qu.:511.5   1st Qu.:1469  
##  Median :2474   Median :5203   Median :584.5   Median :1608  
##  Mean   :2460   Mean   :5091   Mean   :591.1   Mean   :1646  
##  3rd Qu.:2540   3rd Qu.:5336   3rd Qu.:659.2   3rd Qu.:1761  
##  Max.   :2818   Max.   :5572   Max.   :891.0   Max.   :2371  
##                                                              
##        FT            FTA            ORB              DRB      
##  Min.   :1004   Min.   :1359   Min.   : 666.0   Min.   :2359  
##  1st Qu.:1298   1st Qu.:1695   1st Qu.: 882.2   1st Qu.:2452  
##  Median :1357   Median :1786   Median : 927.5   Median :2482  
##  Mean   :1368   Mean   :1819   Mean   : 920.0   Mean   :2533  
##  3rd Qu.:1440   3rd Qu.:1906   3rd Qu.: 989.5   3rd Qu.:2622  
##  Max.   :1819   Max.   :2289   Max.   :1092.0   Max.   :2801  
##                                                               
##       AST            STL             BLK             TOV      
##  Min.   :1579   Min.   :520.0   Min.   :294.0   Min.   : 988  
##  1st Qu.:1737   1st Qu.:590.2   1st Qu.:366.5   1st Qu.:1152  
##  Median :1840   Median :653.5   Median :408.5   Median :1201  
##  Mean   :1819   Mean   :640.4   Mean   :419.4   Mean   :1191  
##  3rd Qu.:1887   3rd Qu.:686.2   3rd Qu.:448.0   3rd Qu.:1233  
##  Max.   :2058   Max.   :784.0   Max.   :624.0   Max.   :1348  
##                                                               
##                Team.fctr  Team.fctr.num       Conf           Conf.fctr
##  Atlanta Hawks      : 1   Min.   : 1.00   Length:28          East:14  
##  Brooklyn Nets      : 1   1st Qu.: 7.75   Class :character   West:14  
##  Charlotte Bobcats  : 1   Median :14.50   Mode  :character            
##  Chicago Bulls      : 1   Mean   :14.50                               
##  Cleveland Cavaliers: 1   3rd Qu.:21.25                               
##  Dallas Mavericks   : 1   Max.   :28.00                               
##  (Other)            :22                                               
##     PTS.diff     
##  Min.   :-757.0  
##  1st Qu.:-284.8  
##  Median : -28.0  
##  Mean   : -11.0  
##  3rd Qu.: 298.8  
##  Max.   : 755.0  
## 
```

```r
script_df <- rbind(script_df, 
                   data.frame(chunk_label="select_features", 
                              chunk_step_major=max(script_df$chunk_step_major)+1, 
                              chunk_step_minor=0))
print(script_df)
```

```
##            chunk_label chunk_step_major chunk_step_minor
## 1          import_data                1                0
## 2         cleanse_data                2                0
## 3 inspect_explore_data                2                1
## 4  manage_missing_data                2                2
## 5   encode_retype_data                2                3
## 6     extract_features                3                0
## 7      select_features                4                0
```

## Step `4`: select features

```r
print(glb_feats_df <- myselect_features())
```

```
##                          id       cor.y  cor.y.abs
## PTS.diff           PTS.diff  0.97074326 0.97074326
## DRB                     DRB  0.47089750 0.47089750
## oppPTS               oppPTS -0.33157294 0.33157294
## AST                     AST  0.32005177 0.32005177
## PTS                     PTS  0.29882561 0.29882561
## TOV                     TOV -0.24318588 0.24318588
## FT                       FT  0.20490600 0.20490600
## BLK                     BLK  0.20392100 0.20392100
## FG                       FG  0.19039642 0.19039642
## FTA                     FTA  0.16188731 0.16188731
## Team.fctr.num Team.fctr.num -0.15119487 0.15119487
## X3P                     X3P  0.11904456 0.11904456
## STL                     STL  0.11619440 0.11619440
## ORB                     ORB -0.09573676 0.09573676
## X2PA                   X2PA -0.08703653 0.08703653
## X3PA                   X3PA  0.08328625 0.08328625
## FGA                     FGA -0.07144566 0.07144566
## X2P                     X2P  0.06927898 0.06927898
## SeasonEnd         SeasonEnd  0.00000000 0.00000000
```

```r
script_df <- rbind(script_df, 
    data.frame(chunk_label="remove_correlated_features", 
        chunk_step_major=max(script_df$chunk_step_major),
        chunk_step_minor=script_df[nrow(script_df), "chunk_step_minor"]+1))        
print(script_df)
```

```
##                  chunk_label chunk_step_major chunk_step_minor
## 1                import_data                1                0
## 2               cleanse_data                2                0
## 3       inspect_explore_data                2                1
## 4        manage_missing_data                2                2
## 5         encode_retype_data                2                3
## 6           extract_features                3                0
## 7            select_features                4                0
## 8 remove_correlated_features                4                1
```

### Step `4`.`1`: remove correlated features

```r
print(glb_feats_df <- orderBy(~-cor.y, 
                    merge(glb_feats_df, mydelete_cor_features(), all.x=TRUE)))
```

```
##                    PTS.diff          DRB      oppPTS         AST
## PTS.diff       1.000000e+00  0.467373785 -0.34003607  0.32340381
## DRB            4.673738e-01  1.000000000 -0.21262703  0.05647709
## oppPTS        -3.400361e-01 -0.212627029  1.00000000  0.54256181
## AST            3.234038e-01  0.056477086  0.54256181  1.00000000
## PTS            3.093789e-01  0.090291366  0.78907471  0.75988915
## TOV           -2.476858e-01 -0.190956902  0.58241272  0.43053288
## FT             2.154451e-01  0.003604883  0.55058757  0.44720848
## BLK            1.851507e-01  0.242990816  0.03076664  0.20308000
## FG             1.966908e-01  0.008966352  0.80446737  0.81226652
## FTA            1.711987e-01 -0.043542668  0.53721485  0.42797223
## Team.fctr.num -1.499552e-01 -0.058154820 -0.10100550 -0.17491216
## X3P            1.212155e-01  0.233353555 -0.56282942 -0.56785931
## STL            1.280813e-01 -0.331127910  0.34348011  0.44313827
## ORB           -9.359240e-02 -0.269179339  0.55187982  0.40676555
## X2PA          -8.672031e-02 -0.165315490  0.75654063  0.67770751
## X3PA           8.333602e-02  0.223060547 -0.56332937 -0.59278305
## FGA           -7.072567e-02 -0.050945458  0.83087856  0.62911736
## X2P            7.235925e-02 -0.098690188  0.76984080  0.77711327
## SeasonEnd      4.173684e-21  0.267127100 -0.63244845 -0.67212300
##                       PTS        TOV           FT         BLK           FG
## PTS.diff       0.30937888 -0.2476858  0.215445119  0.18515068  0.196690814
## DRB            0.09029137 -0.1909569  0.003604883  0.24299082  0.008966352
## oppPTS         0.78907471  0.5824127  0.550587572  0.03076664  0.804467370
## AST            0.75988915  0.4305329  0.447208477  0.20308000  0.812266519
## PTS            1.00000000  0.4271383  0.697483850  0.15205537  0.941954732
## TOV            0.42713832  1.0000000  0.438684976  0.24156872  0.517630444
## FT             0.69748385  0.4386850  1.000000000  0.16465598  0.538342589
## BLK            0.15205537  0.2415687  0.164655982  1.00000000  0.177502748
## FG             0.94195473  0.5176304  0.538342589  0.17750275  1.000000000
## FTA            0.65505876  0.5256698  0.950493246  0.21637171  0.521216830
## Team.fctr.num -0.20008998 -0.1404593 -0.154385609 -0.07311722 -0.198789524
## X3P           -0.48994891 -0.6801733 -0.508712984 -0.23107383 -0.668272893
## STL            0.43099026  0.4640752  0.323156975  0.12045594  0.469730269
## ORB            0.49692073  0.5459866  0.393398702  0.20110479  0.598591143
## X2PA           0.70836140  0.6448524  0.515802012  0.20610678  0.859224632
## X3PA          -0.51519812 -0.6778031 -0.517849574 -0.23403142 -0.688763042
## FGA            0.79397947  0.4499814  0.391691425  0.12109667  0.879662910
## X2P            0.82572457  0.6377157  0.574293954  0.21771153  0.942919965
## SeasonEnd     -0.63952773 -0.7238690 -0.513699180 -0.20416249 -0.761598813
##                       FTA Team.fctr.num        X3P        STL        ORB
## PTS.diff       0.17119872   -0.14995517  0.1212155  0.1280813 -0.0935924
## DRB           -0.04354267   -0.05815482  0.2333536 -0.3311279 -0.2691793
## oppPTS         0.53721485   -0.10100550 -0.5628294  0.3434801  0.5518798
## AST            0.42797223   -0.17491216 -0.5678593  0.4431383  0.4067656
## PTS            0.65505876   -0.20008998 -0.4899489  0.4309903  0.4969207
## TOV            0.52566979   -0.14045929 -0.6801733  0.4640752  0.5459866
## FT             0.95049325   -0.15438561 -0.5087130  0.3231570  0.3933987
## BLK            0.21637171   -0.07311722 -0.2310738  0.1204559  0.2011048
## FG             0.52121683   -0.19878952 -0.6682729  0.4697303  0.5985911
## FTA            1.00000000   -0.13968994 -0.5338969  0.3700552  0.4745584
## Team.fctr.num -0.13968994    1.00000000  0.1423720 -0.1531193 -0.2009572
## X3P           -0.53389685    0.14237198  1.0000000 -0.4168543 -0.6651682
## STL            0.37005522   -0.15311931 -0.4168543  1.0000000  0.4929415
## ORB            0.47455838   -0.20095724 -0.6651682  0.4929415  1.0000000
## X2PA           0.52375675   -0.17580617 -0.9207320  0.4900862  0.7656147
## X3PA          -0.53778289    0.14165189  0.9945109 -0.4090916 -0.6491727
## FGA            0.38212215   -0.17904220 -0.6075645  0.4805329  0.7375094
## X2P            0.57454277   -0.19170714 -0.8778664  0.4890027  0.6831180
## SeasonEnd     -0.54689652    0.21455487  0.8381421 -0.5020598 -0.7020128
##                      X2PA        X3PA         FGA         X2P
## PTS.diff      -0.08672031  0.08333602 -0.07072567  0.07235925
## DRB           -0.16531549  0.22306055 -0.05094546 -0.09869019
## oppPTS         0.75654063 -0.56332937  0.83087856  0.76984080
## AST            0.67770751 -0.59278305  0.62911736  0.77711327
## PTS            0.70836140 -0.51519812  0.79397947  0.82572457
## TOV            0.64485242 -0.67780315  0.44998136  0.63771567
## FT             0.51580201 -0.51784957  0.39169143  0.57429395
## BLK            0.20610678 -0.23403142  0.12109667  0.21771153
## FG             0.85922463 -0.68876304  0.87966291  0.94291996
## FTA            0.52375675 -0.53778289  0.38212215  0.57454277
## Team.fctr.num -0.17580617  0.14165189 -0.17904220 -0.19170714
## X3P           -0.92073200  0.99451088 -0.60756448 -0.87786641
## STL            0.49008620 -0.40909163  0.48053292  0.48900271
## ORB            0.76561469 -0.64917273  0.73750939  0.68311802
## X2PA           1.00000000 -0.92324423  0.86485931  0.96530915
## X3PA          -0.92324423  1.00000000 -0.60559564 -0.88859996
## FGA            0.86485931 -0.60559564  1.00000000  0.83827486
## X2P            0.96530915 -0.88859996  0.83827486  1.00000000
## SeasonEnd     -0.86725681  0.85055227 -0.68490478 -0.86548930
##                   SeasonEnd
## PTS.diff       4.173684e-21
## DRB            2.671271e-01
## oppPTS        -6.324485e-01
## AST           -6.721230e-01
## PTS           -6.395277e-01
## TOV           -7.238690e-01
## FT            -5.136992e-01
## BLK           -2.041625e-01
## FG            -7.615988e-01
## FTA           -5.468965e-01
## Team.fctr.num  2.145549e-01
## X3P            8.381421e-01
## STL           -5.020598e-01
## ORB           -7.020128e-01
## X2PA          -8.672568e-01
## X3PA           8.505523e-01
## FGA           -6.849048e-01
## X2P           -8.654893e-01
## SeasonEnd      1.000000e+00
##                   PTS.diff         DRB     oppPTS        AST        PTS
## PTS.diff      0.000000e+00 0.467373785 0.34003607 0.32340381 0.30937888
## DRB           4.673738e-01 0.000000000 0.21262703 0.05647709 0.09029137
## oppPTS        3.400361e-01 0.212627029 0.00000000 0.54256181 0.78907471
## AST           3.234038e-01 0.056477086 0.54256181 0.00000000 0.75988915
## PTS           3.093789e-01 0.090291366 0.78907471 0.75988915 0.00000000
## TOV           2.476858e-01 0.190956902 0.58241272 0.43053288 0.42713832
## FT            2.154451e-01 0.003604883 0.55058757 0.44720848 0.69748385
## BLK           1.851507e-01 0.242990816 0.03076664 0.20308000 0.15205537
## FG            1.966908e-01 0.008966352 0.80446737 0.81226652 0.94195473
## FTA           1.711987e-01 0.043542668 0.53721485 0.42797223 0.65505876
## Team.fctr.num 1.499552e-01 0.058154820 0.10100550 0.17491216 0.20008998
## X3P           1.212155e-01 0.233353555 0.56282942 0.56785931 0.48994891
## STL           1.280813e-01 0.331127910 0.34348011 0.44313827 0.43099026
## ORB           9.359240e-02 0.269179339 0.55187982 0.40676555 0.49692073
## X2PA          8.672031e-02 0.165315490 0.75654063 0.67770751 0.70836140
## X3PA          8.333602e-02 0.223060547 0.56332937 0.59278305 0.51519812
## FGA           7.072567e-02 0.050945458 0.83087856 0.62911736 0.79397947
## X2P           7.235925e-02 0.098690188 0.76984080 0.77711327 0.82572457
## SeasonEnd     4.173684e-21 0.267127100 0.63244845 0.67212300 0.63952773
##                     TOV          FT        BLK          FG        FTA
## PTS.diff      0.2476858 0.215445119 0.18515068 0.196690814 0.17119872
## DRB           0.1909569 0.003604883 0.24299082 0.008966352 0.04354267
## oppPTS        0.5824127 0.550587572 0.03076664 0.804467370 0.53721485
## AST           0.4305329 0.447208477 0.20308000 0.812266519 0.42797223
## PTS           0.4271383 0.697483850 0.15205537 0.941954732 0.65505876
## TOV           0.0000000 0.438684976 0.24156872 0.517630444 0.52566979
## FT            0.4386850 0.000000000 0.16465598 0.538342589 0.95049325
## BLK           0.2415687 0.164655982 0.00000000 0.177502748 0.21637171
## FG            0.5176304 0.538342589 0.17750275 0.000000000 0.52121683
## FTA           0.5256698 0.950493246 0.21637171 0.521216830 0.00000000
## Team.fctr.num 0.1404593 0.154385609 0.07311722 0.198789524 0.13968994
## X3P           0.6801733 0.508712984 0.23107383 0.668272893 0.53389685
## STL           0.4640752 0.323156975 0.12045594 0.469730269 0.37005522
## ORB           0.5459866 0.393398702 0.20110479 0.598591143 0.47455838
## X2PA          0.6448524 0.515802012 0.20610678 0.859224632 0.52375675
## X3PA          0.6778031 0.517849574 0.23403142 0.688763042 0.53778289
## FGA           0.4499814 0.391691425 0.12109667 0.879662910 0.38212215
## X2P           0.6377157 0.574293954 0.21771153 0.942919965 0.57454277
## SeasonEnd     0.7238690 0.513699180 0.20416249 0.761598813 0.54689652
##               Team.fctr.num       X3P       STL       ORB       X2PA
## PTS.diff         0.14995517 0.1212155 0.1280813 0.0935924 0.08672031
## DRB              0.05815482 0.2333536 0.3311279 0.2691793 0.16531549
## oppPTS           0.10100550 0.5628294 0.3434801 0.5518798 0.75654063
## AST              0.17491216 0.5678593 0.4431383 0.4067656 0.67770751
## PTS              0.20008998 0.4899489 0.4309903 0.4969207 0.70836140
## TOV              0.14045929 0.6801733 0.4640752 0.5459866 0.64485242
## FT               0.15438561 0.5087130 0.3231570 0.3933987 0.51580201
## BLK              0.07311722 0.2310738 0.1204559 0.2011048 0.20610678
## FG               0.19878952 0.6682729 0.4697303 0.5985911 0.85922463
## FTA              0.13968994 0.5338969 0.3700552 0.4745584 0.52375675
## Team.fctr.num    0.00000000 0.1423720 0.1531193 0.2009572 0.17580617
## X3P              0.14237198 0.0000000 0.4168543 0.6651682 0.92073200
## STL              0.15311931 0.4168543 0.0000000 0.4929415 0.49008620
## ORB              0.20095724 0.6651682 0.4929415 0.0000000 0.76561469
## X2PA             0.17580617 0.9207320 0.4900862 0.7656147 0.00000000
## X3PA             0.14165189 0.9945109 0.4090916 0.6491727 0.92324423
## FGA              0.17904220 0.6075645 0.4805329 0.7375094 0.86485931
## X2P              0.19170714 0.8778664 0.4890027 0.6831180 0.96530915
## SeasonEnd        0.21455487 0.8381421 0.5020598 0.7020128 0.86725681
##                     X3PA        FGA        X2P    SeasonEnd
## PTS.diff      0.08333602 0.07072567 0.07235925 4.173684e-21
## DRB           0.22306055 0.05094546 0.09869019 2.671271e-01
## oppPTS        0.56332937 0.83087856 0.76984080 6.324485e-01
## AST           0.59278305 0.62911736 0.77711327 6.721230e-01
## PTS           0.51519812 0.79397947 0.82572457 6.395277e-01
## TOV           0.67780315 0.44998136 0.63771567 7.238690e-01
## FT            0.51784957 0.39169143 0.57429395 5.136992e-01
## BLK           0.23403142 0.12109667 0.21771153 2.041625e-01
## FG            0.68876304 0.87966291 0.94291996 7.615988e-01
## FTA           0.53778289 0.38212215 0.57454277 5.468965e-01
## Team.fctr.num 0.14165189 0.17904220 0.19170714 2.145549e-01
## X3P           0.99451088 0.60756448 0.87786641 8.381421e-01
## STL           0.40909163 0.48053292 0.48900271 5.020598e-01
## ORB           0.64917273 0.73750939 0.68311802 7.020128e-01
## X2PA          0.92324423 0.86485931 0.96530915 8.672568e-01
## X3PA          0.00000000 0.60559564 0.88859996 8.505523e-01
## FGA           0.60559564 0.00000000 0.83827486 6.849048e-01
## X2P           0.88859996 0.83827486 0.00000000 8.654893e-01
## SeasonEnd     0.85055227 0.68490478 0.86548930 0.000000e+00
## [1] "cor(X3P, X3PA)=0.9945"
```

![](NBA_Wins_files/figure-html/remove_correlated_features-1.png) 

```
## [1] "cor(W, X3P)=0.1190"
## [1] "cor(W, X3PA)=0.0833"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in mydelete_cor_features(): Dropping X3PA as a feature
```

![](NBA_Wins_files/figure-html/remove_correlated_features-2.png) 

```
##                          id       cor.y  cor.y.abs
## PTS.diff           PTS.diff  0.97074326 0.97074326
## DRB                     DRB  0.47089750 0.47089750
## oppPTS               oppPTS -0.33157294 0.33157294
## AST                     AST  0.32005177 0.32005177
## PTS                     PTS  0.29882561 0.29882561
## TOV                     TOV -0.24318588 0.24318588
## FT                       FT  0.20490600 0.20490600
## BLK                     BLK  0.20392100 0.20392100
## FG                       FG  0.19039642 0.19039642
## FTA                     FTA  0.16188731 0.16188731
## Team.fctr.num Team.fctr.num -0.15119487 0.15119487
## X3P                     X3P  0.11904456 0.11904456
## STL                     STL  0.11619440 0.11619440
## ORB                     ORB -0.09573676 0.09573676
## X2PA                   X2PA -0.08703653 0.08703653
## FGA                     FGA -0.07144566 0.07144566
## X2P                     X2P  0.06927898 0.06927898
## SeasonEnd         SeasonEnd  0.00000000 0.00000000
##                    PTS.diff          DRB      oppPTS         AST
## PTS.diff       1.000000e+00  0.467373785 -0.34003607  0.32340381
## DRB            4.673738e-01  1.000000000 -0.21262703  0.05647709
## oppPTS        -3.400361e-01 -0.212627029  1.00000000  0.54256181
## AST            3.234038e-01  0.056477086  0.54256181  1.00000000
## PTS            3.093789e-01  0.090291366  0.78907471  0.75988915
## TOV           -2.476858e-01 -0.190956902  0.58241272  0.43053288
## FT             2.154451e-01  0.003604883  0.55058757  0.44720848
## BLK            1.851507e-01  0.242990816  0.03076664  0.20308000
## FG             1.966908e-01  0.008966352  0.80446737  0.81226652
## FTA            1.711987e-01 -0.043542668  0.53721485  0.42797223
## Team.fctr.num -1.499552e-01 -0.058154820 -0.10100550 -0.17491216
## X3P            1.212155e-01  0.233353555 -0.56282942 -0.56785931
## STL            1.280813e-01 -0.331127910  0.34348011  0.44313827
## ORB           -9.359240e-02 -0.269179339  0.55187982  0.40676555
## X2PA          -8.672031e-02 -0.165315490  0.75654063  0.67770751
## FGA           -7.072567e-02 -0.050945458  0.83087856  0.62911736
## X2P            7.235925e-02 -0.098690188  0.76984080  0.77711327
## SeasonEnd      4.173684e-21  0.267127100 -0.63244845 -0.67212300
##                       PTS        TOV           FT         BLK           FG
## PTS.diff       0.30937888 -0.2476858  0.215445119  0.18515068  0.196690814
## DRB            0.09029137 -0.1909569  0.003604883  0.24299082  0.008966352
## oppPTS         0.78907471  0.5824127  0.550587572  0.03076664  0.804467370
## AST            0.75988915  0.4305329  0.447208477  0.20308000  0.812266519
## PTS            1.00000000  0.4271383  0.697483850  0.15205537  0.941954732
## TOV            0.42713832  1.0000000  0.438684976  0.24156872  0.517630444
## FT             0.69748385  0.4386850  1.000000000  0.16465598  0.538342589
## BLK            0.15205537  0.2415687  0.164655982  1.00000000  0.177502748
## FG             0.94195473  0.5176304  0.538342589  0.17750275  1.000000000
## FTA            0.65505876  0.5256698  0.950493246  0.21637171  0.521216830
## Team.fctr.num -0.20008998 -0.1404593 -0.154385609 -0.07311722 -0.198789524
## X3P           -0.48994891 -0.6801733 -0.508712984 -0.23107383 -0.668272893
## STL            0.43099026  0.4640752  0.323156975  0.12045594  0.469730269
## ORB            0.49692073  0.5459866  0.393398702  0.20110479  0.598591143
## X2PA           0.70836140  0.6448524  0.515802012  0.20610678  0.859224632
## FGA            0.79397947  0.4499814  0.391691425  0.12109667  0.879662910
## X2P            0.82572457  0.6377157  0.574293954  0.21771153  0.942919965
## SeasonEnd     -0.63952773 -0.7238690 -0.513699180 -0.20416249 -0.761598813
##                       FTA Team.fctr.num        X3P        STL        ORB
## PTS.diff       0.17119872   -0.14995517  0.1212155  0.1280813 -0.0935924
## DRB           -0.04354267   -0.05815482  0.2333536 -0.3311279 -0.2691793
## oppPTS         0.53721485   -0.10100550 -0.5628294  0.3434801  0.5518798
## AST            0.42797223   -0.17491216 -0.5678593  0.4431383  0.4067656
## PTS            0.65505876   -0.20008998 -0.4899489  0.4309903  0.4969207
## TOV            0.52566979   -0.14045929 -0.6801733  0.4640752  0.5459866
## FT             0.95049325   -0.15438561 -0.5087130  0.3231570  0.3933987
## BLK            0.21637171   -0.07311722 -0.2310738  0.1204559  0.2011048
## FG             0.52121683   -0.19878952 -0.6682729  0.4697303  0.5985911
## FTA            1.00000000   -0.13968994 -0.5338969  0.3700552  0.4745584
## Team.fctr.num -0.13968994    1.00000000  0.1423720 -0.1531193 -0.2009572
## X3P           -0.53389685    0.14237198  1.0000000 -0.4168543 -0.6651682
## STL            0.37005522   -0.15311931 -0.4168543  1.0000000  0.4929415
## ORB            0.47455838   -0.20095724 -0.6651682  0.4929415  1.0000000
## X2PA           0.52375675   -0.17580617 -0.9207320  0.4900862  0.7656147
## FGA            0.38212215   -0.17904220 -0.6075645  0.4805329  0.7375094
## X2P            0.57454277   -0.19170714 -0.8778664  0.4890027  0.6831180
## SeasonEnd     -0.54689652    0.21455487  0.8381421 -0.5020598 -0.7020128
##                      X2PA         FGA         X2P     SeasonEnd
## PTS.diff      -0.08672031 -0.07072567  0.07235925  4.173684e-21
## DRB           -0.16531549 -0.05094546 -0.09869019  2.671271e-01
## oppPTS         0.75654063  0.83087856  0.76984080 -6.324485e-01
## AST            0.67770751  0.62911736  0.77711327 -6.721230e-01
## PTS            0.70836140  0.79397947  0.82572457 -6.395277e-01
## TOV            0.64485242  0.44998136  0.63771567 -7.238690e-01
## FT             0.51580201  0.39169143  0.57429395 -5.136992e-01
## BLK            0.20610678  0.12109667  0.21771153 -2.041625e-01
## FG             0.85922463  0.87966291  0.94291996 -7.615988e-01
## FTA            0.52375675  0.38212215  0.57454277 -5.468965e-01
## Team.fctr.num -0.17580617 -0.17904220 -0.19170714  2.145549e-01
## X3P           -0.92073200 -0.60756448 -0.87786641  8.381421e-01
## STL            0.49008620  0.48053292  0.48900271 -5.020598e-01
## ORB            0.76561469  0.73750939  0.68311802 -7.020128e-01
## X2PA           1.00000000  0.86485931  0.96530915 -8.672568e-01
## FGA            0.86485931  1.00000000  0.83827486 -6.849048e-01
## X2P            0.96530915  0.83827486  1.00000000 -8.654893e-01
## SeasonEnd     -0.86725681 -0.68490478 -0.86548930  1.000000e+00
##                   PTS.diff         DRB     oppPTS        AST        PTS
## PTS.diff      0.000000e+00 0.467373785 0.34003607 0.32340381 0.30937888
## DRB           4.673738e-01 0.000000000 0.21262703 0.05647709 0.09029137
## oppPTS        3.400361e-01 0.212627029 0.00000000 0.54256181 0.78907471
## AST           3.234038e-01 0.056477086 0.54256181 0.00000000 0.75988915
## PTS           3.093789e-01 0.090291366 0.78907471 0.75988915 0.00000000
## TOV           2.476858e-01 0.190956902 0.58241272 0.43053288 0.42713832
## FT            2.154451e-01 0.003604883 0.55058757 0.44720848 0.69748385
## BLK           1.851507e-01 0.242990816 0.03076664 0.20308000 0.15205537
## FG            1.966908e-01 0.008966352 0.80446737 0.81226652 0.94195473
## FTA           1.711987e-01 0.043542668 0.53721485 0.42797223 0.65505876
## Team.fctr.num 1.499552e-01 0.058154820 0.10100550 0.17491216 0.20008998
## X3P           1.212155e-01 0.233353555 0.56282942 0.56785931 0.48994891
## STL           1.280813e-01 0.331127910 0.34348011 0.44313827 0.43099026
## ORB           9.359240e-02 0.269179339 0.55187982 0.40676555 0.49692073
## X2PA          8.672031e-02 0.165315490 0.75654063 0.67770751 0.70836140
## FGA           7.072567e-02 0.050945458 0.83087856 0.62911736 0.79397947
## X2P           7.235925e-02 0.098690188 0.76984080 0.77711327 0.82572457
## SeasonEnd     4.173684e-21 0.267127100 0.63244845 0.67212300 0.63952773
##                     TOV          FT        BLK          FG        FTA
## PTS.diff      0.2476858 0.215445119 0.18515068 0.196690814 0.17119872
## DRB           0.1909569 0.003604883 0.24299082 0.008966352 0.04354267
## oppPTS        0.5824127 0.550587572 0.03076664 0.804467370 0.53721485
## AST           0.4305329 0.447208477 0.20308000 0.812266519 0.42797223
## PTS           0.4271383 0.697483850 0.15205537 0.941954732 0.65505876
## TOV           0.0000000 0.438684976 0.24156872 0.517630444 0.52566979
## FT            0.4386850 0.000000000 0.16465598 0.538342589 0.95049325
## BLK           0.2415687 0.164655982 0.00000000 0.177502748 0.21637171
## FG            0.5176304 0.538342589 0.17750275 0.000000000 0.52121683
## FTA           0.5256698 0.950493246 0.21637171 0.521216830 0.00000000
## Team.fctr.num 0.1404593 0.154385609 0.07311722 0.198789524 0.13968994
## X3P           0.6801733 0.508712984 0.23107383 0.668272893 0.53389685
## STL           0.4640752 0.323156975 0.12045594 0.469730269 0.37005522
## ORB           0.5459866 0.393398702 0.20110479 0.598591143 0.47455838
## X2PA          0.6448524 0.515802012 0.20610678 0.859224632 0.52375675
## FGA           0.4499814 0.391691425 0.12109667 0.879662910 0.38212215
## X2P           0.6377157 0.574293954 0.21771153 0.942919965 0.57454277
## SeasonEnd     0.7238690 0.513699180 0.20416249 0.761598813 0.54689652
##               Team.fctr.num       X3P       STL       ORB       X2PA
## PTS.diff         0.14995517 0.1212155 0.1280813 0.0935924 0.08672031
## DRB              0.05815482 0.2333536 0.3311279 0.2691793 0.16531549
## oppPTS           0.10100550 0.5628294 0.3434801 0.5518798 0.75654063
## AST              0.17491216 0.5678593 0.4431383 0.4067656 0.67770751
## PTS              0.20008998 0.4899489 0.4309903 0.4969207 0.70836140
## TOV              0.14045929 0.6801733 0.4640752 0.5459866 0.64485242
## FT               0.15438561 0.5087130 0.3231570 0.3933987 0.51580201
## BLK              0.07311722 0.2310738 0.1204559 0.2011048 0.20610678
## FG               0.19878952 0.6682729 0.4697303 0.5985911 0.85922463
## FTA              0.13968994 0.5338969 0.3700552 0.4745584 0.52375675
## Team.fctr.num    0.00000000 0.1423720 0.1531193 0.2009572 0.17580617
## X3P              0.14237198 0.0000000 0.4168543 0.6651682 0.92073200
## STL              0.15311931 0.4168543 0.0000000 0.4929415 0.49008620
## ORB              0.20095724 0.6651682 0.4929415 0.0000000 0.76561469
## X2PA             0.17580617 0.9207320 0.4900862 0.7656147 0.00000000
## FGA              0.17904220 0.6075645 0.4805329 0.7375094 0.86485931
## X2P              0.19170714 0.8778664 0.4890027 0.6831180 0.96530915
## SeasonEnd        0.21455487 0.8381421 0.5020598 0.7020128 0.86725681
##                      FGA        X2P    SeasonEnd
## PTS.diff      0.07072567 0.07235925 4.173684e-21
## DRB           0.05094546 0.09869019 2.671271e-01
## oppPTS        0.83087856 0.76984080 6.324485e-01
## AST           0.62911736 0.77711327 6.721230e-01
## PTS           0.79397947 0.82572457 6.395277e-01
## TOV           0.44998136 0.63771567 7.238690e-01
## FT            0.39169143 0.57429395 5.136992e-01
## BLK           0.12109667 0.21771153 2.041625e-01
## FG            0.87966291 0.94291996 7.615988e-01
## FTA           0.38212215 0.57454277 5.468965e-01
## Team.fctr.num 0.17904220 0.19170714 2.145549e-01
## X3P           0.60756448 0.87786641 8.381421e-01
## STL           0.48053292 0.48900271 5.020598e-01
## ORB           0.73750939 0.68311802 7.020128e-01
## X2PA          0.86485931 0.96530915 8.672568e-01
## FGA           0.00000000 0.83827486 6.849048e-01
## X2P           0.83827486 0.00000000 8.654893e-01
## SeasonEnd     0.68490478 0.86548930 0.000000e+00
## [1] "cor(X2PA, X2P)=0.9653"
```

![](NBA_Wins_files/figure-html/remove_correlated_features-3.png) 

```
## [1] "cor(W, X2PA)=-0.0870"
## [1] "cor(W, X2P)=0.0693"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in mydelete_cor_features(): Dropping X2P as a feature
```

![](NBA_Wins_files/figure-html/remove_correlated_features-4.png) 

```
##                          id       cor.y  cor.y.abs
## PTS.diff           PTS.diff  0.97074326 0.97074326
## DRB                     DRB  0.47089750 0.47089750
## oppPTS               oppPTS -0.33157294 0.33157294
## AST                     AST  0.32005177 0.32005177
## PTS                     PTS  0.29882561 0.29882561
## TOV                     TOV -0.24318588 0.24318588
## FT                       FT  0.20490600 0.20490600
## BLK                     BLK  0.20392100 0.20392100
## FG                       FG  0.19039642 0.19039642
## FTA                     FTA  0.16188731 0.16188731
## Team.fctr.num Team.fctr.num -0.15119487 0.15119487
## X3P                     X3P  0.11904456 0.11904456
## STL                     STL  0.11619440 0.11619440
## ORB                     ORB -0.09573676 0.09573676
## X2PA                   X2PA -0.08703653 0.08703653
## FGA                     FGA -0.07144566 0.07144566
## SeasonEnd         SeasonEnd  0.00000000 0.00000000
##                    PTS.diff          DRB      oppPTS         AST
## PTS.diff       1.000000e+00  0.467373785 -0.34003607  0.32340381
## DRB            4.673738e-01  1.000000000 -0.21262703  0.05647709
## oppPTS        -3.400361e-01 -0.212627029  1.00000000  0.54256181
## AST            3.234038e-01  0.056477086  0.54256181  1.00000000
## PTS            3.093789e-01  0.090291366  0.78907471  0.75988915
## TOV           -2.476858e-01 -0.190956902  0.58241272  0.43053288
## FT             2.154451e-01  0.003604883  0.55058757  0.44720848
## BLK            1.851507e-01  0.242990816  0.03076664  0.20308000
## FG             1.966908e-01  0.008966352  0.80446737  0.81226652
## FTA            1.711987e-01 -0.043542668  0.53721485  0.42797223
## Team.fctr.num -1.499552e-01 -0.058154820 -0.10100550 -0.17491216
## X3P            1.212155e-01  0.233353555 -0.56282942 -0.56785931
## STL            1.280813e-01 -0.331127910  0.34348011  0.44313827
## ORB           -9.359240e-02 -0.269179339  0.55187982  0.40676555
## X2PA          -8.672031e-02 -0.165315490  0.75654063  0.67770751
## FGA           -7.072567e-02 -0.050945458  0.83087856  0.62911736
## SeasonEnd      4.173684e-21  0.267127100 -0.63244845 -0.67212300
##                       PTS        TOV           FT         BLK           FG
## PTS.diff       0.30937888 -0.2476858  0.215445119  0.18515068  0.196690814
## DRB            0.09029137 -0.1909569  0.003604883  0.24299082  0.008966352
## oppPTS         0.78907471  0.5824127  0.550587572  0.03076664  0.804467370
## AST            0.75988915  0.4305329  0.447208477  0.20308000  0.812266519
## PTS            1.00000000  0.4271383  0.697483850  0.15205537  0.941954732
## TOV            0.42713832  1.0000000  0.438684976  0.24156872  0.517630444
## FT             0.69748385  0.4386850  1.000000000  0.16465598  0.538342589
## BLK            0.15205537  0.2415687  0.164655982  1.00000000  0.177502748
## FG             0.94195473  0.5176304  0.538342589  0.17750275  1.000000000
## FTA            0.65505876  0.5256698  0.950493246  0.21637171  0.521216830
## Team.fctr.num -0.20008998 -0.1404593 -0.154385609 -0.07311722 -0.198789524
## X3P           -0.48994891 -0.6801733 -0.508712984 -0.23107383 -0.668272893
## STL            0.43099026  0.4640752  0.323156975  0.12045594  0.469730269
## ORB            0.49692073  0.5459866  0.393398702  0.20110479  0.598591143
## X2PA           0.70836140  0.6448524  0.515802012  0.20610678  0.859224632
## FGA            0.79397947  0.4499814  0.391691425  0.12109667  0.879662910
## SeasonEnd     -0.63952773 -0.7238690 -0.513699180 -0.20416249 -0.761598813
##                       FTA Team.fctr.num        X3P        STL        ORB
## PTS.diff       0.17119872   -0.14995517  0.1212155  0.1280813 -0.0935924
## DRB           -0.04354267   -0.05815482  0.2333536 -0.3311279 -0.2691793
## oppPTS         0.53721485   -0.10100550 -0.5628294  0.3434801  0.5518798
## AST            0.42797223   -0.17491216 -0.5678593  0.4431383  0.4067656
## PTS            0.65505876   -0.20008998 -0.4899489  0.4309903  0.4969207
## TOV            0.52566979   -0.14045929 -0.6801733  0.4640752  0.5459866
## FT             0.95049325   -0.15438561 -0.5087130  0.3231570  0.3933987
## BLK            0.21637171   -0.07311722 -0.2310738  0.1204559  0.2011048
## FG             0.52121683   -0.19878952 -0.6682729  0.4697303  0.5985911
## FTA            1.00000000   -0.13968994 -0.5338969  0.3700552  0.4745584
## Team.fctr.num -0.13968994    1.00000000  0.1423720 -0.1531193 -0.2009572
## X3P           -0.53389685    0.14237198  1.0000000 -0.4168543 -0.6651682
## STL            0.37005522   -0.15311931 -0.4168543  1.0000000  0.4929415
## ORB            0.47455838   -0.20095724 -0.6651682  0.4929415  1.0000000
## X2PA           0.52375675   -0.17580617 -0.9207320  0.4900862  0.7656147
## FGA            0.38212215   -0.17904220 -0.6075645  0.4805329  0.7375094
## SeasonEnd     -0.54689652    0.21455487  0.8381421 -0.5020598 -0.7020128
##                      X2PA         FGA     SeasonEnd
## PTS.diff      -0.08672031 -0.07072567  4.173684e-21
## DRB           -0.16531549 -0.05094546  2.671271e-01
## oppPTS         0.75654063  0.83087856 -6.324485e-01
## AST            0.67770751  0.62911736 -6.721230e-01
## PTS            0.70836140  0.79397947 -6.395277e-01
## TOV            0.64485242  0.44998136 -7.238690e-01
## FT             0.51580201  0.39169143 -5.136992e-01
## BLK            0.20610678  0.12109667 -2.041625e-01
## FG             0.85922463  0.87966291 -7.615988e-01
## FTA            0.52375675  0.38212215 -5.468965e-01
## Team.fctr.num -0.17580617 -0.17904220  2.145549e-01
## X3P           -0.92073200 -0.60756448  8.381421e-01
## STL            0.49008620  0.48053292 -5.020598e-01
## ORB            0.76561469  0.73750939 -7.020128e-01
## X2PA           1.00000000  0.86485931 -8.672568e-01
## FGA            0.86485931  1.00000000 -6.849048e-01
## SeasonEnd     -0.86725681 -0.68490478  1.000000e+00
##                   PTS.diff         DRB     oppPTS        AST        PTS
## PTS.diff      0.000000e+00 0.467373785 0.34003607 0.32340381 0.30937888
## DRB           4.673738e-01 0.000000000 0.21262703 0.05647709 0.09029137
## oppPTS        3.400361e-01 0.212627029 0.00000000 0.54256181 0.78907471
## AST           3.234038e-01 0.056477086 0.54256181 0.00000000 0.75988915
## PTS           3.093789e-01 0.090291366 0.78907471 0.75988915 0.00000000
## TOV           2.476858e-01 0.190956902 0.58241272 0.43053288 0.42713832
## FT            2.154451e-01 0.003604883 0.55058757 0.44720848 0.69748385
## BLK           1.851507e-01 0.242990816 0.03076664 0.20308000 0.15205537
## FG            1.966908e-01 0.008966352 0.80446737 0.81226652 0.94195473
## FTA           1.711987e-01 0.043542668 0.53721485 0.42797223 0.65505876
## Team.fctr.num 1.499552e-01 0.058154820 0.10100550 0.17491216 0.20008998
## X3P           1.212155e-01 0.233353555 0.56282942 0.56785931 0.48994891
## STL           1.280813e-01 0.331127910 0.34348011 0.44313827 0.43099026
## ORB           9.359240e-02 0.269179339 0.55187982 0.40676555 0.49692073
## X2PA          8.672031e-02 0.165315490 0.75654063 0.67770751 0.70836140
## FGA           7.072567e-02 0.050945458 0.83087856 0.62911736 0.79397947
## SeasonEnd     4.173684e-21 0.267127100 0.63244845 0.67212300 0.63952773
##                     TOV          FT        BLK          FG        FTA
## PTS.diff      0.2476858 0.215445119 0.18515068 0.196690814 0.17119872
## DRB           0.1909569 0.003604883 0.24299082 0.008966352 0.04354267
## oppPTS        0.5824127 0.550587572 0.03076664 0.804467370 0.53721485
## AST           0.4305329 0.447208477 0.20308000 0.812266519 0.42797223
## PTS           0.4271383 0.697483850 0.15205537 0.941954732 0.65505876
## TOV           0.0000000 0.438684976 0.24156872 0.517630444 0.52566979
## FT            0.4386850 0.000000000 0.16465598 0.538342589 0.95049325
## BLK           0.2415687 0.164655982 0.00000000 0.177502748 0.21637171
## FG            0.5176304 0.538342589 0.17750275 0.000000000 0.52121683
## FTA           0.5256698 0.950493246 0.21637171 0.521216830 0.00000000
## Team.fctr.num 0.1404593 0.154385609 0.07311722 0.198789524 0.13968994
## X3P           0.6801733 0.508712984 0.23107383 0.668272893 0.53389685
## STL           0.4640752 0.323156975 0.12045594 0.469730269 0.37005522
## ORB           0.5459866 0.393398702 0.20110479 0.598591143 0.47455838
## X2PA          0.6448524 0.515802012 0.20610678 0.859224632 0.52375675
## FGA           0.4499814 0.391691425 0.12109667 0.879662910 0.38212215
## SeasonEnd     0.7238690 0.513699180 0.20416249 0.761598813 0.54689652
##               Team.fctr.num       X3P       STL       ORB       X2PA
## PTS.diff         0.14995517 0.1212155 0.1280813 0.0935924 0.08672031
## DRB              0.05815482 0.2333536 0.3311279 0.2691793 0.16531549
## oppPTS           0.10100550 0.5628294 0.3434801 0.5518798 0.75654063
## AST              0.17491216 0.5678593 0.4431383 0.4067656 0.67770751
## PTS              0.20008998 0.4899489 0.4309903 0.4969207 0.70836140
## TOV              0.14045929 0.6801733 0.4640752 0.5459866 0.64485242
## FT               0.15438561 0.5087130 0.3231570 0.3933987 0.51580201
## BLK              0.07311722 0.2310738 0.1204559 0.2011048 0.20610678
## FG               0.19878952 0.6682729 0.4697303 0.5985911 0.85922463
## FTA              0.13968994 0.5338969 0.3700552 0.4745584 0.52375675
## Team.fctr.num    0.00000000 0.1423720 0.1531193 0.2009572 0.17580617
## X3P              0.14237198 0.0000000 0.4168543 0.6651682 0.92073200
## STL              0.15311931 0.4168543 0.0000000 0.4929415 0.49008620
## ORB              0.20095724 0.6651682 0.4929415 0.0000000 0.76561469
## X2PA             0.17580617 0.9207320 0.4900862 0.7656147 0.00000000
## FGA              0.17904220 0.6075645 0.4805329 0.7375094 0.86485931
## SeasonEnd        0.21455487 0.8381421 0.5020598 0.7020128 0.86725681
##                      FGA    SeasonEnd
## PTS.diff      0.07072567 4.173684e-21
## DRB           0.05094546 2.671271e-01
## oppPTS        0.83087856 6.324485e-01
## AST           0.62911736 6.721230e-01
## PTS           0.79397947 6.395277e-01
## TOV           0.44998136 7.238690e-01
## FT            0.39169143 5.136992e-01
## BLK           0.12109667 2.041625e-01
## FG            0.87966291 7.615988e-01
## FTA           0.38212215 5.468965e-01
## Team.fctr.num 0.17904220 2.145549e-01
## X3P           0.60756448 8.381421e-01
## STL           0.48053292 5.020598e-01
## ORB           0.73750939 7.020128e-01
## X2PA          0.86485931 8.672568e-01
## FGA           0.00000000 6.849048e-01
## SeasonEnd     0.68490478 0.000000e+00
## [1] "cor(FT, FTA)=0.9505"
```

![](NBA_Wins_files/figure-html/remove_correlated_features-5.png) 

```
## [1] "cor(W, FT)=0.2049"
## [1] "cor(W, FTA)=0.1619"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in mydelete_cor_features(): Dropping FTA as a feature
```

![](NBA_Wins_files/figure-html/remove_correlated_features-6.png) 

```
##                          id       cor.y  cor.y.abs
## PTS.diff           PTS.diff  0.97074326 0.97074326
## DRB                     DRB  0.47089750 0.47089750
## oppPTS               oppPTS -0.33157294 0.33157294
## AST                     AST  0.32005177 0.32005177
## PTS                     PTS  0.29882561 0.29882561
## TOV                     TOV -0.24318588 0.24318588
## FT                       FT  0.20490600 0.20490600
## BLK                     BLK  0.20392100 0.20392100
## FG                       FG  0.19039642 0.19039642
## Team.fctr.num Team.fctr.num -0.15119487 0.15119487
## X3P                     X3P  0.11904456 0.11904456
## STL                     STL  0.11619440 0.11619440
## ORB                     ORB -0.09573676 0.09573676
## X2PA                   X2PA -0.08703653 0.08703653
## FGA                     FGA -0.07144566 0.07144566
## SeasonEnd         SeasonEnd  0.00000000 0.00000000
##                    PTS.diff          DRB      oppPTS         AST
## PTS.diff       1.000000e+00  0.467373785 -0.34003607  0.32340381
## DRB            4.673738e-01  1.000000000 -0.21262703  0.05647709
## oppPTS        -3.400361e-01 -0.212627029  1.00000000  0.54256181
## AST            3.234038e-01  0.056477086  0.54256181  1.00000000
## PTS            3.093789e-01  0.090291366  0.78907471  0.75988915
## TOV           -2.476858e-01 -0.190956902  0.58241272  0.43053288
## FT             2.154451e-01  0.003604883  0.55058757  0.44720848
## BLK            1.851507e-01  0.242990816  0.03076664  0.20308000
## FG             1.966908e-01  0.008966352  0.80446737  0.81226652
## Team.fctr.num -1.499552e-01 -0.058154820 -0.10100550 -0.17491216
## X3P            1.212155e-01  0.233353555 -0.56282942 -0.56785931
## STL            1.280813e-01 -0.331127910  0.34348011  0.44313827
## ORB           -9.359240e-02 -0.269179339  0.55187982  0.40676555
## X2PA          -8.672031e-02 -0.165315490  0.75654063  0.67770751
## FGA           -7.072567e-02 -0.050945458  0.83087856  0.62911736
## SeasonEnd      4.173684e-21  0.267127100 -0.63244845 -0.67212300
##                       PTS        TOV           FT         BLK           FG
## PTS.diff       0.30937888 -0.2476858  0.215445119  0.18515068  0.196690814
## DRB            0.09029137 -0.1909569  0.003604883  0.24299082  0.008966352
## oppPTS         0.78907471  0.5824127  0.550587572  0.03076664  0.804467370
## AST            0.75988915  0.4305329  0.447208477  0.20308000  0.812266519
## PTS            1.00000000  0.4271383  0.697483850  0.15205537  0.941954732
## TOV            0.42713832  1.0000000  0.438684976  0.24156872  0.517630444
## FT             0.69748385  0.4386850  1.000000000  0.16465598  0.538342589
## BLK            0.15205537  0.2415687  0.164655982  1.00000000  0.177502748
## FG             0.94195473  0.5176304  0.538342589  0.17750275  1.000000000
## Team.fctr.num -0.20008998 -0.1404593 -0.154385609 -0.07311722 -0.198789524
## X3P           -0.48994891 -0.6801733 -0.508712984 -0.23107383 -0.668272893
## STL            0.43099026  0.4640752  0.323156975  0.12045594  0.469730269
## ORB            0.49692073  0.5459866  0.393398702  0.20110479  0.598591143
## X2PA           0.70836140  0.6448524  0.515802012  0.20610678  0.859224632
## FGA            0.79397947  0.4499814  0.391691425  0.12109667  0.879662910
## SeasonEnd     -0.63952773 -0.7238690 -0.513699180 -0.20416249 -0.761598813
##               Team.fctr.num        X3P        STL        ORB        X2PA
## PTS.diff        -0.14995517  0.1212155  0.1280813 -0.0935924 -0.08672031
## DRB             -0.05815482  0.2333536 -0.3311279 -0.2691793 -0.16531549
## oppPTS          -0.10100550 -0.5628294  0.3434801  0.5518798  0.75654063
## AST             -0.17491216 -0.5678593  0.4431383  0.4067656  0.67770751
## PTS             -0.20008998 -0.4899489  0.4309903  0.4969207  0.70836140
## TOV             -0.14045929 -0.6801733  0.4640752  0.5459866  0.64485242
## FT              -0.15438561 -0.5087130  0.3231570  0.3933987  0.51580201
## BLK             -0.07311722 -0.2310738  0.1204559  0.2011048  0.20610678
## FG              -0.19878952 -0.6682729  0.4697303  0.5985911  0.85922463
## Team.fctr.num    1.00000000  0.1423720 -0.1531193 -0.2009572 -0.17580617
## X3P              0.14237198  1.0000000 -0.4168543 -0.6651682 -0.92073200
## STL             -0.15311931 -0.4168543  1.0000000  0.4929415  0.49008620
## ORB             -0.20095724 -0.6651682  0.4929415  1.0000000  0.76561469
## X2PA            -0.17580617 -0.9207320  0.4900862  0.7656147  1.00000000
## FGA             -0.17904220 -0.6075645  0.4805329  0.7375094  0.86485931
## SeasonEnd        0.21455487  0.8381421 -0.5020598 -0.7020128 -0.86725681
##                       FGA     SeasonEnd
## PTS.diff      -0.07072567  4.173684e-21
## DRB           -0.05094546  2.671271e-01
## oppPTS         0.83087856 -6.324485e-01
## AST            0.62911736 -6.721230e-01
## PTS            0.79397947 -6.395277e-01
## TOV            0.44998136 -7.238690e-01
## FT             0.39169143 -5.136992e-01
## BLK            0.12109667 -2.041625e-01
## FG             0.87966291 -7.615988e-01
## Team.fctr.num -0.17904220  2.145549e-01
## X3P           -0.60756448  8.381421e-01
## STL            0.48053292 -5.020598e-01
## ORB            0.73750939 -7.020128e-01
## X2PA           0.86485931 -8.672568e-01
## FGA            1.00000000 -6.849048e-01
## SeasonEnd     -0.68490478  1.000000e+00
##                   PTS.diff         DRB     oppPTS        AST        PTS
## PTS.diff      0.000000e+00 0.467373785 0.34003607 0.32340381 0.30937888
## DRB           4.673738e-01 0.000000000 0.21262703 0.05647709 0.09029137
## oppPTS        3.400361e-01 0.212627029 0.00000000 0.54256181 0.78907471
## AST           3.234038e-01 0.056477086 0.54256181 0.00000000 0.75988915
## PTS           3.093789e-01 0.090291366 0.78907471 0.75988915 0.00000000
## TOV           2.476858e-01 0.190956902 0.58241272 0.43053288 0.42713832
## FT            2.154451e-01 0.003604883 0.55058757 0.44720848 0.69748385
## BLK           1.851507e-01 0.242990816 0.03076664 0.20308000 0.15205537
## FG            1.966908e-01 0.008966352 0.80446737 0.81226652 0.94195473
## Team.fctr.num 1.499552e-01 0.058154820 0.10100550 0.17491216 0.20008998
## X3P           1.212155e-01 0.233353555 0.56282942 0.56785931 0.48994891
## STL           1.280813e-01 0.331127910 0.34348011 0.44313827 0.43099026
## ORB           9.359240e-02 0.269179339 0.55187982 0.40676555 0.49692073
## X2PA          8.672031e-02 0.165315490 0.75654063 0.67770751 0.70836140
## FGA           7.072567e-02 0.050945458 0.83087856 0.62911736 0.79397947
## SeasonEnd     4.173684e-21 0.267127100 0.63244845 0.67212300 0.63952773
##                     TOV          FT        BLK          FG Team.fctr.num
## PTS.diff      0.2476858 0.215445119 0.18515068 0.196690814    0.14995517
## DRB           0.1909569 0.003604883 0.24299082 0.008966352    0.05815482
## oppPTS        0.5824127 0.550587572 0.03076664 0.804467370    0.10100550
## AST           0.4305329 0.447208477 0.20308000 0.812266519    0.17491216
## PTS           0.4271383 0.697483850 0.15205537 0.941954732    0.20008998
## TOV           0.0000000 0.438684976 0.24156872 0.517630444    0.14045929
## FT            0.4386850 0.000000000 0.16465598 0.538342589    0.15438561
## BLK           0.2415687 0.164655982 0.00000000 0.177502748    0.07311722
## FG            0.5176304 0.538342589 0.17750275 0.000000000    0.19878952
## Team.fctr.num 0.1404593 0.154385609 0.07311722 0.198789524    0.00000000
## X3P           0.6801733 0.508712984 0.23107383 0.668272893    0.14237198
## STL           0.4640752 0.323156975 0.12045594 0.469730269    0.15311931
## ORB           0.5459866 0.393398702 0.20110479 0.598591143    0.20095724
## X2PA          0.6448524 0.515802012 0.20610678 0.859224632    0.17580617
## FGA           0.4499814 0.391691425 0.12109667 0.879662910    0.17904220
## SeasonEnd     0.7238690 0.513699180 0.20416249 0.761598813    0.21455487
##                     X3P       STL       ORB       X2PA        FGA
## PTS.diff      0.1212155 0.1280813 0.0935924 0.08672031 0.07072567
## DRB           0.2333536 0.3311279 0.2691793 0.16531549 0.05094546
## oppPTS        0.5628294 0.3434801 0.5518798 0.75654063 0.83087856
## AST           0.5678593 0.4431383 0.4067656 0.67770751 0.62911736
## PTS           0.4899489 0.4309903 0.4969207 0.70836140 0.79397947
## TOV           0.6801733 0.4640752 0.5459866 0.64485242 0.44998136
## FT            0.5087130 0.3231570 0.3933987 0.51580201 0.39169143
## BLK           0.2310738 0.1204559 0.2011048 0.20610678 0.12109667
## FG            0.6682729 0.4697303 0.5985911 0.85922463 0.87966291
## Team.fctr.num 0.1423720 0.1531193 0.2009572 0.17580617 0.17904220
## X3P           0.0000000 0.4168543 0.6651682 0.92073200 0.60756448
## STL           0.4168543 0.0000000 0.4929415 0.49008620 0.48053292
## ORB           0.6651682 0.4929415 0.0000000 0.76561469 0.73750939
## X2PA          0.9207320 0.4900862 0.7656147 0.00000000 0.86485931
## FGA           0.6075645 0.4805329 0.7375094 0.86485931 0.00000000
## SeasonEnd     0.8381421 0.5020598 0.7020128 0.86725681 0.68490478
##                  SeasonEnd
## PTS.diff      4.173684e-21
## DRB           2.671271e-01
## oppPTS        6.324485e-01
## AST           6.721230e-01
## PTS           6.395277e-01
## TOV           7.238690e-01
## FT            5.136992e-01
## BLK           2.041625e-01
## FG            7.615988e-01
## Team.fctr.num 2.145549e-01
## X3P           8.381421e-01
## STL           5.020598e-01
## ORB           7.020128e-01
## X2PA          8.672568e-01
## FGA           6.849048e-01
## SeasonEnd     0.000000e+00
## [1] "cor(PTS, FG)=0.9420"
```

![](NBA_Wins_files/figure-html/remove_correlated_features-7.png) 

```
## [1] "cor(W, PTS)=0.2988"
## [1] "cor(W, FG)=0.1904"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in mydelete_cor_features(): Dropping FG as a feature
```

![](NBA_Wins_files/figure-html/remove_correlated_features-8.png) 

```
##                          id       cor.y  cor.y.abs
## PTS.diff           PTS.diff  0.97074326 0.97074326
## DRB                     DRB  0.47089750 0.47089750
## oppPTS               oppPTS -0.33157294 0.33157294
## AST                     AST  0.32005177 0.32005177
## PTS                     PTS  0.29882561 0.29882561
## TOV                     TOV -0.24318588 0.24318588
## FT                       FT  0.20490600 0.20490600
## BLK                     BLK  0.20392100 0.20392100
## Team.fctr.num Team.fctr.num -0.15119487 0.15119487
## X3P                     X3P  0.11904456 0.11904456
## STL                     STL  0.11619440 0.11619440
## ORB                     ORB -0.09573676 0.09573676
## X2PA                   X2PA -0.08703653 0.08703653
## FGA                     FGA -0.07144566 0.07144566
## SeasonEnd         SeasonEnd  0.00000000 0.00000000
##                    PTS.diff          DRB      oppPTS         AST
## PTS.diff       1.000000e+00  0.467373785 -0.34003607  0.32340381
## DRB            4.673738e-01  1.000000000 -0.21262703  0.05647709
## oppPTS        -3.400361e-01 -0.212627029  1.00000000  0.54256181
## AST            3.234038e-01  0.056477086  0.54256181  1.00000000
## PTS            3.093789e-01  0.090291366  0.78907471  0.75988915
## TOV           -2.476858e-01 -0.190956902  0.58241272  0.43053288
## FT             2.154451e-01  0.003604883  0.55058757  0.44720848
## BLK            1.851507e-01  0.242990816  0.03076664  0.20308000
## Team.fctr.num -1.499552e-01 -0.058154820 -0.10100550 -0.17491216
## X3P            1.212155e-01  0.233353555 -0.56282942 -0.56785931
## STL            1.280813e-01 -0.331127910  0.34348011  0.44313827
## ORB           -9.359240e-02 -0.269179339  0.55187982  0.40676555
## X2PA          -8.672031e-02 -0.165315490  0.75654063  0.67770751
## FGA           -7.072567e-02 -0.050945458  0.83087856  0.62911736
## SeasonEnd      4.173684e-21  0.267127100 -0.63244845 -0.67212300
##                       PTS        TOV           FT         BLK
## PTS.diff       0.30937888 -0.2476858  0.215445119  0.18515068
## DRB            0.09029137 -0.1909569  0.003604883  0.24299082
## oppPTS         0.78907471  0.5824127  0.550587572  0.03076664
## AST            0.75988915  0.4305329  0.447208477  0.20308000
## PTS            1.00000000  0.4271383  0.697483850  0.15205537
## TOV            0.42713832  1.0000000  0.438684976  0.24156872
## FT             0.69748385  0.4386850  1.000000000  0.16465598
## BLK            0.15205537  0.2415687  0.164655982  1.00000000
## Team.fctr.num -0.20008998 -0.1404593 -0.154385609 -0.07311722
## X3P           -0.48994891 -0.6801733 -0.508712984 -0.23107383
## STL            0.43099026  0.4640752  0.323156975  0.12045594
## ORB            0.49692073  0.5459866  0.393398702  0.20110479
## X2PA           0.70836140  0.6448524  0.515802012  0.20610678
## FGA            0.79397947  0.4499814  0.391691425  0.12109667
## SeasonEnd     -0.63952773 -0.7238690 -0.513699180 -0.20416249
##               Team.fctr.num        X3P        STL        ORB        X2PA
## PTS.diff        -0.14995517  0.1212155  0.1280813 -0.0935924 -0.08672031
## DRB             -0.05815482  0.2333536 -0.3311279 -0.2691793 -0.16531549
## oppPTS          -0.10100550 -0.5628294  0.3434801  0.5518798  0.75654063
## AST             -0.17491216 -0.5678593  0.4431383  0.4067656  0.67770751
## PTS             -0.20008998 -0.4899489  0.4309903  0.4969207  0.70836140
## TOV             -0.14045929 -0.6801733  0.4640752  0.5459866  0.64485242
## FT              -0.15438561 -0.5087130  0.3231570  0.3933987  0.51580201
## BLK             -0.07311722 -0.2310738  0.1204559  0.2011048  0.20610678
## Team.fctr.num    1.00000000  0.1423720 -0.1531193 -0.2009572 -0.17580617
## X3P              0.14237198  1.0000000 -0.4168543 -0.6651682 -0.92073200
## STL             -0.15311931 -0.4168543  1.0000000  0.4929415  0.49008620
## ORB             -0.20095724 -0.6651682  0.4929415  1.0000000  0.76561469
## X2PA            -0.17580617 -0.9207320  0.4900862  0.7656147  1.00000000
## FGA             -0.17904220 -0.6075645  0.4805329  0.7375094  0.86485931
## SeasonEnd        0.21455487  0.8381421 -0.5020598 -0.7020128 -0.86725681
##                       FGA     SeasonEnd
## PTS.diff      -0.07072567  4.173684e-21
## DRB           -0.05094546  2.671271e-01
## oppPTS         0.83087856 -6.324485e-01
## AST            0.62911736 -6.721230e-01
## PTS            0.79397947 -6.395277e-01
## TOV            0.44998136 -7.238690e-01
## FT             0.39169143 -5.136992e-01
## BLK            0.12109667 -2.041625e-01
## Team.fctr.num -0.17904220  2.145549e-01
## X3P           -0.60756448  8.381421e-01
## STL            0.48053292 -5.020598e-01
## ORB            0.73750939 -7.020128e-01
## X2PA           0.86485931 -8.672568e-01
## FGA            1.00000000 -6.849048e-01
## SeasonEnd     -0.68490478  1.000000e+00
##                   PTS.diff         DRB     oppPTS        AST        PTS
## PTS.diff      0.000000e+00 0.467373785 0.34003607 0.32340381 0.30937888
## DRB           4.673738e-01 0.000000000 0.21262703 0.05647709 0.09029137
## oppPTS        3.400361e-01 0.212627029 0.00000000 0.54256181 0.78907471
## AST           3.234038e-01 0.056477086 0.54256181 0.00000000 0.75988915
## PTS           3.093789e-01 0.090291366 0.78907471 0.75988915 0.00000000
## TOV           2.476858e-01 0.190956902 0.58241272 0.43053288 0.42713832
## FT            2.154451e-01 0.003604883 0.55058757 0.44720848 0.69748385
## BLK           1.851507e-01 0.242990816 0.03076664 0.20308000 0.15205537
## Team.fctr.num 1.499552e-01 0.058154820 0.10100550 0.17491216 0.20008998
## X3P           1.212155e-01 0.233353555 0.56282942 0.56785931 0.48994891
## STL           1.280813e-01 0.331127910 0.34348011 0.44313827 0.43099026
## ORB           9.359240e-02 0.269179339 0.55187982 0.40676555 0.49692073
## X2PA          8.672031e-02 0.165315490 0.75654063 0.67770751 0.70836140
## FGA           7.072567e-02 0.050945458 0.83087856 0.62911736 0.79397947
## SeasonEnd     4.173684e-21 0.267127100 0.63244845 0.67212300 0.63952773
##                     TOV          FT        BLK Team.fctr.num       X3P
## PTS.diff      0.2476858 0.215445119 0.18515068    0.14995517 0.1212155
## DRB           0.1909569 0.003604883 0.24299082    0.05815482 0.2333536
## oppPTS        0.5824127 0.550587572 0.03076664    0.10100550 0.5628294
## AST           0.4305329 0.447208477 0.20308000    0.17491216 0.5678593
## PTS           0.4271383 0.697483850 0.15205537    0.20008998 0.4899489
## TOV           0.0000000 0.438684976 0.24156872    0.14045929 0.6801733
## FT            0.4386850 0.000000000 0.16465598    0.15438561 0.5087130
## BLK           0.2415687 0.164655982 0.00000000    0.07311722 0.2310738
## Team.fctr.num 0.1404593 0.154385609 0.07311722    0.00000000 0.1423720
## X3P           0.6801733 0.508712984 0.23107383    0.14237198 0.0000000
## STL           0.4640752 0.323156975 0.12045594    0.15311931 0.4168543
## ORB           0.5459866 0.393398702 0.20110479    0.20095724 0.6651682
## X2PA          0.6448524 0.515802012 0.20610678    0.17580617 0.9207320
## FGA           0.4499814 0.391691425 0.12109667    0.17904220 0.6075645
## SeasonEnd     0.7238690 0.513699180 0.20416249    0.21455487 0.8381421
##                     STL       ORB       X2PA        FGA    SeasonEnd
## PTS.diff      0.1280813 0.0935924 0.08672031 0.07072567 4.173684e-21
## DRB           0.3311279 0.2691793 0.16531549 0.05094546 2.671271e-01
## oppPTS        0.3434801 0.5518798 0.75654063 0.83087856 6.324485e-01
## AST           0.4431383 0.4067656 0.67770751 0.62911736 6.721230e-01
## PTS           0.4309903 0.4969207 0.70836140 0.79397947 6.395277e-01
## TOV           0.4640752 0.5459866 0.64485242 0.44998136 7.238690e-01
## FT            0.3231570 0.3933987 0.51580201 0.39169143 5.136992e-01
## BLK           0.1204559 0.2011048 0.20610678 0.12109667 2.041625e-01
## Team.fctr.num 0.1531193 0.2009572 0.17580617 0.17904220 2.145549e-01
## X3P           0.4168543 0.6651682 0.92073200 0.60756448 8.381421e-01
## STL           0.0000000 0.4929415 0.49008620 0.48053292 5.020598e-01
## ORB           0.4929415 0.0000000 0.76561469 0.73750939 7.020128e-01
## X2PA          0.4900862 0.7656147 0.00000000 0.86485931 8.672568e-01
## FGA           0.4805329 0.7375094 0.86485931 0.00000000 6.849048e-01
## SeasonEnd     0.5020598 0.7020128 0.86725681 0.68490478 0.000000e+00
## [1] "cor(X3P, X2PA)=-0.9207"
```

![](NBA_Wins_files/figure-html/remove_correlated_features-9.png) 

```
## [1] "cor(W, X3P)=0.1190"
## [1] "cor(W, X2PA)=-0.0870"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in mydelete_cor_features(): Dropping X2PA as a feature
```

![](NBA_Wins_files/figure-html/remove_correlated_features-10.png) 

```
##                          id       cor.y  cor.y.abs
## PTS.diff           PTS.diff  0.97074326 0.97074326
## DRB                     DRB  0.47089750 0.47089750
## oppPTS               oppPTS -0.33157294 0.33157294
## AST                     AST  0.32005177 0.32005177
## PTS                     PTS  0.29882561 0.29882561
## TOV                     TOV -0.24318588 0.24318588
## FT                       FT  0.20490600 0.20490600
## BLK                     BLK  0.20392100 0.20392100
## Team.fctr.num Team.fctr.num -0.15119487 0.15119487
## X3P                     X3P  0.11904456 0.11904456
## STL                     STL  0.11619440 0.11619440
## ORB                     ORB -0.09573676 0.09573676
## FGA                     FGA -0.07144566 0.07144566
## SeasonEnd         SeasonEnd  0.00000000 0.00000000
##                    PTS.diff          DRB      oppPTS         AST
## PTS.diff       1.000000e+00  0.467373785 -0.34003607  0.32340381
## DRB            4.673738e-01  1.000000000 -0.21262703  0.05647709
## oppPTS        -3.400361e-01 -0.212627029  1.00000000  0.54256181
## AST            3.234038e-01  0.056477086  0.54256181  1.00000000
## PTS            3.093789e-01  0.090291366  0.78907471  0.75988915
## TOV           -2.476858e-01 -0.190956902  0.58241272  0.43053288
## FT             2.154451e-01  0.003604883  0.55058757  0.44720848
## BLK            1.851507e-01  0.242990816  0.03076664  0.20308000
## Team.fctr.num -1.499552e-01 -0.058154820 -0.10100550 -0.17491216
## X3P            1.212155e-01  0.233353555 -0.56282942 -0.56785931
## STL            1.280813e-01 -0.331127910  0.34348011  0.44313827
## ORB           -9.359240e-02 -0.269179339  0.55187982  0.40676555
## FGA           -7.072567e-02 -0.050945458  0.83087856  0.62911736
## SeasonEnd      4.173684e-21  0.267127100 -0.63244845 -0.67212300
##                       PTS        TOV           FT         BLK
## PTS.diff       0.30937888 -0.2476858  0.215445119  0.18515068
## DRB            0.09029137 -0.1909569  0.003604883  0.24299082
## oppPTS         0.78907471  0.5824127  0.550587572  0.03076664
## AST            0.75988915  0.4305329  0.447208477  0.20308000
## PTS            1.00000000  0.4271383  0.697483850  0.15205537
## TOV            0.42713832  1.0000000  0.438684976  0.24156872
## FT             0.69748385  0.4386850  1.000000000  0.16465598
## BLK            0.15205537  0.2415687  0.164655982  1.00000000
## Team.fctr.num -0.20008998 -0.1404593 -0.154385609 -0.07311722
## X3P           -0.48994891 -0.6801733 -0.508712984 -0.23107383
## STL            0.43099026  0.4640752  0.323156975  0.12045594
## ORB            0.49692073  0.5459866  0.393398702  0.20110479
## FGA            0.79397947  0.4499814  0.391691425  0.12109667
## SeasonEnd     -0.63952773 -0.7238690 -0.513699180 -0.20416249
##               Team.fctr.num        X3P        STL        ORB         FGA
## PTS.diff        -0.14995517  0.1212155  0.1280813 -0.0935924 -0.07072567
## DRB             -0.05815482  0.2333536 -0.3311279 -0.2691793 -0.05094546
## oppPTS          -0.10100550 -0.5628294  0.3434801  0.5518798  0.83087856
## AST             -0.17491216 -0.5678593  0.4431383  0.4067656  0.62911736
## PTS             -0.20008998 -0.4899489  0.4309903  0.4969207  0.79397947
## TOV             -0.14045929 -0.6801733  0.4640752  0.5459866  0.44998136
## FT              -0.15438561 -0.5087130  0.3231570  0.3933987  0.39169143
## BLK             -0.07311722 -0.2310738  0.1204559  0.2011048  0.12109667
## Team.fctr.num    1.00000000  0.1423720 -0.1531193 -0.2009572 -0.17904220
## X3P              0.14237198  1.0000000 -0.4168543 -0.6651682 -0.60756448
## STL             -0.15311931 -0.4168543  1.0000000  0.4929415  0.48053292
## ORB             -0.20095724 -0.6651682  0.4929415  1.0000000  0.73750939
## FGA             -0.17904220 -0.6075645  0.4805329  0.7375094  1.00000000
## SeasonEnd        0.21455487  0.8381421 -0.5020598 -0.7020128 -0.68490478
##                   SeasonEnd
## PTS.diff       4.173684e-21
## DRB            2.671271e-01
## oppPTS        -6.324485e-01
## AST           -6.721230e-01
## PTS           -6.395277e-01
## TOV           -7.238690e-01
## FT            -5.136992e-01
## BLK           -2.041625e-01
## Team.fctr.num  2.145549e-01
## X3P            8.381421e-01
## STL           -5.020598e-01
## ORB           -7.020128e-01
## FGA           -6.849048e-01
## SeasonEnd      1.000000e+00
##                   PTS.diff         DRB     oppPTS        AST        PTS
## PTS.diff      0.000000e+00 0.467373785 0.34003607 0.32340381 0.30937888
## DRB           4.673738e-01 0.000000000 0.21262703 0.05647709 0.09029137
## oppPTS        3.400361e-01 0.212627029 0.00000000 0.54256181 0.78907471
## AST           3.234038e-01 0.056477086 0.54256181 0.00000000 0.75988915
## PTS           3.093789e-01 0.090291366 0.78907471 0.75988915 0.00000000
## TOV           2.476858e-01 0.190956902 0.58241272 0.43053288 0.42713832
## FT            2.154451e-01 0.003604883 0.55058757 0.44720848 0.69748385
## BLK           1.851507e-01 0.242990816 0.03076664 0.20308000 0.15205537
## Team.fctr.num 1.499552e-01 0.058154820 0.10100550 0.17491216 0.20008998
## X3P           1.212155e-01 0.233353555 0.56282942 0.56785931 0.48994891
## STL           1.280813e-01 0.331127910 0.34348011 0.44313827 0.43099026
## ORB           9.359240e-02 0.269179339 0.55187982 0.40676555 0.49692073
## FGA           7.072567e-02 0.050945458 0.83087856 0.62911736 0.79397947
## SeasonEnd     4.173684e-21 0.267127100 0.63244845 0.67212300 0.63952773
##                     TOV          FT        BLK Team.fctr.num       X3P
## PTS.diff      0.2476858 0.215445119 0.18515068    0.14995517 0.1212155
## DRB           0.1909569 0.003604883 0.24299082    0.05815482 0.2333536
## oppPTS        0.5824127 0.550587572 0.03076664    0.10100550 0.5628294
## AST           0.4305329 0.447208477 0.20308000    0.17491216 0.5678593
## PTS           0.4271383 0.697483850 0.15205537    0.20008998 0.4899489
## TOV           0.0000000 0.438684976 0.24156872    0.14045929 0.6801733
## FT            0.4386850 0.000000000 0.16465598    0.15438561 0.5087130
## BLK           0.2415687 0.164655982 0.00000000    0.07311722 0.2310738
## Team.fctr.num 0.1404593 0.154385609 0.07311722    0.00000000 0.1423720
## X3P           0.6801733 0.508712984 0.23107383    0.14237198 0.0000000
## STL           0.4640752 0.323156975 0.12045594    0.15311931 0.4168543
## ORB           0.5459866 0.393398702 0.20110479    0.20095724 0.6651682
## FGA           0.4499814 0.391691425 0.12109667    0.17904220 0.6075645
## SeasonEnd     0.7238690 0.513699180 0.20416249    0.21455487 0.8381421
##                     STL       ORB        FGA    SeasonEnd
## PTS.diff      0.1280813 0.0935924 0.07072567 4.173684e-21
## DRB           0.3311279 0.2691793 0.05094546 2.671271e-01
## oppPTS        0.3434801 0.5518798 0.83087856 6.324485e-01
## AST           0.4431383 0.4067656 0.62911736 6.721230e-01
## PTS           0.4309903 0.4969207 0.79397947 6.395277e-01
## TOV           0.4640752 0.5459866 0.44998136 7.238690e-01
## FT            0.3231570 0.3933987 0.39169143 5.136992e-01
## BLK           0.1204559 0.2011048 0.12109667 2.041625e-01
## Team.fctr.num 0.1531193 0.2009572 0.17904220 2.145549e-01
## X3P           0.4168543 0.6651682 0.60756448 8.381421e-01
## STL           0.0000000 0.4929415 0.48053292 5.020598e-01
## ORB           0.4929415 0.0000000 0.73750939 7.020128e-01
## FGA           0.4805329 0.7375094 0.00000000 6.849048e-01
## SeasonEnd     0.5020598 0.7020128 0.68490478 0.000000e+00
## [1] "cor(X3P, SeasonEnd)=0.8381"
```

![](NBA_Wins_files/figure-html/remove_correlated_features-11.png) 

```
## [1] "cor(W, X3P)=0.1190"
## [1] "cor(W, SeasonEnd)=0.0000"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in mydelete_cor_features(): Dropping SeasonEnd as a feature
```

![](NBA_Wins_files/figure-html/remove_correlated_features-12.png) 

```
##                          id       cor.y  cor.y.abs
## PTS.diff           PTS.diff  0.97074326 0.97074326
## DRB                     DRB  0.47089750 0.47089750
## oppPTS               oppPTS -0.33157294 0.33157294
## AST                     AST  0.32005177 0.32005177
## PTS                     PTS  0.29882561 0.29882561
## TOV                     TOV -0.24318588 0.24318588
## FT                       FT  0.20490600 0.20490600
## BLK                     BLK  0.20392100 0.20392100
## Team.fctr.num Team.fctr.num -0.15119487 0.15119487
## X3P                     X3P  0.11904456 0.11904456
## STL                     STL  0.11619440 0.11619440
## ORB                     ORB -0.09573676 0.09573676
## FGA                     FGA -0.07144566 0.07144566
##                  PTS.diff          DRB      oppPTS         AST         PTS
## PTS.diff       1.00000000  0.467373785 -0.34003607  0.32340381  0.30937888
## DRB            0.46737378  1.000000000 -0.21262703  0.05647709  0.09029137
## oppPTS        -0.34003607 -0.212627029  1.00000000  0.54256181  0.78907471
## AST            0.32340381  0.056477086  0.54256181  1.00000000  0.75988915
## PTS            0.30937888  0.090291366  0.78907471  0.75988915  1.00000000
## TOV           -0.24768583 -0.190956902  0.58241272  0.43053288  0.42713832
## FT             0.21544512  0.003604883  0.55058757  0.44720848  0.69748385
## BLK            0.18515068  0.242990816  0.03076664  0.20308000  0.15205537
## Team.fctr.num -0.14995517 -0.058154820 -0.10100550 -0.17491216 -0.20008998
## X3P            0.12121548  0.233353555 -0.56282942 -0.56785931 -0.48994891
## STL            0.12808131 -0.331127910  0.34348011  0.44313827  0.43099026
## ORB           -0.09359240 -0.269179339  0.55187982  0.40676555  0.49692073
## FGA           -0.07072567 -0.050945458  0.83087856  0.62911736  0.79397947
##                      TOV           FT         BLK Team.fctr.num        X3P
## PTS.diff      -0.2476858  0.215445119  0.18515068   -0.14995517  0.1212155
## DRB           -0.1909569  0.003604883  0.24299082   -0.05815482  0.2333536
## oppPTS         0.5824127  0.550587572  0.03076664   -0.10100550 -0.5628294
## AST            0.4305329  0.447208477  0.20308000   -0.17491216 -0.5678593
## PTS            0.4271383  0.697483850  0.15205537   -0.20008998 -0.4899489
## TOV            1.0000000  0.438684976  0.24156872   -0.14045929 -0.6801733
## FT             0.4386850  1.000000000  0.16465598   -0.15438561 -0.5087130
## BLK            0.2415687  0.164655982  1.00000000   -0.07311722 -0.2310738
## Team.fctr.num -0.1404593 -0.154385609 -0.07311722    1.00000000  0.1423720
## X3P           -0.6801733 -0.508712984 -0.23107383    0.14237198  1.0000000
## STL            0.4640752  0.323156975  0.12045594   -0.15311931 -0.4168543
## ORB            0.5459866  0.393398702  0.20110479   -0.20095724 -0.6651682
## FGA            0.4499814  0.391691425  0.12109667   -0.17904220 -0.6075645
##                      STL        ORB         FGA
## PTS.diff       0.1280813 -0.0935924 -0.07072567
## DRB           -0.3311279 -0.2691793 -0.05094546
## oppPTS         0.3434801  0.5518798  0.83087856
## AST            0.4431383  0.4067656  0.62911736
## PTS            0.4309903  0.4969207  0.79397947
## TOV            0.4640752  0.5459866  0.44998136
## FT             0.3231570  0.3933987  0.39169143
## BLK            0.1204559  0.2011048  0.12109667
## Team.fctr.num -0.1531193 -0.2009572 -0.17904220
## X3P           -0.4168543 -0.6651682 -0.60756448
## STL            1.0000000  0.4929415  0.48053292
## ORB            0.4929415  1.0000000  0.73750939
## FGA            0.4805329  0.7375094  1.00000000
##                 PTS.diff         DRB     oppPTS        AST        PTS
## PTS.diff      0.00000000 0.467373785 0.34003607 0.32340381 0.30937888
## DRB           0.46737378 0.000000000 0.21262703 0.05647709 0.09029137
## oppPTS        0.34003607 0.212627029 0.00000000 0.54256181 0.78907471
## AST           0.32340381 0.056477086 0.54256181 0.00000000 0.75988915
## PTS           0.30937888 0.090291366 0.78907471 0.75988915 0.00000000
## TOV           0.24768583 0.190956902 0.58241272 0.43053288 0.42713832
## FT            0.21544512 0.003604883 0.55058757 0.44720848 0.69748385
## BLK           0.18515068 0.242990816 0.03076664 0.20308000 0.15205537
## Team.fctr.num 0.14995517 0.058154820 0.10100550 0.17491216 0.20008998
## X3P           0.12121548 0.233353555 0.56282942 0.56785931 0.48994891
## STL           0.12808131 0.331127910 0.34348011 0.44313827 0.43099026
## ORB           0.09359240 0.269179339 0.55187982 0.40676555 0.49692073
## FGA           0.07072567 0.050945458 0.83087856 0.62911736 0.79397947
##                     TOV          FT        BLK Team.fctr.num       X3P
## PTS.diff      0.2476858 0.215445119 0.18515068    0.14995517 0.1212155
## DRB           0.1909569 0.003604883 0.24299082    0.05815482 0.2333536
## oppPTS        0.5824127 0.550587572 0.03076664    0.10100550 0.5628294
## AST           0.4305329 0.447208477 0.20308000    0.17491216 0.5678593
## PTS           0.4271383 0.697483850 0.15205537    0.20008998 0.4899489
## TOV           0.0000000 0.438684976 0.24156872    0.14045929 0.6801733
## FT            0.4386850 0.000000000 0.16465598    0.15438561 0.5087130
## BLK           0.2415687 0.164655982 0.00000000    0.07311722 0.2310738
## Team.fctr.num 0.1404593 0.154385609 0.07311722    0.00000000 0.1423720
## X3P           0.6801733 0.508712984 0.23107383    0.14237198 0.0000000
## STL           0.4640752 0.323156975 0.12045594    0.15311931 0.4168543
## ORB           0.5459866 0.393398702 0.20110479    0.20095724 0.6651682
## FGA           0.4499814 0.391691425 0.12109667    0.17904220 0.6075645
##                     STL       ORB        FGA
## PTS.diff      0.1280813 0.0935924 0.07072567
## DRB           0.3311279 0.2691793 0.05094546
## oppPTS        0.3434801 0.5518798 0.83087856
## AST           0.4431383 0.4067656 0.62911736
## PTS           0.4309903 0.4969207 0.79397947
## TOV           0.4640752 0.5459866 0.44998136
## FT            0.3231570 0.3933987 0.39169143
## BLK           0.1204559 0.2011048 0.12109667
## Team.fctr.num 0.1531193 0.2009572 0.17904220
## X3P           0.4168543 0.6651682 0.60756448
## STL           0.0000000 0.4929415 0.48053292
## ORB           0.4929415 0.0000000 0.73750939
## FGA           0.4805329 0.7375094 0.00000000
## [1] "cor(oppPTS, FGA)=0.8309"
```

![](NBA_Wins_files/figure-html/remove_correlated_features-13.png) 

```
## [1] "cor(W, oppPTS)=-0.3316"
## [1] "cor(W, FGA)=-0.0714"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in mydelete_cor_features(): Dropping FGA as a feature
```

![](NBA_Wins_files/figure-html/remove_correlated_features-14.png) 

```
##                          id       cor.y  cor.y.abs
## PTS.diff           PTS.diff  0.97074326 0.97074326
## DRB                     DRB  0.47089750 0.47089750
## oppPTS               oppPTS -0.33157294 0.33157294
## AST                     AST  0.32005177 0.32005177
## PTS                     PTS  0.29882561 0.29882561
## TOV                     TOV -0.24318588 0.24318588
## FT                       FT  0.20490600 0.20490600
## BLK                     BLK  0.20392100 0.20392100
## Team.fctr.num Team.fctr.num -0.15119487 0.15119487
## X3P                     X3P  0.11904456 0.11904456
## STL                     STL  0.11619440 0.11619440
## ORB                     ORB -0.09573676 0.09573676
##                 PTS.diff          DRB      oppPTS         AST         PTS
## PTS.diff       1.0000000  0.467373785 -0.34003607  0.32340381  0.30937888
## DRB            0.4673738  1.000000000 -0.21262703  0.05647709  0.09029137
## oppPTS        -0.3400361 -0.212627029  1.00000000  0.54256181  0.78907471
## AST            0.3234038  0.056477086  0.54256181  1.00000000  0.75988915
## PTS            0.3093789  0.090291366  0.78907471  0.75988915  1.00000000
## TOV           -0.2476858 -0.190956902  0.58241272  0.43053288  0.42713832
## FT             0.2154451  0.003604883  0.55058757  0.44720848  0.69748385
## BLK            0.1851507  0.242990816  0.03076664  0.20308000  0.15205537
## Team.fctr.num -0.1499552 -0.058154820 -0.10100550 -0.17491216 -0.20008998
## X3P            0.1212155  0.233353555 -0.56282942 -0.56785931 -0.48994891
## STL            0.1280813 -0.331127910  0.34348011  0.44313827  0.43099026
## ORB           -0.0935924 -0.269179339  0.55187982  0.40676555  0.49692073
##                      TOV           FT         BLK Team.fctr.num        X3P
## PTS.diff      -0.2476858  0.215445119  0.18515068   -0.14995517  0.1212155
## DRB           -0.1909569  0.003604883  0.24299082   -0.05815482  0.2333536
## oppPTS         0.5824127  0.550587572  0.03076664   -0.10100550 -0.5628294
## AST            0.4305329  0.447208477  0.20308000   -0.17491216 -0.5678593
## PTS            0.4271383  0.697483850  0.15205537   -0.20008998 -0.4899489
## TOV            1.0000000  0.438684976  0.24156872   -0.14045929 -0.6801733
## FT             0.4386850  1.000000000  0.16465598   -0.15438561 -0.5087130
## BLK            0.2415687  0.164655982  1.00000000   -0.07311722 -0.2310738
## Team.fctr.num -0.1404593 -0.154385609 -0.07311722    1.00000000  0.1423720
## X3P           -0.6801733 -0.508712984 -0.23107383    0.14237198  1.0000000
## STL            0.4640752  0.323156975  0.12045594   -0.15311931 -0.4168543
## ORB            0.5459866  0.393398702  0.20110479   -0.20095724 -0.6651682
##                      STL        ORB
## PTS.diff       0.1280813 -0.0935924
## DRB           -0.3311279 -0.2691793
## oppPTS         0.3434801  0.5518798
## AST            0.4431383  0.4067656
## PTS            0.4309903  0.4969207
## TOV            0.4640752  0.5459866
## FT             0.3231570  0.3933987
## BLK            0.1204559  0.2011048
## Team.fctr.num -0.1531193 -0.2009572
## X3P           -0.4168543 -0.6651682
## STL            1.0000000  0.4929415
## ORB            0.4929415  1.0000000
##                PTS.diff         DRB     oppPTS        AST        PTS
## PTS.diff      0.0000000 0.467373785 0.34003607 0.32340381 0.30937888
## DRB           0.4673738 0.000000000 0.21262703 0.05647709 0.09029137
## oppPTS        0.3400361 0.212627029 0.00000000 0.54256181 0.78907471
## AST           0.3234038 0.056477086 0.54256181 0.00000000 0.75988915
## PTS           0.3093789 0.090291366 0.78907471 0.75988915 0.00000000
## TOV           0.2476858 0.190956902 0.58241272 0.43053288 0.42713832
## FT            0.2154451 0.003604883 0.55058757 0.44720848 0.69748385
## BLK           0.1851507 0.242990816 0.03076664 0.20308000 0.15205537
## Team.fctr.num 0.1499552 0.058154820 0.10100550 0.17491216 0.20008998
## X3P           0.1212155 0.233353555 0.56282942 0.56785931 0.48994891
## STL           0.1280813 0.331127910 0.34348011 0.44313827 0.43099026
## ORB           0.0935924 0.269179339 0.55187982 0.40676555 0.49692073
##                     TOV          FT        BLK Team.fctr.num       X3P
## PTS.diff      0.2476858 0.215445119 0.18515068    0.14995517 0.1212155
## DRB           0.1909569 0.003604883 0.24299082    0.05815482 0.2333536
## oppPTS        0.5824127 0.550587572 0.03076664    0.10100550 0.5628294
## AST           0.4305329 0.447208477 0.20308000    0.17491216 0.5678593
## PTS           0.4271383 0.697483850 0.15205537    0.20008998 0.4899489
## TOV           0.0000000 0.438684976 0.24156872    0.14045929 0.6801733
## FT            0.4386850 0.000000000 0.16465598    0.15438561 0.5087130
## BLK           0.2415687 0.164655982 0.00000000    0.07311722 0.2310738
## Team.fctr.num 0.1404593 0.154385609 0.07311722    0.00000000 0.1423720
## X3P           0.6801733 0.508712984 0.23107383    0.14237198 0.0000000
## STL           0.4640752 0.323156975 0.12045594    0.15311931 0.4168543
## ORB           0.5459866 0.393398702 0.20110479    0.20095724 0.6651682
##                     STL       ORB
## PTS.diff      0.1280813 0.0935924
## DRB           0.3311279 0.2691793
## oppPTS        0.3434801 0.5518798
## AST           0.4431383 0.4067656
## PTS           0.4309903 0.4969207
## TOV           0.4640752 0.5459866
## FT            0.3231570 0.3933987
## BLK           0.1204559 0.2011048
## Team.fctr.num 0.1531193 0.2009572
## X3P           0.4168543 0.6651682
## STL           0.0000000 0.4929415
## ORB           0.4929415 0.0000000
## [1] "cor(oppPTS, PTS)=0.7891"
```

![](NBA_Wins_files/figure-html/remove_correlated_features-15.png) 

```
## [1] "cor(W, oppPTS)=-0.3316"
## [1] "cor(W, PTS)=0.2988"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in mydelete_cor_features(): Dropping PTS as a feature
```

![](NBA_Wins_files/figure-html/remove_correlated_features-16.png) 

```
##                          id       cor.y  cor.y.abs
## PTS.diff           PTS.diff  0.97074326 0.97074326
## DRB                     DRB  0.47089750 0.47089750
## oppPTS               oppPTS -0.33157294 0.33157294
## AST                     AST  0.32005177 0.32005177
## TOV                     TOV -0.24318588 0.24318588
## FT                       FT  0.20490600 0.20490600
## BLK                     BLK  0.20392100 0.20392100
## Team.fctr.num Team.fctr.num -0.15119487 0.15119487
## X3P                     X3P  0.11904456 0.11904456
## STL                     STL  0.11619440 0.11619440
## ORB                     ORB -0.09573676 0.09573676
##                 PTS.diff          DRB      oppPTS         AST        TOV
## PTS.diff       1.0000000  0.467373785 -0.34003607  0.32340381 -0.2476858
## DRB            0.4673738  1.000000000 -0.21262703  0.05647709 -0.1909569
## oppPTS        -0.3400361 -0.212627029  1.00000000  0.54256181  0.5824127
## AST            0.3234038  0.056477086  0.54256181  1.00000000  0.4305329
## TOV           -0.2476858 -0.190956902  0.58241272  0.43053288  1.0000000
## FT             0.2154451  0.003604883  0.55058757  0.44720848  0.4386850
## BLK            0.1851507  0.242990816  0.03076664  0.20308000  0.2415687
## Team.fctr.num -0.1499552 -0.058154820 -0.10100550 -0.17491216 -0.1404593
## X3P            0.1212155  0.233353555 -0.56282942 -0.56785931 -0.6801733
## STL            0.1280813 -0.331127910  0.34348011  0.44313827  0.4640752
## ORB           -0.0935924 -0.269179339  0.55187982  0.40676555  0.5459866
##                         FT         BLK Team.fctr.num        X3P        STL
## PTS.diff       0.215445119  0.18515068   -0.14995517  0.1212155  0.1280813
## DRB            0.003604883  0.24299082   -0.05815482  0.2333536 -0.3311279
## oppPTS         0.550587572  0.03076664   -0.10100550 -0.5628294  0.3434801
## AST            0.447208477  0.20308000   -0.17491216 -0.5678593  0.4431383
## TOV            0.438684976  0.24156872   -0.14045929 -0.6801733  0.4640752
## FT             1.000000000  0.16465598   -0.15438561 -0.5087130  0.3231570
## BLK            0.164655982  1.00000000   -0.07311722 -0.2310738  0.1204559
## Team.fctr.num -0.154385609 -0.07311722    1.00000000  0.1423720 -0.1531193
## X3P           -0.508712984 -0.23107383    0.14237198  1.0000000 -0.4168543
## STL            0.323156975  0.12045594   -0.15311931 -0.4168543  1.0000000
## ORB            0.393398702  0.20110479   -0.20095724 -0.6651682  0.4929415
##                      ORB
## PTS.diff      -0.0935924
## DRB           -0.2691793
## oppPTS         0.5518798
## AST            0.4067656
## TOV            0.5459866
## FT             0.3933987
## BLK            0.2011048
## Team.fctr.num -0.2009572
## X3P           -0.6651682
## STL            0.4929415
## ORB            1.0000000
##                PTS.diff         DRB     oppPTS        AST       TOV
## PTS.diff      0.0000000 0.467373785 0.34003607 0.32340381 0.2476858
## DRB           0.4673738 0.000000000 0.21262703 0.05647709 0.1909569
## oppPTS        0.3400361 0.212627029 0.00000000 0.54256181 0.5824127
## AST           0.3234038 0.056477086 0.54256181 0.00000000 0.4305329
## TOV           0.2476858 0.190956902 0.58241272 0.43053288 0.0000000
## FT            0.2154451 0.003604883 0.55058757 0.44720848 0.4386850
## BLK           0.1851507 0.242990816 0.03076664 0.20308000 0.2415687
## Team.fctr.num 0.1499552 0.058154820 0.10100550 0.17491216 0.1404593
## X3P           0.1212155 0.233353555 0.56282942 0.56785931 0.6801733
## STL           0.1280813 0.331127910 0.34348011 0.44313827 0.4640752
## ORB           0.0935924 0.269179339 0.55187982 0.40676555 0.5459866
##                        FT        BLK Team.fctr.num       X3P       STL
## PTS.diff      0.215445119 0.18515068    0.14995517 0.1212155 0.1280813
## DRB           0.003604883 0.24299082    0.05815482 0.2333536 0.3311279
## oppPTS        0.550587572 0.03076664    0.10100550 0.5628294 0.3434801
## AST           0.447208477 0.20308000    0.17491216 0.5678593 0.4431383
## TOV           0.438684976 0.24156872    0.14045929 0.6801733 0.4640752
## FT            0.000000000 0.16465598    0.15438561 0.5087130 0.3231570
## BLK           0.164655982 0.00000000    0.07311722 0.2310738 0.1204559
## Team.fctr.num 0.154385609 0.07311722    0.00000000 0.1423720 0.1531193
## X3P           0.508712984 0.23107383    0.14237198 0.0000000 0.4168543
## STL           0.323156975 0.12045594    0.15311931 0.4168543 0.0000000
## ORB           0.393398702 0.20110479    0.20095724 0.6651682 0.4929415
##                     ORB
## PTS.diff      0.0935924
## DRB           0.2691793
## oppPTS        0.5518798
## AST           0.4067656
## TOV           0.5459866
## FT            0.3933987
## BLK           0.2011048
## Team.fctr.num 0.2009572
## X3P           0.6651682
## STL           0.4929415
## ORB           0.0000000
##               id       cor.y  cor.y.abs cor.low
## 11      PTS.diff  0.97074326 0.97074326       1
## 3            DRB  0.47089750 0.47089750       1
## 1            AST  0.32005177 0.32005177       1
## 10           PTS  0.29882561 0.29882561      NA
## 6             FT  0.20490600 0.20490600       1
## 2            BLK  0.20392100 0.20392100       1
## 4             FG  0.19039642 0.19039642      NA
## 7            FTA  0.16188731 0.16188731      NA
## 18           X3P  0.11904456 0.11904456       1
## 13           STL  0.11619440 0.11619440       1
## 19          X3PA  0.08328625 0.08328625      NA
## 16           X2P  0.06927898 0.06927898      NA
## 12     SeasonEnd  0.00000000 0.00000000      NA
## 5            FGA -0.07144566 0.07144566      NA
## 17          X2PA -0.08703653 0.08703653      NA
## 9            ORB -0.09573676 0.09573676       1
## 14 Team.fctr.num -0.15119487 0.15119487       1
## 15           TOV -0.24318588 0.24318588       1
## 8         oppPTS -0.33157294 0.33157294       1
```

```r
script_df <- rbind(script_df, 
                   data.frame(chunk_label="run_models", 
                              chunk_step_major=max(script_df$chunk_step_major)+1, 
                              chunk_step_minor=0))
print(script_df)
```

```
##                  chunk_label chunk_step_major chunk_step_minor
## 1                import_data                1                0
## 2               cleanse_data                2                0
## 3       inspect_explore_data                2                1
## 4        manage_missing_data                2                2
## 5         encode_retype_data                2                3
## 6           extract_features                3                0
## 7            select_features                4                0
## 8 remove_correlated_features                4                1
## 9                 run_models                5                0
```

## Step `5`: run models

```r
max_cor_y_x_var <- subset(glb_feats_df, cor.low == 1)[1, "id"]

#   Regression:
if (glb_is_regression) {
    #   Linear:
    myrun_mdl_fn <- myrun_mdl_lm
}    

#   Classification:
if (glb_is_classification) {
    #   Logit Regression:
    myrun_mdl_fn <- myrun_mdl_glm
}    
    
# Highest cor.y
ret_lst <- myrun_mdl_fn(indep_vars_vctr=max_cor_y_x_var,
                        fit_df=glb_entity_df, OOB_df=glb_predct_df)
```

```
## [1] 269.2104
## [1] 0.941742
## 
## Call:
## lm(formula = reformulate(indep_vars_vctr, response = glb_predct_var), 
##     data = fit_df)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -9.7393 -2.1018 -0.0672  2.0265 10.6026 
## 
## Coefficients:
##              Estimate Std. Error t value Pr(>|t|)    
## (Intercept) 4.100e+01  1.059e-01   387.0   <2e-16 ***
## PTS.diff    3.259e-02  2.793e-04   116.7   <2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3.061 on 833 degrees of freedom
## Multiple R-squared:  0.9423,	Adjusted R-squared:  0.9423 
## F-statistic: 1.361e+04 on 1 and 833 DF,  p-value: < 2.2e-16
## 
##      feats n.fit  R.sq.fit R.sq.OOB Adj.R.sq.fit SSE.fit  SSE.OOB
## 1 PTS.diff   835 0.9423425 0.941742    0.9423425 7805.79 269.2104
##   f.score.OOB
## 1          NA
```

```r
glb_sel_mdl <- glb_mdl

# Enhance Highest cor.y model with additions of interaction terms that were 
#   dropped due to high correlations
ret_lst <- myrun_mdl_fn(indep_vars_vctr=c(max_cor_y_x_var, 
    paste(max_cor_y_x_var, subset(glb_feats_df, is.na(cor.low))[, "id"], sep=":")),
                        fit_df=glb_entity_df, OOB_df=glb_predct_df)    
```

```
## Warning in predict.lm(mdl, newdata = OOB_df): prediction from a
## rank-deficient fit may be misleading
```

```
## [1] 264.38
## [1] 0.9427873
## 
## Call:
## lm(formula = reformulate(indep_vars_vctr, response = glb_predct_var), 
##     data = fit_df)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -9.7026 -2.0681 -0.1593  2.0596  8.8202 
## 
## Coefficients: (1 not defined because of singularities)
##                      Estimate Std. Error t value Pr(>|t|)    
## (Intercept)         4.097e+01  1.210e-01 338.577   <2e-16 ***
## PTS.diff            1.558e-01  1.356e-01   1.149   0.2509    
## PTS.diff:PTS        6.358e-06  4.707e-06   1.351   0.1771    
## PTS.diff:FG        -2.724e-05  2.084e-05  -1.307   0.1916    
## PTS.diff:FTA       -8.192e-06  3.819e-06  -2.145   0.0323 *  
## PTS.diff:X3PA       4.052e-06  6.008e-06   0.674   0.5003    
## PTS.diff:X2P        1.727e-05  1.600e-05   1.079   0.2807    
## PTS.diff:SeasonEnd -5.108e-05  6.671e-05  -0.766   0.4441    
## PTS.diff:FGA       -3.310e-06  1.548e-06  -2.139   0.0328 *  
## PTS.diff:X2PA              NA         NA      NA       NA    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3.051 on 826 degrees of freedom
## Multiple R-squared:  0.9432,	Adjusted R-squared:  0.9426 
## F-statistic:  1714 on 8 and 826 DF,  p-value: < 2.2e-16
## 
##                                                                                                                             feats
## 2 PTS.diff, PTS.diff:PTS, PTS.diff:FG, PTS.diff:FTA, PTS.diff:X3PA, PTS.diff:X2P, PTS.diff:SeasonEnd, PTS.diff:FGA, PTS.diff:X2PA
## 1                                                                                                                        PTS.diff
##   n.fit  R.sq.fit  R.sq.OOB Adj.R.sq.fit  SSE.fit  SSE.OOB f.score.OOB
## 2   835 0.9431903 0.9427873    0.9431903 7691.006 264.3800          NA
## 1   835 0.9423425 0.9417420    0.9423425 7805.790 269.2104          NA
```

```r
# Low correlated X
ret_lst <- myrun_mdl_fn(indep_vars_vctr=subset(glb_feats_df, cor.low == 1)[, "id"],
                        fit_df=glb_entity_df, OOB_df=glb_predct_df)
```

```
## [1] 268.036
## [1] 0.9419961
## 
## Call:
## lm(formula = reformulate(indep_vars_vctr, response = glb_predct_var), 
##     data = fit_df)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -8.7586 -2.0744 -0.1452  1.9465 10.7659 
## 
## Coefficients:
##                 Estimate Std. Error t value Pr(>|t|)    
## (Intercept)    3.675e+01  3.488e+00  10.535   <2e-16 ***
## PTS.diff       3.165e-02  5.766e-04  54.886   <2e-16 ***
## DRB            1.501e-03  1.107e-03   1.356    0.176    
## AST            1.294e-03  8.984e-04   1.441    0.150    
## FT             9.243e-05  8.519e-04   0.109    0.914    
## BLK            3.968e-03  1.440e-03   2.755    0.006 ** 
## X3P            2.729e-04  9.581e-04   0.285    0.776    
## STL           -1.379e-04  1.584e-03  -0.087    0.931    
## ORB           -3.223e-04  1.078e-03  -0.299    0.765    
## Team.fctr.num -9.916e-03  1.182e-02  -0.839    0.402    
## TOV           -1.087e-03  1.099e-03  -0.988    0.323    
## oppPTS        -2.110e-04  3.889e-04  -0.543    0.588    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3.049 on 823 degrees of freedom
## Multiple R-squared:  0.9435,	Adjusted R-squared:  0.9427 
## F-statistic:  1249 on 11 and 823 DF,  p-value: < 2.2e-16
## 
##                                                                                                                             feats
## 2 PTS.diff, PTS.diff:PTS, PTS.diff:FG, PTS.diff:FTA, PTS.diff:X3PA, PTS.diff:X2P, PTS.diff:SeasonEnd, PTS.diff:FGA, PTS.diff:X2PA
## 3                                                          PTS.diff, DRB, AST, FT, BLK, X3P, STL, ORB, Team.fctr.num, TOV, oppPTS
## 1                                                                                                                        PTS.diff
##   n.fit  R.sq.fit  R.sq.OOB Adj.R.sq.fit  SSE.fit  SSE.OOB f.score.OOB
## 2   835 0.9431903 0.9427873    0.9431903 7691.006 264.3800          NA
## 3   835 0.9434823 0.9419961    0.9434823 7651.473 268.0360          NA
## 1   835 0.9423425 0.9417420    0.9423425 7805.790 269.2104          NA
```

```r
# All X that is not user excluded
ret_lst <- myrun_mdl_fn(indep_vars_vctr=setdiff(setdiff(names(glb_entity_df),
                                                        glb_predct_var),
                                                glb_exclude_vars_as_features),
                        fit_df=glb_entity_df, OOB_df=glb_predct_df)
```

```
## Warning in predict.lm(mdl, newdata = OOB_df): prediction from a
## rank-deficient fit may be misleading
```

```
## [1] 279.1548
## [1] 0.93959
## 
## Call:
## lm(formula = reformulate(indep_vars_vctr, response = glb_predct_var), 
##     data = fit_df)
## 
## Residuals:
##    Min     1Q Median     3Q    Max 
## -8.681 -1.993 -0.199  1.981 10.734 
## 
## Coefficients: (5 not defined because of singularities)
##                 Estimate Std. Error t value Pr(>|t|)    
## (Intercept)    1.646e+02  6.289e+01   2.617  0.00903 ** 
## SeasonEnd     -6.195e-02  3.106e-02  -1.994  0.04644 *  
## PTS            3.157e-02  2.018e-03  15.642  < 2e-16 ***
## oppPTS        -3.071e-02  7.134e-04 -43.045  < 2e-16 ***
## FG            -7.369e-03  8.594e-03  -0.858  0.39140    
## FGA            6.575e-04  2.615e-03   0.251  0.80152    
## X2P            8.587e-03  6.449e-03   1.332  0.18336    
## X2PA          -3.607e-03  2.367e-03  -1.524  0.12788    
## X3P                   NA         NA      NA       NA    
## X3PA                  NA         NA      NA       NA    
## FT                    NA         NA      NA       NA    
## FTA           -1.058e-03  1.706e-03  -0.620  0.53551    
## ORB            1.845e-03  1.742e-03   1.059  0.28986    
## DRB            3.815e-03  1.544e-03   2.472  0.01365 *  
## AST            8.025e-04  9.195e-04   0.873  0.38301    
## STL            2.060e-03  1.990e-03   1.035  0.30106    
## BLK            3.779e-03  1.456e-03   2.594  0.00964 ** 
## TOV           -3.582e-03  1.571e-03  -2.281  0.02281 *  
## Team.fctr.num -9.220e-03  1.231e-02  -0.749  0.45393    
## ConfWest       1.864e-01  2.275e-01   0.819  0.41285    
## Conf.fctrWest         NA         NA      NA       NA    
## PTS.diff              NA         NA      NA       NA    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3.041 on 818 degrees of freedom
## Multiple R-squared:  0.9441,	Adjusted R-squared:  0.943 
## F-statistic:   864 on 16 and 818 DF,  p-value: < 2.2e-16
## 
##                                                                                                                                    feats
## 2        PTS.diff, PTS.diff:PTS, PTS.diff:FG, PTS.diff:FTA, PTS.diff:X3PA, PTS.diff:X2P, PTS.diff:SeasonEnd, PTS.diff:FGA, PTS.diff:X2PA
## 3                                                                 PTS.diff, DRB, AST, FT, BLK, X3P, STL, ORB, Team.fctr.num, TOV, oppPTS
## 1                                                                                                                               PTS.diff
## 4 SeasonEnd, PTS, oppPTS, FG, FGA, X2P, X2PA, X3P, X3PA, FT, FTA, ORB, DRB, AST, STL, BLK, TOV, Team.fctr.num, Conf, Conf.fctr, PTS.diff
##   n.fit  R.sq.fit  R.sq.OOB Adj.R.sq.fit  SSE.fit  SSE.OOB f.score.OOB
## 2   835 0.9431903 0.9427873    0.9431903 7691.006 264.3800          NA
## 3   835 0.9434823 0.9419961    0.9434823 7651.473 268.0360          NA
## 1   835 0.9423425 0.9417420    0.9423425 7805.790 269.2104          NA
## 4   835 0.9441333 0.9395900    0.9441333 7563.344 279.1548          NA
```

```r
# PTS, oppPTS
ret_lst <- myrun_mdl_fn(indep_vars_vctr=c("PTS", "oppPTS"),
                        fit_df=glb_entity_df, OOB_df=glb_predct_df)
```

```
## [1] 268.967
## [1] 0.9417946
## 
## Call:
## lm(formula = reformulate(indep_vars_vctr, response = glb_predct_var), 
##     data = fit_df)
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
##                                                                                                                                    feats
## 2        PTS.diff, PTS.diff:PTS, PTS.diff:FG, PTS.diff:FTA, PTS.diff:X3PA, PTS.diff:X2P, PTS.diff:SeasonEnd, PTS.diff:FGA, PTS.diff:X2PA
## 3                                                                 PTS.diff, DRB, AST, FT, BLK, X3P, STL, ORB, Team.fctr.num, TOV, oppPTS
## 5                                                                                                                            PTS, oppPTS
## 1                                                                                                                               PTS.diff
## 4 SeasonEnd, PTS, oppPTS, FG, FGA, X2P, X2PA, X3P, X3PA, FT, FTA, ORB, DRB, AST, STL, BLK, TOV, Team.fctr.num, Conf, Conf.fctr, PTS.diff
##   n.fit  R.sq.fit  R.sq.OOB Adj.R.sq.fit  SSE.fit  SSE.OOB f.score.OOB
## 2   835 0.9431903 0.9427873    0.9431903 7691.006 264.3800          NA
## 3   835 0.9434823 0.9419961    0.9434823 7651.473 268.0360          NA
## 5   835 0.9423450 0.9417946    0.9423450 7805.452 268.9670          NA
## 1   835 0.9423425 0.9417420    0.9423425 7805.790 269.2104          NA
## 4   835 0.9441333 0.9395900    0.9441333 7563.344 279.1548          NA
```

```r
if (glb_is_regression)
    print(myplot_scatter(glb_models_df, "Adj.R.sq.fit", "R.sq.OOB") + 
          geom_text(aes(label=feats), data=glb_models_df, color="NavyBlue", 
                    size=3.5))
```

![](NBA_Wins_files/figure-html/run_models-1.png) 

```r
if (glb_is_classification) {
    plot_models_df <- mutate(glb_models_df, feats.label=substr(feats, 1, 20))
    print(myplot_hbar(df=plot_models_df, xcol_name="feats.label", 
                      ycol_names="f.score.OOB"))
}    

script_df <- rbind(script_df, 
                   data.frame(chunk_label="fit_training.all", 
                              chunk_step_major=max(script_df$chunk_step_major)+1, 
                              chunk_step_minor=0))
print(script_df)
```

```
##                   chunk_label chunk_step_major chunk_step_minor
## 1                 import_data                1                0
## 2                cleanse_data                2                0
## 3        inspect_explore_data                2                1
## 4         manage_missing_data                2                2
## 5          encode_retype_data                2                3
## 6            extract_features                3                0
## 7             select_features                4                0
## 8  remove_correlated_features                4                1
## 9                  run_models                5                0
## 10           fit_training.all                6                0
```

## Step `6`: fit training.all

```r
print(mdl_feats_df <- myextract_mdl_feats())
```

```
##            Estimate   Std. Error  t value Pr.z       id
## PTS.diff 0.03258633 0.0002792779 116.6807    0 PTS.diff
```

```r
if (glb_is_regression) {
    ret_lst <- myrun_mdl_lm(indep_vars_vctr=mdl_feats_df$id, fit_df=glb_entity_df)
    glb_sel_mdl <- glb_mdl    
    glb_entity_df[, glb_predct_var_name] <- predict(glb_sel_mdl, newdata=glb_entity_df)
    print(myplot_scatter(glb_entity_df, glb_predct_var, glb_predct_var_name, 
                         smooth=TRUE))    
}    
```

```
## 
## Call:
## lm(formula = reformulate(indep_vars_vctr, response = glb_predct_var), 
##     data = fit_df)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -9.7393 -2.1018 -0.0672  2.0265 10.6026 
## 
## Coefficients:
##              Estimate Std. Error t value Pr(>|t|)    
## (Intercept) 4.100e+01  1.059e-01   387.0   <2e-16 ***
## PTS.diff    3.259e-02  2.793e-04   116.7   <2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3.061 on 833 degrees of freedom
## Multiple R-squared:  0.9423,	Adjusted R-squared:  0.9423 
## F-statistic: 1.361e+04 on 1 and 833 DF,  p-value: < 2.2e-16
## 
##                                                                                                                                    feats
## 2        PTS.diff, PTS.diff:PTS, PTS.diff:FG, PTS.diff:FTA, PTS.diff:X3PA, PTS.diff:X2P, PTS.diff:SeasonEnd, PTS.diff:FGA, PTS.diff:X2PA
## 3                                                                 PTS.diff, DRB, AST, FT, BLK, X3P, STL, ORB, Team.fctr.num, TOV, oppPTS
## 5                                                                                                                            PTS, oppPTS
## 1                                                                                                                               PTS.diff
## 4 SeasonEnd, PTS, oppPTS, FG, FGA, X2P, X2PA, X3P, X3PA, FT, FTA, ORB, DRB, AST, STL, BLK, TOV, Team.fctr.num, Conf, Conf.fctr, PTS.diff
## 6                                                                                                                               PTS.diff
##   n.fit  R.sq.fit  R.sq.OOB Adj.R.sq.fit  SSE.fit  SSE.OOB f.score.OOB
## 2   835 0.9431903 0.9427873    0.9431903 7691.006 264.3800          NA
## 3   835 0.9434823 0.9419961    0.9434823 7651.473 268.0360          NA
## 5   835 0.9423450 0.9417946    0.9423450 7805.452 268.9670          NA
## 1   835 0.9423425 0.9417420    0.9423425 7805.790 269.2104          NA
## 4   835 0.9441333 0.9395900    0.9441333 7563.344 279.1548          NA
## 6   835 0.9423425        NA    0.9423425 7805.790       NA          NA
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

![](NBA_Wins_files/figure-html/fit_training.all-1.png) 

```r
if (glb_is_classification) {
    ret_lst <- myrun_mdl_glm(indep_vars_vctr=mdl_feats_df$id, fit_df=glb_entity_df)
    glb_sel_mdl <- glb_mdl        
    glb_entity_df[, glb_predct_var_name] <- (predict(glb_sel_mdl, 
                        newdata=glb_entity_df, type="response") >= 0.5) * 1.0
    print(xtabs(reformulate(paste(glb_predct_var, glb_predct_var_name, sep=" + ")),
                glb_entity_df))                        
}    

print(glb_feats_df <- mymerge_feats_Pr.z())
```

```
##               id       cor.y  cor.y.abs cor.low Pr.z
## 11      PTS.diff  0.97074326 0.97074326       1    0
## 1            AST  0.32005177 0.32005177       1   NA
## 2            BLK  0.20392100 0.20392100       1   NA
## 3            DRB  0.47089750 0.47089750       1   NA
## 4             FG  0.19039642 0.19039642      NA   NA
## 5            FGA -0.07144566 0.07144566      NA   NA
## 6             FT  0.20490600 0.20490600       1   NA
## 7            FTA  0.16188731 0.16188731      NA   NA
## 8         oppPTS -0.33157294 0.33157294       1   NA
## 9            ORB -0.09573676 0.09573676       1   NA
## 10           PTS  0.29882561 0.29882561      NA   NA
## 12     SeasonEnd  0.00000000 0.00000000      NA   NA
## 13           STL  0.11619440 0.11619440       1   NA
## 14 Team.fctr.num -0.15119487 0.15119487       1   NA
## 15           TOV -0.24318588 0.24318588       1   NA
## 16           X2P  0.06927898 0.06927898      NA   NA
## 17          X2PA -0.08703653 0.08703653      NA   NA
## 18           X3P  0.11904456 0.11904456       1   NA
## 19          X3PA  0.08328625 0.08328625      NA   NA
```

```r
# Most of this code is used again in predict_newdata chunk
glb_analytics_diag_plots <- function(obs_df) {
    for (var in subset(glb_feats_df, Pr.z < 0.1)$id) {
        plot_df <- melt(obs_df, id.vars=var, 
                        measure.vars=c(glb_predct_var, glb_predct_var_name))
        if (var == "W") print(myplot_scatter(plot_df, var, "value", 
                                             facet_colcol_name="variable") + 
                      geom_vline(xintercept=40, linetype="dotted")) else     
            print(myplot_scatter(plot_df, var, "value", facet_colcol_name="variable"))
    }
    
    if (glb_is_regression) {
        plot_vars_df <- subset(glb_feats_df, Pr.z < 0.1)
        print(myplot_prediction_regression(obs_df, 
                    ifelse(nrow(plot_vars_df) > 1, plot_vars_df$id[2], ".rownames"), 
                                           plot_vars_df$id[1])
              + geom_point(aes_string(color="Conf.fctr"))
              )
    }    
    
    if (glb_is_classification) {
        plot_vars_df <- subset(glb_feats_df, Pr.z < 0.1)
        print(myplot_prediction_classification(obs_df, 
                    ifelse(nrow(plot_vars_df) > 1, plot_vars_df$id[2], ".rownames"),
                                               plot_vars_df$id[1]) + 
                geom_hline(yintercept=40, linetype = "dotted")
                )
    }    
}
glb_analytics_diag_plots(glb_entity_df)
```

![](NBA_Wins_files/figure-html/fit_training.all-2.png) 

```
##                     Team SeasonEnd Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 172     Dallas Mavericks      1993        0 11 8141   9387 3164 7271 2881
## 745  Seattle SuperSonics      1986        0 31 8564   8572 3335 7059 3256
## 356 Los Angeles Clippers      1986        0 32 8907   9475 3388 7165 3324
## 414           Miami Heat      1992        1 38 8608   8953 3256 7061 2999
## 208      Detroit Pistons      1998        0 37 7721   7592 2862 6373 2569
##     X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV
## 172 6434 283  837 1530 2171 1234 2265 1683 649 355 1459
## 745 6759  79  300 1815 2331 1145 2256 1977 745 295 1435
## 356 6936  64  229 2067 2683 1159 2258 1968 694 501 1506
## 414 6310 257  751 1839 2329 1187 2366 1749 670 373 1377
## 208 5435 293  938 1704 2288 1044 2338 1597 678 344 1198
##                Team.fctr Team.fctr.num Conf Conf.fctr PTS.diff  W.predict
## 172     Dallas Mavericks            23 West      West    -1246  0.3974293
## 745  Seattle SuperSonics            20 West      West       -8 40.7393093
## 356 Los Angeles Clippers            24 West      West     -568 22.4909630
## 414           Miami Heat            27 East      East     -345 29.7577152
## 208      Detroit Pistons             6 East      East      129 45.2036369
##     W.predict.err                    .label
## 172     10.602571     1993:Dallas Mavericks
## 745      9.739309  1986:Seattle SuperSonics
## 356      9.509037 1986:Los Angeles Clippers
## 414      8.242285           1992:Miami Heat
## 208      8.203637      1998:Detroit Pistons
```

![](NBA_Wins_files/figure-html/fit_training.all-3.png) 

```r
script_df <- rbind(script_df, 
                   data.frame(chunk_label="predict_newdata", 
                              chunk_step_major=max(script_df$chunk_step_major)+1, 
                              chunk_step_minor=0))
print(script_df)
```

```
##                   chunk_label chunk_step_major chunk_step_minor
## 1                 import_data                1                0
## 2                cleanse_data                2                0
## 3        inspect_explore_data                2                1
## 4         manage_missing_data                2                2
## 5          encode_retype_data                2                3
## 6            extract_features                3                0
## 7             select_features                4                0
## 8  remove_correlated_features                4                1
## 9                  run_models                5                0
## 10           fit_training.all                6                0
## 11            predict_newdata                7                0
```

## Step `7`: predict newdata

```r
if (glb_is_regression)
    glb_predct_df[, glb_predct_var_name] <- predict(glb_sel_mdl, 
                                        newdata=glb_predct_df, type="response")

if (glb_is_classification)
    glb_predct_df[, glb_predct_var_name] <- (predict(glb_sel_mdl, 
                        newdata=glb_predct_df, type="response") >= 0.5) * 1.0
    
myprint_df(glb_predct_df[, c(glb_id_vars, glb_predct_var, glb_predct_var_name)])
```

```
##   SeasonEnd                Team  W W.predict
## 1      2013       Atlanta Hawks 44  42.07535
## 2      2013       Brooklyn Nets 49  45.75760
## 3      2013   Charlotte Bobcats 21  16.33215
## 4      2013       Chicago Bulls 45  41.84724
## 5      2013 Cleveland Cavaliers 24  28.48685
## 6      2013    Dallas Mavericks 41  39.40327
##    SeasonEnd                  Team  W W.predict
## 6       2013      Dallas Mavericks 41  39.40327
## 11      2013  Los Angeles Clippers 56  58.23817
## 12      2013    Los Angeles Lakers 45  44.09570
## 14      2013            Miami Heat 66  62.01818
## 19      2013 Oklahoma City Thunder 60  65.60268
## 28      2013    Washington Wizards 29  34.22204
##    SeasonEnd                   Team  W W.predict
## 23      2013 Portland Trail Blazers 33  32.52755
## 24      2013       Sacramento Kings 28  27.96547
## 25      2013      San Antonio Spurs 58  58.10782
## 26      2013        Toronto Raptors 34  37.05705
## 27      2013              Utah Jazz 43  40.77190
## 28      2013     Washington Wizards 29  34.22204
```

```r
if (glb_is_regression) {
    print(sprintf("Total SSE: %0.4f", 
                  sum((glb_predct_df[, glb_predct_var_name] - 
                        glb_predct_df[, glb_predct_var]) ^ 2)))
    print(myplot_scatter(glb_predct_df, glb_predct_var, glb_predct_var_name, 
                         smooth=TRUE))
}                         
```

```
## [1] "Total SSE: 269.2104"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

![](NBA_Wins_files/figure-html/predict_newdata-1.png) 

```r
if (glb_is_classification)
    print(xtabs(reformulate(paste(glb_predct_var, glb_predct_var_name, sep=" + ")),
                glb_predct_df))
    
glb_analytics_diag_plots(glb_predct_df)
```

![](NBA_Wins_files/figure-html/predict_newdata-2.png) 

```
##                     Team SeasonEnd Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 19 Oklahoma City Thunder      2013        1 60 8669   7914 3126 6504 2528
## 10       Houston Rockets      2013        1 45 8688   8403 3124 6782 2257
## 28    Washington Wizards      2013        0 29 7644   7852 2910 6693 2365
## 3      Charlotte Bobcats      2013        0 21 7661   8418 2823 6649 2354
## 5    Cleveland Cavaliers      2013        0 24 7913   8297 2993 6901 2446
##    X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV
## 19 4916 598 1588 1819 2196  854 2725 1753 679 624 1253
## 10 4413 867 2369 1573 2087  909 2652 1902 679 359 1348
## 28 5198 545 1495 1279 1746  887 2652 1775 598 376 1238
## 3  5250 469 1399 1546 2060  917 2389 1587 591 479 1153
## 5  5320 547 1581 1380 1826 1004 2359 1694 647 334 1149
##                Team.fctr Team.fctr.num Conf Conf.fctr PTS.diff W.predict
## 19 Oklahoma City Thunder            19 West      West      755  65.60268
## 10       Houston Rockets            10 West      West      285  50.28710
## 28    Washington Wizards            28 East      East     -208  34.22204
## 3      Charlotte Bobcats             3 East      East     -757  16.33215
## 5    Cleveland Cavaliers             5 East      East     -384  28.48685
##    W.predict.err                     .label
## 19      5.602681 2013:Oklahoma City Thunder
## 10      5.287105       2013:Houston Rockets
## 28      5.222043    2013:Washington Wizards
## 3       4.667854     2013:Charlotte Bobcats
## 5       4.486848   2013:Cleveland Cavaliers
```

![](NBA_Wins_files/figure-html/predict_newdata-3.png) 

Null Hypothesis ($\sf{H_{0}}$): mpg is not impacted by am_fctr.  
The variance by am_fctr appears to be independent. 

```r
# print(t.test(subset(cars_df, am_fctr == "automatic")$mpg, 
#              subset(cars_df, am_fctr == "manual")$mpg, 
#              var.equal=FALSE)$conf)
```
We reject the null hypothesis i.e. we have evidence to conclude that am_fctr impacts mpg (95% confidence). Manual transmission is better for miles per gallon versus automatic transmission.


```
## R version 3.1.2 (2014-10-31)
## Platform: x86_64-apple-darwin13.4.0 (64-bit)
## 
## locale:
## [1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8
## 
## attached base packages:
## [1] tcltk     stats     graphics  grDevices utils     datasets  methods  
## [8] base     
## 
## other attached packages:
##  [1] sqldf_0.4-10    RSQLite_1.0.0   DBI_0.3.1       gsubfn_0.6-6   
##  [5] proto_0.3-10    reshape2_1.4.1  plyr_1.8.1      doBy_4.5-13    
##  [9] survival_2.38-1 ggplot2_1.0.0  
## 
## loaded via a namespace (and not attached):
##  [1] chron_2.3-45     colorspace_1.2-5 digest_0.6.8     evaluate_0.5.5  
##  [5] formatR_1.0      grid_3.1.2       gtable_0.1.2     htmltools_0.2.6 
##  [9] knitr_1.9        labeling_0.3     lattice_0.20-30  MASS_7.3-39     
## [13] Matrix_1.1-5     munsell_0.4.2    Rcpp_0.11.4      rmarkdown_0.5.1 
## [17] scales_0.2.4     splines_3.1.2    stringr_0.6.2    tools_3.1.2     
## [21] yaml_2.1.13
```
