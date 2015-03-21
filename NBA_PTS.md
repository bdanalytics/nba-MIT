# NBA: PTS regression
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
glb_predct_var <- "PTS"           # or NULL
glb_predct_var_name <- paste0(glb_predct_var, ".predict")
glb_id_vars <- c("SeasonEnd", "Team")                # or NULL

glb_exclude_vars_as_features <- NULL                      
# List chrs converted into factors 
glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, 
                                      c("Team", "Team.fctr",
                                        "Conf", "Conf.fctr")     # or NULL
                                      )
# List feats that shd be excluded due to known causation by prediction variable
glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, 
                                      c("Playoffs", "Wins", 
                                        "PTS.diff", "oppPTS")     # or NULL
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

![](NBA_PTS_files/figure-html/encode_retype_data_1-1.png) 

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

![](NBA_PTS_files/figure-html/encode_retype_data_1-2.png) 

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
## FG                       FG  0.94195473 0.94195473
## X2P                     X2P  0.82572457 0.82572457
## FGA                     FGA  0.79397947 0.79397947
## AST                     AST  0.75988915 0.75988915
## X2PA                   X2PA  0.70836140 0.70836140
## FT                       FT  0.69748385 0.69748385
## FTA                     FTA  0.65505876 0.65505876
## SeasonEnd         SeasonEnd -0.63952773 0.63952773
## X3PA                   X3PA -0.51519812 0.51519812
## ORB                     ORB  0.49692073 0.49692073
## X3P                     X3P -0.48994891 0.48994891
## STL                     STL  0.43099026 0.43099026
## TOV                     TOV  0.42713832 0.42713832
## W                         W  0.29882561 0.29882561
## Team.fctr.num Team.fctr.num -0.20008998 0.20008998
## BLK                     BLK  0.15205537 0.15205537
## DRB                     DRB  0.09029137 0.09029137
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
##                         FG         X2P         FGA         AST        X2PA
## FG             1.000000000  0.94291996  0.87966291  0.81226652  0.85922463
## X2P            0.942919965  1.00000000  0.83827486  0.77711327  0.96530915
## FGA            0.879662910  0.83827486  1.00000000  0.62911736  0.86485931
## AST            0.812266519  0.77711327  0.62911736  1.00000000  0.67770751
## X2PA           0.859224632  0.96530915  0.86485931  0.67770751  1.00000000
## FT             0.538342589  0.57429395  0.39169143  0.44720848  0.51580201
## FTA            0.521216830  0.57454277  0.38212215  0.42797223  0.52375675
## SeasonEnd     -0.761598813 -0.86548930 -0.68490478 -0.67212300 -0.86725681
## X3PA          -0.688763042 -0.88859996 -0.60559564 -0.59278305 -0.92324423
## ORB            0.598591143  0.68311802  0.73750939  0.40676555  0.76561469
## X3P           -0.668272893 -0.87786641 -0.60756448 -0.56785931 -0.92073200
## STL            0.469730269  0.48900271  0.48053292  0.44313827  0.49008620
## TOV            0.517630444  0.63771567  0.44998136  0.43053288  0.64485242
## W              0.190396422  0.06927898 -0.07144566  0.32005177 -0.08703653
## Team.fctr.num -0.198789524 -0.19170714 -0.17904220 -0.17491216 -0.17580617
## BLK            0.177502748  0.21771153  0.12109667  0.20308000  0.20610678
## DRB            0.008966352 -0.09869019 -0.05094546  0.05647709 -0.16531549
##                         FT         FTA  SeasonEnd        X3PA         ORB
## FG             0.538342589  0.52121683 -0.7615988 -0.68876304  0.59859114
## X2P            0.574293954  0.57454277 -0.8654893 -0.88859996  0.68311802
## FGA            0.391691425  0.38212215 -0.6849048 -0.60559564  0.73750939
## AST            0.447208477  0.42797223 -0.6721230 -0.59278305  0.40676555
## X2PA           0.515802012  0.52375675 -0.8672568 -0.92324423  0.76561469
## FT             1.000000000  0.95049325 -0.5136992 -0.51784957  0.39339870
## FTA            0.950493246  1.00000000 -0.5468965 -0.53778289  0.47455838
## SeasonEnd     -0.513699180 -0.54689652  1.0000000  0.85055227 -0.70201284
## X3PA          -0.517849574 -0.53778289  0.8505523  1.00000000 -0.64917273
## ORB            0.393398702  0.47455838 -0.7020128 -0.64917273  1.00000000
## X3P           -0.508712984 -0.53389685  0.8381421  0.99451088 -0.66516818
## STL            0.323156975  0.37005522 -0.5020598 -0.40909163  0.49294148
## TOV            0.438684976  0.52566979 -0.7238690 -0.67780315  0.54598664
## W              0.204906000  0.16188731  0.0000000  0.08328625 -0.09573676
## Team.fctr.num -0.154385609 -0.13968994  0.2145549  0.14165189 -0.20095724
## BLK            0.164655982  0.21637171 -0.2041625 -0.23403142  0.20110479
## DRB            0.003604883 -0.04354267  0.2671271  0.22306055 -0.26917934
##                      X3P        STL        TOV           W Team.fctr.num
## FG            -0.6682729  0.4697303  0.5176304  0.19039642   -0.19878952
## X2P           -0.8778664  0.4890027  0.6377157  0.06927898   -0.19170714
## FGA           -0.6075645  0.4805329  0.4499814 -0.07144566   -0.17904220
## AST           -0.5678593  0.4431383  0.4305329  0.32005177   -0.17491216
## X2PA          -0.9207320  0.4900862  0.6448524 -0.08703653   -0.17580617
## FT            -0.5087130  0.3231570  0.4386850  0.20490600   -0.15438561
## FTA           -0.5338969  0.3700552  0.5256698  0.16188731   -0.13968994
## SeasonEnd      0.8381421 -0.5020598 -0.7238690  0.00000000    0.21455487
## X3PA           0.9945109 -0.4090916 -0.6778031  0.08328625    0.14165189
## ORB           -0.6651682  0.4929415  0.5459866 -0.09573676   -0.20095724
## X3P            1.0000000 -0.4168543 -0.6801733  0.11904456    0.14237198
## STL           -0.4168543  1.0000000  0.4640752  0.11619440   -0.15311931
## TOV           -0.6801733  0.4640752  1.0000000 -0.24318588   -0.14045929
## W              0.1190446  0.1161944 -0.2431859  1.00000000   -0.15119487
## Team.fctr.num  0.1423720 -0.1531193 -0.1404593 -0.15119487    1.00000000
## BLK           -0.2310738  0.1204559  0.2415687  0.20392100   -0.07311722
## DRB            0.2333536 -0.3311279 -0.1909569  0.47089750   -0.05815482
##                       BLK          DRB
## FG             0.17750275  0.008966352
## X2P            0.21771153 -0.098690188
## FGA            0.12109667 -0.050945458
## AST            0.20308000  0.056477086
## X2PA           0.20610678 -0.165315490
## FT             0.16465598  0.003604883
## FTA            0.21637171 -0.043542668
## SeasonEnd     -0.20416249  0.267127100
## X3PA          -0.23403142  0.223060547
## ORB            0.20110479 -0.269179339
## X3P           -0.23107383  0.233353555
## STL            0.12045594 -0.331127910
## TOV            0.24156872 -0.190956902
## W              0.20392100  0.470897497
## Team.fctr.num -0.07311722 -0.058154820
## BLK            1.00000000  0.242990816
## DRB            0.24299082  1.000000000
##                        FG        X2P        FGA        AST       X2PA
## FG            0.000000000 0.94291996 0.87966291 0.81226652 0.85922463
## X2P           0.942919965 0.00000000 0.83827486 0.77711327 0.96530915
## FGA           0.879662910 0.83827486 0.00000000 0.62911736 0.86485931
## AST           0.812266519 0.77711327 0.62911736 0.00000000 0.67770751
## X2PA          0.859224632 0.96530915 0.86485931 0.67770751 0.00000000
## FT            0.538342589 0.57429395 0.39169143 0.44720848 0.51580201
## FTA           0.521216830 0.57454277 0.38212215 0.42797223 0.52375675
## SeasonEnd     0.761598813 0.86548930 0.68490478 0.67212300 0.86725681
## X3PA          0.688763042 0.88859996 0.60559564 0.59278305 0.92324423
## ORB           0.598591143 0.68311802 0.73750939 0.40676555 0.76561469
## X3P           0.668272893 0.87786641 0.60756448 0.56785931 0.92073200
## STL           0.469730269 0.48900271 0.48053292 0.44313827 0.49008620
## TOV           0.517630444 0.63771567 0.44998136 0.43053288 0.64485242
## W             0.190396422 0.06927898 0.07144566 0.32005177 0.08703653
## Team.fctr.num 0.198789524 0.19170714 0.17904220 0.17491216 0.17580617
## BLK           0.177502748 0.21771153 0.12109667 0.20308000 0.20610678
## DRB           0.008966352 0.09869019 0.05094546 0.05647709 0.16531549
##                        FT        FTA SeasonEnd       X3PA        ORB
## FG            0.538342589 0.52121683 0.7615988 0.68876304 0.59859114
## X2P           0.574293954 0.57454277 0.8654893 0.88859996 0.68311802
## FGA           0.391691425 0.38212215 0.6849048 0.60559564 0.73750939
## AST           0.447208477 0.42797223 0.6721230 0.59278305 0.40676555
## X2PA          0.515802012 0.52375675 0.8672568 0.92324423 0.76561469
## FT            0.000000000 0.95049325 0.5136992 0.51784957 0.39339870
## FTA           0.950493246 0.00000000 0.5468965 0.53778289 0.47455838
## SeasonEnd     0.513699180 0.54689652 0.0000000 0.85055227 0.70201284
## X3PA          0.517849574 0.53778289 0.8505523 0.00000000 0.64917273
## ORB           0.393398702 0.47455838 0.7020128 0.64917273 0.00000000
## X3P           0.508712984 0.53389685 0.8381421 0.99451088 0.66516818
## STL           0.323156975 0.37005522 0.5020598 0.40909163 0.49294148
## TOV           0.438684976 0.52566979 0.7238690 0.67780315 0.54598664
## W             0.204906000 0.16188731 0.0000000 0.08328625 0.09573676
## Team.fctr.num 0.154385609 0.13968994 0.2145549 0.14165189 0.20095724
## BLK           0.164655982 0.21637171 0.2041625 0.23403142 0.20110479
## DRB           0.003604883 0.04354267 0.2671271 0.22306055 0.26917934
##                     X3P       STL       TOV          W Team.fctr.num
## FG            0.6682729 0.4697303 0.5176304 0.19039642    0.19878952
## X2P           0.8778664 0.4890027 0.6377157 0.06927898    0.19170714
## FGA           0.6075645 0.4805329 0.4499814 0.07144566    0.17904220
## AST           0.5678593 0.4431383 0.4305329 0.32005177    0.17491216
## X2PA          0.9207320 0.4900862 0.6448524 0.08703653    0.17580617
## FT            0.5087130 0.3231570 0.4386850 0.20490600    0.15438561
## FTA           0.5338969 0.3700552 0.5256698 0.16188731    0.13968994
## SeasonEnd     0.8381421 0.5020598 0.7238690 0.00000000    0.21455487
## X3PA          0.9945109 0.4090916 0.6778031 0.08328625    0.14165189
## ORB           0.6651682 0.4929415 0.5459866 0.09573676    0.20095724
## X3P           0.0000000 0.4168543 0.6801733 0.11904456    0.14237198
## STL           0.4168543 0.0000000 0.4640752 0.11619440    0.15311931
## TOV           0.6801733 0.4640752 0.0000000 0.24318588    0.14045929
## W             0.1190446 0.1161944 0.2431859 0.00000000    0.15119487
## Team.fctr.num 0.1423720 0.1531193 0.1404593 0.15119487    0.00000000
## BLK           0.2310738 0.1204559 0.2415687 0.20392100    0.07311722
## DRB           0.2333536 0.3311279 0.1909569 0.47089750    0.05815482
##                      BLK         DRB
## FG            0.17750275 0.008966352
## X2P           0.21771153 0.098690188
## FGA           0.12109667 0.050945458
## AST           0.20308000 0.056477086
## X2PA          0.20610678 0.165315490
## FT            0.16465598 0.003604883
## FTA           0.21637171 0.043542668
## SeasonEnd     0.20416249 0.267127100
## X3PA          0.23403142 0.223060547
## ORB           0.20110479 0.269179339
## X3P           0.23107383 0.233353555
## STL           0.12045594 0.331127910
## TOV           0.24156872 0.190956902
## W             0.20392100 0.470897497
## Team.fctr.num 0.07311722 0.058154820
## BLK           0.00000000 0.242990816
## DRB           0.24299082 0.000000000
## [1] "cor(X3PA, X3P)=0.9945"
```

![](NBA_PTS_files/figure-html/remove_correlated_features-1.png) 

```
## [1] "cor(PTS, X3PA)=-0.5152"
## [1] "cor(PTS, X3P)=-0.4899"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in mydelete_cor_features(): Dropping X3P as a feature
```

![](NBA_PTS_files/figure-html/remove_correlated_features-2.png) 

```
##                          id       cor.y  cor.y.abs
## FG                       FG  0.94195473 0.94195473
## X2P                     X2P  0.82572457 0.82572457
## FGA                     FGA  0.79397947 0.79397947
## AST                     AST  0.75988915 0.75988915
## X2PA                   X2PA  0.70836140 0.70836140
## FT                       FT  0.69748385 0.69748385
## FTA                     FTA  0.65505876 0.65505876
## SeasonEnd         SeasonEnd -0.63952773 0.63952773
## X3PA                   X3PA -0.51519812 0.51519812
## ORB                     ORB  0.49692073 0.49692073
## STL                     STL  0.43099026 0.43099026
## TOV                     TOV  0.42713832 0.42713832
## W                         W  0.29882561 0.29882561
## Team.fctr.num Team.fctr.num -0.20008998 0.20008998
## BLK                     BLK  0.15205537 0.15205537
## DRB                     DRB  0.09029137 0.09029137
##                         FG         X2P         FGA         AST        X2PA
## FG             1.000000000  0.94291996  0.87966291  0.81226652  0.85922463
## X2P            0.942919965  1.00000000  0.83827486  0.77711327  0.96530915
## FGA            0.879662910  0.83827486  1.00000000  0.62911736  0.86485931
## AST            0.812266519  0.77711327  0.62911736  1.00000000  0.67770751
## X2PA           0.859224632  0.96530915  0.86485931  0.67770751  1.00000000
## FT             0.538342589  0.57429395  0.39169143  0.44720848  0.51580201
## FTA            0.521216830  0.57454277  0.38212215  0.42797223  0.52375675
## SeasonEnd     -0.761598813 -0.86548930 -0.68490478 -0.67212300 -0.86725681
## X3PA          -0.688763042 -0.88859996 -0.60559564 -0.59278305 -0.92324423
## ORB            0.598591143  0.68311802  0.73750939  0.40676555  0.76561469
## STL            0.469730269  0.48900271  0.48053292  0.44313827  0.49008620
## TOV            0.517630444  0.63771567  0.44998136  0.43053288  0.64485242
## W              0.190396422  0.06927898 -0.07144566  0.32005177 -0.08703653
## Team.fctr.num -0.198789524 -0.19170714 -0.17904220 -0.17491216 -0.17580617
## BLK            0.177502748  0.21771153  0.12109667  0.20308000  0.20610678
## DRB            0.008966352 -0.09869019 -0.05094546  0.05647709 -0.16531549
##                         FT         FTA  SeasonEnd        X3PA         ORB
## FG             0.538342589  0.52121683 -0.7615988 -0.68876304  0.59859114
## X2P            0.574293954  0.57454277 -0.8654893 -0.88859996  0.68311802
## FGA            0.391691425  0.38212215 -0.6849048 -0.60559564  0.73750939
## AST            0.447208477  0.42797223 -0.6721230 -0.59278305  0.40676555
## X2PA           0.515802012  0.52375675 -0.8672568 -0.92324423  0.76561469
## FT             1.000000000  0.95049325 -0.5136992 -0.51784957  0.39339870
## FTA            0.950493246  1.00000000 -0.5468965 -0.53778289  0.47455838
## SeasonEnd     -0.513699180 -0.54689652  1.0000000  0.85055227 -0.70201284
## X3PA          -0.517849574 -0.53778289  0.8505523  1.00000000 -0.64917273
## ORB            0.393398702  0.47455838 -0.7020128 -0.64917273  1.00000000
## STL            0.323156975  0.37005522 -0.5020598 -0.40909163  0.49294148
## TOV            0.438684976  0.52566979 -0.7238690 -0.67780315  0.54598664
## W              0.204906000  0.16188731  0.0000000  0.08328625 -0.09573676
## Team.fctr.num -0.154385609 -0.13968994  0.2145549  0.14165189 -0.20095724
## BLK            0.164655982  0.21637171 -0.2041625 -0.23403142  0.20110479
## DRB            0.003604883 -0.04354267  0.2671271  0.22306055 -0.26917934
##                      STL        TOV           W Team.fctr.num         BLK
## FG             0.4697303  0.5176304  0.19039642   -0.19878952  0.17750275
## X2P            0.4890027  0.6377157  0.06927898   -0.19170714  0.21771153
## FGA            0.4805329  0.4499814 -0.07144566   -0.17904220  0.12109667
## AST            0.4431383  0.4305329  0.32005177   -0.17491216  0.20308000
## X2PA           0.4900862  0.6448524 -0.08703653   -0.17580617  0.20610678
## FT             0.3231570  0.4386850  0.20490600   -0.15438561  0.16465598
## FTA            0.3700552  0.5256698  0.16188731   -0.13968994  0.21637171
## SeasonEnd     -0.5020598 -0.7238690  0.00000000    0.21455487 -0.20416249
## X3PA          -0.4090916 -0.6778031  0.08328625    0.14165189 -0.23403142
## ORB            0.4929415  0.5459866 -0.09573676   -0.20095724  0.20110479
## STL            1.0000000  0.4640752  0.11619440   -0.15311931  0.12045594
## TOV            0.4640752  1.0000000 -0.24318588   -0.14045929  0.24156872
## W              0.1161944 -0.2431859  1.00000000   -0.15119487  0.20392100
## Team.fctr.num -0.1531193 -0.1404593 -0.15119487    1.00000000 -0.07311722
## BLK            0.1204559  0.2415687  0.20392100   -0.07311722  1.00000000
## DRB           -0.3311279 -0.1909569  0.47089750   -0.05815482  0.24299082
##                        DRB
## FG             0.008966352
## X2P           -0.098690188
## FGA           -0.050945458
## AST            0.056477086
## X2PA          -0.165315490
## FT             0.003604883
## FTA           -0.043542668
## SeasonEnd      0.267127100
## X3PA           0.223060547
## ORB           -0.269179339
## STL           -0.331127910
## TOV           -0.190956902
## W              0.470897497
## Team.fctr.num -0.058154820
## BLK            0.242990816
## DRB            1.000000000
##                        FG        X2P        FGA        AST       X2PA
## FG            0.000000000 0.94291996 0.87966291 0.81226652 0.85922463
## X2P           0.942919965 0.00000000 0.83827486 0.77711327 0.96530915
## FGA           0.879662910 0.83827486 0.00000000 0.62911736 0.86485931
## AST           0.812266519 0.77711327 0.62911736 0.00000000 0.67770751
## X2PA          0.859224632 0.96530915 0.86485931 0.67770751 0.00000000
## FT            0.538342589 0.57429395 0.39169143 0.44720848 0.51580201
## FTA           0.521216830 0.57454277 0.38212215 0.42797223 0.52375675
## SeasonEnd     0.761598813 0.86548930 0.68490478 0.67212300 0.86725681
## X3PA          0.688763042 0.88859996 0.60559564 0.59278305 0.92324423
## ORB           0.598591143 0.68311802 0.73750939 0.40676555 0.76561469
## STL           0.469730269 0.48900271 0.48053292 0.44313827 0.49008620
## TOV           0.517630444 0.63771567 0.44998136 0.43053288 0.64485242
## W             0.190396422 0.06927898 0.07144566 0.32005177 0.08703653
## Team.fctr.num 0.198789524 0.19170714 0.17904220 0.17491216 0.17580617
## BLK           0.177502748 0.21771153 0.12109667 0.20308000 0.20610678
## DRB           0.008966352 0.09869019 0.05094546 0.05647709 0.16531549
##                        FT        FTA SeasonEnd       X3PA        ORB
## FG            0.538342589 0.52121683 0.7615988 0.68876304 0.59859114
## X2P           0.574293954 0.57454277 0.8654893 0.88859996 0.68311802
## FGA           0.391691425 0.38212215 0.6849048 0.60559564 0.73750939
## AST           0.447208477 0.42797223 0.6721230 0.59278305 0.40676555
## X2PA          0.515802012 0.52375675 0.8672568 0.92324423 0.76561469
## FT            0.000000000 0.95049325 0.5136992 0.51784957 0.39339870
## FTA           0.950493246 0.00000000 0.5468965 0.53778289 0.47455838
## SeasonEnd     0.513699180 0.54689652 0.0000000 0.85055227 0.70201284
## X3PA          0.517849574 0.53778289 0.8505523 0.00000000 0.64917273
## ORB           0.393398702 0.47455838 0.7020128 0.64917273 0.00000000
## STL           0.323156975 0.37005522 0.5020598 0.40909163 0.49294148
## TOV           0.438684976 0.52566979 0.7238690 0.67780315 0.54598664
## W             0.204906000 0.16188731 0.0000000 0.08328625 0.09573676
## Team.fctr.num 0.154385609 0.13968994 0.2145549 0.14165189 0.20095724
## BLK           0.164655982 0.21637171 0.2041625 0.23403142 0.20110479
## DRB           0.003604883 0.04354267 0.2671271 0.22306055 0.26917934
##                     STL       TOV          W Team.fctr.num        BLK
## FG            0.4697303 0.5176304 0.19039642    0.19878952 0.17750275
## X2P           0.4890027 0.6377157 0.06927898    0.19170714 0.21771153
## FGA           0.4805329 0.4499814 0.07144566    0.17904220 0.12109667
## AST           0.4431383 0.4305329 0.32005177    0.17491216 0.20308000
## X2PA          0.4900862 0.6448524 0.08703653    0.17580617 0.20610678
## FT            0.3231570 0.4386850 0.20490600    0.15438561 0.16465598
## FTA           0.3700552 0.5256698 0.16188731    0.13968994 0.21637171
## SeasonEnd     0.5020598 0.7238690 0.00000000    0.21455487 0.20416249
## X3PA          0.4090916 0.6778031 0.08328625    0.14165189 0.23403142
## ORB           0.4929415 0.5459866 0.09573676    0.20095724 0.20110479
## STL           0.0000000 0.4640752 0.11619440    0.15311931 0.12045594
## TOV           0.4640752 0.0000000 0.24318588    0.14045929 0.24156872
## W             0.1161944 0.2431859 0.00000000    0.15119487 0.20392100
## Team.fctr.num 0.1531193 0.1404593 0.15119487    0.00000000 0.07311722
## BLK           0.1204559 0.2415687 0.20392100    0.07311722 0.00000000
## DRB           0.3311279 0.1909569 0.47089750    0.05815482 0.24299082
##                       DRB
## FG            0.008966352
## X2P           0.098690188
## FGA           0.050945458
## AST           0.056477086
## X2PA          0.165315490
## FT            0.003604883
## FTA           0.043542668
## SeasonEnd     0.267127100
## X3PA          0.223060547
## ORB           0.269179339
## STL           0.331127910
## TOV           0.190956902
## W             0.470897497
## Team.fctr.num 0.058154820
## BLK           0.242990816
## DRB           0.000000000
## [1] "cor(X2P, X2PA)=0.9653"
```

![](NBA_PTS_files/figure-html/remove_correlated_features-3.png) 

```
## [1] "cor(PTS, X2P)=0.8257"
## [1] "cor(PTS, X2PA)=0.7084"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in mydelete_cor_features(): Dropping X2PA as a feature
```

![](NBA_PTS_files/figure-html/remove_correlated_features-4.png) 

```
##                          id       cor.y  cor.y.abs
## FG                       FG  0.94195473 0.94195473
## X2P                     X2P  0.82572457 0.82572457
## FGA                     FGA  0.79397947 0.79397947
## AST                     AST  0.75988915 0.75988915
## FT                       FT  0.69748385 0.69748385
## FTA                     FTA  0.65505876 0.65505876
## SeasonEnd         SeasonEnd -0.63952773 0.63952773
## X3PA                   X3PA -0.51519812 0.51519812
## ORB                     ORB  0.49692073 0.49692073
## STL                     STL  0.43099026 0.43099026
## TOV                     TOV  0.42713832 0.42713832
## W                         W  0.29882561 0.29882561
## Team.fctr.num Team.fctr.num -0.20008998 0.20008998
## BLK                     BLK  0.15205537 0.15205537
## DRB                     DRB  0.09029137 0.09029137
##                         FG         X2P         FGA         AST
## FG             1.000000000  0.94291996  0.87966291  0.81226652
## X2P            0.942919965  1.00000000  0.83827486  0.77711327
## FGA            0.879662910  0.83827486  1.00000000  0.62911736
## AST            0.812266519  0.77711327  0.62911736  1.00000000
## FT             0.538342589  0.57429395  0.39169143  0.44720848
## FTA            0.521216830  0.57454277  0.38212215  0.42797223
## SeasonEnd     -0.761598813 -0.86548930 -0.68490478 -0.67212300
## X3PA          -0.688763042 -0.88859996 -0.60559564 -0.59278305
## ORB            0.598591143  0.68311802  0.73750939  0.40676555
## STL            0.469730269  0.48900271  0.48053292  0.44313827
## TOV            0.517630444  0.63771567  0.44998136  0.43053288
## W              0.190396422  0.06927898 -0.07144566  0.32005177
## Team.fctr.num -0.198789524 -0.19170714 -0.17904220 -0.17491216
## BLK            0.177502748  0.21771153  0.12109667  0.20308000
## DRB            0.008966352 -0.09869019 -0.05094546  0.05647709
##                         FT         FTA  SeasonEnd        X3PA         ORB
## FG             0.538342589  0.52121683 -0.7615988 -0.68876304  0.59859114
## X2P            0.574293954  0.57454277 -0.8654893 -0.88859996  0.68311802
## FGA            0.391691425  0.38212215 -0.6849048 -0.60559564  0.73750939
## AST            0.447208477  0.42797223 -0.6721230 -0.59278305  0.40676555
## FT             1.000000000  0.95049325 -0.5136992 -0.51784957  0.39339870
## FTA            0.950493246  1.00000000 -0.5468965 -0.53778289  0.47455838
## SeasonEnd     -0.513699180 -0.54689652  1.0000000  0.85055227 -0.70201284
## X3PA          -0.517849574 -0.53778289  0.8505523  1.00000000 -0.64917273
## ORB            0.393398702  0.47455838 -0.7020128 -0.64917273  1.00000000
## STL            0.323156975  0.37005522 -0.5020598 -0.40909163  0.49294148
## TOV            0.438684976  0.52566979 -0.7238690 -0.67780315  0.54598664
## W              0.204906000  0.16188731  0.0000000  0.08328625 -0.09573676
## Team.fctr.num -0.154385609 -0.13968994  0.2145549  0.14165189 -0.20095724
## BLK            0.164655982  0.21637171 -0.2041625 -0.23403142  0.20110479
## DRB            0.003604883 -0.04354267  0.2671271  0.22306055 -0.26917934
##                      STL        TOV           W Team.fctr.num         BLK
## FG             0.4697303  0.5176304  0.19039642   -0.19878952  0.17750275
## X2P            0.4890027  0.6377157  0.06927898   -0.19170714  0.21771153
## FGA            0.4805329  0.4499814 -0.07144566   -0.17904220  0.12109667
## AST            0.4431383  0.4305329  0.32005177   -0.17491216  0.20308000
## FT             0.3231570  0.4386850  0.20490600   -0.15438561  0.16465598
## FTA            0.3700552  0.5256698  0.16188731   -0.13968994  0.21637171
## SeasonEnd     -0.5020598 -0.7238690  0.00000000    0.21455487 -0.20416249
## X3PA          -0.4090916 -0.6778031  0.08328625    0.14165189 -0.23403142
## ORB            0.4929415  0.5459866 -0.09573676   -0.20095724  0.20110479
## STL            1.0000000  0.4640752  0.11619440   -0.15311931  0.12045594
## TOV            0.4640752  1.0000000 -0.24318588   -0.14045929  0.24156872
## W              0.1161944 -0.2431859  1.00000000   -0.15119487  0.20392100
## Team.fctr.num -0.1531193 -0.1404593 -0.15119487    1.00000000 -0.07311722
## BLK            0.1204559  0.2415687  0.20392100   -0.07311722  1.00000000
## DRB           -0.3311279 -0.1909569  0.47089750   -0.05815482  0.24299082
##                        DRB
## FG             0.008966352
## X2P           -0.098690188
## FGA           -0.050945458
## AST            0.056477086
## FT             0.003604883
## FTA           -0.043542668
## SeasonEnd      0.267127100
## X3PA           0.223060547
## ORB           -0.269179339
## STL           -0.331127910
## TOV           -0.190956902
## W              0.470897497
## Team.fctr.num -0.058154820
## BLK            0.242990816
## DRB            1.000000000
##                        FG        X2P        FGA        AST          FT
## FG            0.000000000 0.94291996 0.87966291 0.81226652 0.538342589
## X2P           0.942919965 0.00000000 0.83827486 0.77711327 0.574293954
## FGA           0.879662910 0.83827486 0.00000000 0.62911736 0.391691425
## AST           0.812266519 0.77711327 0.62911736 0.00000000 0.447208477
## FT            0.538342589 0.57429395 0.39169143 0.44720848 0.000000000
## FTA           0.521216830 0.57454277 0.38212215 0.42797223 0.950493246
## SeasonEnd     0.761598813 0.86548930 0.68490478 0.67212300 0.513699180
## X3PA          0.688763042 0.88859996 0.60559564 0.59278305 0.517849574
## ORB           0.598591143 0.68311802 0.73750939 0.40676555 0.393398702
## STL           0.469730269 0.48900271 0.48053292 0.44313827 0.323156975
## TOV           0.517630444 0.63771567 0.44998136 0.43053288 0.438684976
## W             0.190396422 0.06927898 0.07144566 0.32005177 0.204906000
## Team.fctr.num 0.198789524 0.19170714 0.17904220 0.17491216 0.154385609
## BLK           0.177502748 0.21771153 0.12109667 0.20308000 0.164655982
## DRB           0.008966352 0.09869019 0.05094546 0.05647709 0.003604883
##                      FTA SeasonEnd       X3PA        ORB       STL
## FG            0.52121683 0.7615988 0.68876304 0.59859114 0.4697303
## X2P           0.57454277 0.8654893 0.88859996 0.68311802 0.4890027
## FGA           0.38212215 0.6849048 0.60559564 0.73750939 0.4805329
## AST           0.42797223 0.6721230 0.59278305 0.40676555 0.4431383
## FT            0.95049325 0.5136992 0.51784957 0.39339870 0.3231570
## FTA           0.00000000 0.5468965 0.53778289 0.47455838 0.3700552
## SeasonEnd     0.54689652 0.0000000 0.85055227 0.70201284 0.5020598
## X3PA          0.53778289 0.8505523 0.00000000 0.64917273 0.4090916
## ORB           0.47455838 0.7020128 0.64917273 0.00000000 0.4929415
## STL           0.37005522 0.5020598 0.40909163 0.49294148 0.0000000
## TOV           0.52566979 0.7238690 0.67780315 0.54598664 0.4640752
## W             0.16188731 0.0000000 0.08328625 0.09573676 0.1161944
## Team.fctr.num 0.13968994 0.2145549 0.14165189 0.20095724 0.1531193
## BLK           0.21637171 0.2041625 0.23403142 0.20110479 0.1204559
## DRB           0.04354267 0.2671271 0.22306055 0.26917934 0.3311279
##                     TOV          W Team.fctr.num        BLK         DRB
## FG            0.5176304 0.19039642    0.19878952 0.17750275 0.008966352
## X2P           0.6377157 0.06927898    0.19170714 0.21771153 0.098690188
## FGA           0.4499814 0.07144566    0.17904220 0.12109667 0.050945458
## AST           0.4305329 0.32005177    0.17491216 0.20308000 0.056477086
## FT            0.4386850 0.20490600    0.15438561 0.16465598 0.003604883
## FTA           0.5256698 0.16188731    0.13968994 0.21637171 0.043542668
## SeasonEnd     0.7238690 0.00000000    0.21455487 0.20416249 0.267127100
## X3PA          0.6778031 0.08328625    0.14165189 0.23403142 0.223060547
## ORB           0.5459866 0.09573676    0.20095724 0.20110479 0.269179339
## STL           0.4640752 0.11619440    0.15311931 0.12045594 0.331127910
## TOV           0.0000000 0.24318588    0.14045929 0.24156872 0.190956902
## W             0.2431859 0.00000000    0.15119487 0.20392100 0.470897497
## Team.fctr.num 0.1404593 0.15119487    0.00000000 0.07311722 0.058154820
## BLK           0.2415687 0.20392100    0.07311722 0.00000000 0.242990816
## DRB           0.1909569 0.47089750    0.05815482 0.24299082 0.000000000
## [1] "cor(FT, FTA)=0.9505"
```

![](NBA_PTS_files/figure-html/remove_correlated_features-5.png) 

```
## [1] "cor(PTS, FT)=0.6975"
## [1] "cor(PTS, FTA)=0.6551"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in mydelete_cor_features(): Dropping FTA as a feature
```

![](NBA_PTS_files/figure-html/remove_correlated_features-6.png) 

```
##                          id       cor.y  cor.y.abs
## FG                       FG  0.94195473 0.94195473
## X2P                     X2P  0.82572457 0.82572457
## FGA                     FGA  0.79397947 0.79397947
## AST                     AST  0.75988915 0.75988915
## FT                       FT  0.69748385 0.69748385
## SeasonEnd         SeasonEnd -0.63952773 0.63952773
## X3PA                   X3PA -0.51519812 0.51519812
## ORB                     ORB  0.49692073 0.49692073
## STL                     STL  0.43099026 0.43099026
## TOV                     TOV  0.42713832 0.42713832
## W                         W  0.29882561 0.29882561
## Team.fctr.num Team.fctr.num -0.20008998 0.20008998
## BLK                     BLK  0.15205537 0.15205537
## DRB                     DRB  0.09029137 0.09029137
##                         FG         X2P         FGA         AST
## FG             1.000000000  0.94291996  0.87966291  0.81226652
## X2P            0.942919965  1.00000000  0.83827486  0.77711327
## FGA            0.879662910  0.83827486  1.00000000  0.62911736
## AST            0.812266519  0.77711327  0.62911736  1.00000000
## FT             0.538342589  0.57429395  0.39169143  0.44720848
## SeasonEnd     -0.761598813 -0.86548930 -0.68490478 -0.67212300
## X3PA          -0.688763042 -0.88859996 -0.60559564 -0.59278305
## ORB            0.598591143  0.68311802  0.73750939  0.40676555
## STL            0.469730269  0.48900271  0.48053292  0.44313827
## TOV            0.517630444  0.63771567  0.44998136  0.43053288
## W              0.190396422  0.06927898 -0.07144566  0.32005177
## Team.fctr.num -0.198789524 -0.19170714 -0.17904220 -0.17491216
## BLK            0.177502748  0.21771153  0.12109667  0.20308000
## DRB            0.008966352 -0.09869019 -0.05094546  0.05647709
##                         FT  SeasonEnd        X3PA         ORB        STL
## FG             0.538342589 -0.7615988 -0.68876304  0.59859114  0.4697303
## X2P            0.574293954 -0.8654893 -0.88859996  0.68311802  0.4890027
## FGA            0.391691425 -0.6849048 -0.60559564  0.73750939  0.4805329
## AST            0.447208477 -0.6721230 -0.59278305  0.40676555  0.4431383
## FT             1.000000000 -0.5136992 -0.51784957  0.39339870  0.3231570
## SeasonEnd     -0.513699180  1.0000000  0.85055227 -0.70201284 -0.5020598
## X3PA          -0.517849574  0.8505523  1.00000000 -0.64917273 -0.4090916
## ORB            0.393398702 -0.7020128 -0.64917273  1.00000000  0.4929415
## STL            0.323156975 -0.5020598 -0.40909163  0.49294148  1.0000000
## TOV            0.438684976 -0.7238690 -0.67780315  0.54598664  0.4640752
## W              0.204906000  0.0000000  0.08328625 -0.09573676  0.1161944
## Team.fctr.num -0.154385609  0.2145549  0.14165189 -0.20095724 -0.1531193
## BLK            0.164655982 -0.2041625 -0.23403142  0.20110479  0.1204559
## DRB            0.003604883  0.2671271  0.22306055 -0.26917934 -0.3311279
##                      TOV           W Team.fctr.num         BLK
## FG             0.5176304  0.19039642   -0.19878952  0.17750275
## X2P            0.6377157  0.06927898   -0.19170714  0.21771153
## FGA            0.4499814 -0.07144566   -0.17904220  0.12109667
## AST            0.4305329  0.32005177   -0.17491216  0.20308000
## FT             0.4386850  0.20490600   -0.15438561  0.16465598
## SeasonEnd     -0.7238690  0.00000000    0.21455487 -0.20416249
## X3PA          -0.6778031  0.08328625    0.14165189 -0.23403142
## ORB            0.5459866 -0.09573676   -0.20095724  0.20110479
## STL            0.4640752  0.11619440   -0.15311931  0.12045594
## TOV            1.0000000 -0.24318588   -0.14045929  0.24156872
## W             -0.2431859  1.00000000   -0.15119487  0.20392100
## Team.fctr.num -0.1404593 -0.15119487    1.00000000 -0.07311722
## BLK            0.2415687  0.20392100   -0.07311722  1.00000000
## DRB           -0.1909569  0.47089750   -0.05815482  0.24299082
##                        DRB
## FG             0.008966352
## X2P           -0.098690188
## FGA           -0.050945458
## AST            0.056477086
## FT             0.003604883
## SeasonEnd      0.267127100
## X3PA           0.223060547
## ORB           -0.269179339
## STL           -0.331127910
## TOV           -0.190956902
## W              0.470897497
## Team.fctr.num -0.058154820
## BLK            0.242990816
## DRB            1.000000000
##                        FG        X2P        FGA        AST          FT
## FG            0.000000000 0.94291996 0.87966291 0.81226652 0.538342589
## X2P           0.942919965 0.00000000 0.83827486 0.77711327 0.574293954
## FGA           0.879662910 0.83827486 0.00000000 0.62911736 0.391691425
## AST           0.812266519 0.77711327 0.62911736 0.00000000 0.447208477
## FT            0.538342589 0.57429395 0.39169143 0.44720848 0.000000000
## SeasonEnd     0.761598813 0.86548930 0.68490478 0.67212300 0.513699180
## X3PA          0.688763042 0.88859996 0.60559564 0.59278305 0.517849574
## ORB           0.598591143 0.68311802 0.73750939 0.40676555 0.393398702
## STL           0.469730269 0.48900271 0.48053292 0.44313827 0.323156975
## TOV           0.517630444 0.63771567 0.44998136 0.43053288 0.438684976
## W             0.190396422 0.06927898 0.07144566 0.32005177 0.204906000
## Team.fctr.num 0.198789524 0.19170714 0.17904220 0.17491216 0.154385609
## BLK           0.177502748 0.21771153 0.12109667 0.20308000 0.164655982
## DRB           0.008966352 0.09869019 0.05094546 0.05647709 0.003604883
##               SeasonEnd       X3PA        ORB       STL       TOV
## FG            0.7615988 0.68876304 0.59859114 0.4697303 0.5176304
## X2P           0.8654893 0.88859996 0.68311802 0.4890027 0.6377157
## FGA           0.6849048 0.60559564 0.73750939 0.4805329 0.4499814
## AST           0.6721230 0.59278305 0.40676555 0.4431383 0.4305329
## FT            0.5136992 0.51784957 0.39339870 0.3231570 0.4386850
## SeasonEnd     0.0000000 0.85055227 0.70201284 0.5020598 0.7238690
## X3PA          0.8505523 0.00000000 0.64917273 0.4090916 0.6778031
## ORB           0.7020128 0.64917273 0.00000000 0.4929415 0.5459866
## STL           0.5020598 0.40909163 0.49294148 0.0000000 0.4640752
## TOV           0.7238690 0.67780315 0.54598664 0.4640752 0.0000000
## W             0.0000000 0.08328625 0.09573676 0.1161944 0.2431859
## Team.fctr.num 0.2145549 0.14165189 0.20095724 0.1531193 0.1404593
## BLK           0.2041625 0.23403142 0.20110479 0.1204559 0.2415687
## DRB           0.2671271 0.22306055 0.26917934 0.3311279 0.1909569
##                        W Team.fctr.num        BLK         DRB
## FG            0.19039642    0.19878952 0.17750275 0.008966352
## X2P           0.06927898    0.19170714 0.21771153 0.098690188
## FGA           0.07144566    0.17904220 0.12109667 0.050945458
## AST           0.32005177    0.17491216 0.20308000 0.056477086
## FT            0.20490600    0.15438561 0.16465598 0.003604883
## SeasonEnd     0.00000000    0.21455487 0.20416249 0.267127100
## X3PA          0.08328625    0.14165189 0.23403142 0.223060547
## ORB           0.09573676    0.20095724 0.20110479 0.269179339
## STL           0.11619440    0.15311931 0.12045594 0.331127910
## TOV           0.24318588    0.14045929 0.24156872 0.190956902
## W             0.00000000    0.15119487 0.20392100 0.470897497
## Team.fctr.num 0.15119487    0.00000000 0.07311722 0.058154820
## BLK           0.20392100    0.07311722 0.00000000 0.242990816
## DRB           0.47089750    0.05815482 0.24299082 0.000000000
## [1] "cor(FG, X2P)=0.9429"
```

![](NBA_PTS_files/figure-html/remove_correlated_features-7.png) 

```
## [1] "cor(PTS, FG)=0.9420"
## [1] "cor(PTS, X2P)=0.8257"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in mydelete_cor_features(): Dropping X2P as a feature
```

![](NBA_PTS_files/figure-html/remove_correlated_features-8.png) 

```
##                          id       cor.y  cor.y.abs
## FG                       FG  0.94195473 0.94195473
## FGA                     FGA  0.79397947 0.79397947
## AST                     AST  0.75988915 0.75988915
## FT                       FT  0.69748385 0.69748385
## SeasonEnd         SeasonEnd -0.63952773 0.63952773
## X3PA                   X3PA -0.51519812 0.51519812
## ORB                     ORB  0.49692073 0.49692073
## STL                     STL  0.43099026 0.43099026
## TOV                     TOV  0.42713832 0.42713832
## W                         W  0.29882561 0.29882561
## Team.fctr.num Team.fctr.num -0.20008998 0.20008998
## BLK                     BLK  0.15205537 0.15205537
## DRB                     DRB  0.09029137 0.09029137
##                         FG         FGA         AST           FT  SeasonEnd
## FG             1.000000000  0.87966291  0.81226652  0.538342589 -0.7615988
## FGA            0.879662910  1.00000000  0.62911736  0.391691425 -0.6849048
## AST            0.812266519  0.62911736  1.00000000  0.447208477 -0.6721230
## FT             0.538342589  0.39169143  0.44720848  1.000000000 -0.5136992
## SeasonEnd     -0.761598813 -0.68490478 -0.67212300 -0.513699180  1.0000000
## X3PA          -0.688763042 -0.60559564 -0.59278305 -0.517849574  0.8505523
## ORB            0.598591143  0.73750939  0.40676555  0.393398702 -0.7020128
## STL            0.469730269  0.48053292  0.44313827  0.323156975 -0.5020598
## TOV            0.517630444  0.44998136  0.43053288  0.438684976 -0.7238690
## W              0.190396422 -0.07144566  0.32005177  0.204906000  0.0000000
## Team.fctr.num -0.198789524 -0.17904220 -0.17491216 -0.154385609  0.2145549
## BLK            0.177502748  0.12109667  0.20308000  0.164655982 -0.2041625
## DRB            0.008966352 -0.05094546  0.05647709  0.003604883  0.2671271
##                      X3PA         ORB        STL        TOV           W
## FG            -0.68876304  0.59859114  0.4697303  0.5176304  0.19039642
## FGA           -0.60559564  0.73750939  0.4805329  0.4499814 -0.07144566
## AST           -0.59278305  0.40676555  0.4431383  0.4305329  0.32005177
## FT            -0.51784957  0.39339870  0.3231570  0.4386850  0.20490600
## SeasonEnd      0.85055227 -0.70201284 -0.5020598 -0.7238690  0.00000000
## X3PA           1.00000000 -0.64917273 -0.4090916 -0.6778031  0.08328625
## ORB           -0.64917273  1.00000000  0.4929415  0.5459866 -0.09573676
## STL           -0.40909163  0.49294148  1.0000000  0.4640752  0.11619440
## TOV           -0.67780315  0.54598664  0.4640752  1.0000000 -0.24318588
## W              0.08328625 -0.09573676  0.1161944 -0.2431859  1.00000000
## Team.fctr.num  0.14165189 -0.20095724 -0.1531193 -0.1404593 -0.15119487
## BLK           -0.23403142  0.20110479  0.1204559  0.2415687  0.20392100
## DRB            0.22306055 -0.26917934 -0.3311279 -0.1909569  0.47089750
##               Team.fctr.num         BLK          DRB
## FG              -0.19878952  0.17750275  0.008966352
## FGA             -0.17904220  0.12109667 -0.050945458
## AST             -0.17491216  0.20308000  0.056477086
## FT              -0.15438561  0.16465598  0.003604883
## SeasonEnd        0.21455487 -0.20416249  0.267127100
## X3PA             0.14165189 -0.23403142  0.223060547
## ORB             -0.20095724  0.20110479 -0.269179339
## STL             -0.15311931  0.12045594 -0.331127910
## TOV             -0.14045929  0.24156872 -0.190956902
## W               -0.15119487  0.20392100  0.470897497
## Team.fctr.num    1.00000000 -0.07311722 -0.058154820
## BLK             -0.07311722  1.00000000  0.242990816
## DRB             -0.05815482  0.24299082  1.000000000
##                        FG        FGA        AST          FT SeasonEnd
## FG            0.000000000 0.87966291 0.81226652 0.538342589 0.7615988
## FGA           0.879662910 0.00000000 0.62911736 0.391691425 0.6849048
## AST           0.812266519 0.62911736 0.00000000 0.447208477 0.6721230
## FT            0.538342589 0.39169143 0.44720848 0.000000000 0.5136992
## SeasonEnd     0.761598813 0.68490478 0.67212300 0.513699180 0.0000000
## X3PA          0.688763042 0.60559564 0.59278305 0.517849574 0.8505523
## ORB           0.598591143 0.73750939 0.40676555 0.393398702 0.7020128
## STL           0.469730269 0.48053292 0.44313827 0.323156975 0.5020598
## TOV           0.517630444 0.44998136 0.43053288 0.438684976 0.7238690
## W             0.190396422 0.07144566 0.32005177 0.204906000 0.0000000
## Team.fctr.num 0.198789524 0.17904220 0.17491216 0.154385609 0.2145549
## BLK           0.177502748 0.12109667 0.20308000 0.164655982 0.2041625
## DRB           0.008966352 0.05094546 0.05647709 0.003604883 0.2671271
##                     X3PA        ORB       STL       TOV          W
## FG            0.68876304 0.59859114 0.4697303 0.5176304 0.19039642
## FGA           0.60559564 0.73750939 0.4805329 0.4499814 0.07144566
## AST           0.59278305 0.40676555 0.4431383 0.4305329 0.32005177
## FT            0.51784957 0.39339870 0.3231570 0.4386850 0.20490600
## SeasonEnd     0.85055227 0.70201284 0.5020598 0.7238690 0.00000000
## X3PA          0.00000000 0.64917273 0.4090916 0.6778031 0.08328625
## ORB           0.64917273 0.00000000 0.4929415 0.5459866 0.09573676
## STL           0.40909163 0.49294148 0.0000000 0.4640752 0.11619440
## TOV           0.67780315 0.54598664 0.4640752 0.0000000 0.24318588
## W             0.08328625 0.09573676 0.1161944 0.2431859 0.00000000
## Team.fctr.num 0.14165189 0.20095724 0.1531193 0.1404593 0.15119487
## BLK           0.23403142 0.20110479 0.1204559 0.2415687 0.20392100
## DRB           0.22306055 0.26917934 0.3311279 0.1909569 0.47089750
##               Team.fctr.num        BLK         DRB
## FG               0.19878952 0.17750275 0.008966352
## FGA              0.17904220 0.12109667 0.050945458
## AST              0.17491216 0.20308000 0.056477086
## FT               0.15438561 0.16465598 0.003604883
## SeasonEnd        0.21455487 0.20416249 0.267127100
## X3PA             0.14165189 0.23403142 0.223060547
## ORB              0.20095724 0.20110479 0.269179339
## STL              0.15311931 0.12045594 0.331127910
## TOV              0.14045929 0.24156872 0.190956902
## W                0.15119487 0.20392100 0.470897497
## Team.fctr.num    0.00000000 0.07311722 0.058154820
## BLK              0.07311722 0.00000000 0.242990816
## DRB              0.05815482 0.24299082 0.000000000
## [1] "cor(FG, FGA)=0.8797"
```

![](NBA_PTS_files/figure-html/remove_correlated_features-9.png) 

```
## [1] "cor(PTS, FG)=0.9420"
## [1] "cor(PTS, FGA)=0.7940"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in mydelete_cor_features(): Dropping FGA as a feature
```

![](NBA_PTS_files/figure-html/remove_correlated_features-10.png) 

```
##                          id       cor.y  cor.y.abs
## FG                       FG  0.94195473 0.94195473
## AST                     AST  0.75988915 0.75988915
## FT                       FT  0.69748385 0.69748385
## SeasonEnd         SeasonEnd -0.63952773 0.63952773
## X3PA                   X3PA -0.51519812 0.51519812
## ORB                     ORB  0.49692073 0.49692073
## STL                     STL  0.43099026 0.43099026
## TOV                     TOV  0.42713832 0.42713832
## W                         W  0.29882561 0.29882561
## Team.fctr.num Team.fctr.num -0.20008998 0.20008998
## BLK                     BLK  0.15205537 0.15205537
## DRB                     DRB  0.09029137 0.09029137
##                         FG         AST           FT  SeasonEnd        X3PA
## FG             1.000000000  0.81226652  0.538342589 -0.7615988 -0.68876304
## AST            0.812266519  1.00000000  0.447208477 -0.6721230 -0.59278305
## FT             0.538342589  0.44720848  1.000000000 -0.5136992 -0.51784957
## SeasonEnd     -0.761598813 -0.67212300 -0.513699180  1.0000000  0.85055227
## X3PA          -0.688763042 -0.59278305 -0.517849574  0.8505523  1.00000000
## ORB            0.598591143  0.40676555  0.393398702 -0.7020128 -0.64917273
## STL            0.469730269  0.44313827  0.323156975 -0.5020598 -0.40909163
## TOV            0.517630444  0.43053288  0.438684976 -0.7238690 -0.67780315
## W              0.190396422  0.32005177  0.204906000  0.0000000  0.08328625
## Team.fctr.num -0.198789524 -0.17491216 -0.154385609  0.2145549  0.14165189
## BLK            0.177502748  0.20308000  0.164655982 -0.2041625 -0.23403142
## DRB            0.008966352  0.05647709  0.003604883  0.2671271  0.22306055
##                       ORB        STL        TOV           W Team.fctr.num
## FG             0.59859114  0.4697303  0.5176304  0.19039642   -0.19878952
## AST            0.40676555  0.4431383  0.4305329  0.32005177   -0.17491216
## FT             0.39339870  0.3231570  0.4386850  0.20490600   -0.15438561
## SeasonEnd     -0.70201284 -0.5020598 -0.7238690  0.00000000    0.21455487
## X3PA          -0.64917273 -0.4090916 -0.6778031  0.08328625    0.14165189
## ORB            1.00000000  0.4929415  0.5459866 -0.09573676   -0.20095724
## STL            0.49294148  1.0000000  0.4640752  0.11619440   -0.15311931
## TOV            0.54598664  0.4640752  1.0000000 -0.24318588   -0.14045929
## W             -0.09573676  0.1161944 -0.2431859  1.00000000   -0.15119487
## Team.fctr.num -0.20095724 -0.1531193 -0.1404593 -0.15119487    1.00000000
## BLK            0.20110479  0.1204559  0.2415687  0.20392100   -0.07311722
## DRB           -0.26917934 -0.3311279 -0.1909569  0.47089750   -0.05815482
##                       BLK          DRB
## FG             0.17750275  0.008966352
## AST            0.20308000  0.056477086
## FT             0.16465598  0.003604883
## SeasonEnd     -0.20416249  0.267127100
## X3PA          -0.23403142  0.223060547
## ORB            0.20110479 -0.269179339
## STL            0.12045594 -0.331127910
## TOV            0.24156872 -0.190956902
## W              0.20392100  0.470897497
## Team.fctr.num -0.07311722 -0.058154820
## BLK            1.00000000  0.242990816
## DRB            0.24299082  1.000000000
##                        FG        AST          FT SeasonEnd       X3PA
## FG            0.000000000 0.81226652 0.538342589 0.7615988 0.68876304
## AST           0.812266519 0.00000000 0.447208477 0.6721230 0.59278305
## FT            0.538342589 0.44720848 0.000000000 0.5136992 0.51784957
## SeasonEnd     0.761598813 0.67212300 0.513699180 0.0000000 0.85055227
## X3PA          0.688763042 0.59278305 0.517849574 0.8505523 0.00000000
## ORB           0.598591143 0.40676555 0.393398702 0.7020128 0.64917273
## STL           0.469730269 0.44313827 0.323156975 0.5020598 0.40909163
## TOV           0.517630444 0.43053288 0.438684976 0.7238690 0.67780315
## W             0.190396422 0.32005177 0.204906000 0.0000000 0.08328625
## Team.fctr.num 0.198789524 0.17491216 0.154385609 0.2145549 0.14165189
## BLK           0.177502748 0.20308000 0.164655982 0.2041625 0.23403142
## DRB           0.008966352 0.05647709 0.003604883 0.2671271 0.22306055
##                      ORB       STL       TOV          W Team.fctr.num
## FG            0.59859114 0.4697303 0.5176304 0.19039642    0.19878952
## AST           0.40676555 0.4431383 0.4305329 0.32005177    0.17491216
## FT            0.39339870 0.3231570 0.4386850 0.20490600    0.15438561
## SeasonEnd     0.70201284 0.5020598 0.7238690 0.00000000    0.21455487
## X3PA          0.64917273 0.4090916 0.6778031 0.08328625    0.14165189
## ORB           0.00000000 0.4929415 0.5459866 0.09573676    0.20095724
## STL           0.49294148 0.0000000 0.4640752 0.11619440    0.15311931
## TOV           0.54598664 0.4640752 0.0000000 0.24318588    0.14045929
## W             0.09573676 0.1161944 0.2431859 0.00000000    0.15119487
## Team.fctr.num 0.20095724 0.1531193 0.1404593 0.15119487    0.00000000
## BLK           0.20110479 0.1204559 0.2415687 0.20392100    0.07311722
## DRB           0.26917934 0.3311279 0.1909569 0.47089750    0.05815482
##                      BLK         DRB
## FG            0.17750275 0.008966352
## AST           0.20308000 0.056477086
## FT            0.16465598 0.003604883
## SeasonEnd     0.20416249 0.267127100
## X3PA          0.23403142 0.223060547
## ORB           0.20110479 0.269179339
## STL           0.12045594 0.331127910
## TOV           0.24156872 0.190956902
## W             0.20392100 0.470897497
## Team.fctr.num 0.07311722 0.058154820
## BLK           0.00000000 0.242990816
## DRB           0.24299082 0.000000000
## [1] "cor(SeasonEnd, X3PA)=0.8506"
```

![](NBA_PTS_files/figure-html/remove_correlated_features-11.png) 

```
## [1] "cor(PTS, SeasonEnd)=-0.6395"
## [1] "cor(PTS, X3PA)=-0.5152"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in mydelete_cor_features(): Dropping X3PA as a feature
```

![](NBA_PTS_files/figure-html/remove_correlated_features-12.png) 

```
##                          id       cor.y  cor.y.abs
## FG                       FG  0.94195473 0.94195473
## AST                     AST  0.75988915 0.75988915
## FT                       FT  0.69748385 0.69748385
## SeasonEnd         SeasonEnd -0.63952773 0.63952773
## ORB                     ORB  0.49692073 0.49692073
## STL                     STL  0.43099026 0.43099026
## TOV                     TOV  0.42713832 0.42713832
## W                         W  0.29882561 0.29882561
## Team.fctr.num Team.fctr.num -0.20008998 0.20008998
## BLK                     BLK  0.15205537 0.15205537
## DRB                     DRB  0.09029137 0.09029137
##                         FG         AST           FT  SeasonEnd         ORB
## FG             1.000000000  0.81226652  0.538342589 -0.7615988  0.59859114
## AST            0.812266519  1.00000000  0.447208477 -0.6721230  0.40676555
## FT             0.538342589  0.44720848  1.000000000 -0.5136992  0.39339870
## SeasonEnd     -0.761598813 -0.67212300 -0.513699180  1.0000000 -0.70201284
## ORB            0.598591143  0.40676555  0.393398702 -0.7020128  1.00000000
## STL            0.469730269  0.44313827  0.323156975 -0.5020598  0.49294148
## TOV            0.517630444  0.43053288  0.438684976 -0.7238690  0.54598664
## W              0.190396422  0.32005177  0.204906000  0.0000000 -0.09573676
## Team.fctr.num -0.198789524 -0.17491216 -0.154385609  0.2145549 -0.20095724
## BLK            0.177502748  0.20308000  0.164655982 -0.2041625  0.20110479
## DRB            0.008966352  0.05647709  0.003604883  0.2671271 -0.26917934
##                      STL        TOV           W Team.fctr.num         BLK
## FG             0.4697303  0.5176304  0.19039642   -0.19878952  0.17750275
## AST            0.4431383  0.4305329  0.32005177   -0.17491216  0.20308000
## FT             0.3231570  0.4386850  0.20490600   -0.15438561  0.16465598
## SeasonEnd     -0.5020598 -0.7238690  0.00000000    0.21455487 -0.20416249
## ORB            0.4929415  0.5459866 -0.09573676   -0.20095724  0.20110479
## STL            1.0000000  0.4640752  0.11619440   -0.15311931  0.12045594
## TOV            0.4640752  1.0000000 -0.24318588   -0.14045929  0.24156872
## W              0.1161944 -0.2431859  1.00000000   -0.15119487  0.20392100
## Team.fctr.num -0.1531193 -0.1404593 -0.15119487    1.00000000 -0.07311722
## BLK            0.1204559  0.2415687  0.20392100   -0.07311722  1.00000000
## DRB           -0.3311279 -0.1909569  0.47089750   -0.05815482  0.24299082
##                        DRB
## FG             0.008966352
## AST            0.056477086
## FT             0.003604883
## SeasonEnd      0.267127100
## ORB           -0.269179339
## STL           -0.331127910
## TOV           -0.190956902
## W              0.470897497
## Team.fctr.num -0.058154820
## BLK            0.242990816
## DRB            1.000000000
##                        FG        AST          FT SeasonEnd        ORB
## FG            0.000000000 0.81226652 0.538342589 0.7615988 0.59859114
## AST           0.812266519 0.00000000 0.447208477 0.6721230 0.40676555
## FT            0.538342589 0.44720848 0.000000000 0.5136992 0.39339870
## SeasonEnd     0.761598813 0.67212300 0.513699180 0.0000000 0.70201284
## ORB           0.598591143 0.40676555 0.393398702 0.7020128 0.00000000
## STL           0.469730269 0.44313827 0.323156975 0.5020598 0.49294148
## TOV           0.517630444 0.43053288 0.438684976 0.7238690 0.54598664
## W             0.190396422 0.32005177 0.204906000 0.0000000 0.09573676
## Team.fctr.num 0.198789524 0.17491216 0.154385609 0.2145549 0.20095724
## BLK           0.177502748 0.20308000 0.164655982 0.2041625 0.20110479
## DRB           0.008966352 0.05647709 0.003604883 0.2671271 0.26917934
##                     STL       TOV          W Team.fctr.num        BLK
## FG            0.4697303 0.5176304 0.19039642    0.19878952 0.17750275
## AST           0.4431383 0.4305329 0.32005177    0.17491216 0.20308000
## FT            0.3231570 0.4386850 0.20490600    0.15438561 0.16465598
## SeasonEnd     0.5020598 0.7238690 0.00000000    0.21455487 0.20416249
## ORB           0.4929415 0.5459866 0.09573676    0.20095724 0.20110479
## STL           0.0000000 0.4640752 0.11619440    0.15311931 0.12045594
## TOV           0.4640752 0.0000000 0.24318588    0.14045929 0.24156872
## W             0.1161944 0.2431859 0.00000000    0.15119487 0.20392100
## Team.fctr.num 0.1531193 0.1404593 0.15119487    0.00000000 0.07311722
## BLK           0.1204559 0.2415687 0.20392100    0.07311722 0.00000000
## DRB           0.3311279 0.1909569 0.47089750    0.05815482 0.24299082
##                       DRB
## FG            0.008966352
## AST           0.056477086
## FT            0.003604883
## SeasonEnd     0.267127100
## ORB           0.269179339
## STL           0.331127910
## TOV           0.190956902
## W             0.470897497
## Team.fctr.num 0.058154820
## BLK           0.242990816
## DRB           0.000000000
## [1] "cor(FG, AST)=0.8123"
```

![](NBA_PTS_files/figure-html/remove_correlated_features-13.png) 

```
## [1] "cor(PTS, FG)=0.9420"
## [1] "cor(PTS, AST)=0.7599"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in mydelete_cor_features(): Dropping AST as a feature
```

![](NBA_PTS_files/figure-html/remove_correlated_features-14.png) 

```
##                          id       cor.y  cor.y.abs
## FG                       FG  0.94195473 0.94195473
## FT                       FT  0.69748385 0.69748385
## SeasonEnd         SeasonEnd -0.63952773 0.63952773
## ORB                     ORB  0.49692073 0.49692073
## STL                     STL  0.43099026 0.43099026
## TOV                     TOV  0.42713832 0.42713832
## W                         W  0.29882561 0.29882561
## Team.fctr.num Team.fctr.num -0.20008998 0.20008998
## BLK                     BLK  0.15205537 0.15205537
## DRB                     DRB  0.09029137 0.09029137
##                         FG           FT  SeasonEnd         ORB        STL
## FG             1.000000000  0.538342589 -0.7615988  0.59859114  0.4697303
## FT             0.538342589  1.000000000 -0.5136992  0.39339870  0.3231570
## SeasonEnd     -0.761598813 -0.513699180  1.0000000 -0.70201284 -0.5020598
## ORB            0.598591143  0.393398702 -0.7020128  1.00000000  0.4929415
## STL            0.469730269  0.323156975 -0.5020598  0.49294148  1.0000000
## TOV            0.517630444  0.438684976 -0.7238690  0.54598664  0.4640752
## W              0.190396422  0.204906000  0.0000000 -0.09573676  0.1161944
## Team.fctr.num -0.198789524 -0.154385609  0.2145549 -0.20095724 -0.1531193
## BLK            0.177502748  0.164655982 -0.2041625  0.20110479  0.1204559
## DRB            0.008966352  0.003604883  0.2671271 -0.26917934 -0.3311279
##                      TOV           W Team.fctr.num         BLK
## FG             0.5176304  0.19039642   -0.19878952  0.17750275
## FT             0.4386850  0.20490600   -0.15438561  0.16465598
## SeasonEnd     -0.7238690  0.00000000    0.21455487 -0.20416249
## ORB            0.5459866 -0.09573676   -0.20095724  0.20110479
## STL            0.4640752  0.11619440   -0.15311931  0.12045594
## TOV            1.0000000 -0.24318588   -0.14045929  0.24156872
## W             -0.2431859  1.00000000   -0.15119487  0.20392100
## Team.fctr.num -0.1404593 -0.15119487    1.00000000 -0.07311722
## BLK            0.2415687  0.20392100   -0.07311722  1.00000000
## DRB           -0.1909569  0.47089750   -0.05815482  0.24299082
##                        DRB
## FG             0.008966352
## FT             0.003604883
## SeasonEnd      0.267127100
## ORB           -0.269179339
## STL           -0.331127910
## TOV           -0.190956902
## W              0.470897497
## Team.fctr.num -0.058154820
## BLK            0.242990816
## DRB            1.000000000
##                        FG          FT SeasonEnd        ORB       STL
## FG            0.000000000 0.538342589 0.7615988 0.59859114 0.4697303
## FT            0.538342589 0.000000000 0.5136992 0.39339870 0.3231570
## SeasonEnd     0.761598813 0.513699180 0.0000000 0.70201284 0.5020598
## ORB           0.598591143 0.393398702 0.7020128 0.00000000 0.4929415
## STL           0.469730269 0.323156975 0.5020598 0.49294148 0.0000000
## TOV           0.517630444 0.438684976 0.7238690 0.54598664 0.4640752
## W             0.190396422 0.204906000 0.0000000 0.09573676 0.1161944
## Team.fctr.num 0.198789524 0.154385609 0.2145549 0.20095724 0.1531193
## BLK           0.177502748 0.164655982 0.2041625 0.20110479 0.1204559
## DRB           0.008966352 0.003604883 0.2671271 0.26917934 0.3311279
##                     TOV          W Team.fctr.num        BLK         DRB
## FG            0.5176304 0.19039642    0.19878952 0.17750275 0.008966352
## FT            0.4386850 0.20490600    0.15438561 0.16465598 0.003604883
## SeasonEnd     0.7238690 0.00000000    0.21455487 0.20416249 0.267127100
## ORB           0.5459866 0.09573676    0.20095724 0.20110479 0.269179339
## STL           0.4640752 0.11619440    0.15311931 0.12045594 0.331127910
## TOV           0.0000000 0.24318588    0.14045929 0.24156872 0.190956902
## W             0.2431859 0.00000000    0.15119487 0.20392100 0.470897497
## Team.fctr.num 0.1404593 0.15119487    0.00000000 0.07311722 0.058154820
## BLK           0.2415687 0.20392100    0.07311722 0.00000000 0.242990816
## DRB           0.1909569 0.47089750    0.05815482 0.24299082 0.000000000
## [1] "cor(FG, SeasonEnd)=-0.7616"
```

![](NBA_PTS_files/figure-html/remove_correlated_features-15.png) 

```
## [1] "cor(PTS, FG)=0.9420"
## [1] "cor(PTS, SeasonEnd)=-0.6395"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in mydelete_cor_features(): Dropping SeasonEnd as a feature
```

![](NBA_PTS_files/figure-html/remove_correlated_features-16.png) 

```
##                          id       cor.y  cor.y.abs
## FG                       FG  0.94195473 0.94195473
## FT                       FT  0.69748385 0.69748385
## ORB                     ORB  0.49692073 0.49692073
## STL                     STL  0.43099026 0.43099026
## TOV                     TOV  0.42713832 0.42713832
## W                         W  0.29882561 0.29882561
## Team.fctr.num Team.fctr.num -0.20008998 0.20008998
## BLK                     BLK  0.15205537 0.15205537
## DRB                     DRB  0.09029137 0.09029137
##                         FG           FT         ORB        STL        TOV
## FG             1.000000000  0.538342589  0.59859114  0.4697303  0.5176304
## FT             0.538342589  1.000000000  0.39339870  0.3231570  0.4386850
## ORB            0.598591143  0.393398702  1.00000000  0.4929415  0.5459866
## STL            0.469730269  0.323156975  0.49294148  1.0000000  0.4640752
## TOV            0.517630444  0.438684976  0.54598664  0.4640752  1.0000000
## W              0.190396422  0.204906000 -0.09573676  0.1161944 -0.2431859
## Team.fctr.num -0.198789524 -0.154385609 -0.20095724 -0.1531193 -0.1404593
## BLK            0.177502748  0.164655982  0.20110479  0.1204559  0.2415687
## DRB            0.008966352  0.003604883 -0.26917934 -0.3311279 -0.1909569
##                         W Team.fctr.num         BLK          DRB
## FG             0.19039642   -0.19878952  0.17750275  0.008966352
## FT             0.20490600   -0.15438561  0.16465598  0.003604883
## ORB           -0.09573676   -0.20095724  0.20110479 -0.269179339
## STL            0.11619440   -0.15311931  0.12045594 -0.331127910
## TOV           -0.24318588   -0.14045929  0.24156872 -0.190956902
## W              1.00000000   -0.15119487  0.20392100  0.470897497
## Team.fctr.num -0.15119487    1.00000000 -0.07311722 -0.058154820
## BLK            0.20392100   -0.07311722  1.00000000  0.242990816
## DRB            0.47089750   -0.05815482  0.24299082  1.000000000
##                        FG          FT        ORB       STL       TOV
## FG            0.000000000 0.538342589 0.59859114 0.4697303 0.5176304
## FT            0.538342589 0.000000000 0.39339870 0.3231570 0.4386850
## ORB           0.598591143 0.393398702 0.00000000 0.4929415 0.5459866
## STL           0.469730269 0.323156975 0.49294148 0.0000000 0.4640752
## TOV           0.517630444 0.438684976 0.54598664 0.4640752 0.0000000
## W             0.190396422 0.204906000 0.09573676 0.1161944 0.2431859
## Team.fctr.num 0.198789524 0.154385609 0.20095724 0.1531193 0.1404593
## BLK           0.177502748 0.164655982 0.20110479 0.1204559 0.2415687
## DRB           0.008966352 0.003604883 0.26917934 0.3311279 0.1909569
##                        W Team.fctr.num        BLK         DRB
## FG            0.19039642    0.19878952 0.17750275 0.008966352
## FT            0.20490600    0.15438561 0.16465598 0.003604883
## ORB           0.09573676    0.20095724 0.20110479 0.269179339
## STL           0.11619440    0.15311931 0.12045594 0.331127910
## TOV           0.24318588    0.14045929 0.24156872 0.190956902
## W             0.00000000    0.15119487 0.20392100 0.470897497
## Team.fctr.num 0.15119487    0.00000000 0.07311722 0.058154820
## BLK           0.20392100    0.07311722 0.00000000 0.242990816
## DRB           0.47089750    0.05815482 0.24299082 0.000000000
##               id       cor.y  cor.y.abs cor.low
## 4             FG  0.94195473 0.94195473       1
## 14           X2P  0.82572457 0.82572457      NA
## 5            FGA  0.79397947 0.79397947      NA
## 1            AST  0.75988915 0.75988915      NA
## 15          X2PA  0.70836140 0.70836140      NA
## 6             FT  0.69748385 0.69748385       1
## 7            FTA  0.65505876 0.65505876      NA
## 8            ORB  0.49692073 0.49692073       1
## 10           STL  0.43099026 0.43099026       1
## 12           TOV  0.42713832 0.42713832       1
## 13             W  0.29882561 0.29882561       1
## 2            BLK  0.15205537 0.15205537       1
## 3            DRB  0.09029137 0.09029137       1
## 11 Team.fctr.num -0.20008998 0.20008998       1
## 16           X3P -0.48994891 0.48994891      NA
## 17          X3PA -0.51519812 0.51519812      NA
## 9      SeasonEnd -0.63952773 0.63952773      NA
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
## [1] 1347319
## [1] 0.7663011
## 
## Call:
## lm(formula = reformulate(indep_vars_vctr, response = glb_predct_var), 
##     data = fit_df)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -504.64 -136.59    0.83  130.03  613.68 
## 
## Coefficients:
##              Estimate Std. Error t value Pr(>|t|)    
## (Intercept) 2.271e+03  7.563e+01   30.03   <2e-16 ***
## FG          1.906e+00  2.354e-02   80.97   <2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 195.2 on 833 degrees of freedom
## Multiple R-squared:  0.8873,	Adjusted R-squared:  0.8871 
## F-statistic:  6557 on 1 and 833 DF,  p-value: < 2.2e-16
## 
##   feats n.fit  R.sq.fit  R.sq.OOB Adj.R.sq.fit  SSE.fit SSE.OOB
## 1    FG   835 0.8872787 0.7663011    0.8872787 31738340 1347319
##   f.score.OOB
## 1          NA
```

```r
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
## [1] 110591.8
## [1] 0.9808173
## 
## Call:
## lm(formula = reformulate(indep_vars_vctr, response = glb_predct_var), 
##     data = fit_df)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -265.55  -36.01   -0.58   40.42  178.46 
## 
## Coefficients: (1 not defined because of singularities)
##                Estimate Std. Error t value Pr(>|t|)    
## (Intercept)   2.499e+03  2.487e+02  10.048  < 2e-16 ***
## FG           -2.846e-01  3.540e-01  -0.804   0.4216    
## FG:X2P        7.700e-05  2.471e-05   3.116   0.0019 ** 
## FG:FGA       -6.030e-05  1.314e-05  -4.588 5.18e-06 ***
## FG:AST        2.769e-06  5.282e-06   0.524   0.6002    
## FG:X2PA       5.746e-05  1.423e-05   4.038 5.90e-05 ***
## FG:FTA        2.320e-04  3.250e-06  71.387  < 2e-16 ***
## FG:X3P        5.308e-04  4.408e-05  12.042  < 2e-16 ***
## FG:X3PA              NA         NA      NA       NA    
## FG:SeasonEnd  6.443e-04  1.564e-04   4.121 4.15e-05 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 58.88 on 826 degrees of freedom
## Multiple R-squared:  0.9898,	Adjusted R-squared:  0.9897 
## F-statistic: 1.005e+04 on 8 and 826 DF,  p-value: < 2.2e-16
## 
##                                                                        feats
## 2 FG, FG:X2P, FG:FGA, FG:AST, FG:X2PA, FG:FTA, FG:X3P, FG:X3PA, FG:SeasonEnd
## 1                                                                         FG
##   n.fit  R.sq.fit  R.sq.OOB Adj.R.sq.fit  SSE.fit   SSE.OOB f.score.OOB
## 2   835 0.9898302 0.9808173    0.9898302  2863458  110591.8          NA
## 1   835 0.8872787 0.7663011    0.8872787 31738340 1347318.9          NA
```

```r
# Low correlated X
ret_lst <- myrun_mdl_fn(indep_vars_vctr=subset(glb_feats_df, cor.low == 1)[, "id"],
                        fit_df=glb_entity_df, OOB_df=glb_predct_df)
