# Heikin-Ashi Momentum Trend Strategy Backtester

## Overview

This Python script implements and backtests a quantitative trading strategy based on Heikin-Ashi candles combined with momentum and trend confirmation indicators (EMA, ADX, Volume). It is designed to run across multiple stock tickers using historical data provided in custom CSV files. The goal is to identify potential uptrends and enter long positions, managing risk with an ATR-based stop-loss.

## Features

*   **Multi-Asset Backtesting:** Tests the strategy across several stock tickers (AAPL, META, TSLA, JPM, AMZN).
*   **Custom CSV Data Loading:** Specifically designed to handle CSV files where the header information is on the 3rd row and columns follow a specific order (`Date, Close, High, Low, Open, Volume`).
*   **Heikin-Ashi Candles:** Uses Heikin-Ashi candles (calculated via `pandas_ta`) to smooth price action and potentially provide clearer trend signals. Falls back to regular OHLC if HA calculation fails.
*   **Technical Indicators:** Employs EMA Crossover, ADX, ATR, and Volume SMA for signal generation and filtering.
*   **Configurable Parameters:** Allows easy adjustment of indicator periods, ADX threshold, ATR stop-loss multiplier, initial capital, position sizing, and commission rates.
*   **ATR Stop-Loss:** Implements a dynamic stop-loss based on the Average True Range (ATR) to adapt to market volatility.
*   **Performance Analytics:** Calculates key metrics like Total Return, Annualized Return, Volatility, Sharpe Ratio, Sortino Ratio, Max Drawdown, Win Rate, Profit Factor, etc.
*   **Visualization:** Generates plots for the portfolio equity curve, portfolio drawdown, and individual stock trades overlaid on the price chart.
*   **Trade Log:** Outputs a detailed list of all trades executed during the backtest.

## Trading Strategy Explained

The strategy aims to capture upward momentum in assets that exhibit strong trending characteristics, using Heikin-Ashi candles for smoother signal generation. It is a **long-only** strategy.

1.  **Core Idea:** Enter a long position when the smoothed price (Heikin-Ashi) shows upward momentum (short EMA > long EMA), the trend is strong (ADX > threshold), and there's volume confirmation (Volume > Volume SMA). Exit when the momentum reverses (short EMA < long EMA) or when the stop-loss is hit.

2.  **Heikin-Ashi (HA):**
    *   Calculated using the `pandas_ta` library.
    *   HA candles modify the standard Open, High, Low, Close (OHLC) values to filter out noise and highlight the underlying trend. Consecutive green HA candles suggest an uptrend, while consecutive red candles suggest a downtrend.
    *   The strategy primarily uses the HA Close price for EMA calculations and HA High/Low/Close for ADX and ATR calculations *if* HA calculation is successful. Otherwise, it falls back to using regular OHLC data for these indicators.

3.  **Indicators Used:**
    *   **Exponential Moving Averages (EMA) on HA Close:**
        *   `HA_EMA_short` (e.g., 20-period): Represents the shorter-term trend.
        *   `HA_EMA_long` (e.g., 50-period): Represents the longer-term trend.
        *   *Signal:* A crossover of the short EMA above the long EMA indicates potential bullish momentum.
    *   **Average Directional Index (ADX):**
        *   `ADX` (e.g., 14-period): Measures trend *strength*, not direction.
        *   *Filter:* A value above a threshold (`adx_threshold`, e.g., 25) suggests the market is trending (either up or down), making EMA signals more reliable. The strategy only takes signals when the trend is deemed strong enough.
    *   **Volume Simple Moving Average (SMA):**
        *   `Volume_SMA` (e.g., 20-period): Average volume over the specified period.
        *   *Filter:* Requires the previous day's volume (`Volume`) to be greater than its SMA (`Volume_SMA`) to confirm the momentum with trading activity.
    *   **Average True Range (ATR):**
        *   `ATR` (e.g., 14-period): Measures market volatility.
        *   *Stop-Loss:* Used to set a dynamic stop-loss level below the entry price.

