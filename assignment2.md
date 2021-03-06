Data analysis assignment 2
================
Emilia Cook
06/02/2020

In this assignment you will work with relational data, i.e. data coming
from different data tables that you can combine using keys. Please read
ch.13 from R for Data Science before completing this assignment –
<https://r4ds.had.co.nz/relational-data.html>.

## Read data

We will work with three different tables: household roster from wave 8
(*h\_egoalt*), stable characteristics of individuals (*xwavedat*), and
household data from wave 8 (*h\_hhresp*).

``` r
library(tidyverse)
```

    ## Warning: package 'tidyverse' was built under R version 3.5.3

    ## Warning: package 'ggplot2' was built under R version 3.5.3

    ## Warning: package 'tibble' was built under R version 3.5.3

    ## Warning: package 'tidyr' was built under R version 3.5.3

    ## Warning: package 'readr' was built under R version 3.5.3

    ## Warning: package 'purrr' was built under R version 3.5.3

    ## Warning: package 'dplyr' was built under R version 3.5.3

    ## Warning: package 'stringr' was built under R version 3.5.3

    ## Warning: package 'forcats' was built under R version 3.5.3

``` r
# You need to complete the paths to these files on your computer.
Egoalt8 <- read_tsv("/Users/emili/Documents/Data SOC Term 2 Year 2/UKDA-6614-tab/tab/ukhls_w8/h_egoalt.tab")
Stable <- read_tsv("/Users/emili/Documents/Data SOC Term 2 Year 2/UKDA-6614-tab/tab/ukhls_wx/xwavedat.tab")
Hh8 <- read_tsv("/Users/emili/Documents/Data SOC Term 2 Year 2/UKDA-6614-tab/tab/ukhls_w8/h_hhresp.tab")
```

## Filter household roster data (10 points)

The **egoalt8** data table contains data on the kin and other
relationships between people in the same household. In each row in this
table you will have a pair of individuals in the same household: ego
(identified by *pidp*) and alter (identified by *apidp*).
*h\_relationship\_dv* shows the type of relationship between ego and
alter. You can check the codes in the Understanding Society codebooks
here –
<https://www.understandingsociety.ac.uk/documentation/mainstage/dataset-documentation>.

First we want to select only pairs of individuals who are husbands and
wives or cohabiting partners (codes 1 and 2). For convenience, we also
want to keep only the variables *pidp*, *apidp*, *h\_hidp* (household
identifier), *h\_relationship\_dv*, *h\_esex* (ego’s sex), and *h\_asex*
(alter’s sex).

``` r
Partners8 <- Egoalt8 %>%
        filter(h_relationship_dv == 1, 2) %>%
        select(pidp, apidp, h_hidp, h_relationship_dv, h_asex, h_sex)
```

Each couple now appears in the data twice: 1) with one partner as ego
and the other as alter, 2) the other way round. Now we will only focus
on heterosexual couples, and keep one observation per couple with women
as egos and men as their alters.

``` r
Hetero8 <- Partners8 %>%
        # filter out same-sex couples
        filter(h_sex!=1&h_asex==1|h_sex!=2&h_asex==2) %>%
        # keep only one observation per couple with women as egos
        filter(h_sex==2)
```

## Recode data on ethnicity (10 points)

In this assignment we will explore ethnic endogamy, i.e. marriages and
partnerships within the same ethnic group. First, let us a create a
version of the table with stable individual characteristics with two
variables only: *pidp* and *racel\_dv* (ethnicity).

``` r
Stable2 <- Stable %>%
        select(pidp, racel_dv)
```

Let’s code missing values on ethnicity (-9) as NA.

``` r
Stable2 <- Stable2 %>%
        mutate(racel_dv = recode(racel_dv, '-9' = NA_real_))
```

Now let us recode the variable on ethnicity into a new binary variable
with the following values: “White” (codes 1 to 4) and “non-White” (all
other codes).

``` r
Stable2 <- Stable2 %>%
        mutate(race = case_when(between(racel_dv,1,4)~"white", between(racel_dv,5,97)~"non-white"))
```

## Join data (30 points)

Now we want to join data from the household roster (*Hetero8*) and the
data table with ethnicity (*Stable2*). First let us merge in the data on
ego’s ethnicity. We want to keep all the observations we have in
*Hetero8*, but we don’t want to add any other individuals from
*Stable2*.

``` r
JoinedEthnicity <- Hetero8 %>%
        left_join(Stable2, by="pidp")
```

Let us rename the variables for ethnicity to clearly indicate that they
refer to egos.

``` r
JoinedEthnicity <- JoinedEthnicity %>%
        rename(egoRacel_dv = racel_dv) %>%
        rename(egoRace = race)
```

Now let us merge in the data on alter’s ethnicity. Note that in this
case the key variables have different names in two data tables; please
refer to the documentation for your join function (or the relevant
section from R for Data Science) to check the solution for this problem.

``` r
JoinedEthnicity <- JoinedEthnicity %>%
        inner_join(Stable2, by=c("apidp"="pidp")) 
```

Renaming the variables for alters.

``` r
JoinedEthnicity <- JoinedEthnicity %>%
        rename(alterRacel_dv = racel_dv) %>%
        rename(alterRace = race)
```

## Explore probabilities of racial endogamy (20 points)

Let us start by looking at the joint distribution of race (White
vs. non-White) of both partners.