```

```
## [1] 593244
## [1] 0.897099
## 
## Call:
## lm(formula = reformulate(indep_vars_vctr, response = glb_predct_var), 
##     data = fit_df)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -271.19  -77.38   -9.41   63.64  474.04 
## 
## Coefficients:
##                 Estimate Std. Error t value Pr(>|t|)    
## (Intercept)   1537.08414  109.93038  13.982  < 2e-16 ***
## FG               1.75807    0.02069  84.959  < 2e-16 ***
## FT               0.87077    0.02528  34.449  < 2e-16 ***
## ORB             -0.30190    0.03805  -7.934 6.91e-15 ***
## STL              0.19080    0.05841   3.267  0.00113 ** 
## TOV             -0.39060    0.03850 -10.145  < 2e-16 ***
## W                0.86658    0.43815   1.978  0.04828 *  
## BLK             -0.21172    0.05376  -3.938 8.90e-05 ***
## DRB              0.21760    0.04066   5.352 1.13e-07 ***
## Team.fctr.num   -0.22648    0.44412  -0.510  0.61022    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 114.5 on 825 degrees of freedom
## Multiple R-squared:  0.9616,	Adjusted R-squared:  0.9612 
## F-statistic:  2294 on 9 and 825 DF,  p-value: < 2.2e-16
## 
##                                                                        feats
## 2 FG, FG:X2P, FG:FGA, FG:AST, FG:X2PA, FG:FTA, FG:X3P, FG:X3PA, FG:SeasonEnd
## 3                          FG, FT, ORB, STL, TOV, W, BLK, DRB, Team.fctr.num
## 1                                                                         FG
##   n.fit  R.sq.fit  R.sq.OOB Adj.R.sq.fit  SSE.fit   SSE.OOB f.score.OOB
## 2   835 0.9898302 0.9808173    0.9898302  2863458  110591.8          NA
## 3   835 0.9615788 0.8970990    0.9615788 10818052  593244.0          NA
## 1   835 0.8872787 0.7663011    0.8872787 31738340 1347318.9          NA
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
## [1] 2.787599e-22
## [1] 1
```

```
## Warning in summary.lm(mdl): essentially perfect fit: summary may be
## unreliable
```

```
## Warning in summary.lm(mdl): essentially perfect fit: summary may be
## unreliable
```

```
## Warning in summary.lm(glb_mdl <<- mdl): essentially perfect fit: summary
## may be unreliable
```

```
## 
## Call:
## lm(formula = reformulate(indep_vars_vctr, response = glb_predct_var), 
##     data = fit_df)
## 
## Residuals:
##        Min         1Q     Median         3Q        Max 
## -4.233e-12 -1.654e-13 -6.010e-14  4.300e-14  2.539e-11 
## 
## Coefficients: (2 not defined because of singularities)
##                 Estimate Std. Error    t value Pr(>|t|)    
## (Intercept)   -1.825e-11  2.256e-11 -8.090e-01   0.4189    
## SeasonEnd      1.363e-14  1.111e-14  1.227e+00   0.2202    
## W              2.719e-14  6.949e-15  3.913e+00 9.89e-05 ***
## FG             3.000e+00  2.136e-15  1.405e+15  < 2e-16 ***
## FGA            3.254e-16  8.940e-16  3.640e-01   0.7160    
## X2P           -1.000e+00  2.143e-15 -4.666e+14  < 2e-16 ***
## X2PA           4.227e-17  8.405e-16  5.000e-02   0.9599    
## X3P                   NA         NA         NA       NA    
## X3PA                  NA         NA         NA       NA    
## FT             1.000e+00  7.161e-16  1.397e+15  < 2e-16 ***
## FTA           -5.625e-16  6.122e-16 -9.190e-01   0.3585    
## ORB           -2.786e-18  6.030e-16 -5.000e-03   0.9963    
## DRB            2.008e-16  5.250e-16  3.830e-01   0.7022    
## AST           -3.656e-16  3.287e-16 -1.112e+00   0.2663    
## STL            5.848e-16  6.760e-16  8.650e-01   0.3872    
## BLK            1.942e-16  5.218e-16  3.720e-01   0.7099    
## TOV           -3.790e-17  5.353e-16 -7.100e-02   0.9436    
## Team.fctr.num -9.847e-15  4.289e-15 -2.296e+00   0.0219 *  
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 1.092e-12 on 819 degrees of freedom
## Multiple R-squared:      1,	Adjusted R-squared:      1 
## F-statistic: 1.573e+31 on 15 and 819 DF,  p-value: < 2.2e-16
## 
##                                                                                               feats
## 4 SeasonEnd, W, FG, FGA, X2P, X2PA, X3P, X3PA, FT, FTA, ORB, DRB, AST, STL, BLK, TOV, Team.fctr.num
## 2                        FG, FG:X2P, FG:FGA, FG:AST, FG:X2PA, FG:FTA, FG:X3P, FG:X3PA, FG:SeasonEnd
## 3                                                 FG, FT, ORB, STL, TOV, W, BLK, DRB, Team.fctr.num
## 1                                                                                                FG
##   n.fit  R.sq.fit  R.sq.OOB Adj.R.sq.fit      SSE.fit      SSE.OOB
## 4   835 1.0000000 1.0000000    1.0000000 9.770615e-22 2.787599e-22
## 2   835 0.9898302 0.9808173    0.9898302 2.863458e+06 1.105918e+05
## 3   835 0.9615788 0.8970990    0.9615788 1.081805e+07 5.932440e+05
## 1   835 0.8872787 0.7663011    0.8872787 3.173834e+07 1.347319e+06
##   f.score.OOB
## 4          NA
## 2          NA
## 3          NA
## 1          NA
```

```r
# PTS attempt 1
ret_lst <- myrun_mdl_fn(indep_vars_vctr=c("X2PA", "X3PA", "FTA", "AST",
                                          "ORB", "DRB",
                                          "TOV", "STL", "BLK"),
                        fit_df=glb_entity_df, OOB_df=glb_predct_df)
