# NBA: Playoffs classification
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
glb_predct_var <- "Playoffs"           # or NULL
glb_predct_var_name <- paste0(glb_predct_var, ".predict")
glb_id_vars <- c("SeasonEnd", "Team")                # or NULL

glb_exclude_vars_as_features <- NULL                      
# List chrs converted into factors 
glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, 
                                      c("Team", "Team.fctr")     # or NULL
                                      )
# List feats that shd be excluded due to known causation by prediction variable
# glb_exclude_vars_as_features <- union(glb_exclude_vars_as_features, 
#                                       c("<col_name>")     # or NULL
#                                       )

glb_is_classification <- TRUE; glb_is_regression <- !glb_is_classification

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
#   Potential Enhancements:
#       One code chunk to cycle thru glb_entity_df & glb_predct_df ?
#           Use with / within ?
#           for (df in c(glb_entity_df, glb_predct_df)) cycles thru column names
#           for (df in list(glb_entity_df, glb_predct_df)) does not change the actual dataframes
#
#       Build splines   require(splines); bsBasis <- bs(training$age, df=3)

glb_entity_df <- mutate(glb_entity_df,
#     <col_name>.NA=is.na(<col_name>)
    Team.fctr=factor(Team, 
                as.factor(union(glb_entity_df$Team, glb_predct_df$Team))) 
#     <col_name>.fctr_num=grep(<col_name>, levels(<col_name>.fctr)), # This doesn't work
#     
#     Date.my=as.Date(strptime(Date, "%m/%d/%y %H:%M")),
#     Year=year(Date.my),
#     Month=months(Date.my),
#     Weekday=weekdays(Date.my)
#     
                    )
# 
# If levels of a factor are different across glb_entity_df & glb_predct_df; predict.glm fails  
# Transformations not handled by mutate
glb_entity_df$Team.fctr.num <- sapply(1:nrow(glb_entity_df), 
    function(row_ix) grep(glb_entity_df[row_ix, "Team"],
                          levels(glb_entity_df[row_ix, "Team.fctr"])))

glb_predct_df <- mutate(glb_predct_df, 
    Team.fctr=factor(Team, 
                as.factor(union(glb_entity_df$Team, glb_predct_df$Team))) 
                    )

glb_predct_df$Team.fctr.num <- sapply(1:nrow(glb_predct_df), 
    function(row_ix) grep(glb_predct_df[row_ix, "Team"],
                          levels(glb_predct_df[row_ix, "Team.fctr"])))

print(summary(glb_entity_df))
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
```

```r
print(sapply(names(glb_entity_df), function(col) sum(is.na(glb_entity_df[, col]))))
```

```
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
print(summary(glb_predct_df))
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
##                  Team.fctr  Team.fctr.num  
##  Atlanta Hawks        : 1   Min.   : 1.00  
##  Chicago Bulls        : 1   1st Qu.:10.25  
##  Cleveland Cavaliers  : 1   Median :19.50  
##  Denver Nuggets       : 1   Mean   :19.75  
##  Detroit Pistons      : 1   3rd Qu.:29.25  
##  Golden State Warriors: 1   Max.   :38.00  
##  (Other)              :22
```

```r
print(sapply(names(glb_predct_df), function(col) sum(is.na(glb_predct_df[, col]))))
```

```
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
# 
# glb_entity_df <- mymap_codes(glb_entity_df, "<from_col_name>", "<to_col_name>", 
#     map_<to_col_name>_df, map_join_col_name="<map_join_col_name>", 
#                           map_tgt_col_name="<to_col_name>")
    					
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
# glb_entity_df <- mutate(glb_entity_df,
#     <new_col_name>=
#                     )

# glb_predct_df <- mutate(glb_predct_df,
#     <new_col_name>=
#                     )

# print(summary(glb_entity_df))
# print(summary(glb_predct_df))

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
##                          id        cor.y   cor.y.abs
## W                         W  0.798675595 0.798675595
## DRB                     DRB  0.344749300 0.344749300
## AST                     AST  0.314705084 0.314705084
## PTS                     PTS  0.270601328 0.270601328
## oppPTS               oppPTS -0.232760995 0.232760995
## FT                       FT  0.221799063 0.221799063
## FG                       FG  0.187552799 0.187552799
## BLK                     BLK  0.187056116 0.187056116
## FTA                     FTA  0.179413465 0.179413465
## STL                     STL  0.173822947 0.173822947
## TOV                     TOV -0.168601605 0.168601605
## Team.fctr.num Team.fctr.num -0.127393136 0.127393136
## X2P                     X2P  0.108033881 0.108033881
## SeasonEnd         SeasonEnd -0.059127866 0.059127866
## ORB                     ORB -0.041702843 0.041702843
## X3P                     X3P  0.028382516 0.028382516
## FGA                     FGA -0.011985459 0.011985459
## X2PA                   X2PA -0.009931886 0.009931886
## X3PA                   X3PA  0.006570622 0.006570622
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
##                         W          DRB         AST         PTS      oppPTS
## W              1.00000000  0.470897497  0.32005177  0.29882561 -0.33157294
## DRB            0.47089750  1.000000000  0.05647709  0.09029137 -0.21262703
## AST            0.32005177  0.056477086  1.00000000  0.75988915  0.54256181
## PTS            0.29882561  0.090291366  0.75988915  1.00000000  0.78907471
## oppPTS        -0.33157294 -0.212627029  0.54256181  0.78907471  1.00000000
## FT             0.20490600  0.003604883  0.44720848  0.69748385  0.55058757
## FG             0.19039642  0.008966352  0.81226652  0.94195473  0.80446737
## BLK            0.20392100  0.242990816  0.20308000  0.15205537  0.03076664
## FTA            0.16188731 -0.043542668  0.42797223  0.65505876  0.53721485
## STL            0.11619440 -0.331127910  0.44313827  0.43099026  0.34348011
## TOV           -0.24318588 -0.190956902  0.43053288  0.42713832  0.58241272
## Team.fctr.num -0.15119487 -0.058154820 -0.17491216 -0.20008998 -0.10100550
## X2P            0.06927898 -0.098690188  0.77711327  0.82572457  0.76984080
## SeasonEnd      0.00000000  0.267127100 -0.67212300 -0.63952773 -0.63244845
## ORB           -0.09573676 -0.269179339  0.40676555  0.49692073  0.55187982
## X3P            0.11904456  0.233353555 -0.56785931 -0.48994891 -0.56282942
## FGA           -0.07144566 -0.050945458  0.62911736  0.79397947  0.83087856
## X2PA          -0.08703653 -0.165315490  0.67770751  0.70836140  0.75654063
## X3PA           0.08328625  0.223060547 -0.59278305 -0.51519812 -0.56332937
##                         FT           FG         BLK         FTA        STL
## W              0.204906000  0.190396422  0.20392100  0.16188731  0.1161944
## DRB            0.003604883  0.008966352  0.24299082 -0.04354267 -0.3311279
## AST            0.447208477  0.812266519  0.20308000  0.42797223  0.4431383
## PTS            0.697483850  0.941954732  0.15205537  0.65505876  0.4309903
## oppPTS         0.550587572  0.804467370  0.03076664  0.53721485  0.3434801
## FT             1.000000000  0.538342589  0.16465598  0.95049325  0.3231570
## FG             0.538342589  1.000000000  0.17750275  0.52121683  0.4697303
## BLK            0.164655982  0.177502748  1.00000000  0.21637171  0.1204559
## FTA            0.950493246  0.521216830  0.21637171  1.00000000  0.3700552
## STL            0.323156975  0.469730269  0.12045594  0.37005522  1.0000000
## TOV            0.438684976  0.517630444  0.24156872  0.52566979  0.4640752
## Team.fctr.num -0.154385609 -0.198789524 -0.07311722 -0.13968994 -0.1531193
## X2P            0.574293954  0.942919965  0.21771153  0.57454277  0.4890027
## SeasonEnd     -0.513699180 -0.761598813 -0.20416249 -0.54689652 -0.5020598
## ORB            0.393398702  0.598591143  0.20110479  0.47455838  0.4929415
## X3P           -0.508712984 -0.668272893 -0.23107383 -0.53389685 -0.4168543
## FGA            0.391691425  0.879662910  0.12109667  0.38212215  0.4805329
## X2PA           0.515802012  0.859224632  0.20610678  0.52375675  0.4900862
## X3PA          -0.517849574 -0.688763042 -0.23403142 -0.53778289 -0.4090916
##                      TOV Team.fctr.num         X2P  SeasonEnd         ORB
## W             -0.2431859   -0.15119487  0.06927898  0.0000000 -0.09573676
## DRB           -0.1909569   -0.05815482 -0.09869019  0.2671271 -0.26917934
## AST            0.4305329   -0.17491216  0.77711327 -0.6721230  0.40676555
## PTS            0.4271383   -0.20008998  0.82572457 -0.6395277  0.49692073
## oppPTS         0.5824127   -0.10100550  0.76984080 -0.6324485  0.55187982
## FT             0.4386850   -0.15438561  0.57429395 -0.5136992  0.39339870
## FG             0.5176304   -0.19878952  0.94291996 -0.7615988  0.59859114
## BLK            0.2415687   -0.07311722  0.21771153 -0.2041625  0.20110479
## FTA            0.5256698   -0.13968994  0.57454277 -0.5468965  0.47455838
## STL            0.4640752   -0.15311931  0.48900271 -0.5020598  0.49294148
## TOV            1.0000000   -0.14045929  0.63771567 -0.7238690  0.54598664
## Team.fctr.num -0.1404593    1.00000000 -0.19170714  0.2145549 -0.20095724
## X2P            0.6377157   -0.19170714  1.00000000 -0.8654893  0.68311802
## SeasonEnd     -0.7238690    0.21455487 -0.86548930  1.0000000 -0.70201284
## ORB            0.5459866   -0.20095724  0.68311802 -0.7020128  1.00000000
## X3P           -0.6801733    0.14237198 -0.87786641  0.8381421 -0.66516818
## FGA            0.4499814   -0.17904220  0.83827486 -0.6849048  0.73750939
## X2PA           0.6448524   -0.17580617  0.96530915 -0.8672568  0.76561469
## X3PA          -0.6778031    0.14165189 -0.88859996  0.8505523 -0.64917273
##                      X3P         FGA        X2PA        X3PA
## W              0.1190446 -0.07144566 -0.08703653  0.08328625
## DRB            0.2333536 -0.05094546 -0.16531549  0.22306055
## AST           -0.5678593  0.62911736  0.67770751 -0.59278305
## PTS           -0.4899489  0.79397947  0.70836140 -0.51519812
## oppPTS        -0.5628294  0.83087856  0.75654063 -0.56332937
## FT            -0.5087130  0.39169143  0.51580201 -0.51784957
## FG            -0.6682729  0.87966291  0.85922463 -0.68876304
## BLK           -0.2310738  0.12109667  0.20610678 -0.23403142
## FTA           -0.5338969  0.38212215  0.52375675 -0.53778289
## STL           -0.4168543  0.48053292  0.49008620 -0.40909163
## TOV           -0.6801733  0.44998136  0.64485242 -0.67780315
## Team.fctr.num  0.1423720 -0.17904220 -0.17580617  0.14165189
## X2P           -0.8778664  0.83827486  0.96530915 -0.88859996
## SeasonEnd      0.8381421 -0.68490478 -0.86725681  0.85055227
## ORB           -0.6651682  0.73750939  0.76561469 -0.64917273
## X3P            1.0000000 -0.60756448 -0.92073200  0.99451088
## FGA           -0.6075645  1.00000000  0.86485931 -0.60559564
## X2PA          -0.9207320  0.86485931  1.00000000 -0.92324423
## X3PA           0.9945109 -0.60559564 -0.92324423  1.00000000
##                        W         DRB        AST        PTS     oppPTS
## W             0.00000000 0.470897497 0.32005177 0.29882561 0.33157294
## DRB           0.47089750 0.000000000 0.05647709 0.09029137 0.21262703
## AST           0.32005177 0.056477086 0.00000000 0.75988915 0.54256181
## PTS           0.29882561 0.090291366 0.75988915 0.00000000 0.78907471
## oppPTS        0.33157294 0.212627029 0.54256181 0.78907471 0.00000000
## FT            0.20490600 0.003604883 0.44720848 0.69748385 0.55058757
## FG            0.19039642 0.008966352 0.81226652 0.94195473 0.80446737
## BLK           0.20392100 0.242990816 0.20308000 0.15205537 0.03076664
## FTA           0.16188731 0.043542668 0.42797223 0.65505876 0.53721485
## STL           0.11619440 0.331127910 0.44313827 0.43099026 0.34348011
## TOV           0.24318588 0.190956902 0.43053288 0.42713832 0.58241272
## Team.fctr.num 0.15119487 0.058154820 0.17491216 0.20008998 0.10100550
## X2P           0.06927898 0.098690188 0.77711327 0.82572457 0.76984080
## SeasonEnd     0.00000000 0.267127100 0.67212300 0.63952773 0.63244845
## ORB           0.09573676 0.269179339 0.40676555 0.49692073 0.55187982
## X3P           0.11904456 0.233353555 0.56785931 0.48994891 0.56282942
## FGA           0.07144566 0.050945458 0.62911736 0.79397947 0.83087856
## X2PA          0.08703653 0.165315490 0.67770751 0.70836140 0.75654063
## X3PA          0.08328625 0.223060547 0.59278305 0.51519812 0.56332937
##                        FT          FG        BLK        FTA       STL
## W             0.204906000 0.190396422 0.20392100 0.16188731 0.1161944
## DRB           0.003604883 0.008966352 0.24299082 0.04354267 0.3311279
## AST           0.447208477 0.812266519 0.20308000 0.42797223 0.4431383
## PTS           0.697483850 0.941954732 0.15205537 0.65505876 0.4309903
## oppPTS        0.550587572 0.804467370 0.03076664 0.53721485 0.3434801
## FT            0.000000000 0.538342589 0.16465598 0.95049325 0.3231570
## FG            0.538342589 0.000000000 0.17750275 0.52121683 0.4697303
## BLK           0.164655982 0.177502748 0.00000000 0.21637171 0.1204559
## FTA           0.950493246 0.521216830 0.21637171 0.00000000 0.3700552
## STL           0.323156975 0.469730269 0.12045594 0.37005522 0.0000000
## TOV           0.438684976 0.517630444 0.24156872 0.52566979 0.4640752
## Team.fctr.num 0.154385609 0.198789524 0.07311722 0.13968994 0.1531193
## X2P           0.574293954 0.942919965 0.21771153 0.57454277 0.4890027
## SeasonEnd     0.513699180 0.761598813 0.20416249 0.54689652 0.5020598
## ORB           0.393398702 0.598591143 0.20110479 0.47455838 0.4929415
## X3P           0.508712984 0.668272893 0.23107383 0.53389685 0.4168543
## FGA           0.391691425 0.879662910 0.12109667 0.38212215 0.4805329
## X2PA          0.515802012 0.859224632 0.20610678 0.52375675 0.4900862
## X3PA          0.517849574 0.688763042 0.23403142 0.53778289 0.4090916
##                     TOV Team.fctr.num        X2P SeasonEnd        ORB
## W             0.2431859    0.15119487 0.06927898 0.0000000 0.09573676
## DRB           0.1909569    0.05815482 0.09869019 0.2671271 0.26917934
## AST           0.4305329    0.17491216 0.77711327 0.6721230 0.40676555
## PTS           0.4271383    0.20008998 0.82572457 0.6395277 0.49692073
## oppPTS        0.5824127    0.10100550 0.76984080 0.6324485 0.55187982
## FT            0.4386850    0.15438561 0.57429395 0.5136992 0.39339870
## FG            0.5176304    0.19878952 0.94291996 0.7615988 0.59859114
## BLK           0.2415687    0.07311722 0.21771153 0.2041625 0.20110479
## FTA           0.5256698    0.13968994 0.57454277 0.5468965 0.47455838
## STL           0.4640752    0.15311931 0.48900271 0.5020598 0.49294148
## TOV           0.0000000    0.14045929 0.63771567 0.7238690 0.54598664
## Team.fctr.num 0.1404593    0.00000000 0.19170714 0.2145549 0.20095724
## X2P           0.6377157    0.19170714 0.00000000 0.8654893 0.68311802
## SeasonEnd     0.7238690    0.21455487 0.86548930 0.0000000 0.70201284
## ORB           0.5459866    0.20095724 0.68311802 0.7020128 0.00000000
## X3P           0.6801733    0.14237198 0.87786641 0.8381421 0.66516818
## FGA           0.4499814    0.17904220 0.83827486 0.6849048 0.73750939
## X2PA          0.6448524    0.17580617 0.96530915 0.8672568 0.76561469
## X3PA          0.6778031    0.14165189 0.88859996 0.8505523 0.64917273
##                     X3P        FGA       X2PA       X3PA
## W             0.1190446 0.07144566 0.08703653 0.08328625
## DRB           0.2333536 0.05094546 0.16531549 0.22306055
## AST           0.5678593 0.62911736 0.67770751 0.59278305
## PTS           0.4899489 0.79397947 0.70836140 0.51519812
## oppPTS        0.5628294 0.83087856 0.75654063 0.56332937
## FT            0.5087130 0.39169143 0.51580201 0.51784957
## FG            0.6682729 0.87966291 0.85922463 0.68876304
## BLK           0.2310738 0.12109667 0.20610678 0.23403142
## FTA           0.5338969 0.38212215 0.52375675 0.53778289
## STL           0.4168543 0.48053292 0.49008620 0.40909163
## TOV           0.6801733 0.44998136 0.64485242 0.67780315
## Team.fctr.num 0.1423720 0.17904220 0.17580617 0.14165189
## X2P           0.8778664 0.83827486 0.96530915 0.88859996
## SeasonEnd     0.8381421 0.68490478 0.86725681 0.85055227
## ORB           0.6651682 0.73750939 0.76561469 0.64917273
## X3P           0.0000000 0.60756448 0.92073200 0.99451088
## FGA           0.6075645 0.00000000 0.86485931 0.60559564
## X2PA          0.9207320 0.86485931 0.00000000 0.92324423
## X3PA          0.9945109 0.60559564 0.92324423 0.00000000
## [1] "cor(X3P, X3PA)=0.9945"
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-1.png) 