``` r
TableRace <- JoinedEthnicity %>%
        # filter out observations with missing data
        filter(complete.cases(.)) %>%
        count(egoRace, alterRace)
TableRace
```

    ## # A tibble: 4 x 3
    ##   egoRace   alterRace     n
    ##   <chr>     <chr>     <int>
    ## 1 non-white non-white  1723
    ## 2 non-white white       267
    ## 3 white     non-white   207
    ## 4 white     white      7853

Now calculate the following probabilities: 1) for a White woman to have
a White partner, 2) for a White woman to have a non-White partner, 3)
for a non-White woman to have a White partner, 4) for a non-White woman
to have a non-White partner.

Of course, you can simply calculate these numbers manually. However, the
code will not be reproducible: if the data change the code will need to
be changed, too. Your task is to write reproducible code producing a
table with the required four probabilities.

``` r
TableRace %>%
        # group by ego's race to calculate sums
        group_by(egoRace) %>%
        # create a new variable with the total number of women by race
        mutate(total=sum(n)) %>%
        # create a new variable with the required probabilities 
        mutate(prob= n/total)
```

    ## # A tibble: 4 x 5
    ## # Groups:   egoRace [2]
    ##   egoRace   alterRace     n total   prob
    ##   <chr>     <chr>     <int> <int>  <dbl>
    ## 1 non-white non-white  1723  1990 0.866 
    ## 2 non-white white       267  1990 0.134 
    ## 3 white     non-white   207  8060 0.0257
    ## 4 white     white      7853  8060 0.974

## Join with household data and calculate mean and median number of children by ethnic group (30 points)

1)  Join the individual-level file with the household-level data from
    wave 8 (specifically, we want the variable for the number of
    children in the household).
2)  Select only couples that are ethnically endogamous (i.e. partners
    come from the same ethnic group) for the following groups: White
    British, Indian, and Pakistani.
3)  Produce a table showing the mean and median number of children in
    these households by ethnic group (make sure the table has meaningful
    labels for ethnic groups, not just numerical codes).
4)  Write a short interpretation of your results. What could affect your
    findings?

This table shows that overall Pakistani households have the most
children with the mean being 1.8 and therefore the median averaging 2.
White British households have the least amount of children with the mean
being 0.53, but the median showing 0. Pakistani Households could have
more children due to the tradition of larger families and being against
contraception.

``` r
Hh8 <- Hh8 %>%
        select(h_hidp, h_nkids_dv)
joined <- JoinedEthnicity %>%
        left_join(Hh8, by= "h_hidp")
head(joined)
```

    ## # A tibble: 6 x 11
    ##     pidp  apidp h_hidp h_relationship_~ h_asex h_sex egoRacel_dv egoRace
    ##    <dbl>  <dbl>  <dbl>            <dbl>  <dbl> <dbl>       <dbl> <chr>  
    ## 1 7.62e4 1.42e8 1.42e8                1      1     2           1 white  
    ## 2 2.80e5 7.56e8 7.56e8                1      1     2           1 white  
    ## 3 2.33e6 2.33e6 4.16e8                1      1     2          NA <NA>   
    ## 4 3.49e6 3.49e6 4.17e8                1      1     2          NA <NA>   
    ## 5 6.80e7 6.80e7 6.80e7                1      1     2           1 white  
    ## 6 6.80e7 6.81e7 6.81e7                1      1     2           1 white  
    ## # ... with 3 more variables: alterRacel_dv <dbl>, alterRace <chr>,
    ## #   h_nkids_dv <dbl>

``` r
ethnically.endogamous <- joined %>%
        filter(egoRacel_dv==1&alterRacel_dv==1 | egoRacel_dv==9&alterRacel_dv==9 | egoRacel_dv==10& alterRacel_dv==10)
head(ethnically.endogamous)
```

    ## # A tibble: 6 x 11
    ##     pidp  apidp h_hidp h_relationship_~ h_asex h_sex egoRacel_dv egoRace
    ##    <dbl>  <dbl>  <dbl>            <dbl>  <dbl> <dbl>       <dbl> <chr>  
    ## 1 7.62e4 1.42e8 1.42e8                1      1     2           1 white  
    ## 2 2.80e5 7.56e8 7.56e8                1      1     2           1 white  
    ## 3 6.80e7 6.80e7 6.80e7                1      1     2           1 white  
    ## 4 6.80e7 6.80e7 6.81e7                1      1     2           1 white  
    ## 5 6.80e7 6.80e7 6.81e7                1      1     2           1 white  
    ## 6 6.80e7 6.82e7 6.81e7                1      1     2           1 white  
    ## # ... with 3 more variables: alterRacel_dv <dbl>, alterRace <chr>,
    ## #   h_nkids_dv <dbl>

``` r
table.ethnically.endogamous <- ethnically.endogamous %>%
        group_by(egoRacel_dv) %>%
        summarise(mean=mean(h_nkids_dv, na.rm = TRUE), median=median(h_nkids_dv, na.rm=TRUE))

table.ethnically.endogamous$egoRacel_dv <- factor(table.ethnically.endogamous$egoRacel_dv, levels = c(1,9,10), labels = c(" whiteBritish", "Indian", "Pakistan"))

table.ethnically.endogamous
```

    ## # A tibble: 3 x 3
    ##   egoRacel_dv      mean median
    ##   <fct>           <dbl>  <dbl>
    ## 1 " whiteBritish" 0.529      0
    ## 2 Indian          0.963      1
    ## 3 Pakistan        1.81       2
