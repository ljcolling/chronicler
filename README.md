
<!-- README.md is generated from README.Rmd. Please edit that file -->

# chronicler

<!-- badges: start -->
<!-- badges: end -->

Easily add logs to your functions.

## Installation

You can install the development version from
[GitHub](https://github.com/) with:

``` r
# install.packages("devtools")
devtools::install_github("b-rodrigues/chronicler")
```

## Introduction

{chronicler} allows you to decorate functions to make them provide
enhanced output:

``` r
library(chronicler)

r_sqrt <- record(sqrt)

a <- r_sqrt(1:5)
```

Object `a` is now an object of class `chronicle`. The value of the
`sqrt()` function applied to its arguments can be obtained using
`pick()`:

``` r
pick(a, "value")
#> [1] 1.000000 1.414214 1.732051 2.000000 2.236068
```

A log also gets generated and can be read using `read_log()`:

``` r
read_log(a)
#> [1] "Complete log:"                                      
#> [2] "✔ sqrt(1:5) ran successfully at 2022-04-01 16:04:28"
#> [3] "Total running time: 8.17775726318359e-05 secs"
```

This is especially useful for objects that get created using multiple
calls:

``` r
r_sqrt <- record(sqrt)
r_exp <- record(exp)
r_mean <- record(mean)

b <- 1:10 |>
  r_sqrt() |>
  bind_record(r_exp) |>
  bind_record(r_mean)
```

``` r
read_log(b)
#> [1] "Complete log:"                                           
#> [2] "✔ sqrt(1:10) ran successfully at 2022-04-01 16:04:28"    
#> [3] "✔ exp(.c$value) ran successfully at 2022-04-01 16:04:28" 
#> [4] "✔ mean(.c$value) ran successfully at 2022-04-01 16:04:28"
#> [5] "Total running time: 0.0211019515991211 secs"

pick(b, "value")
#> [1] 11.55345
```

## Composing decorated functions

`bind_record()` is used to pass the output from one decorated function
to the next.

`record()` works with any function:

``` r
library(dplyr)

r_group_by <- record(group_by)
r_select <- record(select)
r_summarise <- record(summarise)
r_filter <- record(filter)

output <- starwars %>%
  r_select(height, mass, species, sex) %>%
  bind_record(r_group_by, species, sex) %>%
  bind_record(r_filter, sex != "male") %>%
  bind_record(r_summarise,
              mass = mean(mass, na.rm = TRUE)
              )
```

``` r
read_log(output)
#> [1] "Complete log:"                                                                         
#> [2] "✔ select(.,height,mass,species,sex) ran successfully at 2022-04-01 16:04:28"           
#> [3] "✔ group_by(.c$value,species,sex) ran successfully at 2022-04-01 16:04:28"              
#> [4] "✔ filter(.c$value,sex != \"male\") ran successfully at 2022-04-01 16:04:28"            
#> [5] "✔ summarise(.c$value,mean(mass, na.rm = TRUE)) ran successfully at 2022-04-01 16:04:28"
#> [6] "Total running time: 0.0484888553619385 secs"
```

The value can then be accessed and worked on as usual using `pick()`:

``` r
pick(output, "value")
#> # A tibble: 9 × 3
#> # Groups:   species [9]
#>   species    sex              mass
#>   <chr>      <chr>           <dbl>
#> 1 Clawdite   female           55  
#> 2 Droid      none             69.8
#> 3 Human      female           56.3
#> 4 Hutt       hermaphroditic 1358  
#> 5 Kaminoan   female          NaN  
#> 6 Mirialan   female           53.1
#> 7 Tholothian female           50  
#> 8 Togruta    female           57  
#> 9 Twi'lek    female           55
```

This package also ships with a dedicated pipe, `%>=%` which you can use
instead of `bind_record()`:

``` r
output_pipe <- starwars %>%
  r_select(height, mass, species, sex) %>=%
  r_group_by(species, sex) %>=%
  r_filter(sex != "male") %>=%
  r_summarise(mass = mean(mass, na.rm = TRUE))
```

``` r
pick(output_pipe, "value")
#> # A tibble: 9 × 3
#> # Groups:   species [9]
#>   species    sex              mass
#>   <chr>      <chr>           <dbl>
#> 1 Clawdite   female           55  
#> 2 Droid      none             69.8
#> 3 Human      female           56.3
#> 4 Hutt       hermaphroditic 1358  
#> 5 Kaminoan   female          NaN  
#> 6 Mirialan   female           53.1
#> 7 Tholothian female           50  
#> 8 Togruta    female           57  
#> 9 Twi'lek    female           55
```

Objects of class `chronicle` have their own print method:

``` r
output_pipe
#> ✔ Value computed successfully:
#> ---------------
#> # A tibble: 9 × 3
#> # Groups:   species [9]
#>   species    sex              mass
#>   <chr>      <chr>           <dbl>
#> 1 Clawdite   female           55  
#> 2 Droid      none             69.8
#> 3 Human      female           56.3
#> 4 Hutt       hermaphroditic 1358  
#> 5 Kaminoan   female          NaN  
#> 6 Mirialan   female           53.1
#> 7 Tholothian female           50  
#> 8 Togruta    female           57  
#> 9 Twi'lek    female           55  
#> 
#> ---------------
#> This is an object of type `chronicle`.
#> Retrieve the value of this object with pick(.c, "value").
#> To read the log of this object, call read_log().
```

## Condition handling

By default, errors and warnings get caught and composed in the log:

``` r
errord_output <- starwars %>%
  r_select(height, mass, species, sex) %>=% 
  r_group_by(species, sx) %>=% # typo, "sx" instead of "sex"
  r_filter(sex != "male") %>=%
  r_summarise(mass = mean(mass, na.rm = TRUE))
```

``` r
errord_output
#> ✖ Value computed unsuccessfully:
#> ---------------
#> [1] NA
#> 
#> ---------------
#> This is an object of type `chronicle`.
#> Retrieve the value of this object with pick(.c, "value").
#> To read the log of this object, call read_log().
```

Reading the log tells you which function failed, and with which error
message:

``` r
read_log(errord_output)
#> [1] "Complete log:"                                                                                                                                                                                    
#> [2] "✔ select(.,height,mass,species,sex) ran successfully at 2022-04-01 16:04:28"                                                                                                                      
#> [3] "✖ group_by(.c$value,species,sx) ran unsuccessfully with following exception: Must group by variables found in `.data`.\n✖ Column `sx` is not found. at 2022-04-01 16:04:28"                       
#> [4] "✖ filter(.c$value,sex != \"male\") ran unsuccessfully with following exception: no applicable method for 'filter' applied to an object of class \"logical\" at 2022-04-01 16:04:28"               
#> [5] "✖ summarise(.c$value,mean(mass, na.rm = TRUE)) ran unsuccessfully with following exception: no applicable method for 'summarise' applied to an object of class \"logical\" at 2022-04-01 16:04:28"
#> [6] "Total running time: 0.0922913551330566 secs"
```

It is also possible to only capture errors, or catpure errors, warnings
and messages using the `strict` parameter of `record()`

``` r
# Only errors:

r_sqrt <- record(sqrt, strict = 1)

r_sqrt(-10) |>
  read_log()
#> Warning in .f(...): NaNs produced
#> [1] "Complete log:"                                                                     
#> [2] "✖ sqrt(-10) ran unsuccessfully with following exception: NA at 2022-04-01 16:04:28"
#> [3] "Total running time: 0.000174283981323242 secs"

# Errors and warnings:

r_sqrt <- record(sqrt, strict = 2)

r_sqrt(-10) |>
  read_log()
#> [1] "Complete log:"                                                                                
#> [2] "✖ sqrt(-10) ran unsuccessfully with following exception: NaNs produced at 2022-04-01 16:04:28"
#> [3] "Total running time: 0.00021052360534668 secs"

# Errors, warnings and messages

my_f <- function(x){
  message("this is a message")
  10
}

record(my_f, strict = 3)(10) |>
                         read_log()
#> [1] "Complete log:"                                                                                     
#> [2] "✖ my_f(10) ran unsuccessfully with following exception: this is a message\n at 2022-04-01 16:04:28"
#> [3] "Total running time: 0.000233650207519531 secs"
```

## Advanced logging

You can provide a function to `record()`, which will be evaluated on the
output. This makes it possible to, for example, monitor the size of a
data frame throughout the pipeline:

``` r
r_group_by <- record(group_by)
r_select <- record(select, .g = dim)
r_summarise <- record(summarise, .g = dim)
r_filter <- record(filter, .g = dim)

output_pipe <- starwars %>%
  r_select(height, mass, species, sex) %>=%
  r_group_by(species, sex) %>=%
  r_filter(sex != "male") %>=%
  r_summarise(mass = mean(mass, na.rm = TRUE))
```

The `$log_df` element of a `chronicle` object contains detailled
information:

``` r
pick(output_pipe, "log_df")
#> # A tibble: 4 × 8
#>   outcome   `function` arguments message start_time          end_time           
#>   <chr>     <chr>      <chr>     <chr>   <dttm>              <dttm>             
#> 1 ✔ Success select     ".,heigh… NA      2022-04-01 16:04:28 2022-04-01 16:04:28
#> 2 ✔ Success group_by   ".c$valu… NA      2022-04-01 16:04:28 2022-04-01 16:04:28
#> 3 ✔ Success filter     ".c$valu… NA      2022-04-01 16:04:28 2022-04-01 16:04:28
#> 4 ✔ Success summarise  ".c$valu… NA      2022-04-01 16:04:28 2022-04-01 16:04:28
#> # … with 2 more variables: run_time <drtn>, g <list>
```

It is thus possible to take a look at the output of the function
provided (`dim()`):

``` r
as.data.frame(output_pipe$log_df[, c("function", "g")])
#>    function     g
#> 1    select 87, 4
#> 2  group_by    NA
#> 3    filter 23, 4
#> 4 summarise  9, 3
```

We can see that the dimension of the dataframe was (87, 4) after the
call to `select()`, (23, 4) after the call to `filter()` and finally (9,
3) after the call to `summarise()`.

## Thanks

I’d like to thank [armcn](https://github.com/armcn),
[Kupac](https://github.com/Kupac) for their blog posts
([here](https://kupac.gitlab.io/biofunctor/2019/05/25/maybe-monad-in-r/))
and packages ([maybe](https://armcn.github.io/maybe/)) which inspired me
to build this package.
