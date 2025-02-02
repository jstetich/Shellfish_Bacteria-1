Revised Graphics Summarizing Shellfish Bacteria Data
================
Curtis C. Bohlen, Casco Bay Estuary Partnership.
02/17/2021

-   [Introduction](#introduction)
-   [Relevant Standards](#relevant-standards)
-   [Load Libraries](#load-libraries)
-   [Load Data](#load-data)
    -   [Main Data](#main-data)
        -   [Remove NAs](#remove-nas)
        -   [Remove Sites not in Region](#remove-sites-not-in-region)
    -   [Summary Statistics Dataframe](#summary-statistics-dataframe)
-   [Graphics for 238 Stations](#graphics-for-238-stations)
    -   [Bootstrap Confidence Interval
        Function](#bootstrap-confidence-interval-function)
    -   [Confidence Intervals for the Geometric
        Mean](#confidence-intervals-for-the-geometric-mean)
    -   [Confidence Intervals for the 90th
        Percentile](#confidence-intervals-for-the-90th-percentile)
    -   [Combining Results](#combining-results)
-   [Assemble Long Data](#assemble-long-data)
-   [Create Graphics](#create-graphics)
    -   [Base Plot](#base-plot)
        -   [Mimicing Graphic As Modified by the Graphic
            Designer](#mimicing-graphic-as-modified-by-the-graphic-designer)
        -   [Alternate Annotations](#alternate-annotations)
    -   [Shapes with oultines Plot](#shapes-with-oultines-plot)
        -   [Mimicing Graphic As Modified by the Graphic
            Designer](#mimicing-graphic-as-modified-by-the-graphic-designer-1)
        -   [Alternate Annotations](#alternate-annotations-1)

<img
    src="https://www.cascobayestuary.org/wp-content/uploads/2014/04/logo_sm.jpg"
    style="position:absolute;top:10px;right:50px;" />

# Introduction

Exploratory analysis highlights the extreme skewness of the distribution
of bacteria data, both here with the shellfish-related data collected by
DMR, and with the data related to recreational beaches, collected by
towns, and managed by DEP.

Here we present graphical summaries of the data, with an emphasis on
reporting observed quantities that relate to regulatory thresholds,
especially geometric means and 90th percentiles related to whether sites
can be classified as “Approved” for harvest.

This Notebook responds to comments on our final draft indicator, and
incorporates some changes suggested by teh graphic designer and final
in-house reviewers.

# Relevant Standards

| Growing Area Classification | Activity Allowed                                                          | Geometric mean FC/100ml | 90th Percentile (P90) FC/100ml |
|-----------------------------|---------------------------------------------------------------------------|-------------------------|--------------------------------|
| Approved                    | Harvesting allowed                                                        | ≤ 14                    | ≤ 31                           |
| Conditionally Approved      | Harvesting allowed except during specified conditions                     | ≤ 14 in open status     | ≤ 31 in open status            |
| Restricted                  | Depuration harvesting or relay only                                       | ≤ 88 and &gt;15         | ≤ 163 and &gt;31               |
| Conditionally Restricted    | Depuration harvesting or relay allowed except during specified conditions | ≤ 88 in open status     | ≤ 163 in open status           |
| Prohibited                  | Aquaculture seed production only                                          | &gt;88                  | &gt;163                        |

# Load Libraries

``` r
library(readr)
#> Warning: package 'readr' was built under R version 4.0.5
library(tidyverse)      # Loads another `select()`
#> Warning: package 'tidyverse' was built under R version 4.0.5
#> -- Attaching packages --------------------------------------- tidyverse 1.3.1 --
#> v ggplot2 3.3.5     v dplyr   1.0.7
#> v tibble  3.1.4     v stringr 1.4.0
#> v tidyr   1.1.3     v forcats 0.5.1
#> v purrr   0.3.4
#> Warning: package 'ggplot2' was built under R version 4.0.5
#> Warning: package 'tibble' was built under R version 4.0.5
#> Warning: package 'tidyr' was built under R version 4.0.5
#> Warning: package 'dplyr' was built under R version 4.0.5
#> Warning: package 'forcats' was built under R version 4.0.5
#> -- Conflicts ------------------------------------------ tidyverse_conflicts() --
#> x dplyr::filter() masks stats::filter()
#> x dplyr::lag()    masks stats::lag()
library(CBEPgraphics)
load_cbep_fonts()
theme_set(theme_cbep())
```

# Load Data

## Main Data

``` r
sibfldnm <- 'Derived_Data'
parent <- dirname(getwd())
sibling <- file.path(parent,sibfldnm)

dir.create(file.path(getwd(), 'figures'), showWarnings = FALSE)
```

``` r
fl1<- "Shellfish data 2015 2018.csv"
path <- file.path(sibling, fl1)

coli_data <- read_csv(path, 
    col_types = cols(SDate = col_date(format = "%Y-%m-%d"), 
        SDateTime = col_datetime(format = "%Y-%m-%dT%H:%M:%SZ"), # Note Format!
        STime = col_time(format = "%H:%M:%S"))) %>%
  mutate_at(c(6:7), factor) %>%
  mutate(Class = factor(Class, levels = c( 'A', 'CA', 'CR',
                                           'R', 'P', 'X' ))) %>%
  mutate(Tide = factor(Tide, levels = c("L", "LF", "F", "HF",
                                        "H", "HE", "E", "LE"))) %>%
  mutate(DOY = as.numeric(format(SDate, format = '%j')),
         Month = as.numeric(format(SDate, format = '%m'))) %>%
  mutate(Month = factor(Month, levels = 1:12, labels = month.abb))
```

### Remove NAs

``` r
coli_data <- coli_data %>%
  filter (! is.na(ColiVal))
```

### Remove Sites not in Region

We have some data that was selected for stations outside of Casco Bay.
To be  
careful, we remove sampling data for any site in th two adjacent Growing
Areas, “WH” and “WM”.

``` r
coli_data <- coli_data %>%
  filter(GROW_AREA != 'WH' & GROW_AREA != "WM") %>%
  mutate(GROW_AREA = fct_drop(GROW_AREA),
         Station = factor(Station))
```

## Summary Statistics Dataframe

We only work with the raw observed summary statistics here.

``` r
sum_data <- coli_data %>%
  mutate(logcoli = log(ColiVal)) %>%
  group_by(Station) %>%
  summarize(p901 = quantile(ColiVal, 0.9),
            meanlog1 = mean(logcoli, na.rm = TRUE),
            sdlog1 = sd(logcoli, na.rm = TRUE),
            nlog1 = sum(! is.na(logcoli)),
            selog1 = sdlog1/sqrt(nlog1),
            gmean1 = exp(meanlog1),
            U_CI1 = exp(meanlog1 + 1.96 * selog1),
            L_CI1 = exp(meanlog1 - 1.96 * selog1)) %>%
  mutate(Station = fct_reorder(Station, gmean1))
```

# Graphics for 238 Stations

## Bootstrap Confidence Interval Function

This is a general function, so needs to be passed log transformed data
to produce geometric means. Note that we can substitute another function
for the mean to generate confidence intervals for other statistics, here
the 90th percentile.

``` r
boot_one <- function (dat, fun = "mean", sz = 1000, width = 0.95) {
  
  low <- (1 - width)/2
  hi <- 1 - low

  vals <- numeric(sz)
  for (i in 1:sz) {
    vals[i] <- eval(call(fun, sample(dat, length(dat), replace = TRUE)))
  }
  return (quantile(vals, probs = c(low, hi)))
}
```

## Confidence Intervals for the Geometric Mean

We need to first calculate confidence intervals on a log scale, then
build a tibble and back transform them.

``` r
gm_ci <- tapply(log(coli_data$ColiVal), coli_data$Station, boot_one)

# Convert to data frame (and then tibble, implicitly...)
# This is convenient because as_tibble() drops the row names,
# which we want to keep.
gm_ci <- as.data.frame(do.call(rbind, gm_ci)) %>%
  rename(gm_lower1 = `2.5%`, gm_upper1 = `97.5%`) %>%
  rownames_to_column('Station')

# Back Transform
gm_ci <- gm_ci %>%
  mutate(gm_lower1 = exp(gm_lower1),
         gm_upper1 = exp(gm_upper1))
```

## Confidence Intervals for the 90th Percentile

Because we use `eval()` and
call()`inside the`boot\_one()`function, we need to pass the function we want to bootstrap as a string. We can't pass in an anonymous function.  The function`call()`assembles a call object (unevaluated).  It's first argument must be a character string.  Then`eval()\`
evaluates the call, seeking the named function among function
identifiers in the current environment.

All of that could be addressed with more advanced R programming, such as
quoting functions or quosures. But for our current purpose, this is more
than sufficient.

``` r
p90 <- function(.x) quantile(.x, 0.9)
```

This takes a while to run because calculating percentiles is harder than
calculating the mean.

``` r
p90_ci <- tapply(coli_data$ColiVal, coli_data$Station,
               function(d) boot_one(d, 'p90'))

# Convert to data frame (and then tibble...) 
p90_ci <- as.data.frame(do.call(rbind, p90_ci)) %>%
  rename(p90_lower1 = `2.5%`, p90_upper1 = `97.5%`) %>%
  rownames_to_column('Station')
```

## Combining Results

We add results to the summary data. (Because this uses `left_join()`,
rerunning it without deleting the old versions will generate errors in
later steps.)

``` r
sum_data <-  sum_data %>% 
  left_join(gm_ci, by = 'Station') %>%
  left_join(p90_ci, by = 'Station') %>%
  mutate(Station = fct_reorder(Station, gmean1))
```

# Assemble Long Data

``` r
long_dat <- sum_data %>%
  select(Station, gmean1, gm_lower1, gm_upper1, p901, p90_lower1, p90_upper1) %>%
  rename(gm_value = gmean1, p90_value = p901) %>%
  rename_with( ~sub('1', '', .x)) %>%
  pivot_longer(gm_value:p90_upper,
               names_to = c('parameter', 'type'), 
               names_sep = '_') %>%
  pivot_wider(c(Station, parameter), names_from = type, values_from = value) %>%
  mutate(parameter = factor(parameter, 
                            levels = c('p90', 'gm'), 
                            labels = c('90th Percentile', 'Geometric Mean'))) %>%
  arrange(parameter, Station)
```

# Create Graphics

## Base Plot

``` r
plt <- ggplot(long_dat, aes(Station, value)) + 

  geom_linerange(aes(ymin = lower, ymax = upper, color = parameter),
                 alpha = 0.25,
                 size = .1) +
  geom_point(aes(color = parameter), size = 0.75) + 
               
  scale_y_log10(labels = scales::comma) +
  
  scale_color_manual(name = '', values = cbep_colors()[c(6,4)]) +

  ylab('Fecal Coliforms (CFU / 100ml)')+

  xlab('Sampling Sites Around Casco Bay') +
  expand_limits(x = 240) +  # this ensures the top dot is not cut off
  
  theme_cbep(base_size = 7) + 
  theme(legend.position = c(.2, 0.8),
        legend.text = element_text(size = 7),
        legend.key.height = unit(1, 'lines'),
        axis.text.x = element_blank(),
        axis.ticks.x = element_blank(),
        axis.text.y = element_text(size = 7),
        axis.line  = element_line(size = 0.5, 
                                  color = 'gray85'),
        panel.grid.major.y = element_line(size = 0.5, 
                                          color = 'gray85', 
                                          linetype = 2)) +
  guides(color = guide_legend(override.aes = list(lty = 0, size = 2)))
```

### Mimicing Graphic As Modified by the Graphic Designer

``` r
plt +
   geom_hline(yintercept = 32, 
             lty = 2, color = 'gray25') +
  geom_hline(yintercept = 14, 
             lty = 2, color = 'gray25') +
  
  annotate('text', x = 3, y = 40, label = '31 CFU',
           hjust = 0,
           size = 1.75, 
           color = cbep_colors()[6]) +
  annotate('text', x = 3, y = 18, label = '14 CFU',
           hjust = 0,
           size = 1.75, 
           color = cbep_colors()[4])
```

<img src="shellfish_bacteria_p90_graphics_revised_files/figure-gfm/bootstrap__graphic_1-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/stations_both_one_revised.pdf', device = cairo_pdf, 
       width = 4, height = 3)
```

### Alternate Annotations

``` r
plt +
   geom_hline(yintercept = 32, 
             lty = 2, color = 'gray25') +
  geom_hline(yintercept = 14, 
             lty = 2, color = 'gray25') +
  
  annotate('text', x = 3, y = 40, label = 'DMR P90 threshold, 31 CFU',
           hjust = 0,
           size = 1.75, 
           color = cbep_colors()[6]) +
  annotate('text', x = 3, y = 17.5, label = 'DMR Geometric Mean threshold, 14 CFU',
           hjust = 0,
           size = 1.75, 
           color = cbep_colors()[4])
```

<img src="shellfish_bacteria_p90_graphics_revised_files/figure-gfm/bootstrap_graphic_2-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/stations_both_one_revised_alt.pdf', device = cairo_pdf, 
       width = 4, height = 3)
```

## Shapes with oultines Plot

These plots look fairly poor here in the Markdown document, but the PDFs
work O.K.

``` r
plt <- ggplot(long_dat, aes(Station, value)) + 

  geom_linerange(aes(ymin = lower, ymax = upper, color = parameter),
                 alpha = 0.25,
                 size = .1) +
  geom_point(aes(fill = parameter), color = 'gray60', 
             stroke = 0.25,  size = 1, shape = 21) + 
               
  scale_y_log10(labels = scales::comma) +
  
  scale_color_manual(name = '', values = cbep_colors()[c(6,4)]) +
  scale_fill_manual(name = '', values = cbep_colors()[c(6,4)]) +

  ylab('Fecal Coliforms (CFU / 100ml)')+

  xlab('Sampling Sites Around Casco Bay') +
  expand_limits(x = 240) +  # this ensures the top dot is not cut off
  
  theme_cbep(base_size = 7) + 
  theme(legend.position = c(.2, 0.8),
        legend.text = element_text(size = 7),
        legend.key.height = unit(1, 'lines'),
        axis.text.x = element_blank(),
        axis.ticks.x = element_blank(),
        axis.text.y = element_text(size = 7),
        axis.line  = element_line(size = 0.5, 
                                  color = 'gray85'),
        panel.grid.major.y = element_line(size = 0.5, 
                                          color = 'gray85', 
                                          linetype = 2)) +
  guides(color = guide_legend(override.aes = list(lty = 0, size = 2)))
```

### Mimicing Graphic As Modified by the Graphic Designer

``` r
plt +
  geom_hline(yintercept = 32, 
             lty = 2, color = 'gray25') +
  geom_hline(yintercept = 14, 
             lty = 2, color = 'gray25') +
  
  annotate('text', x = 3, y = 40, label = '31 CFU',
           hjust = 0,
           size = 1.75, 
           color = cbep_colors()[6]) +
  annotate('text', x = 3, y = 18, label = '14 CFU',
           hjust = 0,
           size = 1.75, 
           color = cbep_colors()[4])
```

<img src="shellfish_bacteria_p90_graphics_revised_files/figure-gfm/bootstrap__graphic_3-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/stations_both_one_revised_outlines.pdf', device = cairo_pdf, 
       width = 4, height = 3)
```

### Alternate Annotations

``` r
plt +
   geom_hline(yintercept = 32, 
             lty = 2, color = 'gray25') +
  geom_hline(yintercept = 14, 
             lty = 2, color = 'gray25') +
  
  annotate('text', x = 3, y = 40, label = 'DMR P90 threshold, 31 CFU',
           hjust = 0,
           size = 1.75, 
           color = cbep_colors()[6]) +
  annotate('text', x = 3, y = 17.5, label = 'DMR Geometric Mean threshold, 14 CFU',
           hjust = 0,
           size = 1.75, 
           color = cbep_colors()[4])
```

<img src="shellfish_bacteria_p90_graphics_revised_files/figure-gfm/bootstrap_graphic_4-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/stations_both_one_revised_alt_outlines.pdf', device = cairo_pdf, 
       width = 4, height = 3)
```