4.  **Entry Logic (Long Position):**
    A buy signal is generated at the end of a trading day (based on `i-1` data) if **ALL** of the following conditions are met:
    *   `HA_EMA_short` (previous day) > `HA_EMA_long` (previous day)
    *   `ADX` (previous day) > `adx_threshold`
    *   `Volume` (previous day) > `Volume_SMA` (previous day)
    *   A valid `ATR` (previous day) exists for stop-loss calculation.
    *   The account currently holds no position in that asset.
    *   *Execution:* If a signal occurs, a long position is entered at the **closing price** of the *current* trading day (`i`).

5.  **Position Sizing:**
    *   When entering a trade, a fixed percentage (`position_size_pct`, e.g., 10%) of the *current portfolio equity* is allocated to the trade.
    *   The number of shares is calculated based on this allocation and the entry price.

6.  **Exit Logic (Long Position):**
    A position is closed if **EITHER** of the following occurs:
    *   **EMA Crossover:** `HA_EMA_short` (previous day) < `HA_EMA_long` (previous day).
        *   *Execution:* Exit at the **closing price** of the *current* trading day (`i`).
    *   **Stop-Loss Hit:** The market's `Low` price during the current day (`i`) touches or goes below the calculated `stop_loss_price`.
        *   `stop_loss_price` = `entry_price` - (`ATR` at entry time * `atr_stop_multiplier`)
        *   *Execution:* Exit at the `stop_loss_price` (or the `Open` price if it gapped below the stop-loss).

7.  **End of Backtest:** Any open positions are closed at the closing price of the last day in the dataset.

8.  **Commissions:** A percentage commission (`commission_pct`) is applied to the value of each trade (both entry and exit).

## Code Structure & Workflow

The script follows a logical flow from configuration to results:

1.  **Imports:** Imports necessary libraries: `pandas` for data manipulation, `numpy` for numerical operations, `pandas_ta` for technical indicators, `matplotlib` for plotting, and `os` for file path handling.
2.  **Configuration:**
    *   Sets file paths for the input CSV data.
    *   Defines strategy parameters (EMA periods, ADX settings, ATR settings, Volume SMA period).
    *   Sets portfolio parameters (initial capital, position size percentage, commission rate).
    *   Defines standard internal column names (`Open`, `High`, `Low`, `Close`, `Volume`, `Date`).
3.  **Data Acquisition & Preparation:**
    *   Iterates through the defined `tickers`.
    *   Reads each CSV file using `pd.read_csv`, specifying `header=2` as the actual headers are on the third row.
    *   **Crucially**, it renames the columns based on the *expected order* in the CSV file's first few rows (`Date, Close, High, Low, Open, Volume`) to the standard internal names (`Date`, `Open`, `High`, `Low`, `Close`, `Volume`).
    *   Converts the `Date` column to datetime objects and sets it as the DataFrame index.
    *   Sorts the data by date.
    *   Checks for the presence of required standard columns (`Open`, `High`, `Low`, `Close`, `Volume`).
    *   Converts OHLCV columns to numeric types, coercing errors.
    *   Handles missing values by dropping rows with NaNs in essential OHLCV columns.
    *   Stores the cleaned DataFrame in `data_dict`.
4.  **Indicator Calculation (`calculate_indicators` function):**
    *   Takes a stock's DataFrame as input.
    *   Calculates Heikin-Ashi candles using `ta.ha`. *Important:* It appends HA columns (`HA_open`, `HA_high`, `HA_low`, `HA_close`) to the DataFrame.
    *   **Conditional Indicator Base:** Checks if HA columns were successfully created.
        *   If YES: Uses HA High, Low, and Close for calculating EMAs, ADX, and ATR.
        *   If NO (or calculation failed): Uses regular High, Low, and Close for these indicators.
    *   Calculates EMAs, ADX, ATR, and Volume SMA using `pandas_ta` functions based on the selected OHLC (HA or regular).
    *   Returns the DataFrame with indicator columns added.
    *   This calculation is performed for all successfully loaded tickers.