```

```
## [1] 1082116
## [1] 0.8123019
## 
## Call:
## lm(formula = reformulate(indep_vars_vctr, response = glb_predct_var), 
##     data = fit_df)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -527.40 -119.83    7.83  120.67  564.71 
## 
## Coefficients:
##               Estimate Std. Error t value Pr(>|t|)    
## (Intercept) -2.051e+03  2.035e+02 -10.078   <2e-16 ***
## X2PA         1.043e+00  2.957e-02  35.274   <2e-16 ***
## X3PA         1.259e+00  3.843e-02  32.747   <2e-16 ***
## FTA          1.128e+00  3.373e-02  33.440   <2e-16 ***
## AST          8.858e-01  4.396e-02  20.150   <2e-16 ***
## ORB         -9.554e-01  7.792e-02 -12.261   <2e-16 ***
## DRB          3.883e-02  6.157e-02   0.631   0.5285    
## TOV         -2.475e-02  6.118e-02  -0.405   0.6859    
## STL         -1.992e-01  9.181e-02  -2.169   0.0303 *  
## BLK         -5.576e-02  8.782e-02  -0.635   0.5256    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 185.5 on 825 degrees of freedom
## Multiple R-squared:  0.8992,	Adjusted R-squared:  0.8981 
## F-statistic: 817.3 on 9 and 825 DF,  p-value: < 2.2e-16
## 
##                                                                                               feats
## 4 SeasonEnd, W, FG, FGA, X2P, X2PA, X3P, X3PA, FT, FTA, ORB, DRB, AST, STL, BLK, TOV, Team.fctr.num
## 2                        FG, FG:X2P, FG:FGA, FG:AST, FG:X2PA, FG:FTA, FG:X3P, FG:X3PA, FG:SeasonEnd
## 3                                                 FG, FT, ORB, STL, TOV, W, BLK, DRB, Team.fctr.num
## 5                                                     X2PA, X3PA, FTA, AST, ORB, DRB, TOV, STL, BLK
## 1                                                                                                FG
##   n.fit  R.sq.fit  R.sq.OOB Adj.R.sq.fit      SSE.fit      SSE.OOB
## 4   835 1.0000000 1.0000000    1.0000000 9.770615e-22 2.787599e-22
## 2   835 0.9898302 0.9808173    0.9898302 2.863458e+06 1.105918e+05
## 3   835 0.9615788 0.8970990    0.9615788 1.081805e+07 5.932440e+05
## 5   835 0.8991553 0.8123019    0.8991553 2.839431e+07 1.082116e+06
## 1   835 0.8872787 0.7663011    0.8872787 3.173834e+07 1.347319e+06
##   f.score.OOB
## 4          NA
## 2          NA
## 3          NA
## 5          NA
## 1          NA
```

```r
# PTS attempt 1 excl. insignificant feats
ret_lst <- myrun_mdl_fn(indep_vars_vctr=c("X2PA", "X3PA", "FTA", "AST",
                                          "ORB", 
                                                 "STL"),
                        fit_df=glb_entity_df, OOB_df=glb_predct_df)
