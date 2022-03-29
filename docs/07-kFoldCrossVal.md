# Standard K-fold Cross-validation



-   **Context and Purpose:**

-   **Upstream:** Section \@ref() -

-   **Downstream:**

-   **Inputs:**

-   **Expected outputs:**

In this section we will run K-fold cross-validation to evaluate the accuracy of predicting the performance of candidate parents (**GEBV**) who have not been phenotyped.

This is always recommended, and there are alternative kinds of predictions that could be set-up to measure this.

Important is the distinction from analyses we will do downstream to assess the accuracy of predicting the performance of *crosses* (i.e. mates).

We will use the `runCrossVal()` function.

We will demonstrate a few of the additional features that it provides in the process:

1.  Support for multiple traits
2.  Computing selection index accuracy

Finally, we'll make a simple plot of the results.

## Process Map

![](images/kfold_crossval_process_map.png){width=100%}

## Set-up for the cross-validation


```r
blups<-readRDS(here::here("output","blups.rds"))
A<-readRDS(file=here::here("output","kinship_add.rds"))
```


```r
blups %<>% 
     # need to rename the "blups" list to comply with the runCrossVal function
     rename(TrainingData=blups) %>% 
     dplyr::select(Trait,TrainingData) %>% 
     # need also to remove phenotyped-but-not-genotyped lines
     # couldn't hurt to also subset the kinship to only phenotyped lines... would save RAM
     mutate(TrainingData=map(TrainingData,
                             ~filter(.,germplasmName %in% rownames(A)) %>% 
                                  # rename the germplasmName column to GID
                                  rename(GID=germplasmName)))

blups
#> # A tibble: 4 × 2
#>   Trait   TrainingData      
#>   <chr>   <list>            
#> 1 DM      <tibble [346 × 6]>
#> 2 MCMDS   <tibble [292 × 6]>
#> 3 logFYLD <tibble [350 × 6]>
#> 4 logDYLD <tibble [348 × 6]>
```

The steps above set-us up almost all the way.


```r
# For fastest, lightest compute of accuracy, remove non-phenotyped from kinship

gids<-blups %>% 
     unnest(TrainingData) %$% unique(GID)
# dim(A) [1] 963 963

A<-A[gids,gids]
```

## Selection indices

Last thing: Let's include selection index weights. You can find an excellent, detailed, open-source chapter from Walsh & Lynch on Selection Index Theory by [**clicking here**](http://nitro.biosci.arizona.edu/zdownload/Volume2/Chapter23.pdf).

$$SI = WT_1 \times Trait_1 + \dots + WT_t \times Trait_t$$ Or in vector form:

$$SI = \boldsymbol{\hat{g}b}$$

