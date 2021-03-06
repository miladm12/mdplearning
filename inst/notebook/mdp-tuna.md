Carl Boettiger  





```r
library("mdplearning")
library("appl")
library("ggplot2")
library("tidyr")
library("purrr")
library("dplyr")
#knitr::opts_chunk$set(cache = TRUE)
```


```r
## choose the state/action/obs space
#states <- seq(0,1.2, length=200)  

#states <- c(seq(0,0.12, length=30), seq(0.12, 1.2, length=200))   # Vector of actions: harvest
p <- 1.5
states <- seq(0, 1.2^(1/p), len=100)^p # Vector of all possible states

actions <- states
obs <- states
K <- 0.9903371
r <- 0.05699246
sigma_g <- 0.01720091
discount <- 0.99

vars <- expand.grid(r = seq(0.025, 0.2, by =0.025), sigma_m = 0.3)
fixed <- data.frame(model = "ricker", sigma_g = sigma_g, discount = discount, K = K, C = NA)
models <- data.frame(vars, fixed)

## Usual assumption at the moment for reward fn
reward_fn <- function(x,h) pmin(x,h)
```



```r
true_m <- appl::fisheries_matrices(states, actions, obs, reward_fn, f =  appl:::ricker(r, K),
                     sigma_g = sigma_g, sigma_m  = 0.3)


tuna_models <- parallel::mclapply(1:dim(models)[1], function(i){
  f <- switch(models[i, "model"],
              allen = appl:::allen(models[i, "r"], models[i, "K"], models[i, "C"]),
              ricker = appl:::ricker(models[i, "r"], models[i, "K"])
  )

  appl::fisheries_matrices(states, actions, obs, reward_fn, f = f,
                     sigma_g = models[i, "sigma_g"], sigma_m  = models[i, "sigma_m"])


}, mc.cores = parallel::detectCores())
```