```

```
## [1] 1079739
## [1] 0.8127142
## 
## Call:
## lm(formula = reformulate(indep_vars_vctr, response = glb_predct_var), 
##     data = fit_df)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -523.33 -122.02    6.93  120.68  568.26 
## 
## Coefficients:
##               Estimate Std. Error t value Pr(>|t|)    
## (Intercept) -2.033e+03  1.629e+02 -12.475  < 2e-16 ***
## X2PA         1.050e+00  2.829e-02  37.117  < 2e-16 ***
## X3PA         1.273e+00  3.441e-02  37.001  < 2e-16 ***
## FTA          1.127e+00  3.260e-02  34.581  < 2e-16 ***
## AST          8.884e-01  4.292e-02  20.701  < 2e-16 ***
## ORB         -9.743e-01  7.465e-02 -13.051  < 2e-16 ***
## STL         -2.268e-01  8.350e-02  -2.717  0.00673 ** 
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 185.3 on 828 degrees of freedom
## Multiple R-squared:  0.8991,	Adjusted R-squared:  0.8983 
## F-statistic:  1229 on 6 and 828 DF,  p-value: < 2.2e-16
## 
##                                                                                               feats
## 4 SeasonEnd, W, FG, FGA, X2P, X2PA, X3P, X3PA, FT, FTA, ORB, DRB, AST, STL, BLK, TOV, Team.fctr.num
## 2                        FG, FG:X2P, FG:FGA, FG:AST, FG:X2PA, FG:FTA, FG:X3P, FG:X3PA, FG:SeasonEnd
## 3                                                 FG, FT, ORB, STL, TOV, W, BLK, DRB, Team.fctr.num
## 6                                                                    X2PA, X3PA, FTA, AST, ORB, STL
## 5                                                     X2PA, X3PA, FTA, AST, ORB, DRB, TOV, STL, BLK
## 1                                                                                                FG
##   n.fit  R.sq.fit  R.sq.OOB Adj.R.sq.fit      SSE.fit      SSE.OOB
## 4   835 1.0000000 1.0000000    1.0000000 9.770615e-22 2.787599e-22
## 2   835 0.9898302 0.9808173    0.9898302 2.863458e+06 1.105918e+05
## 3   835 0.9615788 0.8970990    0.9615788 1.081805e+07 5.932440e+05
## 6   835 0.8990589 0.8127142    0.8990589 2.842146e+07 1.079739e+06
## 5   835 0.8991553 0.8123019    0.8991553 2.839431e+07 1.082116e+06
## 1   835 0.8872787 0.7663011    0.8872787 3.173834e+07 1.347319e+06
##   f.score.OOB
## 4          NA
## 2          NA
## 3          NA
## 6          NA
## 5          NA
## 1          NA
```

```r
glb_sel_mdl <- glb_mdl