```
## [1] "cor(Playoffs, X3P)=0.0284"
## [1] "cor(Playoffs, X3PA)=0.0066"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## Warning in mydelete_cor_features(): Dropping X3PA as a feature
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-2.png) 

```
##                          id        cor.y   cor.y.abs
## W                         W  0.798675595 0.798675595
## DRB                     DRB  0.344749300 0.344749300
## AST                     AST  0.314705084 0.314705084
## PTS                     PTS  0.270601328 0.270601328
## oppPTS               oppPTS -0.232760995 0.232760995
## FT                       FT  0.221799063 0.221799063
## FG                       FG  0.187552799 0.187552799
## BLK                     BLK  0.187056116 0.187056116
## FTA                     FTA  0.179413465 0.179413465
## STL                     STL  0.173822947 0.173822947
## TOV                     TOV -0.168601605 0.168601605
## Team.fctr.num Team.fctr.num -0.127393136 0.127393136
## X2P                     X2P  0.108033881 0.108033881
## SeasonEnd         SeasonEnd -0.059127866 0.059127866
## ORB                     ORB -0.041702843 0.041702843
## X3P                     X3P  0.028382516 0.028382516
## FGA                     FGA -0.011985459 0.011985459
## X2PA                   X2PA -0.009931886 0.009931886
##                         W          DRB         AST         PTS      oppPTS
## W              1.00000000  0.470897497  0.32005177  0.29882561 -0.33157294
## DRB            0.47089750  1.000000000  0.05647709  0.09029137 -0.21262703
## AST            0.32005177  0.056477086  1.00000000  0.75988915  0.54256181
## PTS            0.29882561  0.090291366  0.75988915  1.00000000  0.78907471
## oppPTS        -0.33157294 -0.212627029  0.54256181  0.78907471  1.00000000
## FT             0.20490600  0.003604883  0.44720848  0.69748385  0.55058757
## FG             0.19039642  0.008966352  0.81226652  0.94195473  0.80446737
## BLK            0.20392100  0.242990816  0.20308000  0.15205537  0.03076664
## FTA            0.16188731 -0.043542668  0.42797223  0.65505876  0.53721485
## STL            0.11619440 -0.331127910  0.44313827  0.43099026  0.34348011
## TOV           -0.24318588 -0.190956902  0.43053288  0.42713832  0.58241272
## Team.fctr.num -0.15119487 -0.058154820 -0.17491216 -0.20008998 -0.10100550
## X2P            0.06927898 -0.098690188  0.77711327  0.82572457  0.76984080
## SeasonEnd      0.00000000  0.267127100 -0.67212300 -0.63952773 -0.63244845
## ORB           -0.09573676 -0.269179339  0.40676555  0.49692073  0.55187982
## X3P            0.11904456  0.233353555 -0.56785931 -0.48994891 -0.56282942
## FGA           -0.07144566 -0.050945458  0.62911736  0.79397947  0.83087856
## X2PA          -0.08703653 -0.165315490  0.67770751  0.70836140  0.75654063
##                         FT           FG         BLK         FTA        STL
## W              0.204906000  0.190396422  0.20392100  0.16188731  0.1161944
## DRB            0.003604883  0.008966352  0.24299082 -0.04354267 -0.3311279
## AST            0.447208477  0.812266519  0.20308000  0.42797223  0.4431383
## PTS            0.697483850  0.941954732  0.15205537  0.65505876  0.4309903
## oppPTS         0.550587572  0.804467370  0.03076664  0.53721485  0.3434801
## FT             1.000000000  0.538342589  0.16465598  0.95049325  0.3231570
## FG             0.538342589  1.000000000  0.17750275  0.52121683  0.4697303
## BLK            0.164655982  0.177502748  1.00000000  0.21637171  0.1204559
## FTA            0.950493246  0.521216830  0.21637171  1.00000000  0.3700552
## STL            0.323156975  0.469730269  0.12045594  0.37005522  1.0000000
## TOV            0.438684976  0.517630444  0.24156872  0.52566979  0.4640752
## Team.fctr.num -0.154385609 -0.198789524 -0.07311722 -0.13968994 -0.1531193
## X2P            0.574293954  0.942919965  0.21771153  0.57454277  0.4890027
## SeasonEnd     -0.513699180 -0.761598813 -0.20416249 -0.54689652 -0.5020598
## ORB            0.393398702  0.598591143  0.20110479  0.47455838  0.4929415
## X3P           -0.508712984 -0.668272893 -0.23107383 -0.53389685 -0.4168543
## FGA            0.391691425  0.879662910  0.12109667  0.38212215  0.4805329
## X2PA           0.515802012  0.859224632  0.20610678  0.52375675  0.4900862
##                      TOV Team.fctr.num         X2P  SeasonEnd         ORB
## W             -0.2431859   -0.15119487  0.06927898  0.0000000 -0.09573676
## DRB           -0.1909569   -0.05815482 -0.09869019  0.2671271 -0.26917934
## AST            0.4305329   -0.17491216  0.77711327 -0.6721230  0.40676555
## PTS            0.4271383   -0.20008998  0.82572457 -0.6395277  0.49692073
## oppPTS         0.5824127   -0.10100550  0.76984080 -0.6324485  0.55187982
## FT             0.4386850   -0.15438561  0.57429395 -0.5136992  0.39339870
## FG             0.5176304   -0.19878952  0.94291996 -0.7615988  0.59859114
## BLK            0.2415687   -0.07311722  0.21771153 -0.2041625  0.20110479
## FTA            0.5256698   -0.13968994  0.57454277 -0.5468965  0.47455838
## STL            0.4640752   -0.15311931  0.48900271 -0.5020598  0.49294148
## TOV            1.0000000   -0.14045929  0.63771567 -0.7238690  0.54598664
## Team.fctr.num -0.1404593    1.00000000 -0.19170714  0.2145549 -0.20095724
## X2P            0.6377157   -0.19170714  1.00000000 -0.8654893  0.68311802
## SeasonEnd     -0.7238690    0.21455487 -0.86548930  1.0000000 -0.70201284
## ORB            0.5459866   -0.20095724  0.68311802 -0.7020128  1.00000000
## X3P           -0.6801733    0.14237198 -0.87786641  0.8381421 -0.66516818
## FGA            0.4499814   -0.17904220  0.83827486 -0.6849048  0.73750939
## X2PA           0.6448524   -0.17580617  0.96530915 -0.8672568  0.76561469
##                      X3P         FGA        X2PA
## W              0.1190446 -0.07144566 -0.08703653
## DRB            0.2333536 -0.05094546 -0.16531549
## AST           -0.5678593  0.62911736  0.67770751
## PTS           -0.4899489  0.79397947  0.70836140
## oppPTS        -0.5628294  0.83087856  0.75654063
## FT            -0.5087130  0.39169143  0.51580201
## FG            -0.6682729  0.87966291  0.85922463
## BLK           -0.2310738  0.12109667  0.20610678
## FTA           -0.5338969  0.38212215  0.52375675
## STL           -0.4168543  0.48053292  0.49008620
## TOV           -0.6801733  0.44998136  0.64485242
## Team.fctr.num  0.1423720 -0.17904220 -0.17580617
## X2P           -0.8778664  0.83827486  0.96530915
## SeasonEnd      0.8381421 -0.68490478 -0.86725681
## ORB           -0.6651682  0.73750939  0.76561469
## X3P            1.0000000 -0.60756448 -0.92073200
## FGA           -0.6075645  1.00000000  0.86485931
## X2PA          -0.9207320  0.86485931  1.00000000
##                        W         DRB        AST        PTS     oppPTS
## W             0.00000000 0.470897497 0.32005177 0.29882561 0.33157294
## DRB           0.47089750 0.000000000 0.05647709 0.09029137 0.21262703
## AST           0.32005177 0.056477086 0.00000000 0.75988915 0.54256181
## PTS           0.29882561 0.090291366 0.75988915 0.00000000 0.78907471
## oppPTS        0.33157294 0.212627029 0.54256181 0.78907471 0.00000000
## FT            0.20490600 0.003604883 0.44720848 0.69748385 0.55058757
## FG            0.19039642 0.008966352 0.81226652 0.94195473 0.80446737
## BLK           0.20392100 0.242990816 0.20308000 0.15205537 0.03076664
## FTA           0.16188731 0.043542668 0.42797223 0.65505876 0.53721485
## STL           0.11619440 0.331127910 0.44313827 0.43099026 0.34348011
## TOV           0.24318588 0.190956902 0.43053288 0.42713832 0.58241272
## Team.fctr.num 0.15119487 0.058154820 0.17491216 0.20008998 0.10100550
## X2P           0.06927898 0.098690188 0.77711327 0.82572457 0.76984080
## SeasonEnd     0.00000000 0.267127100 0.67212300 0.63952773 0.63244845
## ORB           0.09573676 0.269179339 0.40676555 0.49692073 0.55187982
## X3P           0.11904456 0.233353555 0.56785931 0.48994891 0.56282942
## FGA           0.07144566 0.050945458 0.62911736 0.79397947 0.83087856
## X2PA          0.08703653 0.165315490 0.67770751 0.70836140 0.75654063
##                        FT          FG        BLK        FTA       STL
## W             0.204906000 0.190396422 0.20392100 0.16188731 0.1161944
## DRB           0.003604883 0.008966352 0.24299082 0.04354267 0.3311279
## AST           0.447208477 0.812266519 0.20308000 0.42797223 0.4431383
## PTS           0.697483850 0.941954732 0.15205537 0.65505876 0.4309903
## oppPTS        0.550587572 0.804467370 0.03076664 0.53721485 0.3434801
## FT            0.000000000 0.538342589 0.16465598 0.95049325 0.3231570
## FG            0.538342589 0.000000000 0.17750275 0.52121683 0.4697303
## BLK           0.164655982 0.177502748 0.00000000 0.21637171 0.1204559
## FTA           0.950493246 0.521216830 0.21637171 0.00000000 0.3700552
## STL           0.323156975 0.469730269 0.12045594 0.37005522 0.0000000
## TOV           0.438684976 0.517630444 0.24156872 0.52566979 0.4640752
## Team.fctr.num 0.154385609 0.198789524 0.07311722 0.13968994 0.1531193
## X2P           0.574293954 0.942919965 0.21771153 0.57454277 0.4890027
## SeasonEnd     0.513699180 0.761598813 0.20416249 0.54689652 0.5020598
## ORB           0.393398702 0.598591143 0.20110479 0.47455838 0.4929415
## X3P           0.508712984 0.668272893 0.23107383 0.53389685 0.4168543
## FGA           0.391691425 0.879662910 0.12109667 0.38212215 0.4805329
## X2PA          0.515802012 0.859224632 0.20610678 0.52375675 0.4900862
##                     TOV Team.fctr.num        X2P SeasonEnd        ORB
## W             0.2431859    0.15119487 0.06927898 0.0000000 0.09573676
## DRB           0.1909569    0.05815482 0.09869019 0.2671271 0.26917934
## AST           0.4305329    0.17491216 0.77711327 0.6721230 0.40676555
## PTS           0.4271383    0.20008998 0.82572457 0.6395277 0.49692073
## oppPTS        0.5824127    0.10100550 0.76984080 0.6324485 0.55187982
## FT            0.4386850    0.15438561 0.57429395 0.5136992 0.39339870
## FG            0.5176304    0.19878952 0.94291996 0.7615988 0.59859114
## BLK           0.2415687    0.07311722 0.21771153 0.2041625 0.20110479
## FTA           0.5256698    0.13968994 0.57454277 0.5468965 0.47455838
## STL           0.4640752    0.15311931 0.48900271 0.5020598 0.49294148
## TOV           0.0000000    0.14045929 0.63771567 0.7238690 0.54598664
## Team.fctr.num 0.1404593    0.00000000 0.19170714 0.2145549 0.20095724
## X2P           0.6377157    0.19170714 0.00000000 0.8654893 0.68311802
## SeasonEnd     0.7238690    0.21455487 0.86548930 0.0000000 0.70201284
## ORB           0.5459866    0.20095724 0.68311802 0.7020128 0.00000000
## X3P           0.6801733    0.14237198 0.87786641 0.8381421 0.66516818
## FGA           0.4499814    0.17904220 0.83827486 0.6849048 0.73750939
## X2PA          0.6448524    0.17580617 0.96530915 0.8672568 0.76561469
##                     X3P        FGA       X2PA
## W             0.1190446 0.07144566 0.08703653
## DRB           0.2333536 0.05094546 0.16531549
## AST           0.5678593 0.62911736 0.67770751
## PTS           0.4899489 0.79397947 0.70836140
## oppPTS        0.5628294 0.83087856 0.75654063
## FT            0.5087130 0.39169143 0.51580201
## FG            0.6682729 0.87966291 0.85922463
## BLK           0.2310738 0.12109667 0.20610678
## FTA           0.5338969 0.38212215 0.52375675
## STL           0.4168543 0.48053292 0.49008620
## TOV           0.6801733 0.44998136 0.64485242
## Team.fctr.num 0.1423720 0.17904220 0.17580617
## X2P           0.8778664 0.83827486 0.96530915
## SeasonEnd     0.8381421 0.68490478 0.86725681
## ORB           0.6651682 0.73750939 0.76561469
## X3P           0.0000000 0.60756448 0.92073200
## FGA           0.6075645 0.00000000 0.86485931
## X2PA          0.9207320 0.86485931 0.00000000
## [1] "cor(X2P, X2PA)=0.9653"
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-3.png) 

