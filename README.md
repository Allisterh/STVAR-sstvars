
<!-- README.md is generated from README.Rmd. Please edit that file -->

# sstvars

<!-- badges: start -->
<!-- badges: end -->

**This software is still in development and does not work yet!**

The goal of `sstvars` is to provide a comprehensive toolkit for maximum
likelihood (ML) estimation and analysis of reduced form and structural
smooth transition vector autoregressive (STVAR) models. Various
transition weight functions, conditional distributions, and
identification methods are and will be accommodated. Also constrained ML
estimation is supported with constraints on the autoregressive
parameters, regimewise means, weight parameters, and the impact matrix.
The package is, however, is limited to endogenous switching variables to
ensure that ergodicity and stationarity of the models is verifiable and
that the true generalized impulse response functions can be estimated.

## Installation

You can install the development version of sstvars from
[GitHub](https://github.com/) with:

``` r
# install.packages("devtools")
devtools::install_github("saviviro/sstvars")
```

## Example

This is a basic example on how to use `sstvars` in time series analysis.
The estimation process is computationally demanding and takes advantage
of parallel computing. After estimating the model, it is shown by simple
examples how to conduct some further analysis.

``` r
# These examples use the data 'gdpdef' which comes with the package, and contains the quarterly percentage growth rate
# of real U.S. GDP and quarterly percentage growth rate of U.S. GDP implicit price deflator, covering the period 
# from 1959Q1 to 2019Q4.
data(gdpdef, package="sstvars")

### Reduced form STVAR models ###

# Estimate a reduced form two-regime Student's t STVAR p=3 model with logistic transition weight function using the first
# lag of the second variable (GDP deflator) as the switching variable. The below estimation is based on 20 estimation
# rounds with seeds set for reproducibility.
# (note: many empirical applications require more estimation rounds, e.g., hundreds
# or thousands).
fit <- fitSTVAR(gdpdef, p=3, M=2, weight_function="logistic", weightfun_pars=c(2, 1), cond_dist="Student",
                nrounds=20, ncores=2, seeds=1:20)

# Estimate the above model but with the autoregressive matrices restricted to be equal in both regimes:
C_mat <- rbind(diag(3*2^2), diag(3*2^2))
fitc <- fitSTVAR(gdpdef, p=3, M=2, weight_function="logistic", weightfun_pars=c(2, 1), cond_dist="Student",
                 AR_constraints=C_mat, nrounds=20, ncores=2, seeds=1:20)

# Estimate the above model but with the autoregressive matrices and unconditional means restricted to be equal
# in both regimes (so that only the conditional covariance matrix varies in time):
fitcm <- fitSTVAR(gdpdef, p=3, M=2, weight_function="logistic", weightfun_pars=c(2, 1), cond_dist="Student",
                  AR_constraints=C_mat, mean_constraints=list(1:2), nrounds=20, ncores=2, seeds=1:20)

# Estimate the above logistic STVAR model without constraints on the autoregressive parameters but with the 
# the location parameter constrained to 1 and scale parameter unconstrained.
fitw <- fitSTVAR(gdpdef, p=3, M=2, weight_function="logistic", weightfun_pars=c(2, 1), cond_dist="Student",
                 weight_constraints=list(R=matrix(c(0, 1), nrow=2), r=c(1, 0)), nrounds=20, ncores=2, seeds=1:20)

# Test the constraint on the location parameter with the likelihood ratio test:
LR_test(fit, fitw) # See also Wald_test and Rao_test

# Further information on the estimated model:
plot(fit) # Plot the estimated transition weight function with data
summary(fit) # Summary prinoout of the estimated model
print_std_errors(fit) # Print standard errors of the estimated model (assuming the standard asymptotics on the ML estimator)
get_foc(fit) # The first order condition (gradient of the log-likelihood function)
get_soc(fit) # The second order condition (eigenvalues of approximated Hessian)
profile_logliks(fit) # Profile log-likelihoods about the estimate

# Check the stationarity condition for the estimated model, i.e., that the 
# upper bound of the joint spectral radius is less than one:
bound_JSR(fit, epsilon=0.01) 

# Residual based model diagnostics:
diagnostics(fit, type="series", resid_type="standardized") # Standardized residual time series
diagnostics(fit, type="ac", resid_type="raw") # Autocorrelation function of unstandardized residuals
diagnostics(fit, type="ch", resid_type="standardized") # Autocorrelation function of squared standardized residuals
diagnostics(fit, type="dist", resid_type="standardized") # Histograms and Q-Q plots of standardized residuals

Portmanteau_test(fit, nlags=20, which_test="autocorr") # Portmanteau test for remaining autocorrelation
Portmanteau_test(fit, nlags=20, which_test="het.sked") # Portmanteau test applied for testing cond. het.kedasticity

# Simulate a sample path from the estimated model:
sim <- simulate(fit, nsim=100)

# Forecast future values of the process:
pred <- predict(fit, nsteps=10, ci=c(0.95, 0.80))
plot(pred)


### Structural STVAR models ###

# stvars implements two identification methods: recursive identification and
# identification by heteroskedasticity. The structural models are estimated 
# based on preliminary estimates from a reduced form model. If the structural model
# is not overidentifying, the model is merely reparametrized and no estimation is
# required (recursively identified models are just reduced form models marked as structural). 

# Identify the above logistic STVAR model by recursive identification:
fitrec <- fitSSTVAR(fit, identification="recursive")
fitrec

# Identify the above logistic STVAR model by heteroskedasticity:
fithet <- fitSSTVAR(fit, identification="heteroskedasticity")
fithet

# Reorder the columns of the impact matrix of fithet to the reverse ordering:
fithet <- reorder_impact_matrix(fithet, perm=c(2, 1))
fithet

# Change all signs of the first column of the impact matrix of fithet:
fithet <- swap_W_signs(fithet, which_to_swap=1)
fithet

# Estimate the generalized impulse response function (GIRF) for the recursively
# identified model to one-standard-error positive shocks with the starting values
# generated form the first regime, N=30 steps ahead and 95% confidence intervals 
# that reflect uncertainty about the initial value within the regime:
girf1 <- GIRF(fitrec, which_shocks=1:2, shock_size=1, N=30, init_regime=1, ci=0.95)
plot(girf1)

# Estimate the above GIRF but instead of drawing initial values form the first regime,
# use tha last p observations of the data as the initial values:
girf2 <- GIRF(fitrec, which_shocks=1:2, shock_size=1, N=30, init_values=fitrec$data)
plot(girf2)

# Estimate the generalized impulse response function (GIRF) for the recursively
# identified model to two-standard-error negative shocks with the starting values
# generated form the second regime, N=30 steps ahead and 95% confidence intervals 
# that reflect uncertainty about the initial value within the regime. Also, scale
# the responses to the first shock to correspond to a 0.3 increase of the first variable.
# Moreover, accumulate the responses of the second variable.
girf3 <- GIRF(fitrec, which_shocks=1:2, shock_size=-2, N=30, init_regime=2, 
              scale=c(1, 1, 0.3), which_cumulative=2, ci=0.95)
plot(girf3)

# Estimate the generalized forecast error variance decomposition (GFEVD) for the 
# recursively identified model with the initial values being all possible length p
# histories in the data, N=30 steps ahead to one-standard-error positive shocks. 
gfevd1 <- GFEVD(fitrec, N=30, initval_type="data", )
```
