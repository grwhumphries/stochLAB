
<!-- README.md is generated from README.Rmd. Please edit that file -->

# stochLAB <img src='man/figures/logo.png' align="right" height="139" />

<!-- badges: start -->
<!-- badges: end -->

`{stochLAB}` is a tool to run Collision Risk Models (CRMs) for seabirds
on offshore wind farms.

## Overview

The `{stochLAB}` package is an adaptation of the [R
code](https://data.marine.gov.scot/dataset/developing-avian-collision-risk-model-incorporate-variability-and-uncertainty-r-code-0)
developed by [Masden
(2015)](https://data.marine.gov.scot/dataset/developing-avian-collision-risk-model-incorporate-variability-and-uncertainty)
to incorporate variability and uncertainty in the avian collision risk
model originally developed by [Band
(2012)](https://www.bto.org/sites/default/files/u28/downloads/Projects/Final_Report_SOSS02_Band1ModelGuidance.pdf).

Code developed under `{stochLAB}` substantially re-factored and
re-structured Masden’s (heavily script-based) implementation into a
user-friendly, streamlined, well documented and easily distributed tool.
Furthermore, the package lays down the code infrastructure for easier
incorporation of new functionality, e.g. extra parameter sampling
features, model expansions, etc.

In addition, previous code underpinning core calculations for the
extended model has been replaced by an alternative approach, resulting
in significant gains in computational speed - over 30x faster than
Masden’s code. This optimization is particularly beneficial under a
stochastic context, when core calculations are called repeatedly during
simulations.

For a more detailed overview type `?stochLAB`, once installed!

## Installation

<!-- You can install the released version of stochLAB from [CRAN](https://CRAN.R-project.org) with: -->
<!-- ``` r -->
<!-- install.packages("stochLAB") -->
<!-- ``` -->

You can install the development version with:

``` r
# install.packages("devtools")
devtools::install_github("HiDef-Aerial-Surveying/stochLAB")
```

## Examples

### Simple example

This is a basic example of running the stochastic collision model for
one seabird species and one turbine/wind-farm scenario, with fictional
input parameter data.

``` r
library(stochLAB)

# ------------------------------------------------------
# Setting some of the required inputs upfront

b_dens <- data.frame(
  month = month.abb,
  mean = runif(12, 0.8, 1.5),
  sd = runif(12, 0.2, 0.3))

# Generic FHD bootstraps for one species, from Johnson et al (2014)
fhd_boots <- generic_fhd_bootstraps[[1]]

# wind speed vs rotation speed vs pitch
wind_rtn_ptch <- data.frame(
  wind_speed = seq_len(30),
  rtn_speed = 10/(30:1),
  bld_pitch = c(rep(90, 4), rep(0, 8), 5:22))

# wind availability
windavb <- data.frame(
  month = month.abb,
  pctg = runif(12, 85, 98))

# maintenance downtime
dwntm <- data.frame(
  month = month.abb,
  mean = runif(12, 6, 10),
  sd = rep(2, 12))

# seasons specification
seas_dt <- data.frame(
  season_id = c("a", "b", "c"),
  start_month = c("Jan", "May", "Oct"), end_month = c("Apr", "Sep", "Dec"))

# ----------------------------------------------------------
# Run stochastic CRM, treating rotor radius, air gap and
# blade width as fixed parameters (i.e. not stochastic)

stoch_crm(
  model_options = c(1, 2, 3),
  n_iter = 1000,
  flt_speed_pars = data.frame(mean = 7.26, sd = 1.5),
  body_lt_pars = data.frame(mean = 0.39, sd = 0.005),
  wing_span_pars = data.frame(mean = 1.08, sd = 0.04),
  avoid_bsc_pars = data.frame(mean = 0.99, sd = 0.001),
  avoid_ext_pars = data.frame(mean = 0.96, sd = 0.002),
  noct_act_pars = data.frame(mean = 0.033, sd = 0.005),
  prop_crh_pars = data.frame(mean = 0.06, sd = 0.009),
  bird_dens_opt = "tnorm",
  bird_dens_dt = b_dens,
  flight_type = "flapping",
  prop_upwind = 0.5,
  gen_fhd_boots = fhd_boots,
  n_blades = 3,
  rtr_radius_pars = data.frame(mean = 80, sd = 0), # sd = 0, rotor radius is fixed
  air_gap_pars = data.frame(mean = 36, sd = 0),    # sd = 0, air gap is fixed
  bld_width_pars = data.frame(mean = 8, sd = 0),   # sd = 0, blade width is fixed
  rtn_pitch_opt = "windSpeedReltn",
  windspd_pars = data.frame(mean = 7.74, sd = 3),
  rtn_pitch_windspd_dt = wind_rtn_ptch,
  trb_wind_avbl = windavb,
  trb_downtime_pars = dwntm,
  wf_n_trbs = 200,
  wf_width = 15,
  wf_latitude = 56.9,
  tidal_offset = 2.5,
  lrg_arr_corr = TRUE,
  verbose = TRUE,
  seed = 1234,
  out_format = "summaries",
  out_sampled_pars = TRUE,
  out_period = "seasons",
  season_specs = seas_dt,
  log_file = paste0(getwd(), "scrm_example.log")
)
```

### Multiscenario example

This is an example usage of `stoch_crm()` for multiple scenarios. The
aim is two-fold:

1.  Suggest how input parameter datasets used in the previous
    implementation can be reshaped to fit `stoch_crm()`’s interface.
    Suggested code is also relevant in the context of multiple scenarios
    applications, since the wide tabular structure of these datasets is
    likely the favoured format for users to compile input parameters
    under different scenarios.

2.  Propose a functional programming framework to run `stoch_crm()` for
    multiple species and wind-farm/turbines features.

Please note the example runs on fictional data.

``` r
library(stochLAB)

# --------------------------------------------------------- #
# ----      Reshaping into list-column data frames       ----
# --------------------------------------------------------- #
#
# Here embracing list-columns tibbles, but lists could be used instead

# --- bird features
bird_pars <- bird_pars_wide_example %>%
  dplyr::relocate(Flight, .after = dplyr::last_col()) %>%
  tidyr::pivot_longer(AvoidanceBasic:Prop_CRH_ObsSD) %>%
  dplyr::mutate(
    par = dplyr::if_else(grepl("SD|sd|Sd", name), "sd", "mean"),
    feature = gsub("SD|sd|Sd","", name)) %>%
  dplyr::select(-name) %>%
  tidyr::pivot_wider(names_from = par, values_from = value) %>%
  tidyr::nest(pars = c(mean, sd)) %>%
  tidyr::pivot_wider(names_from = feature, values_from = pars) %>%
  tibble::add_column(prop_upwind = 0.5)

# --- bird densities: provided as mean and sd Parameters for Truncated Normal lower
# bounded at 0
dens_pars <- dens_tnorm_wide_example %>%
  tibble::add_column(
    dens_opt = rep("tnorm", nrow(.)),
    .after = 1) %>%
  tidyr::pivot_longer(Jan:DecSD) %>%
  dplyr::mutate(
    par = dplyr::if_else(grepl("SD|sd|Sd", name), "sd", "mean"),
    month = gsub("SD|sd|Sd","", name)) %>%
  dplyr::select(-name) %>%
  tidyr::pivot_wider(names_from = par, values_from = value) %>%
  tidyr::nest(mth_dens = c(month, mean, sd))

# --- FHD data from Johnson et al (2014) for the species under analysis
gen_fhd_boots <- generic_fhd_bootstraps[bird_pars$Species]

# --- seasons definitions (made up)
season_dt <- list(
  Arctic_Tern = data.frame(
    season_id = c("breeding", "feeding", "migrating"),
    start_month = c("May", "Sep", "Jan"),
    end_month = c("Aug", "Dec", "Apr")),
  Black_headed_Gull = data.frame(
    season_id = c("breeding", "feeding", "migrating"),
    start_month = c("Jan", "May", "Oct"),
    end_month = c("Apr", "Sep", "Dec")),
  Black_legged_Kittiwake = data.frame(
    season_id = c("breeding", "feeding", "migrating"),
    start_month = c("Dec", "Mar", "Sep"),
    end_month = c("Feb", "Aug", "Nov")))

# --- turbine parameters
## address operation parameters first
trb_opr_pars <- turb_pars_wide_example %>%
  dplyr::select(TurbineModel, JanOp:DecOpSD) %>%
  tidyr::pivot_longer(JanOp:DecOpSD) %>%
  dplyr::mutate(
    month = substr(name, 1, 3),
    par = dplyr::case_when(
      grepl("SD|sd|Sd", name) ~ "sd",
      grepl("Mean|MEAN|mean", name) ~ "mean",
      TRUE ~ "pctg"
    )) %>%
  dplyr::select(-name) %>%
  tidyr::pivot_wider(names_from = par, values_from = value) %>%
  tidyr::nest(
    wind_avbl = c(month, pctg),
    trb_dwntm = c(month, mean, sd))

## address turbine features and subsequently merge operation parameters
trb_pars <- turb_pars_wide_example %>%
  dplyr::select(TurbineModel:windSpeedSD ) %>%
  dplyr::relocate(RotorSpeedAndPitch_SimOption, .after = 1) %>%
  tidyr::pivot_longer(RotorRadius:windSpeedSD) %>%
  dplyr::mutate(
    par = dplyr::if_else(grepl("SD|sd|Sd", name), "sd", "mean"),
    feature = gsub("(SD|sd|Sd)|(Mean|MEAN|mean)","", name)
  ) %>%
  dplyr::select(-name) %>%
  tidyr::pivot_wider(names_from = par, values_from = value) %>%
  tidyr::nest(pars = c(mean, sd)) %>%
  tidyr::pivot_wider(names_from = feature, values_from = pars) %>%
  dplyr::left_join(., trb_opr_pars)

# --- windspeed, rotation speed and blade pitch relationship
wndspd_rtn_ptch_example

# --- windfarm parameters
wf_pars <- data.frame(
  wf_id = c("wf_1", "wf_2"),
  n_turbs = c(200, 400),
  wf_width = c(4, 10),
  wf_lat = c(55.8, 55.0),
  td_off = c(2.5, 2),
  large_array_corr = c(FALSE, TRUE)
)


# -------------------------------------------------------------- #
# ----      Run stoch_crm() for multiple scenarios           ----
# -------------------------------------------------------------- #

# --- Set up scenario combinations
scenarios_specs <- tidyr::expand_grid(
  spp = bird_pars$Species,
  turb_id = trb_pars$TurbineModel,
  wf_id = wf_pars$wf_id) %>%
  tibble::add_column(
    scenario_id = paste0("scenario_", 1:nrow(.)),
    .before = 1)

# --- Set up progress bar for the upcoming iterative mapping step
pb <- progress::progress_bar$new(
  format = "Running Scenario: :what [:bar] :percent eta: :eta",
  width = 100,
  total = nrow(scenarios_specs))

# --- Map stoch_crm() to each scenario specification via purrr::pmap
outputs <- scenarios_specs %>%
  purrr::pmap(function(scenario_id, spp, turb_id, wf_id, ...){

    pb$tick(tokens = list(what = scenario_id))

    # params for current species
    c_spec <- bird_pars %>%
      dplyr::filter(Species == {{spp}}) # {{}} to avoid issues with data masking

    # density for current species
    c_dens <- dens_pars %>%
      dplyr::filter(Species == {{spp}})

    # params for current turbine scenario
    c_turb <- trb_pars %>%
      dplyr::filter(TurbineModel == {{turb_id}})

    # params for current windfarm scenario
    c_wf <- wf_pars %>%
      dplyr::filter(wf_id == {{wf_id}})

    # inputs in list-columns need to be unlisted, either via `unlist()` or
    # indexing `[[1]]`
    # switching off `verbose`, otherwise console will be # cramped with log
    messages
    stoch_crm(
      model_options = c(1, 2, 3),
      n_iter = 1000,
      flt_speed_pars = c_spec$Flight_Speed[[1]],
      body_lt_pars = c_spec$Body_Length[[1]],
      wing_span_pars = c_spec$Wingspan[[1]],
      avoid_bsc_pars = c_spec$AvoidanceBasic[[1]],
      avoid_ext_pars = c_spec$AvoidanceExtended[[1]],
      noct_act_pars = c_spec$Nocturnal_Activity[[1]],
      prop_crh_pars = c_spec$Prop_CRH_Obs[[1]],
      bird_dens_opt = c_dens$dens_opt,
      bird_dens_dt = c_dens$mth_dens[[1]],
      flight_type = c_spec$Flight,
      prop_upwind = c_spec$prop_upwind,
      gen_fhd_boots = gen_fhd_boots[[spp]],
      n_blades = c_turb$Blades,
      rtr_radius_pars = c_turb$RotorRadius[[1]],
      air_gap_pars = c_turb$HubHeightAdd[[1]],
      bld_width_pars = c_turb$BladeWidth[[1]],
      rtn_pitch_opt = c_turb$RotorSpeedAndPitch_SimOption,
      bld_pitch_pars = c_turb$Pitch[[1]],
      rtn_speed_pars = c_turb$RotationSpeed[[1]],
      windspd_pars = c_turb$windSpeed[[1]],
      rtn_pitch_windspd_dt = wndspd_rtn_ptch_example,
      trb_wind_avbl = c_turb$wind_avbl[[1]],
      trb_downtime_pars = c_turb$trb_dwntm[[1]],
      wf_n_trbs = c_wf$n_turbs,
      wf_width = c_wf$wf_width,
      wf_latitude = c_wf$wf_lat,
      tidal_offset = c_wf$td_off,
      lrg_arr_corr = c_wf$large_array_corr,
      verbose = FALSE,
      seed = 1234,
      out_format = "summaries",
      out_sampled_pars = FALSE,
      out_period = "seasons",
      season_specs = season_dt[[spp]],
      log_file = NULL
    )
  })

# --- close progress bar
pb$terminate()

# --- identify elements of output list
names(outputs) <- scenarios_specs$scenario_id

outputs
```
