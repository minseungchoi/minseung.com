---
title: "How to (Meticulously) Convert Participant-Level Dispute Data to Dyadic Dispute-Year Data in R"
output:
  md_document:
    variant: gfm
    preserve_yaml: TRUE
author: "steve"
date: '2021-05-25'
excerpt: "Use some R code (and SQL code) to create dyadic dispute-year data from participant-level summaries."
layout: post
categories:
  - R
  - Political Science
image: "putin-visit-denmark-2011.jpg"
---





{% include image.html url="/images/putin-visit-denmark-2011.jpg" caption="Queen Margrethe and Prince Henrik of Denmark with Vladimir Putin on a State visit to Russia, Sept 7, 2011. The visit coincided with an increase in hostilities between Denmark and Russia that is recorded in the CoW-MID data." width=400 align="right" %}

<!-- <img src="http://svmiller.com/images/steveproj-hexlogo.png" alt="My steveproj hexlogo" align="right" width="250" style="padding: 0 15px; float: right;"/> -->

<style>
img[src*='#center'] { 
    display: block;
    margin: auto;
}
</style>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>


I had to recently dive into the Correlates of War (CoW) Militarized Interstate Dispute (MID) data for a co-authored project under a second round of revision. This project compared the latest release of the CoW-MID data ([v. 5.0, as of writing](https://correlatesofwar.org/data-sets/MIDs)) with what will be the latest evolution of [my conflict data with Doug Gibler](http://svmiller.com/gml-mid-data/). The task for this comparison was to convert CoW's MID data into a [EUGene](https://journals.sagepub.com/doi/abs/10.1177/0738894211413055)-inspired dyad-year data frame and highlight how my conflict data with Gibler offer substantial differences and improvements on what is available in CoW-MID. I won't belabor that point here---but please use our data instead for reasons we discuss across [multiple](http://dmgibler.people.ua.edu/mid-replication.html) [manuscripts](https://doi.org/10.1093/isq/sqaa011). Instead, I want to show users how to create these data in R. What follows will closely resemble the process by which we create dispute-year and dyad-year data as part of our data repository, which is why the application here will be CoW-MID rather than our data. They will also give a preview to how these data are prepared in my R package---[`{peacesciencer}`](http://svmiller.com/peacesciencer/). Check out the vignettes on the package's website for a follow-up tutorial on how to whittle dyadic dispute-year data into regular dyad-year data for your analyses.

I will start with a caveat. The CoW-MID data are limited in the information they provide in the context of multilateral conflicts. A classic case here is Brazil-Japan during World War II. Brazil and Japan were on opposite sides of World War II for about two years, but never directly interacted. Instead, Brazil's participation in World War II was against the Germans, incidentally fighting against the Germans largely on the Italian peninsula. The tutorial that follows will *not* tell you this kind of information. Instead, it will just create all dyadic conflict pairings in which there is a temporal overlap. It will also not inform the user as to what exactly happened in a given dyad-year. Consider the Belgium-Germany dyad in World War II as another example here. In 1939, Belgium mobilized against Germany and in 1940 the two fought an (operationalized) war. The participant-level data have no way of recording this because the highest action for Belgium's participation in World War II across both years was war. Thus, it's basically a way of mimicking what EUGene was able to do for conflict researchers a decade ago. If you want a more informative dyadic dispute-year data set, check out [Zeev Maoz' dyadic MID (DYMID) data](https://correlatesofwar.org/news/dyadic-mid-4-01-data-available) or, better yet, wait a little bit for the latest release of our conflict data. We have this information and much, much more.

With that in mind, let's load the participant-level data into an R session. Let's also load the R packages we'll be using here.

```r
library(lubridate) # for some date formatting
library(tidyverse) # for most things
library(sqldf)     # for some early-on SQL magic

# This is how it works in my setup.
# MIDA: dispute-level. MIDB: participant-level
MIDB <- haven::read_dta("~/Dropbox/data/cow/mid/5/MIDB 5.0.dta")
```

## Convert Participant-Level Data to "Long" Participant-Level Data

The first step in this process will require converting the participant-level data to a "long" format. This, by the way, is the greatest parlor trick for creating these types of data and it's everywhere in the [`{peacesciencer}`](http://svmiller.com/peacesciencer/) package. I've used it before a few times on my blog for [creating strategic rivalry data](http://svmiller.com/blog/2019/10/create-extend-strategic-rivalry-data-r/) and for [creating state-year/dyad-year data](http://svmiller.com/blog/2019/01/create-country-year-dyad-year-from-country-data/). 

However, there is a wrinkle you'll want to add here because one feature of the CoW-MID data is that some dates aren't known with precision. For example, assume a newspaper article dated Aug. 8, 1834 details a border incident between the United States and (British) Canada. Given the relative remoteness of these regions and the technology of the time, the newspaper article can pretty definitely say that a militarized incident took place, let's say, "earlier this week." The newspaper article was published August 8, but that doesn't mean the skirmish took place August 8 or even August 7. Under those conditions, the start day gets a `-9` (CoW-MID code for missing data). Given what we know about CoW-MID dispute-coding rules and how incidents must occur within six months of each other to connect them within one dispute, you can reasonably generate the data as follows. First, cases where the start day is missing will get a start day of 1 (first of the month). Cases where an end date is missing will get an end date that is the last day of the month. This could be either a 28 (February), 29 (leap year February), 30 (April, June, September, November), or 31 (every other month).

The next step is to use some `{lubridate}` functions to create a daily sequence of these days and extract (for the next step) the year, month, and day from these dates. The rest is going to look familiar to the other stuff I've done on this blog.

```r
MIDB %>%
  mutate(stdate = make_date(styear, stmon, ifelse(stday == -9, 1, stday)),
         enddate = make_date(endyear, endmon, 1),
         endday2 = ifelse(endday != -9, endday, day(ceiling_date(enddate, unit = "month")-1)),
         enddate = make_date(endyear, endmon, endday2)) %>% 
  rowwise() %>%
  mutate(date = list(seq(stdate, enddate, by = "1 day"))) %>%
  unnest(date) %>%
  arrange(dispnum, date, ccode) %>%
  mutate(year = year(date),
         month = month(date),
         day = day(date)) -> longPart
```

Let's use the Kargil War (MID#4007) as an illustration of what this did. First, here's how it's recorded in the participant-level data.


```r
MIDB %>% filter(dispnum == 4007)
#> # A tibble: 2 x 19
#>   dispnum stabb ccode stday stmon styear endday endmon endyear sidea revstate
#>     <dbl> <chr> <dbl> <dbl> <dbl>  <dbl>  <dbl>  <dbl>   <dbl> <dbl>    <dbl>
#> 1    4007 PAK     770    17     9   1993     17      7    1999     1        1
#> 2    4007 IND     750    17     9   1993     17      7    1999     0        0
#> # … with 8 more variables: revtype1 <dbl>, revtype2 <dbl>, fatality <dbl>,
#> #   fatalpre <dbl>, hiact <dbl>, hostlev <dbl>, orig <dbl>, version <dbl>
```

And here is what it looks like when made a little bit longer. Observe the new `year`, `month`, and `day` variables. We'll be making great use of those next.


```r
longPart %>% filter(dispnum == 4007) %>%
  select(dispnum:ccode, date:day, everything())
#> # A tibble: 4,260 x 26
#>    dispnum stabb ccode date        year month   day stday stmon styear endday
#>      <dbl> <chr> <dbl> <date>     <dbl> <dbl> <int> <dbl> <dbl>  <dbl>  <dbl>
#>  1    4007 IND     750 1993-09-17  1993     9    17    17     9   1993     17
#>  2    4007 PAK     770 1993-09-17  1993     9    17    17     9   1993     17
#>  3    4007 IND     750 1993-09-18  1993     9    18    17     9   1993     17
#>  4    4007 PAK     770 1993-09-18  1993     9    18    17     9   1993     17
#>  5    4007 IND     750 1993-09-19  1993     9    19    17     9   1993     17
#>  6    4007 PAK     770 1993-09-19  1993     9    19    17     9   1993     17
#>  7    4007 IND     750 1993-09-20  1993     9    20    17     9   1993     17
#>  8    4007 PAK     770 1993-09-20  1993     9    20    17     9   1993     17
#>  9    4007 IND     750 1993-09-21  1993     9    21    17     9   1993     17
#> 10    4007 PAK     770 1993-09-21  1993     9    21    17     9   1993     17
#> # … with 4,250 more rows, and 15 more variables: endmon <dbl>, endyear <dbl>,
#> #   sidea <dbl>, revstate <dbl>, revtype1 <dbl>, revtype2 <dbl>,
#> #   fatality <dbl>, fatalpre <dbl>, hiact <dbl>, hostlev <dbl>, orig <dbl>,
#> #   version <dbl>, stdate <date>, enddate <date>, endday2 <dbl>
```

## Create Directed Dyadic Dispute-Year Pairings with Some SQL Magic

The second step requires some SQL, at least as I do it. I will note, humbly, that this will look a bit convoluted and there is likely an R/"tidy" conversion for this SQL code (that's wrapped in the `sqldf()` function). However, it looks convoluted because it's the first thing I had to figure out when preparing our first batches of conflict data for release several years ago. This could be made prettier, but it works, and that's ultimately what I care about.

```r
sqldf("select A.dispnum dispnum, A.ccode ccode1, B.ccode ccode2,
      A.year year1, B.year year2, A.month month1, B.month month2, A.day day1, B.day day2,
      A.sidea sidea1, B.sidea sidea2,
             A.fatality fatality1, B.fatality fatality2,
             A.fatalpre fatalpre1, B.fatalpre fatalpre2,
             A.hiact hiact1, B.hiact hiact2, A.hostlev hostlev1, B.hostlev hostlev2,
             A.orig orig1, B.orig orig2
             from longPart A join longPart B using (dispnum)
             where A.ccode != B.ccode AND
             sidea1 != sidea2 AND year1 == year2 AND month1 == month2 AND day1 == day2
             order by A.ccode, B.ccode") %>% as_tibble() %>%
  # Some nip-and-tuck...
  rename(year = year1) %>%
  select(-year2, -month1:-day2) %>%
  group_by(dispnum, ccode1, ccode2, year, sidea1, sidea2) %>% slice(1) %>%
  arrange(dispnum) %>%
  ungroup() -> dDisp

```

I'll explain what this is doing. This SQL code is going to look into the `longPart` data and create every single dyadic pairing within a dispute (i.e. `using (dispnum)`). It will add the familiar variable suffices (sic?) of 1 and 2 denoting variable information for the respective countries in the dyadic pairing. It's only going to select the participant-level data that we care about in this context (i.e. we don't care about start months or end days in dyadic dispute-year data). To make it fully dyadic, it will importantly eliminate `ccode1` and `ccode2` are the same (i.e. `where A.ccode != B.ccode`) and it will (i.e. must) remove cases where both participants where on the same side of the conflict. Finally, it will only keep the observations where the years, months, and days align. Some nip-and-tuck after that will rename one of the year variables and remove the other one. We'll then drop the other month and day variables because we only needed them for matching. It will finally assign it to an object name of `dDisp`, indicating this is---and it is---a rudimentary directed dispute-year data set.

Here's an example of what this will look like. The Paraguayan War (MID#1590) was a multilateral war pitting Paraguay (`ccode`: 150) against a coalition of Argentina (`ccode`: 160) and Brazil (`ccode`: 140). Brazil and Paraguay initiate the MID (that became a war) in 1863. Argentina joined in 1865 on the side of Brazil. Here's what this looks like in these data (suppressing the follow-up code that formats the table).

```r
dDisp %>% filter(dispnum == 1590) %>%
  select(dispnum:fatality2, orig1, orig2, hiact1, hiact2)
```

<table id="stevetable">
<caption>The Paraguayan War (MID#1590) as Directed Dyadic Dispute-Year Data</caption>
 <thead>
  <tr>
   <th style="text-align:center;"> dispnum </th>
   <th style="text-align:center;"> ccode1 </th>
   <th style="text-align:center;"> ccode2 </th>
   <th style="text-align:center;"> year </th>
   <th style="text-align:center;"> sidea1 </th>
   <th style="text-align:center;"> sidea2 </th>
   <th style="text-align:center;"> fatality1 </th>
   <th style="text-align:center;"> fatality2 </th>
   <th style="text-align:center;"> orig1 </th>
   <th style="text-align:center;"> orig2 </th>
   <th style="text-align:center;"> hiact1 </th>
   <th style="text-align:center;"> hiact2 </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 140 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 1863 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 140 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 1864 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 140 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 1865 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 140 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 1866 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 140 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 1867 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 140 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 1868 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 140 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 1869 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 140 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 1870 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 140 </td>
   <td style="text-align:center;"> 1863 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 140 </td>
   <td style="text-align:center;"> 1864 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 140 </td>
   <td style="text-align:center;"> 1865 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 140 </td>
   <td style="text-align:center;"> 1866 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 140 </td>
   <td style="text-align:center;"> 1867 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 140 </td>
   <td style="text-align:center;"> 1868 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 140 </td>
   <td style="text-align:center;"> 1869 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 140 </td>
   <td style="text-align:center;"> 1870 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 160 </td>
   <td style="text-align:center;"> 1865 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 160 </td>
   <td style="text-align:center;"> 1866 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 160 </td>
   <td style="text-align:center;"> 1867 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 160 </td>
   <td style="text-align:center;"> 1868 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 160 </td>
   <td style="text-align:center;"> 1869 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 160 </td>
   <td style="text-align:center;"> 1870 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 160 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 1865 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 160 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 1866 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 160 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 1867 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 160 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 1868 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 160 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 1869 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 1590 </td>
   <td style="text-align:center;"> 160 </td>
   <td style="text-align:center;"> 150 </td>
   <td style="text-align:center;"> 1870 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 20 </td>
  </tr>
</tbody>
</table>

Nailed it. The data capture that Paraguay was Side A of this dispute, that Brazil and Argentina were Side B, and that Brazil and Paraguay were originators of the dispute (whereas Argentina joined two calendar years after it started).

## Check and Eliminate Duplicates (Where Necessary)

Always check for duplicate observations in your peace science data. This is a big part of [`{peacesciencer}`](http://svmiller.com/peacesciencer/)'s internal unit testing and few things indicate that I'm doing a lazy job on my data munging quite like the proliferation of duplicates. If you're not careful, and you're doing just matching by years, you'll invariably create duplicate dyadic dispute-year observations in the following two scenarios. The first is the case where there is a multilateral conflict featured multiple participants switching sides in the same calendar year. This certainly happened in World War II. Therein, Bulgaria, Finland, and Romania all switched from the Axis to the Allies around the same time in 1944. The second is a still possible---but much rarer---case of a participant dropping in and out of the dispute on the same side of the dispute in one calendar year. CoW-MID coding rules permit such a scenario when six months separate militarized incidents on the same underlying issue in the dispute. 

On the first scenario, observe the participant-level entries for Bulgaria, Finland, and Romania in World War II.

```r
MIDB %>%
  filter(dispnum == 258 & ccode %in% c(355, 360, 375)) %>%
  arrange(ccode, styear) %>%
  select(dispnum:sidea)
```

<table id="stevetable">
<caption>Bulgaria, Romania, and Finland Participant-Level Summaries for World War II</caption>
 <thead>
  <tr>
   <th style="text-align:center;"> dispnum </th>
   <th style="text-align:left;"> stabb </th>
   <th style="text-align:center;"> ccode </th>
   <th style="text-align:center;"> stday </th>
   <th style="text-align:center;"> stmon </th>
   <th style="text-align:center;"> styear </th>
   <th style="text-align:center;"> endday </th>
   <th style="text-align:center;"> endmon </th>
   <th style="text-align:center;"> endyear </th>
   <th style="text-align:center;"> sidea </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:left;"> BUL </td>
   <td style="text-align:center;"> 355 </td>
   <td style="text-align:center;"> 13 </td>
   <td style="text-align:center;"> 12 </td>
   <td style="text-align:center;"> 1941 </td>
   <td style="text-align:center;"> 26 </td>
   <td style="text-align:center;"> 8 </td>
   <td style="text-align:center;"> 1944 </td>
   <td style="text-align:center;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:left;"> BUL </td>
   <td style="text-align:center;"> 355 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 9 </td>
   <td style="text-align:center;"> 1944 </td>
   <td style="text-align:center;"> 7 </td>
   <td style="text-align:center;"> 5 </td>
   <td style="text-align:center;"> 1945 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:left;"> RUM </td>
   <td style="text-align:center;"> 360 </td>
   <td style="text-align:center;"> 22 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1941 </td>
   <td style="text-align:center;"> 23 </td>
   <td style="text-align:center;"> 8 </td>
   <td style="text-align:center;"> 1944 </td>
   <td style="text-align:center;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:left;"> RUM </td>
   <td style="text-align:center;"> 360 </td>
   <td style="text-align:center;"> 9 </td>
   <td style="text-align:center;"> 9 </td>
   <td style="text-align:center;"> 1944 </td>
   <td style="text-align:center;"> 7 </td>
   <td style="text-align:center;"> 5 </td>
   <td style="text-align:center;"> 1945 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:left;"> FIN </td>
   <td style="text-align:center;"> 375 </td>
   <td style="text-align:center;"> 25 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 1941 </td>
   <td style="text-align:center;"> 19 </td>
   <td style="text-align:center;"> 9 </td>
   <td style="text-align:center;"> 1944 </td>
   <td style="text-align:center;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:left;"> FIN </td>
   <td style="text-align:center;"> 375 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 9 </td>
   <td style="text-align:center;"> 1944 </td>
   <td style="text-align:center;"> 7 </td>
   <td style="text-align:center;"> 5 </td>
   <td style="text-align:center;"> 1945 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
</tbody>
</table>

The second scenario is a new scenario in the data that appears with the release of v. 5.0. MID#4676 is a NATO v. Russia MID in which Denmark appears twice in the same dispute in the same calendar year. This is the only instance of it happening in the history of the data.


```r
MIDB %>% filter(dispnum == 4676) %>%
  select(dispnum:sidea, hiact) %>%
     kable(., format="html",
        table.attr='id="stevetable"',
        caption = "A Participant-Level Summary of MID#4676 in CoW-MID (v. 5.0)",
        align=c("c", "l","c","c","c","c","c","c","c","c","c","c"))
```

<table id="stevetable">
<caption>A Participant-Level Summary of MID#4676 in CoW-MID (v. 5.0)</caption>
 <thead>
  <tr>
   <th style="text-align:center;"> dispnum </th>
   <th style="text-align:left;"> stabb </th>
   <th style="text-align:center;"> ccode </th>
   <th style="text-align:center;"> stday </th>
   <th style="text-align:center;"> stmon </th>
   <th style="text-align:center;"> styear </th>
   <th style="text-align:center;"> endday </th>
   <th style="text-align:center;"> endmon </th>
   <th style="text-align:center;"> endyear </th>
   <th style="text-align:center;"> sidea </th>
   <th style="text-align:center;"> hiact </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:center;"> 4676 </td>
   <td style="text-align:left;"> UKG </td>
   <td style="text-align:center;"> 200 </td>
   <td style="text-align:center;"> 19 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 2011 </td>
   <td style="text-align:center;"> 8 </td>
   <td style="text-align:center;"> 2 </td>
   <td style="text-align:center;"> 2011 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 7 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 4676 </td>
   <td style="text-align:left;"> DEN </td>
   <td style="text-align:center;"> 390 </td>
   <td style="text-align:center;"> 19 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 2011 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 2011 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 7 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 4676 </td>
   <td style="text-align:left;"> USA </td>
   <td style="text-align:center;"> 2 </td>
   <td style="text-align:center;"> 23 </td>
   <td style="text-align:center;"> 6 </td>
   <td style="text-align:center;"> 2011 </td>
   <td style="text-align:center;"> 18 </td>
   <td style="text-align:center;"> 10 </td>
   <td style="text-align:center;"> 2011 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 7 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 4676 </td>
   <td style="text-align:left;"> DEN </td>
   <td style="text-align:center;"> 390 </td>
   <td style="text-align:center;"> 7 </td>
   <td style="text-align:center;"> 11 </td>
   <td style="text-align:center;"> 2011 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 4 </td>
   <td style="text-align:center;"> 2012 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 7 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 4676 </td>
   <td style="text-align:left;"> FIN </td>
   <td style="text-align:center;"> 375 </td>
   <td style="text-align:center;"> 7 </td>
   <td style="text-align:center;"> 11 </td>
   <td style="text-align:center;"> 2011 </td>
   <td style="text-align:center;"> 7 </td>
   <td style="text-align:center;"> 11 </td>
   <td style="text-align:center;"> 2011 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 4676 </td>
   <td style="text-align:left;"> NOR </td>
   <td style="text-align:center;"> 385 </td>
   <td style="text-align:center;"> 19 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 2011 </td>
   <td style="text-align:center;"> 13 </td>
   <td style="text-align:center;"> 7 </td>
   <td style="text-align:center;"> 2011 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 7 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 4676 </td>
   <td style="text-align:left;"> NOR </td>
   <td style="text-align:center;"> 385 </td>
   <td style="text-align:center;"> 19 </td>
   <td style="text-align:center;"> 4 </td>
   <td style="text-align:center;"> 2012 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 4 </td>
   <td style="text-align:center;"> 2012 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 7 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 4676 </td>
   <td style="text-align:left;"> BEL </td>
   <td style="text-align:center;"> 211 </td>
   <td style="text-align:center;"> 19 </td>
   <td style="text-align:center;"> 4 </td>
   <td style="text-align:center;"> 2012 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 4 </td>
   <td style="text-align:center;"> 2012 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 7 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 4676 </td>
   <td style="text-align:left;"> LIT </td>
   <td style="text-align:center;"> 368 </td>
   <td style="text-align:center;"> 27 </td>
   <td style="text-align:center;"> 3 </td>
   <td style="text-align:center;"> 2012 </td>
   <td style="text-align:center;"> 27 </td>
   <td style="text-align:center;"> 3 </td>
   <td style="text-align:center;"> 2012 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 4676 </td>
   <td style="text-align:left;"> CAN </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 27 </td>
   <td style="text-align:center;"> 9 </td>
   <td style="text-align:center;"> 2011 </td>
   <td style="text-align:center;"> 27 </td>
   <td style="text-align:center;"> 9 </td>
   <td style="text-align:center;"> 2011 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 7 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 4676 </td>
   <td style="text-align:left;"> RUS </td>
   <td style="text-align:center;"> 365 </td>
   <td style="text-align:center;"> 19 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 2011 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 4 </td>
   <td style="text-align:center;"> 2012 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 7 </td>
  </tr>
</tbody>
</table>

We need to take inventory of what duplicates may have emerged in these data to see if anything needs a more careful inspection.


```r
dDisp %>%
  group_by(dispnum, ccode1, ccode2, year) %>%
  tally() %>% filter(n > 1)
#> # A tibble: 0 x 5
#> # Groups:   dispnum, ccode1, ccode2 [0]
#> # … with 5 variables: dispnum <dbl>, ccode1 <dbl>, ccode2 <dbl>, year <dbl>,
#> #   n <int>
```

Fortunately, this came up empty. Because we made the sequence in SQL on dates and not years, we don't have an incorrect scenario where Finland is fighting Bulgaria and Romania where Finland is on the Allies and Bulgaria and Romania are on the Axis. That never happened.



```r
dDisp %>%
    select(dispnum:sidea2) %>%
    filter(dispnum == 258 & ccode1 %in% c(355, 360, 375) & ccode2 %in% c(355, 360, 375))
#> # A tibble: 4 x 6
#>   dispnum ccode1 ccode2  year sidea1 sidea2
#>     <dbl>  <dbl>  <dbl> <dbl>  <dbl>  <dbl>
#> 1     258    355    375  1944      1      0
#> 2     258    360    375  1944      1      0
#> 3     258    375    355  1944      0      1
#> 4     258    375    360  1944      0      1
```

We also have just one observation year in 2011 for Denmark-Russia in MID#4676.


```r
dDisp %>%
    select(dispnum:sidea2) %>%
    filter(dispnum == 4676 & ccode1 %in% c(365, 390) & ccode2 %in% c(365, 390))
#> # A tibble: 4 x 6
#>   dispnum ccode1 ccode2  year sidea1 sidea2
#>     <dbl>  <dbl>  <dbl> <dbl>  <dbl>  <dbl>
#> 1    4676    365    390  2011      0      1
#> 2    4676    365    390  2012      0      1
#> 3    4676    390    365  2011      1      0
#> 4    4676    390    365  2012      1      0
```


Duplicate *dyad-year* observations will remain in the data, but these are all cases where a dyad had multiple disputes ongoing in the same year (e.g. Italy and France had three unique dispute onsets in 1860). However, there are no duplicate dyadic dispute-year data.

## Create Onset/Ongoing Variables

Already, these chunks of code created directed dyadic dispute-year data from the dispute-level and participant-level data. We also scrubbed these data from some erroneous dyadic pairings that were created from some SQL code. The next step in the process is to create two important variables for conflict researchers: whether a dispute was ongoing in the dyadic dispute-year and whether it was the start of the dyadic dispute. The first is a simple case. By virtue of the data we're using, the ongoing variable must always be 1 and researchers who are interested in removing irrelevant dyadic dispute-year pairings like Brazil-Japan during World War II or China-South Korea in the Vietnam War should look into Zeev Maoz' dyadic MID data (or, better yet, wait for the next release of our conflict data).


```r
dDisp %>%
  # Simple ongoing variable. This one is easy.
  mutate(dispongoing = 1) -> dDisp
```

The onset variable will require a little bit more work. In most (really: almost all) applications, the first dyadic dispute-year observation will be the onset. Most conflicts are bilateral and very few conflicts are as complex as World War II or as weird as that 2011-12 NATO-Russia MID referenced above. Further, cases where a side is dropping in/out in the same calendar year will routinely have more than a calendar year of separation between entry points. The Netherlands in what became the Iraq War (MID#4273) is a nice illustration of this. The Netherlands first appeared in MID#4273 in 1999 for a one-day incident. It next appears in March 2003, leading into the war itself.

In other words, you can get almost all onset cases in the data by assigning the onset to the first dyadic observation and cases where the previous dyadic dispute-year is separated by more than a year.


```r
dDisp %>%
  group_by(dispnum, ccode1, ccode2) %>%
  # Now, here are two ways for calculating onsets:
  # grouping by dispnum, ccode1, and ccode2, find the first observation
  # This should, btw, take care of those  cases where there was side-switching. Mostly in WW2, naturally.
  mutate(first = ifelse(row_number() == 1, 1, 0)) %>%
  # Now, for cases where states dropped out and dropped in again, let's do a quick solution here.
  # If the lag difference between years is > 1, it's a new onset.
  # Thus, midonset is any first observation or a case where the ydiff > 1.
  # This will help a little bit with MID#4273.
  mutate(ydiff = year - lag(year, 1),
         disponset = ifelse(first == 1 | ydiff > 1, 1, 0)) %>%
  select(dispnum:year,  dispongoing, disponset, everything(), -first, -ydiff) %>%
  ungroup() -> dDisp
```

Observe what this will do with a tricky case like France in World War II. France appears three times in this war. It starts on the Allies' side until Germany eliminates it in 1940. It then reappears on the Axis' side from 1940 to 1941. It next appears again on the Allies' side from 1944 to 1945. Notice how well we captured the onsets with that in mind. Also notice there is now France-U.S. dyadic pairing in World War II. While France was on the Axis' side in 1941 and the U.S. joined the conflict in 1941, France's participation in the Axis ends in July and the U.S. entry happens in December. There is no overlap and, thus, no dyadic pairing in these data.

```r
dDisp %>%
  filter(dispnum == 258 & ccode1 == 220) %>%
  select(dispnum:sidea2) %>%
  arrange(year)
```
<table id="stevetable">
<caption>Dyadic Dispute-Year Data for France in World War II</caption>
 <thead>
  <tr>
   <th style="text-align:center;"> dispnum </th>
   <th style="text-align:center;"> ccode1 </th>
   <th style="text-align:center;"> ccode2 </th>
   <th style="text-align:center;"> year </th>
   <th style="text-align:center;"> dispongoing </th>
   <th style="text-align:center;"> disponset </th>
   <th style="text-align:center;"> sidea1 </th>
   <th style="text-align:center;"> sidea2 </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 255 </td>
   <td style="text-align:center;"> 1939 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 365 </td>
   <td style="text-align:center;"> 1939 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 1940 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 200 </td>
   <td style="text-align:center;"> 1940 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 255 </td>
   <td style="text-align:center;"> 1940 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 325 </td>
   <td style="text-align:center;"> 1940 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 350 </td>
   <td style="text-align:center;"> 1940 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 560 </td>
   <td style="text-align:center;"> 1940 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 900 </td>
   <td style="text-align:center;"> 1940 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 920 </td>
   <td style="text-align:center;"> 1940 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 20 </td>
   <td style="text-align:center;"> 1941 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 200 </td>
   <td style="text-align:center;"> 1941 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 345 </td>
   <td style="text-align:center;"> 1941 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 350 </td>
   <td style="text-align:center;"> 1941 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 365 </td>
   <td style="text-align:center;"> 1941 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 530 </td>
   <td style="text-align:center;"> 1941 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 560 </td>
   <td style="text-align:center;"> 1941 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 900 </td>
   <td style="text-align:center;"> 1941 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 920 </td>
   <td style="text-align:center;"> 1941 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 255 </td>
   <td style="text-align:center;"> 1944 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 310 </td>
   <td style="text-align:center;"> 1944 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 740 </td>
   <td style="text-align:center;"> 1944 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 255 </td>
   <td style="text-align:center;"> 1945 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 310 </td>
   <td style="text-align:center;"> 1945 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:center;"> 258 </td>
   <td style="text-align:center;"> 220 </td>
   <td style="text-align:center;"> 740 </td>
   <td style="text-align:center;"> 1945 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
   <td style="text-align:center;"> 1 </td>
   <td style="text-align:center;"> 0 </td>
  </tr>
</tbody>
</table>

So far this isn't bad, but it's not complete. There are several cases of states dropping in/out of a MID on the same side, but where participation is not separated by more than a year. Syria in MID#4182 is a nice illustration of this wrinkle. Conflict researchers should recognize MID#4182 as the longest-running MID in the post-Congress of Vienna history of the world. In the CoW-MID coding of this dispute, Syria appears five separate times (as a joiner) in the 13-year dispute. Only its last appearance (2000) was separated by more than a calendar year from its previous participation.


```r
MIDB %>% filter(dispnum == 4182) %>%
  arrange(-sidea, -orig,  ccode, styear)
#> # A tibble: 7 x 19
#>   dispnum stabb ccode stday stmon styear endday endmon endyear sidea revstate
#>     <dbl> <chr> <dbl> <dbl> <dbl>  <dbl>  <dbl>  <dbl>   <dbl> <dbl>    <dbl>
#> 1    4182 ISR     666     6     4   1993      8      9    2006     1        1
#> 2    4182 LEB     660     6     4   1993      8      9    2006     0        1
#> 3    4182 SYR     652    12     7   1993     19      8    1993     0        0
#> 4    4182 SYR     652    23     5   1994     22      8    1994     0        0
#> 5    4182 SYR     652    22     5   1995     15      9    1995     0        0
#> 6    4182 SYR     652    11     4   1996     -9      8    1996     0        0
#> 7    4182 SYR     652    24     5   2000      1      7    2001     0        0
#> # … with 8 more variables: revtype1 <dbl>, revtype2 <dbl>, fatality <dbl>,
#> #   fatalpre <dbl>, hiact <dbl>, hostlev <dbl>, orig <dbl>, version <dbl>
```

Here's what it will look like more generally.


```r
MIDB %>%
  group_by(dispnum, ccode) %>%
  # find side-switchers and the drop-ins/outs.
  mutate(n = n()) %>%
  filter(n > 1) %>%
   # arrange chronologically
  arrange(dispnum, ccode, styear, stmon) %>%
  # what was the previous end year?
  # did it switch sides? We already have side-switchers covered.
  mutate(prevendyr = lag(endyear),
         lagsidea = lag(sidea)) %>%
  select(dispnum:endyear, sidea, prevendyr, lagsidea) %>%
  mutate(ydiff = styear - prevendyr) %>%
  # Give me the cases where the separation was a year or less AND not a side-switcher
  filter(ydiff <= 1 & (sidea - lagsidea == 0))
#> # A tibble: 16 x 13
#> # Groups:   dispnum, ccode [12]
#>    dispnum stabb ccode stday stmon styear endday endmon endyear sidea prevendyr
#>      <dbl> <chr> <dbl> <dbl> <dbl>  <dbl>  <dbl>  <dbl>   <dbl> <dbl>     <dbl>
#>  1    4095 GRC     350    21     7   1999      7     10    1999     0      1998
#>  2    4095 GRC     350    20    10   2000     26      4    2001     0      1999
#>  3    4137 MAC     343    16     3   1999     10      6    1999     1      1998
#>  4    4182 SYR     652    23     5   1994     22      8    1994     0      1993
#>  5    4182 SYR     652    22     5   1995     15      9    1995     0      1994
#>  6    4182 SYR     652    11     4   1996     -9      8    1996     0      1995
#>  7    4273 KUW     690    26     9   2002     18      3    2003     1      2001
#>  8    4676 NOR     385    19     4   2012     20      4    2012     1      2011
#>  9    4676 DEN     390     7    11   2011     20      4    2012     1      2011
#> 10    4678 UKG     200    29     4   2014     29     10    2014     1      2013
#> 11    4678 EST     366    21    10   2014     12     12    2014     1      2013
#> 12    4678 FIN     375    20     5   2014     28     10    2014     1      2013
#> 13    4678 NOR     385    29    10   2014     31     10    2014     1      2013
#> 14    4680 ROK     732    20     5   2014     21      8    2014     0      2013
#> 15    4691 USA       2    30     8   2013      1      9    2013     0      2012
#> 16    4691 USA       2    15     9   2014     15      9    2014     0      2013
#> # … with 2 more variables: lagsidea <dbl>, ydiff <dbl>
```

There is a way to automate this, but sometimes the safest thing is to take care to do it yourself. Use this information and manually insert some onsets in a `case_when()` wrapper.


```r
dDisp %>%
  mutate(disponset = case_when(
    dispnum == 4095 & (ccode1 == 350 | ccode2 == 350) & year %in% c(1999, 2000) ~ 1,
    dispnum == 4137 & (ccode1 == 343 | ccode2 == 343) & year == 1999 ~ 1,
    dispnum == 4182 & (ccode1 == 652 | ccode2 == 652) & year %in% c(1994, 1995, 1996) ~ 1,
    dispnum == 4273 & (ccode1 == 690 | ccode2 == 690) & year == 2002 ~ 1,
    dispnum == 4676 & (ccode1 == 385 | ccode2 == 385) & year == 2012 ~ 1,
    # The Denmark example will be overkill, but there's no harm in doing this.
    dispnum == 4676 & (ccode1 == 390 | ccode2 == 390) & year == 2011 ~ 1,
    dispnum == 4678 & (ccode1 == 200 | ccode2 == 200) & year == 2014 ~ 1,
    dispnum == 4678 & (ccode1 == 366 | ccode2 == 366) & year == 2014 ~ 1,
    dispnum == 4678 & (ccode1 == 385 | ccode2 == 385) & year == 2014 ~ 1,
    dispnum == 4680 & (ccode1 == 732 | ccode2 == 732) & year == 2014 ~ 1,
    dispnum == 4691 & (ccode1 == 2 | ccode2 == 2) & year %in% c(2013, 2014) ~ 1,
    TRUE ~ disponset
  )) -> dDisp
```


Let's check our work with that case of MID#4182, looking at the `disponset` column.


```r
dDisp %>% filter(dispnum == 4182 & ccode1 == 652)
#> # A tibble: 6 x 18
#>   dispnum ccode1 ccode2  year dispongoing disponset sidea1 sidea2 fatality1
#>     <dbl>  <dbl>  <dbl> <dbl>       <dbl>     <dbl>  <dbl>  <dbl>     <dbl>
#> 1    4182    652    666  1993           1         1      0      1         1
#> 2    4182    652    666  1994           1         1      0      1         1
#> 3    4182    652    666  1995           1         1      0      1         1
#> 4    4182    652    666  1996           1         1      0      1         1
#> 5    4182    652    666  2000           1         1      0      1         1
#> 6    4182    652    666  2001           1         0      0      1         1
#> # … with 9 more variables: fatality2 <dbl>, fatalpre1 <dbl>, fatalpre2 <dbl>,
#> #   hiact1 <dbl>, hiact2 <dbl>, hostlev1 <dbl>, hostlev2 <dbl>, orig1 <dbl>,
#> #   orig2 <dbl>
```

Nailed it.

This is all the code you need to create directed dyadic dispute-year data from the participant-level CoW-MID data. I'll reiterate the caveat with which I started. These functions in R will basically recreate what EUGene created for conflict researchers over a decade ago. It is, admittedly, a naive extension of dispute-level data and participant-level data into dyadic dispute-year data. It won't tell you that Brazil and Japan never directly fought each other in World War II. If you want that kind of information, Zeev Maoz has it available as part of his dyadic MID data. I recommend waiting for the newest release of the conflict data I jointly maintain with Doug Gibler. That data will have a truly dyadic component along with other goodies.

The next step for conflict researchers is to coerce dyadic dispute-year data into a dyad-year data frame, which requires incorporating some reasonable case exclusion rules. [`{peacesciencer}`](http://svmiller.com/peacesciencer/) already does this. The next vignette for it will explain how and why it does this.