if (glb_is_regression)
    print(myplot_scatter(glb_models_df, "Adj.R.sq.fit", "R.sq.OOB") + 
          geom_text(aes(label=feats), data=glb_models_df, color="NavyBlue", 
                    size=3.5))
```

![](NBA_PTS_files/figure-html/run_models-1.png) 

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
##        Estimate Std. Error    t value          Pr.z   id
## X2PA  1.0499679 0.02828816  37.116862 2.421882e-178 X2PA
## X3PA  1.2730613 0.03440659  37.000508 1.227527e-177 X3PA
## FTA   1.1273097 0.03259879  34.581335 7.273642e-163  FTA
## AST   0.8884401 0.04291845  20.700655  4.833077e-77  AST
## ORB  -0.9742959 0.07465229 -13.051118  1.545419e-35  ORB
## STL  -0.2268411 0.08350426  -2.716521  6.734815e-03  STL
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
## -523.33 -122.02    6.93  120.68  568.26 
## 
## Coefficients:
##               Estimate Std. Error t value Pr(>|t|)    
## (Intercept) -2.033e+03  1.629e+02 -12.475  < 2e-16 ***
## X2PA         1.050e+00  2.829e-02  37.117  < 2e-16 ***
## X3PA         1.273e+00  3.441e-02  37.001  < 2e-16 ***
## FTA          1.127e+00  3.260e-02  34.581  < 2e-16 ***
## AST          8.884e-01  4.292e-02  20.701  < 2e-16 ***
## ORB         -9.743e-01  7.465e-02 -13.051  < 2e-16 ***
## STL         -2.268e-01  8.350e-02  -2.717  0.00673 ** 
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 185.3 on 828 degrees of freedom
## Multiple R-squared:  0.8991,	Adjusted R-squared:  0.8983 
## F-statistic:  1229 on 6 and 828 DF,  p-value: < 2.2e-16
## 
##                                                                                               feats
## 4 SeasonEnd, W, FG, FGA, X2P, X2PA, X3P, X3PA, FT, FTA, ORB, DRB, AST, STL, BLK, TOV, Team.fctr.num
## 2                        FG, FG:X2P, FG:FGA, FG:AST, FG:X2PA, FG:FTA, FG:X3P, FG:X3PA, FG:SeasonEnd
## 3                                                 FG, FT, ORB, STL, TOV, W, BLK, DRB, Team.fctr.num
## 6                                                                    X2PA, X3PA, FTA, AST, ORB, STL
## 5                                                     X2PA, X3PA, FTA, AST, ORB, DRB, TOV, STL, BLK
## 1                                                                                                FG
## 7                                                                    X2PA, X3PA, FTA, AST, ORB, STL
##   n.fit  R.sq.fit  R.sq.OOB Adj.R.sq.fit      SSE.fit      SSE.OOB
## 4   835 1.0000000 1.0000000    1.0000000 9.770615e-22 2.787599e-22
## 2   835 0.9898302 0.9808173    0.9898302 2.863458e+06 1.105918e+05
## 3   835 0.9615788 0.8970990    0.9615788 1.081805e+07 5.932440e+05
## 6   835 0.8990589 0.8127142    0.8990589 2.842146e+07 1.079739e+06
## 5   835 0.8991553 0.8123019    0.8991553 2.839431e+07 1.082116e+06
## 1   835 0.8872787 0.7663011    0.8872787 3.173834e+07 1.347319e+06
## 7   835 0.8990589        NA    0.8990589 2.842146e+07           NA
##   f.score.OOB
## 4          NA
## 2          NA
## 3          NA
## 6          NA
## 5          NA
## 1          NA
## 7          NA
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

![](NBA_PTS_files/figure-html/fit_training.all-1.png) 

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
##               id       cor.y  cor.y.abs cor.low          Pr.z
## 15          X2PA  0.70836140 0.70836140      NA 2.421882e-178
## 17          X3PA -0.51519812 0.51519812      NA 1.227527e-177
## 7            FTA  0.65505876 0.65505876      NA 7.273642e-163
## 1            AST  0.75988915 0.75988915      NA  4.833077e-77
## 8            ORB  0.49692073 0.49692073       1  1.545419e-35
## 10           STL  0.43099026 0.43099026       1  6.734815e-03
## 2            BLK  0.15205537 0.15205537       1            NA
## 3            DRB  0.09029137 0.09029137       1            NA
## 4             FG  0.94195473 0.94195473       1            NA
## 5            FGA  0.79397947 0.79397947      NA            NA
## 6             FT  0.69748385 0.69748385       1            NA
## 9      SeasonEnd -0.63952773 0.63952773      NA            NA
## 11 Team.fctr.num -0.20008998 0.20008998       1            NA
## 12           TOV  0.42713832 0.42713832       1            NA
## 13             W  0.29882561 0.29882561       1            NA
## 14           X2P  0.82572457 0.82572457      NA            NA
## 16           X3P -0.48994891 0.48994891      NA            NA
```