```
## [1] "cor(Playoffs, X2P)=0.1080"
## [1] "cor(Playoffs, X2PA)=-0.0099"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## Warning in mydelete_cor_features(): Dropping X2PA as a feature
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-4.png) 

```
##                          id       cor.y  cor.y.abs
## W                         W  0.79867559 0.79867559
## DRB                     DRB  0.34474930 0.34474930
## AST                     AST  0.31470508 0.31470508
## PTS                     PTS  0.27060133 0.27060133
## oppPTS               oppPTS -0.23276100 0.23276100
## FT                       FT  0.22179906 0.22179906
## FG                       FG  0.18755280 0.18755280
## BLK                     BLK  0.18705612 0.18705612
## FTA                     FTA  0.17941347 0.17941347
## STL                     STL  0.17382295 0.17382295
## TOV                     TOV -0.16860161 0.16860161
## Team.fctr.num Team.fctr.num -0.12739314 0.12739314
## X2P                     X2P  0.10803388 0.10803388
## SeasonEnd         SeasonEnd -0.05912787 0.05912787
## ORB                     ORB -0.04170284 0.04170284
## X3P                     X3P  0.02838252 0.02838252
## FGA                     FGA -0.01198546 0.01198546
##                         W          DRB         AST         PTS      oppPTS
## W              1.00000000  0.470897497  0.32005177  0.29882561 -0.33157294
## DRB            0.47089750  1.000000000  0.05647709  0.09029137 -0.21262703
## AST            0.32005177  0.056477086  1.00000000  0.75988915  0.54256181
## PTS            0.29882561  0.090291366  0.75988915  1.00000000  0.78907471
## oppPTS        -0.33157294 -0.212627029  0.54256181  0.78907471  1.00000000
## FT             0.20490600  0.003604883  0.44720848  0.69748385  0.55058757
## FG             0.19039642  0.008966352  0.81226652  0.94195473  0.80446737
## BLK            0.20392100  0.242990816  0.20308000  0.15205537  0.03076664
## FTA            0.16188731 -0.043542668  0.42797223  0.65505876  0.53721485
## STL            0.11619440 -0.331127910  0.44313827  0.43099026  0.34348011
## TOV           -0.24318588 -0.190956902  0.43053288  0.42713832  0.58241272
## Team.fctr.num -0.15119487 -0.058154820 -0.17491216 -0.20008998 -0.10100550
## X2P            0.06927898 -0.098690188  0.77711327  0.82572457  0.76984080
## SeasonEnd      0.00000000  0.267127100 -0.67212300 -0.63952773 -0.63244845
## ORB           -0.09573676 -0.269179339  0.40676555  0.49692073  0.55187982
## X3P            0.11904456  0.233353555 -0.56785931 -0.48994891 -0.56282942
## FGA           -0.07144566 -0.050945458  0.62911736  0.79397947  0.83087856
##                         FT           FG         BLK         FTA        STL
## W              0.204906000  0.190396422  0.20392100  0.16188731  0.1161944
## DRB            0.003604883  0.008966352  0.24299082 -0.04354267 -0.3311279
## AST            0.447208477  0.812266519  0.20308000  0.42797223  0.4431383
## PTS            0.697483850  0.941954732  0.15205537  0.65505876  0.4309903
## oppPTS         0.550587572  0.804467370  0.03076664  0.53721485  0.3434801
## FT             1.000000000  0.538342589  0.16465598  0.95049325  0.3231570
## FG             0.538342589  1.000000000  0.17750275  0.52121683  0.4697303
## BLK            0.164655982  0.177502748  1.00000000  0.21637171  0.1204559
## FTA            0.950493246  0.521216830  0.21637171  1.00000000  0.3700552
## STL            0.323156975  0.469730269  0.12045594  0.37005522  1.0000000
## TOV            0.438684976  0.517630444  0.24156872  0.52566979  0.4640752
## Team.fctr.num -0.154385609 -0.198789524 -0.07311722 -0.13968994 -0.1531193
## X2P            0.574293954  0.942919965  0.21771153  0.57454277  0.4890027
## SeasonEnd     -0.513699180 -0.761598813 -0.20416249 -0.54689652 -0.5020598
## ORB            0.393398702  0.598591143  0.20110479  0.47455838  0.4929415
## X3P           -0.508712984 -0.668272893 -0.23107383 -0.53389685 -0.4168543
## FGA            0.391691425  0.879662910  0.12109667  0.38212215  0.4805329
##                      TOV Team.fctr.num         X2P  SeasonEnd         ORB
## W             -0.2431859   -0.15119487  0.06927898  0.0000000 -0.09573676
## DRB           -0.1909569   -0.05815482 -0.09869019  0.2671271 -0.26917934
## AST            0.4305329   -0.17491216  0.77711327 -0.6721230  0.40676555
## PTS            0.4271383   -0.20008998  0.82572457 -0.6395277  0.49692073
## oppPTS         0.5824127   -0.10100550  0.76984080 -0.6324485  0.55187982
## FT             0.4386850   -0.15438561  0.57429395 -0.5136992  0.39339870
## FG             0.5176304   -0.19878952  0.94291996 -0.7615988  0.59859114
## BLK            0.2415687   -0.07311722  0.21771153 -0.2041625  0.20110479
## FTA            0.5256698   -0.13968994  0.57454277 -0.5468965  0.47455838
## STL            0.4640752   -0.15311931  0.48900271 -0.5020598  0.49294148
## TOV            1.0000000   -0.14045929  0.63771567 -0.7238690  0.54598664
## Team.fctr.num -0.1404593    1.00000000 -0.19170714  0.2145549 -0.20095724
## X2P            0.6377157   -0.19170714  1.00000000 -0.8654893  0.68311802
## SeasonEnd     -0.7238690    0.21455487 -0.86548930  1.0000000 -0.70201284
## ORB            0.5459866   -0.20095724  0.68311802 -0.7020128  1.00000000
## X3P           -0.6801733    0.14237198 -0.87786641  0.8381421 -0.66516818
## FGA            0.4499814   -0.17904220  0.83827486 -0.6849048  0.73750939
##                      X3P         FGA
## W              0.1190446 -0.07144566
## DRB            0.2333536 -0.05094546
## AST           -0.5678593  0.62911736
## PTS           -0.4899489  0.79397947
## oppPTS        -0.5628294  0.83087856
## FT            -0.5087130  0.39169143
## FG            -0.6682729  0.87966291
## BLK           -0.2310738  0.12109667
## FTA           -0.5338969  0.38212215
## STL           -0.4168543  0.48053292
## TOV           -0.6801733  0.44998136
## Team.fctr.num  0.1423720 -0.17904220
## X2P           -0.8778664  0.83827486
## SeasonEnd      0.8381421 -0.68490478
## ORB           -0.6651682  0.73750939
## X3P            1.0000000 -0.60756448
## FGA           -0.6075645  1.00000000
##                        W         DRB        AST        PTS     oppPTS
## W             0.00000000 0.470897497 0.32005177 0.29882561 0.33157294
## DRB           0.47089750 0.000000000 0.05647709 0.09029137 0.21262703
## AST           0.32005177 0.056477086 0.00000000 0.75988915 0.54256181
## PTS           0.29882561 0.090291366 0.75988915 0.00000000 0.78907471
## oppPTS        0.33157294 0.212627029 0.54256181 0.78907471 0.00000000
## FT            0.20490600 0.003604883 0.44720848 0.69748385 0.55058757
## FG            0.19039642 0.008966352 0.81226652 0.94195473 0.80446737
## BLK           0.20392100 0.242990816 0.20308000 0.15205537 0.03076664
## FTA           0.16188731 0.043542668 0.42797223 0.65505876 0.53721485
## STL           0.11619440 0.331127910 0.44313827 0.43099026 0.34348011
## TOV           0.24318588 0.190956902 0.43053288 0.42713832 0.58241272
## Team.fctr.num 0.15119487 0.058154820 0.17491216 0.20008998 0.10100550
## X2P           0.06927898 0.098690188 0.77711327 0.82572457 0.76984080
## SeasonEnd     0.00000000 0.267127100 0.67212300 0.63952773 0.63244845
## ORB           0.09573676 0.269179339 0.40676555 0.49692073 0.55187982
## X3P           0.11904456 0.233353555 0.56785931 0.48994891 0.56282942
## FGA           0.07144566 0.050945458 0.62911736 0.79397947 0.83087856
##                        FT          FG        BLK        FTA       STL
## W             0.204906000 0.190396422 0.20392100 0.16188731 0.1161944
## DRB           0.003604883 0.008966352 0.24299082 0.04354267 0.3311279
## AST           0.447208477 0.812266519 0.20308000 0.42797223 0.4431383
## PTS           0.697483850 0.941954732 0.15205537 0.65505876 0.4309903
## oppPTS        0.550587572 0.804467370 0.03076664 0.53721485 0.3434801
## FT            0.000000000 0.538342589 0.16465598 0.95049325 0.3231570
## FG            0.538342589 0.000000000 0.17750275 0.52121683 0.4697303
## BLK           0.164655982 0.177502748 0.00000000 0.21637171 0.1204559
## FTA           0.950493246 0.521216830 0.21637171 0.00000000 0.3700552
## STL           0.323156975 0.469730269 0.12045594 0.37005522 0.0000000
## TOV           0.438684976 0.517630444 0.24156872 0.52566979 0.4640752
## Team.fctr.num 0.154385609 0.198789524 0.07311722 0.13968994 0.1531193
## X2P           0.574293954 0.942919965 0.21771153 0.57454277 0.4890027
## SeasonEnd     0.513699180 0.761598813 0.20416249 0.54689652 0.5020598
## ORB           0.393398702 0.598591143 0.20110479 0.47455838 0.4929415
## X3P           0.508712984 0.668272893 0.23107383 0.53389685 0.4168543
## FGA           0.391691425 0.879662910 0.12109667 0.38212215 0.4805329
##                     TOV Team.fctr.num        X2P SeasonEnd        ORB
## W             0.2431859    0.15119487 0.06927898 0.0000000 0.09573676
## DRB           0.1909569    0.05815482 0.09869019 0.2671271 0.26917934
## AST           0.4305329    0.17491216 0.77711327 0.6721230 0.40676555
## PTS           0.4271383    0.20008998 0.82572457 0.6395277 0.49692073
## oppPTS        0.5824127    0.10100550 0.76984080 0.6324485 0.55187982
## FT            0.4386850    0.15438561 0.57429395 0.5136992 0.39339870
## FG            0.5176304    0.19878952 0.94291996 0.7615988 0.59859114
## BLK           0.2415687    0.07311722 0.21771153 0.2041625 0.20110479
## FTA           0.5256698    0.13968994 0.57454277 0.5468965 0.47455838
## STL           0.4640752    0.15311931 0.48900271 0.5020598 0.49294148
## TOV           0.0000000    0.14045929 0.63771567 0.7238690 0.54598664
## Team.fctr.num 0.1404593    0.00000000 0.19170714 0.2145549 0.20095724
## X2P           0.6377157    0.19170714 0.00000000 0.8654893 0.68311802
## SeasonEnd     0.7238690    0.21455487 0.86548930 0.0000000 0.70201284
## ORB           0.5459866    0.20095724 0.68311802 0.7020128 0.00000000
## X3P           0.6801733    0.14237198 0.87786641 0.8381421 0.66516818
## FGA           0.4499814    0.17904220 0.83827486 0.6849048 0.73750939
##                     X3P        FGA
## W             0.1190446 0.07144566
## DRB           0.2333536 0.05094546
## AST           0.5678593 0.62911736
## PTS           0.4899489 0.79397947
## oppPTS        0.5628294 0.83087856
## FT            0.5087130 0.39169143
## FG            0.6682729 0.87966291
## BLK           0.2310738 0.12109667
## FTA           0.5338969 0.38212215
## STL           0.4168543 0.48053292
## TOV           0.6801733 0.44998136
## Team.fctr.num 0.1423720 0.17904220
## X2P           0.8778664 0.83827486
## SeasonEnd     0.8381421 0.68490478
## ORB           0.6651682 0.73750939
## X3P           0.0000000 0.60756448
## FGA           0.6075645 0.00000000
## [1] "cor(FT, FTA)=0.9505"
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-5.png) 

