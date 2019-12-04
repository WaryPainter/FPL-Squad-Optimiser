FPL Squad Optimiser
================

``` r
library(tidyverse)
```

    ## -- Attaching packages --------------------------------------------------------------------------------------- tidyverse 1.3.0 --

    ## v ggplot2 3.2.1     v purrr   0.3.3
    ## v tibble  2.1.3     v dplyr   0.8.3
    ## v tidyr   1.0.0     v stringr 1.4.0
    ## v readr   1.3.1     v forcats 0.4.0

    ## -- Conflicts ------------------------------------------------------------------------------------------ tidyverse_conflicts() --
    ## x dplyr::filter() masks stats::filter()
    ## x dplyr::lag()    masks stats::lag()

``` r
library(fplscrapR)
library(lpSolve)
```

``` r
player_info <- get_player_info()
#player_info <- read_csv('player_info.csv')
team_ref <- read_csv('data/team_ref.csv')
```

    ## Parsed with column specification:
    ## cols(
    ##   team_id = col_double(),
    ##   team_name = col_character()
    ## )

``` r
# team_ids <- unique(player_info$team_code)
# team_names <- c('Arsenal', 'Aston Villa', 'Bournemouth', 'Brighton', 'Burnley', 'Chelsea', 'Crystal Palace',
#                 'Everton', 'Leicester City', 'Liverpool', 'Manchester United', 'Manchester City', 'Newcastle United',
#                 'Norwich City', 'Sheffield United', 'Southampton', 'Tottenham Hotspur', 'Watford', 'West Ham United',
#                 'Wolverhampton Wanderers')
# 
# team_ref <- tibble(team_id = team_ids, team_name = team_names)
# 
# write_csv(team_ref, 'data/team_ref.csv')
```

``` r
player_info_cln <- player_info %>% 
  mutate(start_cost = (now_cost - cost_change_start)) %>% 
  select(playername, total_points, team_code, element_type, start_cost, now_cost) %>% 
  left_join(team_ref, by = c('team_code' = 'team_id'))

one_hot_encode <- function(df, column) {
  column <- enquo(column)
  
  df %>% 
    separate_rows(!!column) %>% 
    mutate(count = 1) %>% 
    pivot_wider(names_from = !!column, values_from = count, values_fill = list(count = 0))

}
```

``` r
# Assume no subs and 87m to spend as 17 is required for bench

# Setup variables
num_gk <- 1
num_def_min <- 3
num_def_max <- 5
num_mid_min <- 2
num_mid_max <- 5
num_fwd_min <- 1
num_fwd_max <- 3


squad_max <- 11
budget <- 870
```

``` r
input <- player_info_cln %>% 
  mutate(position = case_when(
    element_type == 1 ~ 'goalkeeper',
    element_type == 2 ~ 'defender',
    element_type == 3 ~ 'midfielder',
    element_type == 4 ~ 'forward'
  )) %>% 
  select(-element_type) %>% 
  one_hot_encode(position) %>% 
  one_hot_encode(team_code)
  
team_const <- input %>% 
  select(which(!is.na(as.numeric(colnames(input))))) %>% 
  as.matrix() %>% 
  as.vector()
```

    ## Warning in which(!is.na(as.numeric(colnames(input)))): NAs introduced by
    ## coercion

``` r
optimise_squad <- function(input, cost) {

  cost <- enquo(cost)
  
  #if(!(cost %in% c(sym(now_cost), sym(start_cost)))) stop('use a proper cost variable')
  
  goalkeeper <- input$goalkeeper
  defender <- input$defender
  midfielder <- input$midfielder
  forward <- input$forward
  team_vector <- rep(1, nrow(player_info))
  player_cost <- pull(input, !!cost)
  
  
  
  const_vector <- c(goalkeeper, defender, defender, midfielder, midfielder, forward, forward, team_vector, player_cost, team_const)
  
  objective <- input$total_points
  
  const_mat <- matrix(
    data = const_vector,
    nrow = 29,
    byrow = TRUE
  )
  
  const_dir <- c('=', rep(c('>=', '<='), 3), '=', rep('<=', 21))
  const_rhs <-  c(num_gk, num_def_min, num_def_max, num_mid_min, num_mid_max, num_fwd_min, num_fwd_max, squad_max, budget, rep(3, 20))
  
  result <- lp(
    direction = 'max', 
    objective.in = objective,
    const.mat = const_mat,
    const.dir = const_dir,
    const.rhs = const_rhs,
    all.bin = TRUE,
    all.int = TRUE
  )
  
  
  player_info_cln %>% 
    mutate(start_cost = start_cost /10, now_cost = now_cost /10) %>% 
    mutate(position = case_when(
      element_type == 1 ~ 'Goalkeeper',
      element_type == 2 ~ 'Defender',
      element_type == 3 ~ 'Midfielder',
      element_type == 4 ~ 'Forward'
    )) %>% 
    mutate(solution = result$solution) %>% 
    filter(solution == 1) %>% 
    arrange(element_type, desc(total_points)) %>% 
    select(-element_type, -team_code, -solution)
  
}
```

``` r
now_solution <- optimise_squad(input, now_cost)
start_solution <- optimise_squad(input, start_cost) 
```