```r
# Most of this code is used again in predict_newdata chunk
glb_analytics_diag_plots <- function(obs_df) {
    for (var in subset(glb_feats_df, Pr.z < 0.1)$id) {
        plot_df <- melt(obs_df, id.vars=var, 
                        measure.vars=c(glb_predct_var, glb_predct_var_name))
        if (var == "W") print(myplot_scatter(plot_df, var, "value", 
                                             facet_colcol_name="variable") + 
                      geom_vline(xintercept=41, linetype="dotted")) else     
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
                                               plot_vars_df$id[1])
                + geom_hline(yintercept=41, linetype = "dotted")
                )
    }    
}
glb_analytics_diag_plots(glb_entity_df)
```

![](NBA_PTS_files/figure-html/fit_training.all-2.png) ![](NBA_PTS_files/figure-html/fit_training.all-3.png) ![](NBA_PTS_files/figure-html/fit_training.all-4.png) ![](NBA_PTS_files/figure-html/fit_training.all-5.png) ![](NBA_PTS_files/figure-html/fit_training.all-6.png) ![](NBA_PTS_files/figure-html/fit_training.all-7.png) 

```
##                      Team SeasonEnd Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 60         Boston Celtics      1991        1 56 9145   8668 3695 7214 3586
## 264 Golden State Warriors      2001        0 17 7584   8326 2937 7175 2655
## 81      Charlotte Hornets      2000        1 49 8072   7853 2935 6533 2596
## 194        Denver Nuggets      2003        0 17 6901   7580 2689 6544 2460
## 241 Golden State Warriors      2000        0 19 7834   8512 2996 7140 2651
##     X2PA X3P X3PA   FT  FTA  ORB  DRB  AST STL BLK  TOV
## 60  6868 109  346 1646 1997 1088 2697 2160 672 565 1320
## 264 6211 282  964 1428 2024 1345 2385 1788 742 410 1301
## 81  5532 339 1001 1863 2458  884 2635 2023 732 480 1206
## 194 5720 229  824 1294 1850 1112 2363 1737 712 422 1514
## 241 6071 345 1069 1497 2147 1300 2438 1851 731 356 1302
##                 Team.fctr Team.fctr.num Conf Conf.fctr PTS.diff
## 60         Boston Celtics             2 East      East      477
## 264 Golden State Warriors             7 West      West     -742
## 81      Charlotte Hornets            26 East      East      219
## 194        Denver Nuggets             5 West      West     -679
## 241 Golden State Warriors             7 West      West     -678
##     PTS.predict PTS.predict.err                     .label
## 60     8576.739        568.2605        1991:Boston Celtics
## 264    8107.327        523.3272 2001:Golden State Warriors
## 81     8590.957        518.9569     2000:Charlotte Hornets
## 194    7405.918        504.9182        2003:Denver Nuggets
## 241    8334.972        500.9725 2000:Golden State Warriors
```