```
## [1] "cor(Playoffs, FT)=0.2218"
## [1] "cor(Playoffs, FTA)=0.1794"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## Warning in mydelete_cor_features(): Dropping FTA as a feature
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-6.png) 

```
##                          id       cor.y  cor.y.abs
## W                         W  0.79867559 0.79867559
## DRB                     DRB  0.34474930 0.34474930
## AST                     AST  0.31470508 0.31470508
## PTS                     PTS  0.27060133 0.27060133
## oppPTS               oppPTS -0.23276100 0.23276100
## FT                       FT  0.22179906 0.22179906
## FG                       FG  0.18755280 0.18755280
## BLK                     BLK  0.18705612 0.18705612
## STL                     STL  0.17382295 0.17382295
## TOV                     TOV -0.16860161 0.16860161
## Team.fctr.num Team.fctr.num -0.12739314 0.12739314
## X2P                     X2P  0.10803388 0.10803388
## SeasonEnd         SeasonEnd -0.05912787 0.05912787
## ORB                     ORB -0.04170284 0.04170284
## X3P                     X3P  0.02838252 0.02838252
## FGA                     FGA -0.01198546 0.01198546
##                         W          DRB         AST         PTS      oppPTS
## W              1.00000000  0.470897497  0.32005177  0.29882561 -0.33157294
## DRB            0.47089750  1.000000000  0.05647709  0.09029137 -0.21262703
## AST            0.32005177  0.056477086  1.00000000  0.75988915  0.54256181
## PTS            0.29882561  0.090291366  0.75988915  1.00000000  0.78907471
## oppPTS        -0.33157294 -0.212627029  0.54256181  0.78907471  1.00000000
## FT             0.20490600  0.003604883  0.44720848  0.69748385  0.55058757
## FG             0.19039642  0.008966352  0.81226652  0.94195473  0.80446737
## BLK            0.20392100  0.242990816  0.20308000  0.15205537  0.03076664
## STL            0.11619440 -0.331127910  0.44313827  0.43099026  0.34348011
## TOV           -0.24318588 -0.190956902  0.43053288  0.42713832  0.58241272
## Team.fctr.num -0.15119487 -0.058154820 -0.17491216 -0.20008998 -0.10100550
## X2P            0.06927898 -0.098690188  0.77711327  0.82572457  0.76984080
## SeasonEnd      0.00000000  0.267127100 -0.67212300 -0.63952773 -0.63244845
## ORB           -0.09573676 -0.269179339  0.40676555  0.49692073  0.55187982
## X3P            0.11904456  0.233353555 -0.56785931 -0.48994891 -0.56282942
## FGA           -0.07144566 -0.050945458  0.62911736  0.79397947  0.83087856
##                         FT           FG         BLK        STL        TOV
## W              0.204906000  0.190396422  0.20392100  0.1161944 -0.2431859
## DRB            0.003604883  0.008966352  0.24299082 -0.3311279 -0.1909569
## AST            0.447208477  0.812266519  0.20308000  0.4431383  0.4305329
## PTS            0.697483850  0.941954732  0.15205537  0.4309903  0.4271383
## oppPTS         0.550587572  0.804467370  0.03076664  0.3434801  0.5824127
## FT             1.000000000  0.538342589  0.16465598  0.3231570  0.4386850
## FG             0.538342589  1.000000000  0.17750275  0.4697303  0.5176304
## BLK            0.164655982  0.177502748  1.00000000  0.1204559  0.2415687
## STL            0.323156975  0.469730269  0.12045594  1.0000000  0.4640752
## TOV            0.438684976  0.517630444  0.24156872  0.4640752  1.0000000
## Team.fctr.num -0.154385609 -0.198789524 -0.07311722 -0.1531193 -0.1404593
## X2P            0.574293954  0.942919965  0.21771153  0.4890027  0.6377157
## SeasonEnd     -0.513699180 -0.761598813 -0.20416249 -0.5020598 -0.7238690
## ORB            0.393398702  0.598591143  0.20110479  0.4929415  0.5459866
## X3P           -0.508712984 -0.668272893 -0.23107383 -0.4168543 -0.6801733
## FGA            0.391691425  0.879662910  0.12109667  0.4805329  0.4499814
##               Team.fctr.num         X2P  SeasonEnd         ORB        X3P
## W               -0.15119487  0.06927898  0.0000000 -0.09573676  0.1190446
## DRB             -0.05815482 -0.09869019  0.2671271 -0.26917934  0.2333536
## AST             -0.17491216  0.77711327 -0.6721230  0.40676555 -0.5678593
## PTS             -0.20008998  0.82572457 -0.6395277  0.49692073 -0.4899489
## oppPTS          -0.10100550  0.76984080 -0.6324485  0.55187982 -0.5628294
## FT              -0.15438561  0.57429395 -0.5136992  0.39339870 -0.5087130
## FG              -0.19878952  0.94291996 -0.7615988  0.59859114 -0.6682729
## BLK             -0.07311722  0.21771153 -0.2041625  0.20110479 -0.2310738
## STL             -0.15311931  0.48900271 -0.5020598  0.49294148 -0.4168543
## TOV             -0.14045929  0.63771567 -0.7238690  0.54598664 -0.6801733
## Team.fctr.num    1.00000000 -0.19170714  0.2145549 -0.20095724  0.1423720
## X2P             -0.19170714  1.00000000 -0.8654893  0.68311802 -0.8778664
## SeasonEnd        0.21455487 -0.86548930  1.0000000 -0.70201284  0.8381421
## ORB             -0.20095724  0.68311802 -0.7020128  1.00000000 -0.6651682
## X3P              0.14237198 -0.87786641  0.8381421 -0.66516818  1.0000000
## FGA             -0.17904220  0.83827486 -0.6849048  0.73750939 -0.6075645
##                       FGA
## W             -0.07144566
## DRB           -0.05094546
## AST            0.62911736
## PTS            0.79397947
## oppPTS         0.83087856
## FT             0.39169143
## FG             0.87966291
## BLK            0.12109667
## STL            0.48053292
## TOV            0.44998136
## Team.fctr.num -0.17904220
## X2P            0.83827486
## SeasonEnd     -0.68490478
## ORB            0.73750939
## X3P           -0.60756448
## FGA            1.00000000
##                        W         DRB        AST        PTS     oppPTS
## W             0.00000000 0.470897497 0.32005177 0.29882561 0.33157294
## DRB           0.47089750 0.000000000 0.05647709 0.09029137 0.21262703
## AST           0.32005177 0.056477086 0.00000000 0.75988915 0.54256181
## PTS           0.29882561 0.090291366 0.75988915 0.00000000 0.78907471
## oppPTS        0.33157294 0.212627029 0.54256181 0.78907471 0.00000000
## FT            0.20490600 0.003604883 0.44720848 0.69748385 0.55058757
## FG            0.19039642 0.008966352 0.81226652 0.94195473 0.80446737
## BLK           0.20392100 0.242990816 0.20308000 0.15205537 0.03076664
## STL           0.11619440 0.331127910 0.44313827 0.43099026 0.34348011
## TOV           0.24318588 0.190956902 0.43053288 0.42713832 0.58241272
## Team.fctr.num 0.15119487 0.058154820 0.17491216 0.20008998 0.10100550
## X2P           0.06927898 0.098690188 0.77711327 0.82572457 0.76984080
## SeasonEnd     0.00000000 0.267127100 0.67212300 0.63952773 0.63244845
## ORB           0.09573676 0.269179339 0.40676555 0.49692073 0.55187982
## X3P           0.11904456 0.233353555 0.56785931 0.48994891 0.56282942
## FGA           0.07144566 0.050945458 0.62911736 0.79397947 0.83087856
##                        FT          FG        BLK       STL       TOV
## W             0.204906000 0.190396422 0.20392100 0.1161944 0.2431859
## DRB           0.003604883 0.008966352 0.24299082 0.3311279 0.1909569
## AST           0.447208477 0.812266519 0.20308000 0.4431383 0.4305329
## PTS           0.697483850 0.941954732 0.15205537 0.4309903 0.4271383
## oppPTS        0.550587572 0.804467370 0.03076664 0.3434801 0.5824127
## FT            0.000000000 0.538342589 0.16465598 0.3231570 0.4386850
## FG            0.538342589 0.000000000 0.17750275 0.4697303 0.5176304
## BLK           0.164655982 0.177502748 0.00000000 0.1204559 0.2415687
## STL           0.323156975 0.469730269 0.12045594 0.0000000 0.4640752
## TOV           0.438684976 0.517630444 0.24156872 0.4640752 0.0000000
## Team.fctr.num 0.154385609 0.198789524 0.07311722 0.1531193 0.1404593
## X2P           0.574293954 0.942919965 0.21771153 0.4890027 0.6377157
## SeasonEnd     0.513699180 0.761598813 0.20416249 0.5020598 0.7238690
## ORB           0.393398702 0.598591143 0.20110479 0.4929415 0.5459866
## X3P           0.508712984 0.668272893 0.23107383 0.4168543 0.6801733
## FGA           0.391691425 0.879662910 0.12109667 0.4805329 0.4499814
##               Team.fctr.num        X2P SeasonEnd        ORB       X3P
## W                0.15119487 0.06927898 0.0000000 0.09573676 0.1190446
## DRB              0.05815482 0.09869019 0.2671271 0.26917934 0.2333536
## AST              0.17491216 0.77711327 0.6721230 0.40676555 0.5678593
## PTS              0.20008998 0.82572457 0.6395277 0.49692073 0.4899489
## oppPTS           0.10100550 0.76984080 0.6324485 0.55187982 0.5628294
## FT               0.15438561 0.57429395 0.5136992 0.39339870 0.5087130
## FG               0.19878952 0.94291996 0.7615988 0.59859114 0.6682729
## BLK              0.07311722 0.21771153 0.2041625 0.20110479 0.2310738
## STL              0.15311931 0.48900271 0.5020598 0.49294148 0.4168543
## TOV              0.14045929 0.63771567 0.7238690 0.54598664 0.6801733
## Team.fctr.num    0.00000000 0.19170714 0.2145549 0.20095724 0.1423720
## X2P              0.19170714 0.00000000 0.8654893 0.68311802 0.8778664
## SeasonEnd        0.21455487 0.86548930 0.0000000 0.70201284 0.8381421
## ORB              0.20095724 0.68311802 0.7020128 0.00000000 0.6651682
## X3P              0.14237198 0.87786641 0.8381421 0.66516818 0.0000000
## FGA              0.17904220 0.83827486 0.6849048 0.73750939 0.6075645
##                      FGA
## W             0.07144566
## DRB           0.05094546
## AST           0.62911736
## PTS           0.79397947
## oppPTS        0.83087856
## FT            0.39169143
## FG            0.87966291
## BLK           0.12109667
## STL           0.48053292
## TOV           0.44998136
## Team.fctr.num 0.17904220
## X2P           0.83827486
## SeasonEnd     0.68490478
## ORB           0.73750939
## X3P           0.60756448
## FGA           0.00000000
## [1] "cor(FG, X2P)=0.9429"
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-7.png) 

```
## [1] "cor(Playoffs, FG)=0.1876"
## [1] "cor(Playoffs, X2P)=0.1080"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## Warning in mydelete_cor_features(): Dropping X2P as a feature
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-8.png) 

```
##                          id       cor.y  cor.y.abs
## W                         W  0.79867559 0.79867559
## DRB                     DRB  0.34474930 0.34474930
## AST                     AST  0.31470508 0.31470508
## PTS                     PTS  0.27060133 0.27060133
## oppPTS               oppPTS -0.23276100 0.23276100
## FT                       FT  0.22179906 0.22179906
## FG                       FG  0.18755280 0.18755280
## BLK                     BLK  0.18705612 0.18705612
## STL                     STL  0.17382295 0.17382295
## TOV                     TOV -0.16860161 0.16860161
## Team.fctr.num Team.fctr.num -0.12739314 0.12739314
## SeasonEnd         SeasonEnd -0.05912787 0.05912787
## ORB                     ORB -0.04170284 0.04170284
## X3P                     X3P  0.02838252 0.02838252
## FGA                     FGA -0.01198546 0.01198546
##                         W          DRB         AST         PTS      oppPTS
## W              1.00000000  0.470897497  0.32005177  0.29882561 -0.33157294
## DRB            0.47089750  1.000000000  0.05647709  0.09029137 -0.21262703
## AST            0.32005177  0.056477086  1.00000000  0.75988915  0.54256181
## PTS            0.29882561  0.090291366  0.75988915  1.00000000  0.78907471
## oppPTS        -0.33157294 -0.212627029  0.54256181  0.78907471  1.00000000
## FT             0.20490600  0.003604883  0.44720848  0.69748385  0.55058757
## FG             0.19039642  0.008966352  0.81226652  0.94195473  0.80446737
## BLK            0.20392100  0.242990816  0.20308000  0.15205537  0.03076664
## STL            0.11619440 -0.331127910  0.44313827  0.43099026  0.34348011
## TOV           -0.24318588 -0.190956902  0.43053288  0.42713832  0.58241272
## Team.fctr.num -0.15119487 -0.058154820 -0.17491216 -0.20008998 -0.10100550
## SeasonEnd      0.00000000  0.267127100 -0.67212300 -0.63952773 -0.63244845
## ORB           -0.09573676 -0.269179339  0.40676555  0.49692073  0.55187982
## X3P            0.11904456  0.233353555 -0.56785931 -0.48994891 -0.56282942
## FGA           -0.07144566 -0.050945458  0.62911736  0.79397947  0.83087856
##                         FT           FG         BLK        STL        TOV
## W              0.204906000  0.190396422  0.20392100  0.1161944 -0.2431859
## DRB            0.003604883  0.008966352  0.24299082 -0.3311279 -0.1909569
## AST            0.447208477  0.812266519  0.20308000  0.4431383  0.4305329
## PTS            0.697483850  0.941954732  0.15205537  0.4309903  0.4271383
## oppPTS         0.550587572  0.804467370  0.03076664  0.3434801  0.5824127
## FT             1.000000000  0.538342589  0.16465598  0.3231570  0.4386850
## FG             0.538342589  1.000000000  0.17750275  0.4697303  0.5176304
## BLK            0.164655982  0.177502748  1.00000000  0.1204559  0.2415687
## STL            0.323156975  0.469730269  0.12045594  1.0000000  0.4640752
## TOV            0.438684976  0.517630444  0.24156872  0.4640752  1.0000000
## Team.fctr.num -0.154385609 -0.198789524 -0.07311722 -0.1531193 -0.1404593
## SeasonEnd     -0.513699180 -0.761598813 -0.20416249 -0.5020598 -0.7238690
## ORB            0.393398702  0.598591143  0.20110479  0.4929415  0.5459866
## X3P           -0.508712984 -0.668272893 -0.23107383 -0.4168543 -0.6801733
## FGA            0.391691425  0.879662910  0.12109667  0.4805329  0.4499814
##               Team.fctr.num  SeasonEnd         ORB        X3P         FGA
## W               -0.15119487  0.0000000 -0.09573676  0.1190446 -0.07144566
## DRB             -0.05815482  0.2671271 -0.26917934  0.2333536 -0.05094546
## AST             -0.17491216 -0.6721230  0.40676555 -0.5678593  0.62911736
## PTS             -0.20008998 -0.6395277  0.49692073 -0.4899489  0.79397947
## oppPTS          -0.10100550 -0.6324485  0.55187982 -0.5628294  0.83087856
## FT              -0.15438561 -0.5136992  0.39339870 -0.5087130  0.39169143
## FG              -0.19878952 -0.7615988  0.59859114 -0.6682729  0.87966291
## BLK             -0.07311722 -0.2041625  0.20110479 -0.2310738  0.12109667
## STL             -0.15311931 -0.5020598  0.49294148 -0.4168543  0.48053292
## TOV             -0.14045929 -0.7238690  0.54598664 -0.6801733  0.44998136
## Team.fctr.num    1.00000000  0.2145549 -0.20095724  0.1423720 -0.17904220
## SeasonEnd        0.21455487  1.0000000 -0.70201284  0.8381421 -0.68490478
## ORB             -0.20095724 -0.7020128  1.00000000 -0.6651682  0.73750939
## X3P              0.14237198  0.8381421 -0.66516818  1.0000000 -0.60756448
## FGA             -0.17904220 -0.6849048  0.73750939 -0.6075645  1.00000000
##                        W         DRB        AST        PTS     oppPTS
## W             0.00000000 0.470897497 0.32005177 0.29882561 0.33157294
## DRB           0.47089750 0.000000000 0.05647709 0.09029137 0.21262703
## AST           0.32005177 0.056477086 0.00000000 0.75988915 0.54256181
## PTS           0.29882561 0.090291366 0.75988915 0.00000000 0.78907471
## oppPTS        0.33157294 0.212627029 0.54256181 0.78907471 0.00000000
## FT            0.20490600 0.003604883 0.44720848 0.69748385 0.55058757
## FG            0.19039642 0.008966352 0.81226652 0.94195473 0.80446737
## BLK           0.20392100 0.242990816 0.20308000 0.15205537 0.03076664
## STL           0.11619440 0.331127910 0.44313827 0.43099026 0.34348011
## TOV           0.24318588 0.190956902 0.43053288 0.42713832 0.58241272
## Team.fctr.num 0.15119487 0.058154820 0.17491216 0.20008998 0.10100550
## SeasonEnd     0.00000000 0.267127100 0.67212300 0.63952773 0.63244845
## ORB           0.09573676 0.269179339 0.40676555 0.49692073 0.55187982
## X3P           0.11904456 0.233353555 0.56785931 0.48994891 0.56282942
## FGA           0.07144566 0.050945458 0.62911736 0.79397947 0.83087856
##                        FT          FG        BLK       STL       TOV
## W             0.204906000 0.190396422 0.20392100 0.1161944 0.2431859
## DRB           0.003604883 0.008966352 0.24299082 0.3311279 0.1909569
## AST           0.447208477 0.812266519 0.20308000 0.4431383 0.4305329
## PTS           0.697483850 0.941954732 0.15205537 0.4309903 0.4271383
## oppPTS        0.550587572 0.804467370 0.03076664 0.3434801 0.5824127
## FT            0.000000000 0.538342589 0.16465598 0.3231570 0.4386850
## FG            0.538342589 0.000000000 0.17750275 0.4697303 0.5176304
## BLK           0.164655982 0.177502748 0.00000000 0.1204559 0.2415687
## STL           0.323156975 0.469730269 0.12045594 0.0000000 0.4640752
## TOV           0.438684976 0.517630444 0.24156872 0.4640752 0.0000000
## Team.fctr.num 0.154385609 0.198789524 0.07311722 0.1531193 0.1404593
## SeasonEnd     0.513699180 0.761598813 0.20416249 0.5020598 0.7238690
## ORB           0.393398702 0.598591143 0.20110479 0.4929415 0.5459866
## X3P           0.508712984 0.668272893 0.23107383 0.4168543 0.6801733
## FGA           0.391691425 0.879662910 0.12109667 0.4805329 0.4499814
##               Team.fctr.num SeasonEnd        ORB       X3P        FGA
## W                0.15119487 0.0000000 0.09573676 0.1190446 0.07144566
## DRB              0.05815482 0.2671271 0.26917934 0.2333536 0.05094546
## AST              0.17491216 0.6721230 0.40676555 0.5678593 0.62911736
## PTS              0.20008998 0.6395277 0.49692073 0.4899489 0.79397947
## oppPTS           0.10100550 0.6324485 0.55187982 0.5628294 0.83087856
## FT               0.15438561 0.5136992 0.39339870 0.5087130 0.39169143
## FG               0.19878952 0.7615988 0.59859114 0.6682729 0.87966291
## BLK              0.07311722 0.2041625 0.20110479 0.2310738 0.12109667
## STL              0.15311931 0.5020598 0.49294148 0.4168543 0.48053292
## TOV              0.14045929 0.7238690 0.54598664 0.6801733 0.44998136
## Team.fctr.num    0.00000000 0.2145549 0.20095724 0.1423720 0.17904220
## SeasonEnd        0.21455487 0.0000000 0.70201284 0.8381421 0.68490478
## ORB              0.20095724 0.7020128 0.00000000 0.6651682 0.73750939
## X3P              0.14237198 0.8381421 0.66516818 0.0000000 0.60756448
## FGA              0.17904220 0.6849048 0.73750939 0.6075645 0.00000000
## [1] "cor(PTS, FG)=0.9420"
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-9.png) 

