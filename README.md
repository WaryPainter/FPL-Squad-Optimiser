FPL Squad Optimiser
================

This file contains code to calculate the optimal FPL squad possible
based on the historical points obtained for each player. It is
calculated against the starting budget of 100m.

Some assumptions

1)  This only solves for 11 players, a bench of minimum cost is assumed,
    so the budget provided is 83m (17m is the minimum amount needed to
    fill the bench slots).
2)  Substitions are not taken into account as that takes the scope of
    the problem beyond linear programming.

This code can be re-run to include any updated information as it scrapes
from the FPL website

``` r
library(tidyverse)
library(fplscrapR)
library(lpSolve)
library(knitr)
```

``` r
player_info <- get_player_info()
team_ref <- read_csv('data/team_ref.csv')
```

``` r
# Setting up a function for convenience later

one_hot_encode <- function(df, column) {
  column <- enquo(column)
  
  df %>% 
    separate_rows(!!column) %>% 
    mutate(count = 1) %>% 
    pivot_wider(names_from = !!column, values_from = count, values_fill = list(count = 0))

}
```

``` r
player_info_cln <- player_info %>% 
  mutate(start_cost = (now_cost - cost_change_start)) %>% 
  select(playername, total_points, team_code, element_type, start_cost, now_cost) %>% 
  left_join(team_ref, by = c('team_code' = 'team_id'))
```

How the code works: This problem can be summarised into a series of
linear equations. The equation to be optimised is essentially a equation
where the coefficient is the points of the player with a binary variable
indicating the selection of the player.

There are a few constrains based on the rules of the game, they can all
be represented as linear equations, similarly to above but with
different coefficients. List of constraints:

1)  Only 3 players can be selected from each team.

This can be written as 20 separate equations (1 for each team) where the
coefficients are binary indicators of whether player n is in the team.

2)  As stated earlier, there is a budget constraint, this is written as
    an equation where the coefficient are the cost of each player, and
    the total cost has to be below 830 (83m in-game)

3)  There are limits on the possible formations in the game. You can
    only play 1 goalkeeper, and for the other 3 positions, there is a
    lower and upper limit. Each limit is written as one equation too.
    The code here ensures that the final output is of a valid formation.

<!-- end list -->

``` r
# Assume no subs and 83m to spend as 17 is required for bench

# Setup variables
num_gk <- 1
num_def_min <- 3

# Note: the max amount of defenders if 4 instead of 5 as only defenders can cost 4.0m at the start of a season, so there is an assumption that we have a 4.0m defender on the bench in this code

num_def_max <- 4
num_mid_min <- 2
num_mid_max <- 5
num_fwd_min <- 1
num_fwd_max <- 3


squad_max <- 11
budget <- 830
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


optimise_squad <- function(input, cost = start_cost) {

  cost <- enquo(cost)
  
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
    select(-element_type, -team_code, -solution) %>% 
    rename(
      'Player' = playername, 
      'Total Points' = total_points, 
      'Starting Cost' = start_cost,
      'Current Cost' = now_cost,
      'Team' = team_name,
      'Position' = position
    )
  
}
```

Solution 1: Optimal squad based on starting prices of
players

| Player                           | Total Points | Starting Cost | Team              | Position   |
| :------------------------------- | -----------: | ------------: | :---------------- | :--------- |
| Mathew Ryan                      |           60 |           4.5 | Brighton          | Goalkeeper |
| Ricardo Domingos Barbosa Pereira |           75 |           6.0 | Leicester City    | Defender   |
| John Lundstram                   |           75 |           4.0 | Sheffield United  | Defender   |
| Benjamin Chilwell                |           64 |           5.5 | Leicester City    | Defender   |
| Kevin De Bruyne                  |           96 |           9.5 | Manchester United | Midfielder |
| Sadio Mané                       |           94 |          11.5 | Liverpool         | Midfielder |
| Raheem Sterling                  |           79 |          12.0 | Manchester United | Midfielder |
| David Silva                      |           77 |           7.5 | Manchester United | Midfielder |
| Jamie Vardy                      |          110 |           9.0 | Leicester City    | Forward    |
| Tammy Abraham                    |           83 |           7.0 | Chelsea           | Forward    |
| Teemu Pukki                      |           74 |           6.5 | Norwich City      | Forward    |

Solution 2: Optimal squad based on current prices of players (with
starting
budget)

| Player                           | Total Points | Current Cost | Team              | Position   |
| :------------------------------- | -----------: | -----------: | :---------------- | :--------- |
| Mathew Ryan                      |           60 |          4.8 | Brighton          | Goalkeeper |
| Ricardo Domingos Barbosa Pereira |           75 |          6.4 | Leicester City    | Defender   |
| John Lundstram                   |           75 |          5.1 | Sheffield United  | Defender   |
| Andrew Robertson                 |           65 |          7.1 | Liverpool         | Defender   |
| Çaglar Söyüncü                   |           63 |          5.1 | Leicester City    | Defender   |
| Kevin De Bruyne                  |           96 |         10.3 | Manchester United | Midfielder |
| Sadio Mané                       |           94 |         12.2 | Liverpool         | Midfielder |
| David Silva                      |           77 |          7.6 | Manchester United | Midfielder |
| Jamie Vardy                      |          110 |          9.9 | Leicester City    | Forward    |
| Tammy Abraham                    |           83 |          7.9 | Chelsea           | Forward    |
| Teemu Pukki                      |           74 |          6.6 | Norwich City      | Forward    |