![](NBA_PTS_files/figure-html/fit_training.all-8.png) 

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
##   SeasonEnd                Team  PTS PTS.predict
## 1      2013       Atlanta Hawks 8032    8086.446
## 2      2013       Brooklyn Nets 7944    7764.143
## 3      2013   Charlotte Bobcats 7661    7965.348
## 4      2013       Chicago Bulls 7641    7784.034
## 5      2013 Cleveland Cavaliers 7913    8004.349
## 6      2013    Dallas Mavericks 8293    8247.427
##    SeasonEnd                  Team  PTS PTS.predict
## 6       2013      Dallas Mavericks 8293    8247.427
## 11      2013  Los Angeles Clippers 8289    8072.525
## 12      2013    Los Angeles Lakers 8381    8535.753
## 14      2013            Miami Heat 8436    8022.760
## 19      2013 Oklahoma City Thunder 8669    8197.481
## 28      2013    Washington Wizards 7644    7873.656
##    SeasonEnd                   Team  PTS PTS.predict
## 23      2013 Portland Trail Blazers 7995    7947.870
## 24      2013       Sacramento Kings 8219    8144.708
## 25      2013      San Antonio Spurs 8448    8335.840
## 26      2013        Toronto Raptors 7971    8006.388
## 27      2013              Utah Jazz 8038    7975.788
## 28      2013     Washington Wizards 7644    7873.656
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
## [1] "Total SSE: 1079738.7293"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

