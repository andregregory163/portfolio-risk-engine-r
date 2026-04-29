# ==========================================================
# GLOBAL PORTFOLIO RISK ENGINE (R)
# Uses your real portfolio market value weights
# Author: Andre Gregory
# ==========================================================

# INSTALL PACKAGES FIRST IF NEEDED:

install.packages(c(
  "quantmod",
  "PerformanceAnalytics",
  "tidyverse",
  "lubridate",
  "corrplot",
  "scales"
))

library(quantmod)
library(PerformanceAnalytics)
library(tidyverse)
library(lubridate)
library(corrplot)
library(scales)

# ==========================================================
# 1. PORTFOLIO HOLDINGS (OPTION A: MARKET VALUE WEIGHTS)
# ==========================================================

holdings <- tibble(
  ticker = c("GOLD.AX","HMC.AX","IJR.AX","IVV.AX","IXJ.AX","NEU.AX",
             "NXG.AX","SMLL.AX","VAS.AX","VGS.AX","WIRE.AX","XRO.AX",
             "XYZ.AX","ZIP.AX","AMD","ANET","APH","AVGO","UNH","VRT"),
  
  value = c(2821.44,3232.50,2090.77,18021.50,1901.70,4803.44,
            4180.00,1006.74,13040.17,1936.48,2751.10,1123.92,
            4788.28,3874.09,2908.89,2172.72,1074.67,1665.05,
            1100.31,3050.30)
)

holdings$weight <- holdings$value / sum(holdings$value)

# ==========================================================
# 2. DOWNLOAD 10 YEARS OF PRICE DATA
# ==========================================================

start_date <- Sys.Date() - years(10)

prices <- list()

for(i in holdings$ticker){
  tryCatch({
    data <- Ad(getSymbols(i, src = "yahoo",
                          from = start_date,
                          auto.assign = FALSE))
    prices[[i]] <- data
  }, error = function(e) cat("Failed:", i, "\n"))
}

prices_xts <- do.call(merge, prices)
colnames(prices_xts) <- names(prices)

# ==========================================================
# 3. RETURNS
# ==========================================================

returns <- na.omit(Return.calculate(prices_xts, method = "log"))

# Align weights to downloaded assets only
valid_assets <- colnames(returns)

weights <- holdings %>%
  filter(ticker %in% valid_assets) %>%
  arrange(match(ticker, valid_assets)) %>%
  pull(weight)

weights <- weights / sum(weights)

# Portfolio returns
port_ret <- Return.portfolio(returns, weights = weights)

# ==========================================================
# 4. PERFORMANCE METRICS
# ==========================================================

cat("====================================\n")
cat("PORTFOLIO PERFORMANCE METRICS\n")
cat("====================================\n")

cat("Annual Return:",
    round(Return.annualized(port_ret)*100,2), "%\n")

cat("Annual Volatility:",
    round(StdDev.annualized(port_ret)*100,2), "%\n")

cat("Sharpe Ratio:",
    round(SharpeRatio.annualized(port_ret),2), "\n")

cat("Sortino Ratio:",
    round(SortinoRatio(port_ret),2), "\n")

cat("Max Drawdown:",
    round(maxDrawdown(port_ret)*100,2), "%\n")

# ==========================================================
# 5. HISTORICAL VAR
# ==========================================================

hist_var95 <- quantile(port_ret, 0.05)
hist_var99 <- quantile(port_ret, 0.01)

cat("\nHistorical VaR 95%:",
    round(hist_var95*100,2), "%\n")

cat("Historical VaR 99%:",
    round(hist_var99*100,2), "%\n")

# ==========================================================
# 6. MONTE CARLO VAR
# ==========================================================

mu <- mean(port_ret)
sigma <- sd(port_ret)

sim <- rnorm(10000, mean = mu, sd = sigma)

mc_var95 <- quantile(sim, 0.05)
mc_var99 <- quantile(sim, 0.01)

cat("\nMonte Carlo VaR 95%:",
    round(mc_var95*100,2), "%\n")

cat("Monte Carlo VaR 99%:",
    round(mc_var99*100,2), "%\n")

# Expected Shortfall
es95 <- mean(sim[sim < mc_var95])

cat("Expected Shortfall 95%:",
    round(es95*100,2), "%\n")

# ==========================================================
# 7. CHARTS
# ==========================================================

png("portfolio_growth.png", width=900, height=500)
charts.PerformanceSummary(port_ret,
                          main="Portfolio Performance")
dev.off()

png("correlation_heatmap.png", width=900, height=700)
corrplot(cor(returns, use="complete.obs"),
         method="color",
         tl.cex=.7)
dev.off()

png("var_distribution.png", width=900, height=500)
hist(sim, breaks=50,
     main="Monte Carlo Return Distribution",
     xlab="Simulated Returns",
     col="lightblue")
abline(v=mc_var95, lwd=2, col="red")
dev.off()

# ==========================================================
# 8. TOP CONCENTRATION RISK
# ==========================================================

cat("\n====================================\n")
cat("TOP HOLDINGS BY WEIGHT\n")
cat("====================================\n")

holdings %>%
  mutate(weight = percent(weight)) %>%
  arrange(desc(value)) %>%
  select(ticker, weight) %>%
  head(10) %>%
  print()

# ==========================================================
# END
# ==========================================================

getwd()