```
## [1] "cor(Playoffs, PTS)=0.2706"
## [1] "cor(Playoffs, FG)=0.1876"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## Warning in mydelete_cor_features(): Dropping FG as a feature
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-10.png) 

```
##                          id       cor.y  cor.y.abs
## W                         W  0.79867559 0.79867559
## DRB                     DRB  0.34474930 0.34474930
## AST                     AST  0.31470508 0.31470508
## PTS                     PTS  0.27060133 0.27060133
## oppPTS               oppPTS -0.23276100 0.23276100
## FT                       FT  0.22179906 0.22179906
## BLK                     BLK  0.18705612 0.18705612
## STL                     STL  0.17382295 0.17382295
## TOV                     TOV -0.16860161 0.16860161
## Team.fctr.num Team.fctr.num -0.12739314 0.12739314
## SeasonEnd         SeasonEnd -0.05912787 0.05912787
## ORB                     ORB -0.04170284 0.04170284
## X3P                     X3P  0.02838252 0.02838252
## FGA                     FGA -0.01198546 0.01198546
##                         W          DRB         AST         PTS      oppPTS
## W              1.00000000  0.470897497  0.32005177  0.29882561 -0.33157294
## DRB            0.47089750  1.000000000  0.05647709  0.09029137 -0.21262703
## AST            0.32005177  0.056477086  1.00000000  0.75988915  0.54256181
## PTS            0.29882561  0.090291366  0.75988915  1.00000000  0.78907471
## oppPTS        -0.33157294 -0.212627029  0.54256181  0.78907471  1.00000000
## FT             0.20490600  0.003604883  0.44720848  0.69748385  0.55058757
## BLK            0.20392100  0.242990816  0.20308000  0.15205537  0.03076664
## STL            0.11619440 -0.331127910  0.44313827  0.43099026  0.34348011
## TOV           -0.24318588 -0.190956902  0.43053288  0.42713832  0.58241272
## Team.fctr.num -0.15119487 -0.058154820 -0.17491216 -0.20008998 -0.10100550
## SeasonEnd      0.00000000  0.267127100 -0.67212300 -0.63952773 -0.63244845
## ORB           -0.09573676 -0.269179339  0.40676555  0.49692073  0.55187982
## X3P            0.11904456  0.233353555 -0.56785931 -0.48994891 -0.56282942
## FGA           -0.07144566 -0.050945458  0.62911736  0.79397947  0.83087856
##                         FT         BLK        STL        TOV Team.fctr.num
## W              0.204906000  0.20392100  0.1161944 -0.2431859   -0.15119487
## DRB            0.003604883  0.24299082 -0.3311279 -0.1909569   -0.05815482
## AST            0.447208477  0.20308000  0.4431383  0.4305329   -0.17491216
## PTS            0.697483850  0.15205537  0.4309903  0.4271383   -0.20008998
## oppPTS         0.550587572  0.03076664  0.3434801  0.5824127   -0.10100550
## FT             1.000000000  0.16465598  0.3231570  0.4386850   -0.15438561
## BLK            0.164655982  1.00000000  0.1204559  0.2415687   -0.07311722
## STL            0.323156975  0.12045594  1.0000000  0.4640752   -0.15311931
## TOV            0.438684976  0.24156872  0.4640752  1.0000000   -0.14045929
## Team.fctr.num -0.154385609 -0.07311722 -0.1531193 -0.1404593    1.00000000
## SeasonEnd     -0.513699180 -0.20416249 -0.5020598 -0.7238690    0.21455487
## ORB            0.393398702  0.20110479  0.4929415  0.5459866   -0.20095724
## X3P           -0.508712984 -0.23107383 -0.4168543 -0.6801733    0.14237198
## FGA            0.391691425  0.12109667  0.4805329  0.4499814   -0.17904220
##                SeasonEnd         ORB        X3P         FGA
## W              0.0000000 -0.09573676  0.1190446 -0.07144566
## DRB            0.2671271 -0.26917934  0.2333536 -0.05094546
## AST           -0.6721230  0.40676555 -0.5678593  0.62911736
## PTS           -0.6395277  0.49692073 -0.4899489  0.79397947
## oppPTS        -0.6324485  0.55187982 -0.5628294  0.83087856
## FT            -0.5136992  0.39339870 -0.5087130  0.39169143
## BLK           -0.2041625  0.20110479 -0.2310738  0.12109667
## STL           -0.5020598  0.49294148 -0.4168543  0.48053292
## TOV           -0.7238690  0.54598664 -0.6801733  0.44998136
## Team.fctr.num  0.2145549 -0.20095724  0.1423720 -0.17904220
## SeasonEnd      1.0000000 -0.70201284  0.8381421 -0.68490478
## ORB           -0.7020128  1.00000000 -0.6651682  0.73750939
## X3P            0.8381421 -0.66516818  1.0000000 -0.60756448
## FGA           -0.6849048  0.73750939 -0.6075645  1.00000000
##                        W         DRB        AST        PTS     oppPTS
## W             0.00000000 0.470897497 0.32005177 0.29882561 0.33157294
## DRB           0.47089750 0.000000000 0.05647709 0.09029137 0.21262703
## AST           0.32005177 0.056477086 0.00000000 0.75988915 0.54256181
## PTS           0.29882561 0.090291366 0.75988915 0.00000000 0.78907471
## oppPTS        0.33157294 0.212627029 0.54256181 0.78907471 0.00000000
## FT            0.20490600 0.003604883 0.44720848 0.69748385 0.55058757
## BLK           0.20392100 0.242990816 0.20308000 0.15205537 0.03076664
## STL           0.11619440 0.331127910 0.44313827 0.43099026 0.34348011
## TOV           0.24318588 0.190956902 0.43053288 0.42713832 0.58241272
## Team.fctr.num 0.15119487 0.058154820 0.17491216 0.20008998 0.10100550
## SeasonEnd     0.00000000 0.267127100 0.67212300 0.63952773 0.63244845
## ORB           0.09573676 0.269179339 0.40676555 0.49692073 0.55187982
## X3P           0.11904456 0.233353555 0.56785931 0.48994891 0.56282942
## FGA           0.07144566 0.050945458 0.62911736 0.79397947 0.83087856
##                        FT        BLK       STL       TOV Team.fctr.num
## W             0.204906000 0.20392100 0.1161944 0.2431859    0.15119487
## DRB           0.003604883 0.24299082 0.3311279 0.1909569    0.05815482
## AST           0.447208477 0.20308000 0.4431383 0.4305329    0.17491216
## PTS           0.697483850 0.15205537 0.4309903 0.4271383    0.20008998
## oppPTS        0.550587572 0.03076664 0.3434801 0.5824127    0.10100550
## FT            0.000000000 0.16465598 0.3231570 0.4386850    0.15438561
## BLK           0.164655982 0.00000000 0.1204559 0.2415687    0.07311722
## STL           0.323156975 0.12045594 0.0000000 0.4640752    0.15311931
## TOV           0.438684976 0.24156872 0.4640752 0.0000000    0.14045929
## Team.fctr.num 0.154385609 0.07311722 0.1531193 0.1404593    0.00000000
## SeasonEnd     0.513699180 0.20416249 0.5020598 0.7238690    0.21455487
## ORB           0.393398702 0.20110479 0.4929415 0.5459866    0.20095724
## X3P           0.508712984 0.23107383 0.4168543 0.6801733    0.14237198
## FGA           0.391691425 0.12109667 0.4805329 0.4499814    0.17904220
##               SeasonEnd        ORB       X3P        FGA
## W             0.0000000 0.09573676 0.1190446 0.07144566
## DRB           0.2671271 0.26917934 0.2333536 0.05094546
## AST           0.6721230 0.40676555 0.5678593 0.62911736
## PTS           0.6395277 0.49692073 0.4899489 0.79397947
## oppPTS        0.6324485 0.55187982 0.5628294 0.83087856
## FT            0.5136992 0.39339870 0.5087130 0.39169143
## BLK           0.2041625 0.20110479 0.2310738 0.12109667
## STL           0.5020598 0.49294148 0.4168543 0.48053292
## TOV           0.7238690 0.54598664 0.6801733 0.44998136
## Team.fctr.num 0.2145549 0.20095724 0.1423720 0.17904220
## SeasonEnd     0.0000000 0.70201284 0.8381421 0.68490478
## ORB           0.7020128 0.00000000 0.6651682 0.73750939
## X3P           0.8381421 0.66516818 0.0000000 0.60756448
## FGA           0.6849048 0.73750939 0.6075645 0.00000000
## [1] "cor(SeasonEnd, X3P)=0.8381"
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-11.png) 

```
## [1] "cor(Playoffs, SeasonEnd)=-0.0591"
## [1] "cor(Playoffs, X3P)=0.0284"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## Warning in mydelete_cor_features(): Dropping X3P as a feature
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-12.png) 

```
##                          id       cor.y  cor.y.abs
## W                         W  0.79867559 0.79867559
## DRB                     DRB  0.34474930 0.34474930
## AST                     AST  0.31470508 0.31470508
## PTS                     PTS  0.27060133 0.27060133
## oppPTS               oppPTS -0.23276100 0.23276100
## FT                       FT  0.22179906 0.22179906
## BLK                     BLK  0.18705612 0.18705612
## STL                     STL  0.17382295 0.17382295
## TOV                     TOV -0.16860161 0.16860161
## Team.fctr.num Team.fctr.num -0.12739314 0.12739314
## SeasonEnd         SeasonEnd -0.05912787 0.05912787
## ORB                     ORB -0.04170284 0.04170284
## FGA                     FGA -0.01198546 0.01198546
##                         W          DRB         AST         PTS      oppPTS
## W              1.00000000  0.470897497  0.32005177  0.29882561 -0.33157294
## DRB            0.47089750  1.000000000  0.05647709  0.09029137 -0.21262703
## AST            0.32005177  0.056477086  1.00000000  0.75988915  0.54256181
## PTS            0.29882561  0.090291366  0.75988915  1.00000000  0.78907471
## oppPTS        -0.33157294 -0.212627029  0.54256181  0.78907471  1.00000000
## FT             0.20490600  0.003604883  0.44720848  0.69748385  0.55058757
## BLK            0.20392100  0.242990816  0.20308000  0.15205537  0.03076664
## STL            0.11619440 -0.331127910  0.44313827  0.43099026  0.34348011
## TOV           -0.24318588 -0.190956902  0.43053288  0.42713832  0.58241272
## Team.fctr.num -0.15119487 -0.058154820 -0.17491216 -0.20008998 -0.10100550
## SeasonEnd      0.00000000  0.267127100 -0.67212300 -0.63952773 -0.63244845
## ORB           -0.09573676 -0.269179339  0.40676555  0.49692073  0.55187982
## FGA           -0.07144566 -0.050945458  0.62911736  0.79397947  0.83087856
##                         FT         BLK        STL        TOV Team.fctr.num
## W              0.204906000  0.20392100  0.1161944 -0.2431859   -0.15119487
## DRB            0.003604883  0.24299082 -0.3311279 -0.1909569   -0.05815482
## AST            0.447208477  0.20308000  0.4431383  0.4305329   -0.17491216
## PTS            0.697483850  0.15205537  0.4309903  0.4271383   -0.20008998
## oppPTS         0.550587572  0.03076664  0.3434801  0.5824127   -0.10100550
## FT             1.000000000  0.16465598  0.3231570  0.4386850   -0.15438561
## BLK            0.164655982  1.00000000  0.1204559  0.2415687   -0.07311722
## STL            0.323156975  0.12045594  1.0000000  0.4640752   -0.15311931
## TOV            0.438684976  0.24156872  0.4640752  1.0000000   -0.14045929
## Team.fctr.num -0.154385609 -0.07311722 -0.1531193 -0.1404593    1.00000000
## SeasonEnd     -0.513699180 -0.20416249 -0.5020598 -0.7238690    0.21455487
## ORB            0.393398702  0.20110479  0.4929415  0.5459866   -0.20095724
## FGA            0.391691425  0.12109667  0.4805329  0.4499814   -0.17904220
##                SeasonEnd         ORB         FGA
## W              0.0000000 -0.09573676 -0.07144566
## DRB            0.2671271 -0.26917934 -0.05094546
## AST           -0.6721230  0.40676555  0.62911736
## PTS           -0.6395277  0.49692073  0.79397947
## oppPTS        -0.6324485  0.55187982  0.83087856
## FT            -0.5136992  0.39339870  0.39169143
## BLK           -0.2041625  0.20110479  0.12109667
## STL           -0.5020598  0.49294148  0.48053292
## TOV           -0.7238690  0.54598664  0.44998136
## Team.fctr.num  0.2145549 -0.20095724 -0.17904220
## SeasonEnd      1.0000000 -0.70201284 -0.68490478
## ORB           -0.7020128  1.00000000  0.73750939
## FGA           -0.6849048  0.73750939  1.00000000
##                        W         DRB        AST        PTS     oppPTS
## W             0.00000000 0.470897497 0.32005177 0.29882561 0.33157294
## DRB           0.47089750 0.000000000 0.05647709 0.09029137 0.21262703
## AST           0.32005177 0.056477086 0.00000000 0.75988915 0.54256181
## PTS           0.29882561 0.090291366 0.75988915 0.00000000 0.78907471
## oppPTS        0.33157294 0.212627029 0.54256181 0.78907471 0.00000000
## FT            0.20490600 0.003604883 0.44720848 0.69748385 0.55058757
## BLK           0.20392100 0.242990816 0.20308000 0.15205537 0.03076664
## STL           0.11619440 0.331127910 0.44313827 0.43099026 0.34348011
## TOV           0.24318588 0.190956902 0.43053288 0.42713832 0.58241272
## Team.fctr.num 0.15119487 0.058154820 0.17491216 0.20008998 0.10100550
## SeasonEnd     0.00000000 0.267127100 0.67212300 0.63952773 0.63244845
## ORB           0.09573676 0.269179339 0.40676555 0.49692073 0.55187982
## FGA           0.07144566 0.050945458 0.62911736 0.79397947 0.83087856
##                        FT        BLK       STL       TOV Team.fctr.num
## W             0.204906000 0.20392100 0.1161944 0.2431859    0.15119487
## DRB           0.003604883 0.24299082 0.3311279 0.1909569    0.05815482
## AST           0.447208477 0.20308000 0.4431383 0.4305329    0.17491216
## PTS           0.697483850 0.15205537 0.4309903 0.4271383    0.20008998
## oppPTS        0.550587572 0.03076664 0.3434801 0.5824127    0.10100550
## FT            0.000000000 0.16465598 0.3231570 0.4386850    0.15438561
## BLK           0.164655982 0.00000000 0.1204559 0.2415687    0.07311722
## STL           0.323156975 0.12045594 0.0000000 0.4640752    0.15311931
## TOV           0.438684976 0.24156872 0.4640752 0.0000000    0.14045929
## Team.fctr.num 0.154385609 0.07311722 0.1531193 0.1404593    0.00000000
## SeasonEnd     0.513699180 0.20416249 0.5020598 0.7238690    0.21455487
## ORB           0.393398702 0.20110479 0.4929415 0.5459866    0.20095724
## FGA           0.391691425 0.12109667 0.4805329 0.4499814    0.17904220
##               SeasonEnd        ORB        FGA
## W             0.0000000 0.09573676 0.07144566
## DRB           0.2671271 0.26917934 0.05094546
## AST           0.6721230 0.40676555 0.62911736
## PTS           0.6395277 0.49692073 0.79397947
## oppPTS        0.6324485 0.55187982 0.83087856
## FT            0.5136992 0.39339870 0.39169143
## BLK           0.2041625 0.20110479 0.12109667
## STL           0.5020598 0.49294148 0.48053292
## TOV           0.7238690 0.54598664 0.44998136
## Team.fctr.num 0.2145549 0.20095724 0.17904220
## SeasonEnd     0.0000000 0.70201284 0.68490478
## ORB           0.7020128 0.00000000 0.73750939
## FGA           0.6849048 0.73750939 0.00000000
## [1] "cor(oppPTS, FGA)=0.8309"
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-13.png) 