5.  **Backtesting Engine (`run_backtest` function):**
    *   Takes the indicator-rich DataFrame, ticker symbol, allocated capital, position sizing percentage, and commission rate as input.
    *   Initializes portfolio variables (cash, equity history, position status, trades list).
    *   Iterates through the DataFrame, starting from the second row (index 1).
    *   **Signal Check:** Uses data from the *previous* day (`i-1`) to check for entry/exit signals based on the strategy logic (EMA cross, ADX, Volume).
    *   **Stop-Loss Check:** Checks if the *current* day's `Low` price hits the active stop-loss level.
    *   **Execution:** If a signal occurs or a stop-loss is hit:
        *   Calculates trade size based on `position_size_pct` and current equity.
        *   Simulates entry/exit at the *current* day's `Close` price (or stop-loss price).
        *   Calculates P&L, factoring in commissions.
        *   Updates cash and position status.
        *   Records the trade details (entry/exit dates/prices, PNL, reason).
    *   **Equity Tracking:** Updates the portfolio equity daily based on cash and the market value of the current position (using the daily `Close` price).
    *   Returns the daily equity curve (DataFrame) and a list of executed trades (DataFrame).
6.  **Running Backtests:**
    *   Determines the capital allocated per stock (initial capital / number of valid stocks).
    *   Iterates through the tickers that have valid indicator data.
    *   Calls `run_backtest` for each ticker.
    *   Stores the resulting equity curves and trade lists.
7.  **Results Aggregation & Portfolio Calculation:**
    *   Combines individual trade lists into one master `all_trades_df`.
    *   Aligns the individual equity curves on a common date index (using intersection).
    *   Sums the equity curves daily (after forward-filling and handling initial NaNs) to create the overall `portfolio_equity` curve.
8.  **Performance Analytics (`calculate_performance_analytics` function):**
    *   Takes an equity curve (Series or DataFrame) and a trades DataFrame as input.
    *   Calculates various performance metrics (Total Return, Annualized Metrics, Drawdown, Trade Stats).
    *   Includes robust handling for empty data, NaNs, zero standard deviation in returns, and edge cases in calculations.
    *   Calculates analytics for the overall portfolio and for each individual stock.
    *   Prints the analytics results.
9.  **Visualization:**
    *   Uses `matplotlib` to plot:
        *   The overall portfolio equity curve over time.
        *   The portfolio drawdown curve over time.
        *   An example plot for a single stock (e.g., AAPL) showing the closing price with trade entry (^) and exit (v) markers.
10. **Output Trade Log:**
    *   Prints the complete `all_trades_df` DataFrame to the console, showing details for every simulated trade.

## Configuration Parameters (Section 2)

*   `csv_directory`: Path to the folder containing the CSV data files.
*   `csv_files`: Dictionary mapping tickers to their respective CSV file paths.
*   `tickers`: List of stock symbols to backtest.
*   `ema_short_period`: Lookback period for the short-term EMA.
*   `ema_long_period`: Lookback period for the long-term EMA.
*   `adx_period`: Lookback period for the ADX calculation.
*   `adx_threshold`: Minimum ADX value required to confirm a trend.
*   `atr_period`: Lookback period for the ATR calculation.
*   `atr_stop_multiplier`: Factor multiplied by ATR to set the stop-loss distance.
*   `volume_sma_period`: Lookback period for the Volume SMA.
*   `initial_capital`: Starting capital for the entire portfolio.
*   `position_size_pct`: Percentage of current portfolio equity allocated to each new trade.
*   `commission_pct`: Transaction cost applied to the value of each trade.

## Dependencies

*   pandas
*   numpy
*   pandas_ta
*   matplotlib

## Disclaimer

This script is for educational and informational purposes only. Trading financial markets involves significant risk. Past performance is not indicative of future results. The strategy and code provided do not constitute financial advice. Use this script at your own risk and consult with a qualified financial advisor before making any investment decisions. Download the dataset from the link specified andn put that into the respective folder to start running the code https://drive.google.com/drive/folders/1n6w_DTFzHeeMOb3hRjcbpShjls75qXe5
