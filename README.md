# FIRE Calculator

A single-page FIRE calculator for estimating financial independence timelines.

The app runs entirely in the browser. It does not send user-entered financial
data to a server.

## Features

- Annual income and expense modeling
- Cash, stock, bond, real estate, and other asset inputs
- Mortgage and personal loan projection
- FIRE target based on configurable withdrawal rate
- Retirement withdrawal strategies: inflation-adjusted fixed spending,
  Vanguard dynamic spending, guardrails, and fixed-percent withdrawals
- Historical retirement withdrawal backtest with success rate, worst starting
  year, terminal value distribution, and worst sequence table
- Best, median, and worst historical window cards selected from real rolling
  sequences, including real terminal assets, real lifestyle withdrawal cuts,
  and years below first-year purchasing power
- Optional debt-at-retirement backtest mode that pays mortgage/loan cashflows
  before lifestyle withdrawals
- Coast FIRE milestone based on target retirement age
- Scenario and sensitivity tables
- Interactive yearly chart tooltip
- JSON export

## Backtest data

The historical backtest is embedded for offline browser use. It uses the U.S.
annual stocks, 3-month Treasury bill, 10-year Treasury bond, and CPI inflation
series from NYU Stern / Aswath Damodaran's historical returns data set:

https://pages.stern.nyu.edu/~adamodar/New_Home_Page/data.html

Backtests are scenario estimates, not forecasts. The app keeps all entered
financial data in the browser.

## Deploy

This repository deploys to GitHub Pages through GitHub Actions.