```
## [1] "cor(Playoffs, oppPTS)=-0.2328"
## [1] "cor(Playoffs, FGA)=-0.0120"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## Warning in mydelete_cor_features(): Dropping FGA as a feature
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-14.png) 

```
##                          id       cor.y  cor.y.abs
## W                         W  0.79867559 0.79867559
## DRB                     DRB  0.34474930 0.34474930
## AST                     AST  0.31470508 0.31470508
## PTS                     PTS  0.27060133 0.27060133
## oppPTS               oppPTS -0.23276100 0.23276100
## FT                       FT  0.22179906 0.22179906
## BLK                     BLK  0.18705612 0.18705612
## STL                     STL  0.17382295 0.17382295
## TOV                     TOV -0.16860161 0.16860161
## Team.fctr.num Team.fctr.num -0.12739314 0.12739314
## SeasonEnd         SeasonEnd -0.05912787 0.05912787
## ORB                     ORB -0.04170284 0.04170284
##                         W          DRB         AST         PTS      oppPTS
## W              1.00000000  0.470897497  0.32005177  0.29882561 -0.33157294
## DRB            0.47089750  1.000000000  0.05647709  0.09029137 -0.21262703
## AST            0.32005177  0.056477086  1.00000000  0.75988915  0.54256181
## PTS            0.29882561  0.090291366  0.75988915  1.00000000  0.78907471
## oppPTS        -0.33157294 -0.212627029  0.54256181  0.78907471  1.00000000
## FT             0.20490600  0.003604883  0.44720848  0.69748385  0.55058757
## BLK            0.20392100  0.242990816  0.20308000  0.15205537  0.03076664
## STL            0.11619440 -0.331127910  0.44313827  0.43099026  0.34348011
## TOV           -0.24318588 -0.190956902  0.43053288  0.42713832  0.58241272
## Team.fctr.num -0.15119487 -0.058154820 -0.17491216 -0.20008998 -0.10100550
## SeasonEnd      0.00000000  0.267127100 -0.67212300 -0.63952773 -0.63244845
## ORB           -0.09573676 -0.269179339  0.40676555  0.49692073  0.55187982
##                         FT         BLK        STL        TOV Team.fctr.num
## W              0.204906000  0.20392100  0.1161944 -0.2431859   -0.15119487
## DRB            0.003604883  0.24299082 -0.3311279 -0.1909569   -0.05815482
## AST            0.447208477  0.20308000  0.4431383  0.4305329   -0.17491216
## PTS            0.697483850  0.15205537  0.4309903  0.4271383   -0.20008998
## oppPTS         0.550587572  0.03076664  0.3434801  0.5824127   -0.10100550
## FT             1.000000000  0.16465598  0.3231570  0.4386850   -0.15438561
## BLK            0.164655982  1.00000000  0.1204559  0.2415687   -0.07311722
## STL            0.323156975  0.12045594  1.0000000  0.4640752   -0.15311931
## TOV            0.438684976  0.24156872  0.4640752  1.0000000   -0.14045929
## Team.fctr.num -0.154385609 -0.07311722 -0.1531193 -0.1404593    1.00000000
## SeasonEnd     -0.513699180 -0.20416249 -0.5020598 -0.7238690    0.21455487
## ORB            0.393398702  0.20110479  0.4929415  0.5459866   -0.20095724
##                SeasonEnd         ORB
## W              0.0000000 -0.09573676
## DRB            0.2671271 -0.26917934
## AST           -0.6721230  0.40676555
## PTS           -0.6395277  0.49692073
## oppPTS        -0.6324485  0.55187982
## FT            -0.5136992  0.39339870
## BLK           -0.2041625  0.20110479
## STL           -0.5020598  0.49294148
## TOV           -0.7238690  0.54598664
## Team.fctr.num  0.2145549 -0.20095724
## SeasonEnd      1.0000000 -0.70201284
## ORB           -0.7020128  1.00000000
##                        W         DRB        AST        PTS     oppPTS
## W             0.00000000 0.470897497 0.32005177 0.29882561 0.33157294
## DRB           0.47089750 0.000000000 0.05647709 0.09029137 0.21262703
## AST           0.32005177 0.056477086 0.00000000 0.75988915 0.54256181
## PTS           0.29882561 0.090291366 0.75988915 0.00000000 0.78907471
## oppPTS        0.33157294 0.212627029 0.54256181 0.78907471 0.00000000
## FT            0.20490600 0.003604883 0.44720848 0.69748385 0.55058757
## BLK           0.20392100 0.242990816 0.20308000 0.15205537 0.03076664
## STL           0.11619440 0.331127910 0.44313827 0.43099026 0.34348011
## TOV           0.24318588 0.190956902 0.43053288 0.42713832 0.58241272
## Team.fctr.num 0.15119487 0.058154820 0.17491216 0.20008998 0.10100550
## SeasonEnd     0.00000000 0.267127100 0.67212300 0.63952773 0.63244845
## ORB           0.09573676 0.269179339 0.40676555 0.49692073 0.55187982
##                        FT        BLK       STL       TOV Team.fctr.num
## W             0.204906000 0.20392100 0.1161944 0.2431859    0.15119487
## DRB           0.003604883 0.24299082 0.3311279 0.1909569    0.05815482
## AST           0.447208477 0.20308000 0.4431383 0.4305329    0.17491216
## PTS           0.697483850 0.15205537 0.4309903 0.4271383    0.20008998
## oppPTS        0.550587572 0.03076664 0.3434801 0.5824127    0.10100550
## FT            0.000000000 0.16465598 0.3231570 0.4386850    0.15438561
## BLK           0.164655982 0.00000000 0.1204559 0.2415687    0.07311722
## STL           0.323156975 0.12045594 0.0000000 0.4640752    0.15311931
## TOV           0.438684976 0.24156872 0.4640752 0.0000000    0.14045929
## Team.fctr.num 0.154385609 0.07311722 0.1531193 0.1404593    0.00000000
## SeasonEnd     0.513699180 0.20416249 0.5020598 0.7238690    0.21455487
## ORB           0.393398702 0.20110479 0.4929415 0.5459866    0.20095724
##               SeasonEnd        ORB
## W             0.0000000 0.09573676
## DRB           0.2671271 0.26917934
## AST           0.6721230 0.40676555
## PTS           0.6395277 0.49692073
## oppPTS        0.6324485 0.55187982
## FT            0.5136992 0.39339870
## BLK           0.2041625 0.20110479
## STL           0.5020598 0.49294148
## TOV           0.7238690 0.54598664
## Team.fctr.num 0.2145549 0.20095724
## SeasonEnd     0.0000000 0.70201284
## ORB           0.7020128 0.00000000
## [1] "cor(PTS, oppPTS)=0.7891"
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-15.png) 

```
## [1] "cor(Playoffs, PTS)=0.2706"
## [1] "cor(Playoffs, oppPTS)=-0.2328"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## Warning in mydelete_cor_features(): Dropping oppPTS as a feature
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-16.png) 

```
##                          id       cor.y  cor.y.abs
## W                         W  0.79867559 0.79867559
## DRB                     DRB  0.34474930 0.34474930
## AST                     AST  0.31470508 0.31470508
## PTS                     PTS  0.27060133 0.27060133
## FT                       FT  0.22179906 0.22179906
## BLK                     BLK  0.18705612 0.18705612
## STL                     STL  0.17382295 0.17382295
## TOV                     TOV -0.16860161 0.16860161
## Team.fctr.num Team.fctr.num -0.12739314 0.12739314
## SeasonEnd         SeasonEnd -0.05912787 0.05912787
## ORB                     ORB -0.04170284 0.04170284
##                         W          DRB         AST         PTS
## W              1.00000000  0.470897497  0.32005177  0.29882561
## DRB            0.47089750  1.000000000  0.05647709  0.09029137
## AST            0.32005177  0.056477086  1.00000000  0.75988915
## PTS            0.29882561  0.090291366  0.75988915  1.00000000
## FT             0.20490600  0.003604883  0.44720848  0.69748385
## BLK            0.20392100  0.242990816  0.20308000  0.15205537
## STL            0.11619440 -0.331127910  0.44313827  0.43099026
## TOV           -0.24318588 -0.190956902  0.43053288  0.42713832
## Team.fctr.num -0.15119487 -0.058154820 -0.17491216 -0.20008998
## SeasonEnd      0.00000000  0.267127100 -0.67212300 -0.63952773
## ORB           -0.09573676 -0.269179339  0.40676555  0.49692073
##                         FT         BLK        STL        TOV Team.fctr.num
## W              0.204906000  0.20392100  0.1161944 -0.2431859   -0.15119487
## DRB            0.003604883  0.24299082 -0.3311279 -0.1909569   -0.05815482
## AST            0.447208477  0.20308000  0.4431383  0.4305329   -0.17491216
## PTS            0.697483850  0.15205537  0.4309903  0.4271383   -0.20008998
## FT             1.000000000  0.16465598  0.3231570  0.4386850   -0.15438561
## BLK            0.164655982  1.00000000  0.1204559  0.2415687   -0.07311722
## STL            0.323156975  0.12045594  1.0000000  0.4640752   -0.15311931
## TOV            0.438684976  0.24156872  0.4640752  1.0000000   -0.14045929
## Team.fctr.num -0.154385609 -0.07311722 -0.1531193 -0.1404593    1.00000000
## SeasonEnd     -0.513699180 -0.20416249 -0.5020598 -0.7238690    0.21455487
## ORB            0.393398702  0.20110479  0.4929415  0.5459866   -0.20095724
##                SeasonEnd         ORB
## W              0.0000000 -0.09573676
## DRB            0.2671271 -0.26917934
## AST           -0.6721230  0.40676555
## PTS           -0.6395277  0.49692073
## FT            -0.5136992  0.39339870
## BLK           -0.2041625  0.20110479
## STL           -0.5020598  0.49294148
## TOV           -0.7238690  0.54598664
## Team.fctr.num  0.2145549 -0.20095724
## SeasonEnd      1.0000000 -0.70201284
## ORB           -0.7020128  1.00000000
##                        W         DRB        AST        PTS          FT
## W             0.00000000 0.470897497 0.32005177 0.29882561 0.204906000
## DRB           0.47089750 0.000000000 0.05647709 0.09029137 0.003604883
## AST           0.32005177 0.056477086 0.00000000 0.75988915 0.447208477
## PTS           0.29882561 0.090291366 0.75988915 0.00000000 0.697483850
## FT            0.20490600 0.003604883 0.44720848 0.69748385 0.000000000
## BLK           0.20392100 0.242990816 0.20308000 0.15205537 0.164655982
## STL           0.11619440 0.331127910 0.44313827 0.43099026 0.323156975
## TOV           0.24318588 0.190956902 0.43053288 0.42713832 0.438684976
## Team.fctr.num 0.15119487 0.058154820 0.17491216 0.20008998 0.154385609
## SeasonEnd     0.00000000 0.267127100 0.67212300 0.63952773 0.513699180
## ORB           0.09573676 0.269179339 0.40676555 0.49692073 0.393398702
##                      BLK       STL       TOV Team.fctr.num SeasonEnd
## W             0.20392100 0.1161944 0.2431859    0.15119487 0.0000000
## DRB           0.24299082 0.3311279 0.1909569    0.05815482 0.2671271
## AST           0.20308000 0.4431383 0.4305329    0.17491216 0.6721230
## PTS           0.15205537 0.4309903 0.4271383    0.20008998 0.6395277
## FT            0.16465598 0.3231570 0.4386850    0.15438561 0.5136992
## BLK           0.00000000 0.1204559 0.2415687    0.07311722 0.2041625
## STL           0.12045594 0.0000000 0.4640752    0.15311931 0.5020598
## TOV           0.24156872 0.4640752 0.0000000    0.14045929 0.7238690
## Team.fctr.num 0.07311722 0.1531193 0.1404593    0.00000000 0.2145549
## SeasonEnd     0.20416249 0.5020598 0.7238690    0.21455487 0.0000000
## ORB           0.20110479 0.4929415 0.5459866    0.20095724 0.7020128
##                      ORB
## W             0.09573676
## DRB           0.26917934
## AST           0.40676555
## PTS           0.49692073
## FT            0.39339870
## BLK           0.20110479
## STL           0.49294148
## TOV           0.54598664
## Team.fctr.num 0.20095724
## SeasonEnd     0.70201284
## ORB           0.00000000
## [1] "cor(AST, PTS)=0.7599"
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-17.png) 

