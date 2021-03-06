Analysis of in vivo FRET recordings
================
Ryan Martin

### Required libraries

``` r
library(tidyverse)
library(modelr)
library(readxl)
library(here)
```

### Set recording interval time

``` r
interval_time <- 2
```

### Set length of baseline reading

``` r
baseline_time <- 10
```

### List of files in directory

``` r
files <- list.files(path = here(), pattern = "*.xls", full.names = T, recursive = T)
```

### Import CFP and YFP channel readings and name elements of list

``` r
all_cfp <- lapply(files, read_xls, sheet = 4)
names(all_cfp) <- gsub(".*/(.*)\\..*", "\\1", files)

all_yfp <- lapply(files, read_xls, sheet = 5)
names(all_yfp) <- gsub(".*/(.*)\\..*", "\\1", files)
```

### Calculate CFP and YFP mean fluorescence at each time point for the control files

``` r
gather_fluorescence <- function(n, z) {
  n %>%
    pivot_longer(cols = 2:ncol(n), names_to = 'Event', values_to = 'Fluorescence') %>%
    group_by(Time) %>%
    summarise(Fluorescence = mean(Fluorescence)) 
}

control_cfp <- map(all_cfp[grep("Control", files)], gather_fluorescence) 
control_yfp <- map(all_yfp[grep("Control", files)], gather_fluorescence)
```

### Calculate CFP readings from control animals and do linear regression

``` r
control_cfp_linear <- control_cfp %>%
  reduce(left_join, by = "Time") %>%
  pivot_longer(2:ncol(.), names_to = 'Fluorescence', values_to = 'value') %>%
  mutate(Time = as.numeric(factor(Time)) * interval_time)

linear_cfp <- lm(value ~ Time, data = control_cfp_linear)

background_cfp <- select(control_cfp_linear, Time) %>%
  distinct() %>%
  add_predictions(linear_cfp)
```

### Average YFP readings from control animals and do linear regression

``` r
control_yfp_linear <- control_yfp %>%
  reduce(left_join, by = "Time") %>%
  pivot_longer(2:ncol(.), names_to = 'Fluorescence', values_to = 'value') %>%
  mutate(Time = as.numeric(factor(Time)) * interval_time)

linear_yfp <- lm(value ~ Time, data = control_yfp_linear)

background_yfp <- select(control_yfp_linear, Time) %>%
  distinct() %>%
  add_predictions(linear_yfp)
```

### Extract datasets of treated animals in the list and name them

``` r
treat_cfp <- map(all_cfp[grep("Control", files, invert = T)], gather_fluorescence)
treat_yfp <- map(all_yfp[grep("Control", files, invert = T)], gather_fluorescence)
```

### Adjust CFP and YFP but substracting off predicted values from control linear regression

``` r
adj_cfp <- map(treat_cfp, function(x) {
  x %>% 
    mutate(Time = as.numeric(factor(Time)) * interval_time) %>%
    left_join(background_cfp, by = "Time")  %>%
    mutate(fluorescence_adj = Fluorescence - pred) %>%
    select(Time, fluorescence_adj)
})

adj_yfp <- map(treat_yfp, function(x) {
  x %>%
    mutate(Time = as.numeric(factor(Time)) * interval_time) %>%
    left_join(background_yfp, by = "Time")  %>%
    mutate(fluorescence_adj = Fluorescence - pred) %>%
    select(Time, fluorescence_adj)
})
```

### Merge YFP and CFP channels for each animal

``` r
merged_fluorescence <- list()

for(i in 1:length(adj_cfp)) {
  a <- adj_cfp[i][[1]]
  b <- adj_yfp[i][[1]]
  merged_fluorescence[i][[1]] <- left_join(a, b, by = "Time", suffix = c('.cfp', '.yfp'))
}
```

### Obtain baseline fluorescence for each animal, adjust baseline\_fret function for time of baseline reading

``` r
names(merged_fluorescence) <- names(adj_cfp)
baseline_fret <- lapply(merged_fluorescence, function(x) {
  x %>% 
    mutate(fret = fluorescence_adj.yfp / fluorescence_adj.cfp) %>%
    filter(Time <= 10) %>%
    summarise(baseline = mean(fret))
})
```

### Obtain FRET ratios for each animal

``` r
fret_ratio <- lapply(merged_fluorescence, function(x) {
  x %>% 
    mutate(fret = fluorescence_adj.yfp / fluorescence_adj.cfp) %>%
    select(Time, fret)
  })
```

### Merge FRET ratios with the baseline FRET for each animal

``` r
delta_fret <- list()

for(i in 1:length(fret_ratio)) {
  a <- fret_ratio[i][[1]]
  b <- baseline_fret[i][[1]]
  delta_fret[i][[1]] <- cbind(a, b)
}
```

### Calculate dFRET and %dFRET/FRET

``` r
dfret <- function(x) {
  x %>%
    mutate(dfret = fret - baseline,
           df_fret = dfret / fret * 100) %>%
    select(Time, dfret, df_fret)
}

fret_final <- lapply(delta_fret, dfret)
names(fret_final) <- names(adj_cfp)
```

### Collapse list into a matrix maintaining which dataset the values came from

``` r
fret_matrix <- plyr::ldply(fret_final, .id = 'names')
```

### Quick plot of all the animals in the current directory

``` r
ggplot(fret_matrix, aes(x = Time, y = df_fret, color = names)) + 
  geom_line() 
```