![](NBA_PTS_files/figure-html/predict_newdata-1.png) 

```r
if (glb_is_classification)
    print(xtabs(reformulate(paste(glb_predct_var, glb_predct_var_name, sep=" + ")),
                glb_predct_df))
    
glb_analytics_diag_plots(glb_predct_df)
```

![](NBA_PTS_files/figure-html/predict_newdata-2.png) ![](NBA_PTS_files/figure-html/predict_newdata-3.png) ![](NBA_PTS_files/figure-html/predict_newdata-4.png) ![](NBA_PTS_files/figure-html/predict_newdata-5.png) ![](NBA_PTS_files/figure-html/predict_newdata-6.png) ![](NBA_PTS_files/figure-html/predict_newdata-7.png) 

```
##                      Team SeasonEnd Playoffs  W  PTS oppPTS   FG  FGA  X2P
## 19  Oklahoma City Thunder      2013        1 60 8669   7914 3126 6504 2528
## 14             Miami Heat      2013        1 66 8436   7791 3148 6348 2431
## 18        New York Knicks      2013        1 54 8196   7849 2996 6689 2105
## 16 Minnesota Timberwolves      2013        0 31 7851   8045 2943 6702 2493
## 3       Charlotte Bobcats      2013        0 21 7661   8418 2823 6649 2354
##    X2PA X3P X3PA   FT  FTA ORB  DRB  AST STL BLK  TOV
## 19 4916 598 1588 1819 2196 854 2725 1753 679 624 1253
## 14 4539 717 1809 1423 1887 676 2490 1890 710 441 1143
## 18 4318 891 2371 1313 1729 890 2436 1579 672 294  988
## 16 5227 450 1475 1515 2042 973 2473 1836 700 387 1214
## 3  5250 469 1399 1546 2060 917 2389 1587 591 479 1153
##                 Team.fctr Team.fctr.num Conf Conf.fctr PTS.diff
## 19  Oklahoma City Thunder            19 West      West      755
## 14             Miami Heat            14 East      East      645
## 18        New York Knicks            18 East      East      347
## 16 Minnesota Timberwolves            16 West      West     -194
## 3       Charlotte Bobcats             3 East      East     -757
##    PTS.predict PTS.predict.err                      .label
## 19    8197.481        471.5189  2013:Oklahoma City Thunder
## 14    8022.760        413.2401             2013:Miami Heat
## 18    7851.878        344.1217        2013:New York Knicks
## 16    8159.595        308.5952 2013:Minnesota Timberwolves
## 3     7965.348        304.3480      2013:Charlotte Bobcats
```

![](NBA_PTS_files/figure-html/predict_newdata-8.png) 

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