```
## [1] "cor(Playoffs, AST)=0.3147"
## [1] "cor(Playoffs, PTS)=0.2706"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## Warning in mydelete_cor_features(): Dropping PTS as a feature
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-18.png) 

```
##                          id       cor.y  cor.y.abs
## W                         W  0.79867559 0.79867559
## DRB                     DRB  0.34474930 0.34474930
## AST                     AST  0.31470508 0.31470508
## FT                       FT  0.22179906 0.22179906
## BLK                     BLK  0.18705612 0.18705612
## STL                     STL  0.17382295 0.17382295
## TOV                     TOV -0.16860161 0.16860161
## Team.fctr.num Team.fctr.num -0.12739314 0.12739314
## SeasonEnd         SeasonEnd -0.05912787 0.05912787
## ORB                     ORB -0.04170284 0.04170284
##                         W          DRB         AST           FT
## W              1.00000000  0.470897497  0.32005177  0.204906000
## DRB            0.47089750  1.000000000  0.05647709  0.003604883
## AST            0.32005177  0.056477086  1.00000000  0.447208477
## FT             0.20490600  0.003604883  0.44720848  1.000000000
## BLK            0.20392100  0.242990816  0.20308000  0.164655982
## STL            0.11619440 -0.331127910  0.44313827  0.323156975
## TOV           -0.24318588 -0.190956902  0.43053288  0.438684976
## Team.fctr.num -0.15119487 -0.058154820 -0.17491216 -0.154385609
## SeasonEnd      0.00000000  0.267127100 -0.67212300 -0.513699180
## ORB           -0.09573676 -0.269179339  0.40676555  0.393398702
##                       BLK        STL        TOV Team.fctr.num  SeasonEnd
## W              0.20392100  0.1161944 -0.2431859   -0.15119487  0.0000000
## DRB            0.24299082 -0.3311279 -0.1909569   -0.05815482  0.2671271
## AST            0.20308000  0.4431383  0.4305329   -0.17491216 -0.6721230
## FT             0.16465598  0.3231570  0.4386850   -0.15438561 -0.5136992
## BLK            1.00000000  0.1204559  0.2415687   -0.07311722 -0.2041625
## STL            0.12045594  1.0000000  0.4640752   -0.15311931 -0.5020598
## TOV            0.24156872  0.4640752  1.0000000   -0.14045929 -0.7238690
## Team.fctr.num -0.07311722 -0.1531193 -0.1404593    1.00000000  0.2145549
## SeasonEnd     -0.20416249 -0.5020598 -0.7238690    0.21455487  1.0000000
## ORB            0.20110479  0.4929415  0.5459866   -0.20095724 -0.7020128
##                       ORB
## W             -0.09573676
## DRB           -0.26917934
## AST            0.40676555
## FT             0.39339870
## BLK            0.20110479
## STL            0.49294148
## TOV            0.54598664
## Team.fctr.num -0.20095724
## SeasonEnd     -0.70201284
## ORB            1.00000000
##                        W         DRB        AST          FT        BLK
## W             0.00000000 0.470897497 0.32005177 0.204906000 0.20392100
## DRB           0.47089750 0.000000000 0.05647709 0.003604883 0.24299082
## AST           0.32005177 0.056477086 0.00000000 0.447208477 0.20308000
## FT            0.20490600 0.003604883 0.44720848 0.000000000 0.16465598
## BLK           0.20392100 0.242990816 0.20308000 0.164655982 0.00000000
## STL           0.11619440 0.331127910 0.44313827 0.323156975 0.12045594
## TOV           0.24318588 0.190956902 0.43053288 0.438684976 0.24156872
## Team.fctr.num 0.15119487 0.058154820 0.17491216 0.154385609 0.07311722
## SeasonEnd     0.00000000 0.267127100 0.67212300 0.513699180 0.20416249
## ORB           0.09573676 0.269179339 0.40676555 0.393398702 0.20110479
##                     STL       TOV Team.fctr.num SeasonEnd        ORB
## W             0.1161944 0.2431859    0.15119487 0.0000000 0.09573676
## DRB           0.3311279 0.1909569    0.05815482 0.2671271 0.26917934
## AST           0.4431383 0.4305329    0.17491216 0.6721230 0.40676555
## FT            0.3231570 0.4386850    0.15438561 0.5136992 0.39339870
## BLK           0.1204559 0.2415687    0.07311722 0.2041625 0.20110479
## STL           0.0000000 0.4640752    0.15311931 0.5020598 0.49294148
## TOV           0.4640752 0.0000000    0.14045929 0.7238690 0.54598664
## Team.fctr.num 0.1531193 0.1404593    0.00000000 0.2145549 0.20095724
## SeasonEnd     0.5020598 0.7238690    0.21455487 0.0000000 0.70201284
## ORB           0.4929415 0.5459866    0.20095724 0.7020128 0.00000000
## [1] "cor(TOV, SeasonEnd)=-0.7239"
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-19.png) 

```
## [1] "cor(Playoffs, TOV)=-0.1686"
## [1] "cor(Playoffs, SeasonEnd)=-0.0591"
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use 'method = x' to change the smoothing method.
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : pseudoinverse used at -0.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : neighborhood radius 1.005
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : reciprocal condition number 1.3451e-30
```

```
## Warning in simpleLoess(y, x, w, span, degree, parametric, drop.square,
## normalize, : There are other near singularities as well. 1.01
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : pseudoinverse used at -0.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : neighborhood radius 1.005
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : reciprocal condition number 1.3451e-30
```

```
## Warning in predLoess(y, x, newx, s, weights, pars$robust, pars$span,
## pars$degree, : There are other near singularities as well. 1.01
```

```
## Warning in mydelete_cor_features(): Dropping SeasonEnd as a feature
```

![](NBA_Playoffs_files/figure-html/remove_correlated_features-20.png) 

```
##                          id       cor.y  cor.y.abs
## W                         W  0.79867559 0.79867559
## DRB                     DRB  0.34474930 0.34474930
## AST                     AST  0.31470508 0.31470508
## FT                       FT  0.22179906 0.22179906
## BLK                     BLK  0.18705612 0.18705612
## STL                     STL  0.17382295 0.17382295
## TOV                     TOV -0.16860161 0.16860161
## Team.fctr.num Team.fctr.num -0.12739314 0.12739314
## ORB                     ORB -0.04170284 0.04170284
##                         W          DRB         AST           FT
## W              1.00000000  0.470897497  0.32005177  0.204906000
## DRB            0.47089750  1.000000000  0.05647709  0.003604883
## AST            0.32005177  0.056477086  1.00000000  0.447208477
## FT             0.20490600  0.003604883  0.44720848  1.000000000
## BLK            0.20392100  0.242990816  0.20308000  0.164655982
## STL            0.11619440 -0.331127910  0.44313827  0.323156975
## TOV           -0.24318588 -0.190956902  0.43053288  0.438684976
## Team.fctr.num -0.15119487 -0.058154820 -0.17491216 -0.154385609
## ORB           -0.09573676 -0.269179339  0.40676555  0.393398702
##                       BLK        STL        TOV Team.fctr.num         ORB
## W              0.20392100  0.1161944 -0.2431859   -0.15119487 -0.09573676
## DRB            0.24299082 -0.3311279 -0.1909569   -0.05815482 -0.26917934
## AST            0.20308000  0.4431383  0.4305329   -0.17491216  0.40676555
## FT             0.16465598  0.3231570  0.4386850   -0.15438561  0.39339870
## BLK            1.00000000  0.1204559  0.2415687   -0.07311722  0.20110479
## STL            0.12045594  1.0000000  0.4640752   -0.15311931  0.49294148
## TOV            0.24156872  0.4640752  1.0000000   -0.14045929  0.54598664
## Team.fctr.num -0.07311722 -0.1531193 -0.1404593    1.00000000 -0.20095724
## ORB            0.20110479  0.4929415  0.5459866   -0.20095724  1.00000000
##                        W         DRB        AST          FT        BLK
## W             0.00000000 0.470897497 0.32005177 0.204906000 0.20392100
## DRB           0.47089750 0.000000000 0.05647709 0.003604883 0.24299082
## AST           0.32005177 0.056477086 0.00000000 0.447208477 0.20308000
## FT            0.20490600 0.003604883 0.44720848 0.000000000 0.16465598
## BLK           0.20392100 0.242990816 0.20308000 0.164655982 0.00000000
## STL           0.11619440 0.331127910 0.44313827 0.323156975 0.12045594
## TOV           0.24318588 0.190956902 0.43053288 0.438684976 0.24156872
## Team.fctr.num 0.15119487 0.058154820 0.17491216 0.154385609 0.07311722
## ORB           0.09573676 0.269179339 0.40676555 0.393398702 0.20110479
##                     STL       TOV Team.fctr.num        ORB
## W             0.1161944 0.2431859    0.15119487 0.09573676
## DRB           0.3311279 0.1909569    0.05815482 0.26917934
## AST           0.4431383 0.4305329    0.17491216 0.40676555
## FT            0.3231570 0.4386850    0.15438561 0.39339870
## BLK           0.1204559 0.2415687    0.07311722 0.20110479
## STL           0.0000000 0.4640752    0.15311931 0.49294148
## TOV           0.4640752 0.0000000    0.14045929 0.54598664
## Team.fctr.num 0.1531193 0.1404593    0.00000000 0.20095724
## ORB           0.4929415 0.5459866    0.20095724 0.00000000
##               id        cor.y   cor.y.abs cor.low
## 15             W  0.798675595 0.798675595       1
## 3            DRB  0.344749300 0.344749300       1
## 1            AST  0.314705084 0.314705084       1
## 10           PTS  0.270601328 0.270601328      NA
## 6             FT  0.221799063 0.221799063       1
## 4             FG  0.187552799 0.187552799      NA
## 2            BLK  0.187056116 0.187056116       1
## 7            FTA  0.179413465 0.179413465      NA
## 12           STL  0.173822947 0.173822947       1
## 16           X2P  0.108033881 0.108033881      NA
## 18           X3P  0.028382516 0.028382516      NA
## 19          X3PA  0.006570622 0.006570622      NA
## 17          X2PA -0.009931886 0.009931886      NA
## 5            FGA -0.011985459 0.011985459      NA
## 9            ORB -0.041702843 0.041702843       1
## 11     SeasonEnd -0.059127866 0.059127866      NA
## 13 Team.fctr.num -0.127393136 0.127393136       1
## 14           TOV -0.168601605 0.168601605       1
## 8         oppPTS -0.232760995 0.232760995      NA
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

    # Highest cor.y
    ret_lst <- myrun_mdl_lm(indep_vars_vctr=max_cor_y_x_var,
                            fit_df=glb_entity_df, OOB_df=glb_predct_df)

    # Enhance Highest cor.y model with additions of interaction terms that were 
    #   dropped due to high correlations
    ret_lst <- myrun_mdl_lm(indep_vars_vctr=c(max_cor_y_x_var, 
        paste(max_cor_y_x_var, subset(glb_feats_df, is.na(cor.low))[, "id"], sep=":")),
                            fit_df=glb_entity_df, OOB_df=glb_predct_df)    

    # Low correlated X
    ret_lst <- myrun_mdl_lm(indep_vars_vctr=subset(glb_feats_df, cor.low == 1)[, "id"],
                            fit_df=glb_entity_df, OOB_df=glb_predct_df)
    glb_sel_mdl <- glb_mdl
    
    # All X that is not missing
    ret_lst <- myrun_mdl_lm(indep_vars_vctr=setdiff(setdiff(names(glb_entity_df),
                                                             glb_predct_var),
                                                     glb_exclude_vars_as_features),
                            fit_df=glb_entity_df, OOB_df=glb_predct_df)

}    

#   Classification:
if (glb_is_classification) {
    #   Logit Regression:
    
    # Highest cor.y
    ret_lst <- myrun_mdl_glm(indep_vars_vctr=max_cor_y_x_var,
                            fit_df=glb_entity_df, OOB_df=glb_predct_df)        
    glb_sel_mdl <- glb_mdl

    # Enhance Highest cor.y model with additions of interaction terms that were 
    #   dropped due to high correlations
    ret_lst <- myrun_mdl_glm(indep_vars_vctr=c(max_cor_y_x_var, 
        paste(max_cor_y_x_var, subset(glb_feats_df, is.na(cor.low))[, "id"], sep=":")),
                            fit_df=glb_entity_df, OOB_df=glb_predct_df)    

    # Low correlated X
    ret_lst <- myrun_mdl_glm(indep_vars_vctr=subset(glb_feats_df, cor.low == 1)[, "id"],
                            fit_df=glb_entity_df, OOB_df=glb_predct_df)        
    
    # All X that is not missing
    ret_lst <- myrun_mdl_glm(indep_vars_vctr=setdiff(setdiff(names(glb_entity_df),
                                                             glb_predct_var),
                                                     glb_exclude_vars_as_features),
                            fit_df=glb_entity_df, OOB_df=glb_predct_df)
                            
}
```

```
## [1] 3
## [1] 0.5808225
## 
## Call:
## glm(formula = reformulate(indep_vars_vctr, response = glb_predct_var), 
##     family = "binomial", data = fit_df)
## 
## Deviance Residuals: 
##      Min        1Q    Median        3Q       Max  
## -2.89406  -0.09084   0.01310   0.17491   2.94469  
## 
## Coefficients:
##              Estimate Std. Error z value Pr(>|z|)    
## (Intercept) -18.48061    1.62139  -11.40   <2e-16 ***
## W             0.47194    0.04059   11.63   <2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for binomial family taken to be 1)
## 
##     Null deviance: 1138.8  on 834  degrees of freedom
## Residual deviance:  308.9  on 833  degrees of freedom
## AIC: 312.9
## 
## Number of Fisher Scoring iterations: 8
## 
##   feats n.fit R.sq.fit  R.sq.OOB Adj.R.sq.fit  SSE.fit SSE.OOB f.score.OOB
## 1     W   835       NA 0.5808225           NA 16105.57       3   0.9433962
```

```
## Warning in predict.lm(object, newdata, se.fit, scale = 1, type =
## ifelse(type == : prediction from a rank-deficient fit may be misleading
```

```
## [1] 2
## [1] 0.7205483
## 
## Call:
## glm(formula = reformulate(indep_vars_vctr, response = glb_predct_var), 
##     family = "binomial", data = fit_df)
## 
## Deviance Residuals: 
##      Min        1Q    Median        3Q       Max  
## -2.71047  -0.07253   0.00958   0.14906   2.94933  
## 
## Coefficients: (2 not defined because of singularities)
##               Estimate Std. Error z value Pr(>|z|)    
## (Intercept) -1.870e+01  2.532e+00  -7.384 1.53e-13 ***
## W            3.108e+00  1.851e+00   1.679  0.09312 .  
## W:PTS        2.244e-04  8.247e-05   2.721  0.00651 ** 
## W:FG        -9.766e-04  3.205e-04  -3.047  0.00231 ** 
## W:FTA       -1.300e-04  5.656e-05  -2.298  0.02154 *  
## W:X2P        5.770e-04  2.375e-04   2.430  0.01511 *  
## W:X3P               NA         NA      NA       NA    
## W:X3PA       1.277e-04  7.952e-05   1.606  0.10829    
## W:X2PA      -1.102e-05  2.616e-05  -0.421  0.67353    
## W:FGA               NA         NA      NA       NA    
## W:SeasonEnd -1.298e-03  9.037e-04  -1.437  0.15085    
## W:oppPTS    -2.644e-05  4.164e-05  -0.635  0.52556    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for binomial family taken to be 1)
## 
##     Null deviance: 1138.77  on 834  degrees of freedom
## Residual deviance:  284.87  on 825  degrees of freedom
## AIC: 304.87
## 
## Number of Fisher Scoring iterations: 8
## 
##                                                                               feats
## 2 W, W:PTS, W:FG, W:FTA, W:X2P, W:X3P, W:X3PA, W:X2PA, W:FGA, W:SeasonEnd, W:oppPTS
## 1                                                                                 W
##   n.fit R.sq.fit  R.sq.OOB Adj.R.sq.fit  SSE.fit SSE.OOB f.score.OOB
## 2   835       NA 0.7205483           NA 13343.56       2   0.9629630
## 1   835       NA 0.5808225           NA 16105.57       3   0.9433962
## [1] 3
## [1] 0.5808225
## 
## Call:
## glm(formula = reformulate(indep_vars_vctr, response = glb_predct_var), 
##     family = "binomial", data = fit_df)
## 
## Deviance Residuals: 
##      Min        1Q    Median        3Q       Max  
## -3.01114  -0.06986   0.00824   0.14670   2.94822  
## 
## Coefficients:
##                 Estimate Std. Error z value Pr(>|z|)    
## (Intercept)   -2.712e+01  5.024e+00  -5.398 6.73e-08 ***
## W              4.818e-01  4.612e-02  10.447  < 2e-16 ***
## DRB            2.718e-04  1.605e-03   0.169   0.8655    
## AST            2.249e-03  9.246e-04   2.433   0.0150 *  
## FT             2.092e-03  9.604e-04   2.178   0.0294 *  
## BLK            1.612e-03  1.987e-03   0.811   0.4171    
## STL            4.314e-03  2.327e-03   1.854   0.0637 .  
## ORB           -2.227e-03  1.363e-03  -1.633   0.1024    
## Team.fctr.num -1.245e-02  1.674e-02  -0.743   0.4573    
## TOV           -8.386e-04  1.602e-03  -0.523   0.6007    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for binomial family taken to be 1)
## 
##     Null deviance: 1138.77  on 834  degrees of freedom
## Residual deviance:  281.36  on 825  degrees of freedom
## AIC: 301.36
## 
## Number of Fisher Scoring iterations: 8
## 
##                                                                               feats
## 2 W, W:PTS, W:FG, W:FTA, W:X2P, W:X3P, W:X3PA, W:X2PA, W:FGA, W:SeasonEnd, W:oppPTS
## 1                                                                                 W
## 3                                W, DRB, AST, FT, BLK, STL, ORB, Team.fctr.num, TOV
##   n.fit R.sq.fit  R.sq.OOB Adj.R.sq.fit  SSE.fit SSE.OOB f.score.OOB
## 2   835       NA 0.7205483           NA 13343.56       2   0.9629630
## 1   835       NA 0.5808225           NA 16105.57       3   0.9433962
## 3   835       NA 0.5808225           NA 19417.07       3   0.9433962
```