```r
models <- tuna_models
reward <- models[[1]]$reward
transitions <- lapply(models, `[[`, "transition")
observation <- models[[1]]$observation
```


## MDP Policies



```r
unif <- mdp_compute_policy(transitions, reward, discount)
prior <- numeric(length(models))
prior[1] <- 1
low <- mdp_compute_policy(transitions, reward, discount, prior)
prior <- numeric(length(models))
prior[2] <- 1
true <- mdp_compute_policy(transitions, reward, discount, prior)

bind_rows(unif = unif, low = low, true = true, .id = "model") %>%
  ggplot(aes(states[state], states[state] - actions[policy], col = model)) + geom_line()
```

![](mdp-tuna_files/figure-html/unnamed-chunk-5-1.png)<!-- -->



## Hindcast: Historical catch and stock


```r
set.seed(123)
data("scaled_data")
y <- sapply(scaled_data$y, function(y) which.min(abs(states - y)))
a <- sapply(scaled_data$a, function(a) which.min(abs(actions - a)))
Tmax <- length(y)

data("bluefin_tuna")
to_mt <- max(bluefin_tuna$total) # 1178363 # scaling factor for data
states_mt <- to_mt * states
actions_mt <- to_mt * actions
year <- 1952:2009
future <- 2009:2067
```



```r
mdp_hindcast <- mdp_historical(transitions, reward, discount, state = y, action = a)
```


### Merge and plot resulting optimal solutions


```r
mdp_hindcast$df %>%
mutate("actual catch" = actions_mt[action], 
       "estimated stock" = states_mt[state], 
       optimal = actions_mt[optimal], 
       time = year[time]) %>%
       select(-state, -action) %>%
gather(variable, stock, -time) %>% 
ggplot(aes(time, stock, color = variable)) + geom_line(lwd=1) #  + geom_point()
```

```
## Warning: Removed 1 rows containing missing values (geom_path).
```

![](mdp-tuna_files/figure-html/unnamed-chunk-8-1.png)<!-- -->


### Final beliefs

Show the final belief over models for pomdp and mdp:



```r
barplot(as.numeric(mdp_hindcast$posterior[Tmax,]))
```

![](mdp-tuna_files/figure-html/unnamed-chunk-9-1.png)<!-- -->

## Compare rates of learning


```r
# delta function for true model distribution
h_star = array(0,dim = length(models)) 
h_star[2] = 1
## Fn for the base-2 KL divergence from true model, in a friendly format
kl2 <- function(value) seewave::kl.dist(value, h_star, base = 2)[[2]]

mdp_hindcast$posterior %>%
mutate(time = year[1:Tmax], rep = 1) %>%
gather(model, value, -time, -rep) %>%
group_by(time, rep) %>% 
summarise(kl = kl2(value)) %>%
ggplot(aes(time, kl)) + 
  stat_summary(geom="line", fun.y = mean, lwd = 1)
```

![](mdp-tuna_files/figure-html/unnamed-chunk-10-1.png)<!-- -->




## Forecast simulations under MDP-learning

All forecasts start from final stock, go forward an equal length of time:


```r
#x0 <- which.min(abs(1 - states)) # Initial stock, for hindcasts
x0 <- y[length(y)] # Final stock, e.g. forcasts, = 9

Tmax <- length(y)
set.seed(123)
```

We simulate replicates under MDP learning (with observation uncertainty):


```r
Tmax <- 150
future <- 2009:(2009+Tmax)
mdp_forecast <- 
pomdpplus::plus_replicate(25, 
               mdp_learning(transition = transitions, reward = reward, 
                            model_prior = as.numeric(mdp_hindcast$posterior[length(y),]),
                            discount = discount, x0 = x0,  Tmax = Tmax,
                            true_transition = true_m$transition, 
                            observation = observation),
               mc.cores = parallel::detectCores())
```

## Compare forecasts


```r
historical <- bluefin_tuna[c("tsyear", "total", "catch_landings")] %>% 
  rename(time = tsyear, state = total, action = catch_landings)

bind_rows(
  historical = historical, 
  mdp = mdp_forecast$df %>% 
    select(-value, -obs) %>% 
    mutate(state = states_mt[state], action = actions_mt[action], time = future[time]),
  .id = "method") %>%
rename("catch (MT)" = action, "stock (MT)" = state) %>%  
gather(variable, stock, -time, -rep, -method) %>%
ggplot(aes(time, stock)) + 
  geom_line(aes(group = interaction(rep,method), color = method), alpha=0.1) +
  stat_summary(aes(color = method), geom="line", fun.y = mean, lwd=1) +
  stat_summary(aes(fill = method), geom="ribbon", fun.data = mean_sdl, fun.args = list(mult=1), alpha = 0.25) + 
  facet_wrap(~variable, ncol = 1, scales = "free_y")
```

![](mdp-tuna_files/figure-html/unnamed-chunk-13-1.png)<!-- -->




## MDP Planning

#### Planning only solution given posterior model belief (e.g. almost certain knowledge of model)


```r
model_prior <- as.numeric(mdp_hindcast$posterior[length(y),])
planning <- mdp_compute_policy(transitions, reward, discount, model_prior)

df <- mdp_planning(true_m$transition, reward, discount, x0 = x0, Tmax = Tmax, 
              policy = planning$policy, observation = observation)
```



```r
df %>% 
  select(-value) %>%
  mutate(state = states[state], obs = states[obs], action = actions[action]) %>% 
  gather(series, stock, -time) %>% 
  ggplot(aes(time, stock, color = series)) + geom_line()
```

![](mdp-tuna_files/figure-html/unnamed-chunk-15-1.png)<!-- -->




#### Det policy


```r
f <- appl:::ricker(r, K)
S_star <- optimize(function(x) x / discount - f(x,0), c(min(states),max(states)))$minimum
h <- pmax(states - S_star,  0)
policy <- sapply(h, function(h) which.min((abs(h - actions))))
det <- data.frame(policy, value = 1:length(states), state = 1:length(states))


df <- mdp_planning(true_m$transition, reward, discount, x0 = x0, Tmax = Tmax, 
              policy = det$policy, observation = observation)
```



```r
df %>% 
  select(-value) %>%
  mutate(state = states[state], obs = states[obs], action = actions[action]) %>% 
  gather(series, stock, -time) %>% 
  ggplot(aes(time, stock, color = series)) + geom_line()
```

![](mdp-tuna_files/figure-html/unnamed-chunk-17-1.png)<!-- -->

