# R Stats GSoC 2021 Student Test

## Tests for Project: Expected Returns: FactorAnalytics

### System Configuration

- OS: MacOS Big Sur 11.2.1
- R: Version 4.0.4
- Important Dependencies:
  - ExpectedReturns: commit `39bc03d`
  - FactorAnalytics: commit `4499a1f`

### Task 1 (Easy)

#### Problem Statement

Easy: Begin by downloading and building the `ExpectedReturns` and `FactorAnalytics` packages locally. Work through, 
and list any build errors or issues you encounter on install.

```R
library(remotes)
install_github("JustinMShea/ExpectedReturns")
install_github("braverock/FactorAnalytics")
```

#### Solution

I have tried using the suggested installation command on my computer configuration. The problems that I have
encounter are the following

1. **Request Timeout**

    During the run of `install_github("braverock/ExpectedReturns")`, it tried to install
    `braverock/FactorAnalytics` but it has failed with the following error logs.

    ```bash
    ERROR: dependency ‘FactorAnalytics’ is not available for package ‘ExpectedReturns’
    * removing ‘/private/var/folders/97/bq5tb99n6cq9lmg8bxrp_33h0000gp/T/Rtmp7mCrpI/Rinstf4943408a02/ExpectedReturns’
    ```

    So I tried to install `FactorAnalytics` first instead.
    ```R
    install_github("braverock/FactorAnalytics")
    ```
    The following error was raised.
    ```bash
    > install_github("braverock/FactorAnalytics") 
    Downloading GitHub repo braverock/FactorAnalytics@HEAD
    Error in utils::download.file(url, path, method = method, quiet = quiet,  :
      download from 'https://api.github.com/repos/braverock/FactorAnalytics/tarball/HEAD' failed
    ```

    This was when I noticed that it cannot download the repository from Github and it is probably caused by
    timeout limit of `install_github`. It is probably set to 60 seconds. As I could not find a way to increase
    the limit. I download the files directly instead by clicking the link
    (https://api.github.com/repos/braverock/FactorAnalytics/tarball/HEAD) on an internet browser.

    Then I use `install_local` to install `FactorAnalytics`
    ```R
    install_local('/path/to/braverock-FactorAnalytics-4499a1f')
    ```

2. **Namespace Conflict and warning-converted error**

    ```R
    install_github("JustinMShea/ExpectedReturns")
    ```

    When I tried to come back to install `ExpectedReturn`, I found another error caused by namespace conflict.
    This was an error raised by `remotes` packages. I consult its documentation at https://remotes.r-lib.org/.
    At the very end of the page, I found that there is an environment variable that prevented converting a warning
    to be an error. 

    I checked the current setting of the env by using
    ```R
    Sys.getenv('R_REMOTES_NO_ERRORS_FROM_WARNINGS')
    ```
    and I got an empty string as an output `''`. So I fix this by using
    ```R
    Sys.setenv(R_REMOTES_NO_ERRORS_FROM_WARNINGS = TRUE)
    ```
    After solving this, I was able to install the packages successfully.


### Task 2 (Intermediate)



#### Problem Statement
Intermediate: Locate the `expected-returns-replications.Rmd` file in the vignettes directory. Refactor sections
of this vignettes to replace functions from the plm package with the `fitFfm` or `fitFfmDT` functions associated
with the `FactorAnalytics` package. This may include debugging upstream issues with merging data series, as well
as reformatting data to match requirements of the new function arguments.

#### Solution

The answer file is [here (r_stats_gsoc2021_task.html).](/task/r_stats_gsoc2021_task.html)

##### Additional Explanation on the process of solving the problem

In the vignette 'expected-returns-replications.Rmd', I notice a strange behavior at `Fama-French` section.
I noticed that the output Factor returns across periods of the factors fitted by `fitFfm` were all NA.

```R
fit.test <- fitFfm(data=test.data, asset.var="TICKER", ret.var="EXC.RETURN", date.var="DATE", 
                  exposure.vars=exposure.vars, addIntercept=TRUE)
```
Output
```
Call:
fitFfm(data = test.data, asset.var = "TICKER", ret.var = "EXC.RETURN", 
    date.var = "DATE", exposure.vars = exposure.vars, addIntercept = TRUE)

Model dimensions:
Factors  Assets Periods 
      4      22      59 

Factor returns across periods:
     Alpha               MKT.RF         SMB           HML     
 Min.   :-0.269555   Min.   : NA   Min.   : NA   Min.   : NA  
 1st Qu.:-0.065922   1st Qu.: NA   1st Qu.: NA   1st Qu.: NA  
 Median :-0.004664   Median : NA   Median : NA   Median : NA  
 Mean   :-0.023463   Mean   :NaN   Mean   :NaN   Mean   :NaN  
 3rd Qu.: 0.027509   3rd Qu.: NA   3rd Qu.: NA   3rd Qu.: NA  
 Max.   : 0.109357   Max.   : NA   Max.   : NA   Max.   : NA  
                     NA's   :59    NA's   :59    NA's   :59   

R-squared values across periods:
   Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
      0       0       0       0       0       0 

Residual Variances across assets:
    Min.  1st Qu.   Median     Mean  3rd Qu.     Max. 
0.001165 0.002585 0.002865 0.004530 0.004432 0.025192 
```

I visited `https://rdrr.io/rforge/factorAnalytics/man/fitFfm.html` and `https://github.com/braverock/FactorAnalytics/blob/master/R/fitFfm.R`.


This was to find out exactly why is the caused and I found this at `https://github.com/braverock/FactorAnalytics/blob/master/R/fitFfm.R#L416`

```R
if (grepl("LS",fit.method)) 
{
  reg.list <- by(data=data, INDICES=data[[date.var]], FUN=lm, 
                 formula=fm.formula, contrasts=contrasts.list, 
                 na.action=na.fail)
} 
```

The function is designed to fit the data with cross sectional regression.
It separates the data by date first and fit them with a linear model.
As the `Fama-French Model` is a time series model, this cannot be used.  

Also at `https://github.com/braverock/FactorAnalytics/blob/master/R/fitFfm.R#L260`,
the date column are forced to be a `Date` vector. So the same trick that was used with `plm` package by changing
the order of the indexes cannot be done.
```R
data[[date.var]] <- as.Date(data[[date.var]])
```

Thus, this function, `fitFfm`, cannot be used here and also the first path of the `Fama-MacBeth`.
However, It still can be used for fitting the second part of the `Fama-MacBeth`.

I looked into the `FactorAnalytics` to see if there are any other better function to use.
And I found `fitTsfm` at `https://rdrr.io/rforge/factorAnalytics/man/fitTsfm.html` and used this for the first part
of the `Fama-MacBeth` instead. 

In conclusion, I decided to add a section that refactor the `Fama-MacBeth` to use both of the mentioned functions.
I also confirm the result of both part at the end.

### Task 3 (Hard)

#### Problem Statement
Harder: Reflect on the steps above. How do you interpret the results of the new functions? In addition, was there
any repetitious code in the vignette that may be written as a function for future use? If so please include it as
an example. What data transformations or models might have benefited from writing unit tests? Please include
examples for these as well.

#### Solution
The answer is at the very bottom of [the same file (r_stats_gsoc2021_task.html).](/task/r_stats_gsoc2021_task.html)