$SI$ is the selection index, dimension $[n \times 1]$. $b$ is a $[t \times 1]$ vector of selection index "economic weights" designed to value each trait relative to its impact on the economic potential of changing the corresponding trait by one unit. Finally, $\boldsymbol{\hat{g}$ is a matrix $[n \times t]$ with the (in this case) **GEBV** for each trait on the columns.

`runCrossVal()` will accept a named vector of selection index weights where names must match the "Trait" variable in `blups` using the `SIwts=` argument and setting `selInd=TRUE`.

Here are example weights, I'll use. These are *not* to be taken as canonical. *Weights should be determined for each target population of environments and product profile!*


```r
# I chose to remove MCMDS 
## our preliminary analysis showed it to have ~0 heritability in this dataset
## initial test of cross-val. showed the models do not fit
SIwts<-c(DM=15,
         #MCMDS=-10,
         logFYLD=20,
         logDYLD=20)
SIwts
#>      DM logFYLD logDYLD 
#>      15      20      20
```

I'll run a meager 2 repetitions of 5-fold cross-validation, which means 10 predictions per trait overall. I've got a 16-core laptop so I can use `ncores=10` to do all 10 predictions per trait at the same time. `runCrossVal()` will process all four traits *and* compute the selection index accuracy at the end.

## Execute cross-validation


```r
starttime<-proc.time()[3]
standardCV<-runCrossVal(blups=blups %>% filter(Trait != "MCMDS"),
                        modelType="A",
                        selInd=TRUE,SIwts=SIwts,
                        grms=list(A=A),
                        nrepeats=2,nfolds=5,
                        gid="GID",seed=424242,
                        ncores=10)
#> Loading required package: rsample
#> Loading required package: furrr
#> Loading required package: future
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -150.932   6:58:10      0           0
#>     2      -150.587   6:58:10      0           0
#>     3      -150.456   6:58:10      0           0
#>     4      -150.431   6:58:11      1           0
#>     5      -150.429   6:58:11      1           0
#>     6      -150.429   6:58:11      1           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -109.584   6:58:11      0           0
#>     2      -109.57   6:58:11      0           0
#>     3      -109.562   6:58:11      0           0
#>     4      -109.56   6:58:11      0           0
#>     5      -109.559   6:58:11      0           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -115.829   6:58:12      0           0
#>     2      -115.829   6:58:12      0           0
#>     3      -115.828   6:58:12      0           0
#>     4      -115.828   6:58:12      0           0
#> [1] "GBLUP model complete - one trait"
#> [1] "Genomic predictions done for all traits in one repeat-fold"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -153.247   6:58:11      0           0
#>     2      -153.244   6:58:11      0           0
#>     3      -153.243   6:58:11      0           0
#>     4      -153.243   6:58:11      0           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -108.226   6:58:11      0           0
#>     2      -108.147   6:58:12      1           0
#>     3      -108.101   6:58:12      1           0
#>     4      -108.087   6:58:12      1           0
#>     5      -108.085   6:58:12      1           0
#>     6      -108.085   6:58:12      1           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -117.592   6:58:12      0           0
#>     2      -117.537   6:58:12      0           0
#>     3      -117.513   6:58:12      0           0
#>     4      -117.509   6:58:12      0           0
#>     5      -117.508   6:58:12      0           0
#> [1] "GBLUP model complete - one trait"
#> [1] "Genomic predictions done for all traits in one repeat-fold"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -150.198   6:58:12      1           0
#>     2      -149.363   6:58:12      1           0
#>     3      -148.987   6:58:12      1           0
#>     4      -148.881   6:58:12      1           0
#>     5      -148.865   6:58:12      1           0
#>     6      -148.863   6:58:12      1           0
#>     7      -148.862   6:58:12      1           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -106.107   6:58:12      0           0
#>     2      -105.581   6:58:12      0           0
#>     3      -105.152   6:58:12      0           0
#>     4      -104.92   6:58:12      0           0
#>     5      -104.852   6:58:12      0           0
#>     6      -104.832   6:58:13      1           0
#>     7      -104.827   6:58:13      1           0
#>     8      -104.825   6:58:13      1           0
#>     9      -104.825   6:58:13      1           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -118.481   6:58:13      0           0
#>     2      -118.255   6:58:13      0           0
#>     3      -118.106   6:58:13      0           0
#>     4      -118.047   6:58:13      0           0
#>     5      -118.035   6:58:13      0           0
#>     6      -118.032   6:58:13      0           0
#>     7      -118.032   6:58:13      0           0
#> [1] "GBLUP model complete - one trait"
#> [1] "Genomic predictions done for all traits in one repeat-fold"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -144.958   6:58:12      0           0
#>     2      -144.946   6:58:12      0           0
#>     3      -144.94   6:58:12      0           0
#>     4      -144.939   6:58:12      0           0
#>     5      -144.939   6:58:12      0           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -107.241   6:58:13      1           0
#>     2      -107.24   6:58:13      1           0
#>     3      -107.24   6:58:13      1           0
#>     4      -107.24   6:58:13      1           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -114.776   6:58:13      0           0
#>     2      -114.775   6:58:13      0           0
#>     3      -114.775   6:58:13      0           0
#>     4      -114.775   6:58:13      0           0
#> [1] "GBLUP model complete - one trait"
#> [1] "Genomic predictions done for all traits in one repeat-fold"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -150.502   6:58:13      1           0
#>     2      -150.404   6:58:13      1           0
#>     3      -150.354   6:58:13      1           0
#>     4      -150.339   6:58:13      1           0
#>     5      -150.336   6:58:13      1           0
#>     6      -150.336   6:58:13      1           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -112.48   6:58:13      0           0
#>     2      -112.42   6:58:13      0           0
#>     3      -112.38   6:58:13      0           0
#>     4      -112.364   6:58:13      0           0
#>     5      -112.36   6:58:13      0           0
#>     6      -112.358   6:58:13      0           0
#>     7      -112.358   6:58:14      1           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -118.347   6:58:14      0           0
#>     2      -118.041   6:58:14      0           0
#>     3      -117.869   6:58:14      0           0
#>     4      -117.803   6:58:14      0           0
#>     5      -117.787   6:58:14      0           0
#>     6      -117.784   6:58:14      0           0
#>     7      -117.783   6:58:14      0           0
#> [1] "GBLUP model complete - one trait"
#> [1] "Genomic predictions done for all traits in one repeat-fold"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -150.226   6:58:13      0           0
#>     2      -149.466   6:58:13      0           0
#>     3      -149.138   6:58:13      0           0
#>     4      -149.063   6:58:13      0           0
#>     5      -149.056   6:58:13      0           0
#>     6      -149.055   6:58:13      0           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -111.205   6:58:14      0           0
#>     2      -111.2   6:58:14      0           0
#>     3      -111.196   6:58:14      0           0
#>     4      -111.193   6:58:14      0           0
#>     5      -111.193   6:58:14      0           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -115.15   6:58:14      0           0
#>     2      -115.132   6:58:14      0           0
#>     3      -115.119   6:58:14      0           0
#>     4      -115.114   6:58:14      0           0
#>     5      -115.113   6:58:14      0           0
#>     6      -115.112   6:58:14      0           0
#> [1] "GBLUP model complete - one trait"
#> [1] "Genomic predictions done for all traits in one repeat-fold"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -150.983   6:58:14      1           0
#>     2      -150.511   6:58:14      1           0
#>     3      -150.265   6:58:14      1           0
#>     4      -150.179   6:58:14      1           0
#>     5      -150.162   6:58:14      1           0
#>     6      -150.158   6:58:14      1           0
#>     7      -150.157   6:58:14      1           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -109.264   6:58:14      0           0
#>     2      -109.264   6:58:14      0           0
#>     3      -109.264   6:58:14      0           0
#>     4      -109.263   6:58:14      0           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -116.271   6:58:15      0           0
#>     2      -116.238   6:58:15      0           0
#>     3      -116.225   6:58:15      0           0
#>     4      -116.223   6:58:15      0           0
#>     5      -116.223   6:58:15      0           0
#> [1] "GBLUP model complete - one trait"
#> [1] "Genomic predictions done for all traits in one repeat-fold"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -146.729   6:58:14      0           0
#>     2      -146.707   6:58:14      0           0
#>     3      -146.695   6:58:14      0           0
#>     4      -146.691   6:58:14      0           0
#>     5      -146.691   6:58:14      0           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -105.14   6:58:15      0           0
#>     2      -105.116   6:58:15      0           0
#>     3      -105.101   6:58:15      0           0
#>     4      -105.095   6:58:15      0           0
#>     5      -105.095   6:58:15      0           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -116.469   6:58:15      0           0
#>     2      -116.439   6:58:15      0           0
#>     3      -116.428   6:58:15      0           0
#>     4      -116.426   6:58:15      0           0
#>     5      -116.426   6:58:15      0           0
#> [1] "GBLUP model complete - one trait"
#> [1] "Genomic predictions done for all traits in one repeat-fold"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -146.167   6:58:15      0           0
#>     2      -145.784   6:58:15      0           0
#>     3      -145.645   6:58:15      0           0
#>     4      -145.618   6:58:15      0           0
#>     5      -145.616   6:58:15      0           0
#>     6      -145.616   6:58:15      0           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -108.335   6:58:15      0           0
#>     2      -108.255   6:58:15      0           0
#>     3      -108.205   6:58:15      0           0
#>     4      -108.187   6:58:15      0           0
#>     5      -108.184   6:58:15      0           0
#>     6      -108.184   6:58:15      0           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -115.606   6:58:16      0           0
#>     2      -115.563   6:58:16      0           0
#>     3      -115.541   6:58:16      0           0
#>     4      -115.535   6:58:16      0           0
#>     5      -115.534   6:58:16      0           0
#> [1] "GBLUP model complete - one trait"
#> [1] "Genomic predictions done for all traits in one repeat-fold"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -152   6:58:15      0           0
#>     2      -151.698   6:58:15      0           0
#>     3      -151.579   6:58:15      0           0
#>     4      -151.555   6:58:15      0           0
#>     5      -151.554   6:58:15      0           0
#>     6      -151.553   6:58:15      0           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -107.98   6:58:16      0           0
#>     2      -107.972   6:58:16      0           0
#>     3      -107.968   6:58:16      0           0
#>     4      -107.967   6:58:16      0           0
#> [1] "GBLUP model complete - one trait"
#> iteration    LogLik     wall    cpu(sec)   restrained
#>     1      -119.501   6:58:16      0           0
#>     2      -119.452   6:58:16      0           0
#>     3      -119.431   6:58:16      0           0
#>     4      -119.426   6:58:16      0           0
#>     5      -119.425   6:58:16      0           0
#> [1] "GBLUP model complete - one trait"
#> [1] "Genomic predictions done for all traits in one repeat-fold"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
#> Joining, by = "GID"
timeelapsed<-proc.time()[3]-starttime; 
timeelapsed/60
#>   elapsed 
#> 0.1421833
```

Save the results


```r
saveRDS(standardCV,file = here::here("output","standardCV.rds"))
```

## Plot results


```r
standardCV %>% 
     unnest(accuracyEstOut) %>% 
     dplyr::select(repeats,id,predOf,Trait,Accuracy) %>% 
     ggplot(.,aes(x=Trait,y=Accuracy,fill=Trait)) + 
     geom_boxplot() + theme_bw()
```

<img src="07-kFoldCrossVal_files/figure-html/unnamed-chunk-7-1.png" width="672" />

This result is not what I would expect. SELIND should be similar to individual trait accuracies.

Best guess: SELIND requires the BLUPs for each trait to be observed, so only the clones with complete data will be included.

Your results will be different if you choose a different dataset, hopefully better.
