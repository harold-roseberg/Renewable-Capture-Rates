# Renewable-Capture-Rates
Examining the actual market value of utility-scale solar power across European geographies.

Several energy market terms I have described in more detail in my previous power markets modelling (European-Power-Trading, "EPT") repository. Please refer to that in case some of the terminology (eg. day-ahead, imbalance pricing) is troublesome. 

# 1. Introduction 

I have a long-term thesis that standalone utility-scale solar power (solar plants greater than ~5MW, connected to transmission or distribution grids) have a structural economics issue.  

Grid-scale solar power suffers from the "cannibalisation" effect: 
- Solar assets in a given geography tend to produce in a highly correlated fashion, ie. they produce roughly all at the same time (when it is sunny in Spain it tends to be sunny across most of the country). In the case of solar, this is in a sinusoidal curve peaking around midday, stronger in summer, weaker in winter. 
- Solar power has historically been a price taker on energy markets. Compared to technologies like coal and gas which have a fuel cost per unit MWh produced, solar has an almost-zero marginal cost of energy. In practice this means that solar operators bid their assets into energy markets (eg. day-ahead markets) at zero bid price, and  take whatever resulting clearing price occurs. This is a lazy (but historically effective) bidding strategy when you are pretty sure the marginal cost of electricity is fuel-driven.
- However, when demand is entirely met (or exceeded) by supply of solar power, this bidding strategy produces zero (or negative) power prices, causing solar operators to receive no revenues, or even pay to discharge electricity.
- As Europe crosses the 400GW threshold of solar power, the massive injection of solar power into European grids has exacerbated the occurence of zero/negative power prices during peak solar months (Apr-Aug). Every additional solar plant built reduces the revenues of existing plants, leading to the affectionate term of **"price cannibalisation"**.

**It goes without saying that this is a major economics issue for solar assets.** Whilst solar plants do not need to pay fuel costs, and only a small amount of operational costs, they do need to recoup their capital expenditures with a certain level of return on capital. If they cannot meet this return on capital due to cannibalisation, capital will cease to flow to building out utility-scale solar.

# 2. Project aims

**Project aims:**
- The aim of this project is to use publically available market and generation data from ENTSO-E to determine the capture price for utility-scale solar in European geographies, defined as the total revenue a solar plant receives divided by its total generation across a year (in €/MWh).
- Capture prices calculated on day-ahead market prices are a well trodden research avenue, but I aim to add two additional pieces of analysis:
  1. Implementing forecasting error costs into the capture price. I discuss this at greater length in the EPT repository, but summarised briefly here: solar assets sell energy into day-ahead markets based on forecasted production (driven by weather forecasts etc.), but when they miss their forecasts and are long/short on their contracted volumes they need to compensate by selling/buying energy on the intra-day markets, with any further errors corrected on imbalance markets. These forecast errors can cost assets a meaningful sum.
  2. Comparison of resulting capture rates against historical LCOEs for European solar. LCOE is the flawed but favourite metric of the renewables industry. Briefly, it is a metric that illustrates (given a set of assumptions on capital expenditures, operating costs, cost of capital, etc.) the price a solar asset would need to sell its power at to recoup its required cost of capital. **If a solar asset generates less revenue than its LCOE, it will not generate enough return on capital, and vice-versa.** My reference source for LCOEs is from the International Renewable Energy Agency: https://www.irena.org/Publications/2025/Jun/Renewable-Power-Generation-Costs-in-2024
 
**NOTE**: this project is done on a national fleet level basis, which naturally evens out the errors of an aggregated set of solar plants. I will deal with this in more depth in one of the later sections.

# 3. Data sources
**Data source:**
This project uses data from the ENTSO-E Transparency Platform. To replicate the code you need to create a login and an individual API key.

**Python API client:**
THe ENTSO-E API is difficult to use so I have used a Python client for the API, developed by EnergieID: https://github.com/EnergieID/entsoe-py. The `entsoe-py` package is MIT licensed.

**Geographies and technology:**
To begin I have chosen to analyse the top-3 solar producers in Europe for the years 2020-2025: Germany, Spain and France. I have made the code able to pull offshore and onshore wind, but for this project will restrict it to solar power.

**Generation forecast and actual data:**
For the project I pull three types of generation data:
1. **Day-ahead generation forecast**: the renewables generation forecast used in the day-ahead market. Solar assets use this forecast to place their bids to sell energy.
2. **Intra-day generation forecast:** after the day-ahead market closes a further generation forecast is used for the intra-day market. This updated forecast allows generators to update their volumes if they expect there will be a difference in generation to their day-ahead contracts.
3. **Actual generation:** the actual generation by solar assets.

**Market data:**
To illustrate the cost of forecasting errors, I pull the relevant market prices that a generator would be beholden to. More detail on these in my EPT repository.
1. **Day-ahead energy prices**
2. **Intra-day energy prices**: NOTE: ENTSO-E do not publish these, so I have used the day-ahead prices as a proxy. This is not strictly market-accurate but remains a good proxy for a majority of cases.
3. **Imbalance prices**

# 4. Methodology
The code runs through the following logic:
- For a given country (eg. Germany) and technology (here solar), the code pulls the generation forecast/actual data and market prices, and computes through the steps below.
- The code computes the error from the day-ahead to the intra-day forecast. This is the error that a solar operator would seek to remedy on the intra-day market. The error is multiplied by the day-ahead price (proxying for intra-day) to compute the error cost.
  - Example: solar asset contracts to sell 400MWh on the day-ahead market based on the DA generation forecast. The intra-day forecast suggests that the generation will instead be 350MWh. The solar asset therefore has to buy 50MWh of energy on the intra-day market to fulfil its day-ahead contract. 
- The code then computes the error from the intra-day forecast to actual generation. The residual error has to be rectified at the imbalance price. The imbalance price (depending on country) has a long price (if you exceed your forecast) or a short price (if you undershoot your forecast)
  - Example: the intra-day forecast suggested 350MWh production, but the actual final generation was 300MWh. The solar asset has to cover the 50MWh delta on the imbalance short market, given the asset was short on its predicted generation. If it had been long (eg. 375MWh generation) it would have been paid the long imbalance price for the positive delta of its production.
- 