```
## Warning in predict.lm(object, newdata, se.fit, scale = 1, type =
## ifelse(type == : prediction from a rank-deficient fit may be misleading
```

```
## [1] 3
## [1] 0.5808225
## 
## Call:
## glm(formula = reformulate(indep_vars_vctr, response = glb_predct_var), 
##     family = "binomial", data = fit_df)
## 
## Deviance Residuals: 
##      Min        1Q    Median        3Q       Max  
## -2.64597  -0.06714   0.00853   0.13599   3.00851  
## 
## Coefficients: (3 not defined because of singularities)
##                 Estimate Std. Error z value Pr(>|z|)    
## (Intercept)   54.4835195 94.1146409   0.579  0.56265    
## SeasonEnd     -0.0387860  0.0464922  -0.834  0.40414    
## W              0.4882423  0.0685069   7.127 1.03e-12 ***
## PTS            0.0069788  0.0037067   1.883  0.05973 .  
## oppPTS        -0.0001793  0.0019573  -0.092  0.92701    
## FG            -0.0342528  0.0140581  -2.437  0.01483 *  
## FGA            0.0035754  0.0039836   0.898  0.36943    
## X2P            0.0177184  0.0103021   1.720  0.08545 .  
## X2PA          -0.0037101  0.0037134  -0.999  0.31775    
## X3P                   NA         NA      NA       NA    
## X3PA                  NA         NA      NA       NA    
## FT                    NA         NA      NA       NA    
## FTA           -0.0038417  0.0027600  -1.392  0.16394    
## ORB           -0.0014959  0.0024916  -0.600  0.54824    
## DRB            0.0014214  0.0023731   0.599  0.54920    
## AST            0.0039012  0.0014715   2.651  0.00802 ** 
## STL            0.0050787  0.0031166   1.630  0.10319    
## BLK            0.0012202  0.0020926   0.583  0.55982    
## TOV           -0.0012361  0.0024210  -0.511  0.60964    
## Team.fctr.num -0.0074274  0.0175425  -0.423  0.67201    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for binomial family taken to be 1)
## 
##     Null deviance: 1138.77  on 834  degrees of freedom
## Residual deviance:  271.28  on 818  degrees of freedom
## AIC: 305.28
## 
## Number of Fisher Scoring iterations: 8
## 
##                                                                                                            feats
## 2                              W, W:PTS, W:FG, W:FTA, W:X2P, W:X3P, W:X3PA, W:X2PA, W:FGA, W:SeasonEnd, W:oppPTS
## 1                                                                                                              W
## 3                                                             W, DRB, AST, FT, BLK, STL, ORB, Team.fctr.num, TOV
## 4 SeasonEnd, W, PTS, oppPTS, FG, FGA, X2P, X2PA, X3P, X3PA, FT, FTA, ORB, DRB, AST, STL, BLK, TOV, Team.fctr.num
##   n.fit R.sq.fit  R.sq.OOB Adj.R.sq.fit  SSE.fit SSE.OOB f.score.OOB
## 2   835       NA 0.7205483           NA 13343.56       2   0.9629630
## 1   835       NA 0.5808225           NA 16105.57       3   0.9433962
## 3   835       NA 0.5808225           NA 19417.07       3   0.9433962
## 4   835       NA 0.5808225           NA 13683.51       3   0.9433962
```

```r
if (glb_is_regression)
    print(myplot_scatter(glb_models_df, "Adj.R.sq.fit", "R.sq.OOB") + 
          geom_text(aes(label=feats), data=glb_models_df, color="NavyBlue", 
                    size=3.5))

if (glb_is_classification) {
    plot_models_df <- mutate(glb_models_df, feats.label=substr(feats, 1, 20))
    print(myplot_hbar(df=plot_models_df, xcol_name="feats.label", 
                      ycol_names="f.score.OOB"))
}    
```

![](NBA_Playoffs_files/figure-html/run_models-1.png) 

```r
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
##    Estimate Std. Error  z value         Pr.z id
## W 0.4719398 0.04059482 11.62562 3.053832e-31  W
```

```r
if (glb_is_regression) {
    ret_lst <- myrun_mdl_lm(indep_vars_vctr=mdl_feats_df$id, fit_df=glb_entity_df)
    glb_sel_mdl <- glb_mdl    
    glb_entity_df[, glb_predct_var_name] <- predict(glb_sel_mdl, newdata=glb_entity_df)
    print(myplot_scatter(glb_entity_df, glb_predct_var, glb_predct_var_name, 
                         smooth=TRUE))    
}    

if (glb_is_classification) {
    ret_lst <- myrun_mdl_glm(indep_vars_vctr=mdl_feats_df$id, fit_df=glb_entity_df)
    glb_sel_mdl <- glb_mdl        
    glb_entity_df[, glb_predct_var_name] <- (predict(glb_sel_mdl, 
                        newdata=glb_entity_df, type="response") >= 0.5) * 1.0
    print(xtabs(reformulate(paste(glb_predct_var, glb_predct_var_name, sep=" + ")),
                glb_entity_df))                        
}    
```

```
## 
## Call:
## glm(formula = reformulate(indep_vars_vctr, response = glb_predct_var), 
##     family = "binomial", data = fit_df)
## 
## Deviance Residuals: 
##      Min        1Q    Median        3Q       Max  
## -2.89406  -0.09084   0.01310   0.17491   2.94469  
## 
## Coefficients:
##              Estimate Std. Error z value Pr(>|z|)    
## (Intercept) -18.48061    1.62139  -11.40   <2e-16 ***
## W             0.47194    0.04059   11.63   <2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for binomial family taken to be 1)
## 
##     Null deviance: 1138.8  on 834  degrees of freedom
## Residual deviance:  308.9  on 833  degrees of freedom
## AIC: 312.9
## 
## Number of Fisher Scoring iterations: 8
## 
##                                                                                                            feats
## 2                              W, W:PTS, W:FG, W:FTA, W:X2P, W:X3P, W:X3PA, W:X2PA, W:FGA, W:SeasonEnd, W:oppPTS
## 1                                                                                                              W
## 3                                                             W, DRB, AST, FT, BLK, STL, ORB, Team.fctr.num, TOV
## 4 SeasonEnd, W, PTS, oppPTS, FG, FGA, X2P, X2PA, X3P, X3PA, FT, FTA, ORB, DRB, AST, STL, BLK, TOV, Team.fctr.num
## 5                                                                                                              W
##   n.fit R.sq.fit  R.sq.OOB Adj.R.sq.fit  SSE.fit SSE.OOB f.score.OOB
## 2   835       NA 0.7205483           NA 13343.56       2   0.9629630
## 1   835       NA 0.5808225           NA 16105.57       3   0.9433962
## 3   835       NA 0.5808225           NA 19417.07       3   0.9433962
## 4   835       NA 0.5808225           NA 13683.51       3   0.9433962
## 5   835       NA        NA           NA 16105.57      NA          NA
##         Playoffs.predict
## Playoffs   0   1
##        0 318  37
##        1  30 450
```

```r
print(glb_feats_df <- mymerge_feats_Pr.z())
```

```
##               id        cor.y   cor.y.abs cor.low         Pr.z
## 15             W  0.798675595 0.798675595       1 3.053832e-31
## 1            AST  0.314705084 0.314705084       1           NA
## 2            BLK  0.187056116 0.187056116       1           NA
## 3            DRB  0.344749300 0.344749300       1           NA
## 4             FG  0.187552799 0.187552799      NA           NA
## 5            FGA -0.011985459 0.011985459      NA           NA
## 6             FT  0.221799063 0.221799063       1           NA
## 7            FTA  0.179413465 0.179413465      NA           NA
## 8         oppPTS -0.232760995 0.232760995      NA           NA
## 9            ORB -0.041702843 0.041702843       1           NA
## 10           PTS  0.270601328 0.270601328      NA           NA
## 11     SeasonEnd -0.059127866 0.059127866      NA           NA
## 12           STL  0.173822947 0.173822947       1           NA
## 13 Team.fctr.num -0.127393136 0.127393136       1           NA
## 14           TOV -0.168601605 0.168601605       1           NA
## 16           X2P  0.108033881 0.108033881      NA           NA
## 17          X2PA -0.009931886 0.009931886      NA           NA
## 18           X3P  0.028382516 0.028382516      NA           NA
## 19          X3PA  0.006570622 0.006570622      NA           NA
```

```r
for (var in subset(glb_feats_df, Pr.z < 0.1)$id) {
    plot_df <- melt(glb_entity_df, id.vars=var, 
                    measure.vars=c(glb_predct_var, glb_predct_var_name))
    if (var == "W") print(myplot_scatter(plot_df, var, "value", 
                                         facet_colcol_name="variable") + 
                  geom_vline(xintercept=40, linetype="dotted")) else     
        print(myplot_scatter(plot_df, var, "value", facet_colcol_name="variable"))
}
```

![](NBA_Playoffs_files/figure-html/fit_training.all-1.png) 

```r
if (glb_is_regression) {
    plot_vars_df <- subset(glb_feats_df, Pr.z < 0.1)
    print(myplot_prediction_regression(glb_entity_df, 
                ifelse(nrow(plot_vars_df) > 1, plot_vars_df$id[2], ".rownames"), 
                                       plot_vars_df$id[1]))
}    

if (glb_is_classification) {
    plot_vars_df <- subset(glb_feats_df, Pr.z < 0.1)
    print(myplot_prediction_classification(glb_entity_df, 
                ifelse(nrow(plot_vars_df) > 1, plot_vars_df$id[2], ".rownames"),
                                           plot_vars_df$id[1]) + 
            geom_hline(yintercept=40, linetype = "dotted"))
}    
```

![](NBA_Playoffs_files/figure-html/fit_training.all-2.png) 

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
##   SeasonEnd                Team Playoffs Playoffs.predict
## 1      2013       Atlanta Hawks        1                1
## 2      2013       Brooklyn Nets        1                1
## 3      2013   Charlotte Bobcats        0                0
## 4      2013       Chicago Bulls        1                1
## 5      2013 Cleveland Cavaliers        0                0
## 6      2013    Dallas Mavericks        0                1
##    SeasonEnd                 Team Playoffs Playoffs.predict
## 4       2013        Chicago Bulls        1                1
## 10      2013      Houston Rockets        1                1
## 11      2013 Los Angeles Clippers        1                1
## 13      2013    Memphis Grizzlies        1                1
## 17      2013  New Orleans Hornets        0                0
## 20      2013        Orlando Magic        0                0
##    SeasonEnd                   Team Playoffs Playoffs.predict
## 23      2013 Portland Trail Blazers        0                0
## 24      2013       Sacramento Kings        0                0
## 25      2013      San Antonio Spurs        1                1
## 26      2013        Toronto Raptors        0                0
## 27      2013              Utah Jazz        0                1
## 28      2013     Washington Wizards        0                0
```

```r
if (glb_is_regression) {
    print(sprintf("Total SSE: %0.4f", 
                  sum((glb_predct_df[, glb_predct_var_name] - 
                        glb_predct_df[, glb_predct_var]) ^ 2)))
    print(myplot_scatter(glb_predct_df, glb_predct_var, glb_predct_var_name, 
                         smooth=TRUE))
}                         

if (glb_is_classification)
    print(xtabs(reformulate(paste(glb_predct_var, glb_predct_var_name, sep=" + ")),
                glb_predct_df))
```

```
##         Playoffs.predict
## Playoffs  0  1
##        0 12  2
##        1  1 13
```

```r
for (var in subset(glb_feats_df, Pr.z < 0.1)$id) {
    plot_df <- melt(glb_predct_df, id.vars=var, 
                    measure.vars=c(glb_predct_var, glb_predct_var_name))
    if (var == "W") print(myplot_scatter(plot_df, var, "value", 
                                         facet_colcol_name="variable") + 
                  geom_vline(xintercept=40, linetype="dotted")) else     
        print(myplot_scatter(plot_df, var, "value", facet_colcol_name="variable"))
}
```

![](NBA_Playoffs_files/figure-html/predict_newdata-1.png) 

```r
# Add choose(, 2) functionality to select feature pairs to plot 
if (glb_is_regression) {
    plot_vars_df <- subset(glb_feats_df, Pr.z < 0.1)
    print(myplot_prediction_regression(glb_predct_df, 
                ifelse(nrow(plot_vars_df) > 1, plot_vars_df$id[2], ".rownames"),
                                        plot_vars_df$id[1]))
}    

if (glb_is_classification) {
    plot_vars_df <- subset(glb_feats_df, Pr.z < 0.1)
    print(myplot_prediction_classification(glb_predct_df,                
                ifelse(nrow(plot_vars_df) > 1, plot_vars_df$id[2], ".rownames"),
                                           plot_vars_df$id[1]) + 
            geom_hline(yintercept=40, linetype = "dotted"))
}    
```

![](NBA_Playoffs_files/figure-html/predict_newdata-2.png) 

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
## [1] stats     graphics  grDevices utils     datasets  methods   base     
## 
## other attached packages:
## [1] reshape2_1.4.1  plyr_1.8.1      doBy_4.5-13     survival_2.38-1
## [5] ggplot2_1.0.0  
## 
## loaded via a namespace (and not attached):
##  [1] colorspace_1.2-5 digest_0.6.8     evaluate_0.5.5   formatR_1.0     
##  [5] grid_3.1.2       gtable_0.1.2     htmltools_0.2.6  knitr_1.9       
##  [9] labeling_0.3     lattice_0.20-30  MASS_7.3-39      Matrix_1.1-5    
## [13] munsell_0.4.2    proto_0.3-10     Rcpp_0.11.4      rmarkdown_0.5.1 
## [17] scales_0.2.4     splines_3.1.2    stringr_0.6.2    tcltk_3.1.2     
## [21] tools_3.1.2      yaml_2.1.13
```
