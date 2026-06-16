# Renewable-Capture-Rates
Examining the actual market value of utility-scale solar power across European geographies.

Several energy market terms I have described in more detail in my previous power markets modelling (European-Power-Trading, "EPT") repository. Please refer to that in case some of the terminology (eg. day-ahead, imbalance pricing) is troublesome. 

# 1. Introduction 

I have a long-term thesis that standalone utility-scale solar power (solar plants greater than ~5MW, connected to transmission or distribution grids) have a structural economics issue.  

Grid-scale solar power suffers from the "cannibalisation" effect: 
- Solar assets in a given geography tend to produce in a highly correlated fashion, ie. they produce roughly all at the same time (when it is sunny in Spain it tends to be sunny across most of the country). In the case of solar, this is in a sinusoidal curve peaking around midday, stronger in summer, weaker in winter. 
- Solar power has historically been a price taker on energy markets. Compared to technologies like coal and gas which have a fuel cost per unit MWh produced, solar has an almost-zero marginal cost of energy. In practice this means that solar operators bid their assets into energy markets (eg. day-ahead markets) at zero bid price, and  take whatever resulting clearing price occurs. This is a lazy (but historically effective) bidding strategy when you are pretty sure the marginal cost of electricity is fuel-driven.
- However, when demand is entirely met (or exceeded) by supply of solar power, this bidding strategy produces zero (or negative) power prices, causing solar operators to receive no revenues, or even pay to discharge electricity.
- As the EU crosses the 400GW threshold of solar power, the massive injection of solar power into European grids has exacerbated the occurence of zero/negative power prices during peak solar months (Apr-Aug). Every additional solar plant built reduces the revenues of existing plants, leading to the affectionate term of **"price cannibalisation"**.

**It goes without saying that this is a major economics issue for solar assets.** Whilst solar plants do not need to pay fuel costs, and only a small amount of operational costs, they do need to recoup their capital expenditures with a certain level of return on capital. If they cannot meet this return on capital due to cannibalisation, capital will cease to flow to building out utility-scale solar.

# 2. Project aims

**Project aims:**
- The aim of this project is to use publically available market and generation data from ENTSO-E to determine the capture price for utility-scale solar in European geographies, defined as the total revenue a solar plant receives divided by its total generation across a year (in €/MWh).
- Capture prices calculated on day-ahead market prices are a well trodden research avenue, but I aim to add two additional pieces of analysis:
  1. Implementing forecasting error costs into the capture price. I discuss this at greater length in the EPT repository, but summarised briefly here: solar assets sell energy into day-ahead markets based on forecasted production (driven by weather forecasts etc.), but when they miss their forecasts and are long/short on their contracted volumes they need to compensate by selling/buying energy on the intra-day markets, with any further errors corrected on imbalance markets. These forecast errors can cost assets a meaningful sum.
  2. Comparison of resulting capture rates against historical LCOEs for European solar. LCOE is the flawed but favourite metric of the renewables industry. Briefly, it is a metric that illustrates (given a set of assumptions on capital expenditures, operating costs, cost of capital, etc.) the price a solar asset would need to sell its power at to recoup its required cost of capital. **If a solar asset generates less revenue than its LCOE, it will not generate enough return on capital, and vice-versa.** My reference source for LCOEs is from the International Renewable Energy Agency: https://www.irena.org/Publications/2025/Jun/Renewable-Power-Generation-Costs-in-2024
 
**NOTE**: the data for this project is downloaded on a national fleet level basis, which naturally evens out the errors of an aggregated set of solar plants. I will deal with this in more depth in one of the later sections.

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

# 4. Methodology - basic data pull

The code runs through the following basic logic:
- For a given country (eg. Germany) and technology (here solar), the code pulls the generation forecast/actual data and market prices, and computes through the steps below.
- The code computes the error from the day-ahead to the intra-day forecast. This is the error that a solar operator would seek to remedy on the intra-day market. The error is multiplied by the day-ahead price (proxying for intra-day) to compute the error cost.
  - Example: solar asset contracts to sell 400MWh on the day-ahead market based on the DA generation forecast. The intra-day forecast suggests that the generation will instead be 350MWh. The solar asset therefore has to buy 50MWh of energy on the intra-day market to fulfil its day-ahead contract. 
- The code then computes the error from the intra-day forecast to actual generation. The residual error has to be rectified at the imbalance price. The imbalance price (depending on country) has a long price (if you exceed your forecast) or a short price (if you undershoot your forecast)
  - Example: the intra-day forecast suggested 350MWh production, but the actual final generation was 300MWh. The solar asset has to cover the 50MWh delta on the imbalance short market, given the asset was short on its predicted generation. If it had been long (eg. 375MWh generation) it would have been paid the long imbalance price for the positive delta of its production.
- Finally, the code computes the net revenue to the solar asset, here composed of:
  - Its contracted volume sales on the day-ahead market
  - Less: any adjustments to be made on the intra-day market (here proxied by the DA price)
  - Less: any adjustments to be made on the imbalance market

# 5. Methodology - fleet versus asset-level errors

Without spoiling the results section too early, I discovered early in the journey that working with fleet level aggregated data can mask the size of errors at the asset level. For example, if two assets both miss their forecast by +50MWh for one, and -50MWh for the other, the fleet level will still register a net 0/MWh forecast error. **An individual asset is usually more exposed to forecast errors than a national fleet.**

For the DA-ID forecast errors, this has little to no effect on the net revenue per MWh of a fleet versus an asset. Any divergence in errors from the fleet level error can be corrected at the same price: the intra-day price (here proxied with the DA price). Continuing with the example above, if the ID price for the two assets is €100/MWh, asset A would sell its long position for €5,000 and asset B would have to pay €5000 for its shortfall. If on the long run asset errors revert to the mean of the fleet level error, the net financial position should be the same due to a symmetric long/short price. However this gets interesting for systems (like FR, ES) which have different imbalance prices whether long or short, and therefore where assets diverging from fleet-level ID->Actual errors can register non-linear differences in imbalance cost corrections. Note I would expect DE solar assets not to be effected as the DE system has a symmetric imbalance price (same long and short).

To estimate asset-level forecast errors I subdivided the national fleets into N=1000 equal-sized assets. This produces representative asset sizes of roughly 20–50 MW across DE/ES/FR in recent years, which is a reasonable stylised utility-scale asset size, though actual projects vary widely. The second key assumption is a statistical assumption called rho (size 0-1), which represents how correlated forecast errors are between different assets. A rho of 1 means all assets make the same error at the same time, a rho of 0 means asset errors are effectively uncorrleated so errors cancel out strongly. In practice renewable assets are between these extremes, and I have run a simulation with a rho = 0.75, to mimic a decently correlated fleet. In practice the rho will strongly differ by countries and geographies.

Given a fleet level forecast error, here measured as an RMSE (root-mean-squared-error), I grouped errors by month and hour and calculated the RMSE within each month-hour bucket. This gives a seasonal and intraday profile of forecasting errors magnitude (eg. 13.00 in June has an average error of 500MWh). This adjustment is needed to convert a yearly non-seasonal RMSE value to errors that make sense on an hourly basis (eg. the fleet level RMSE for DE in 2025 is c. 1.5GW, it would make no sense to vary the nighttime production by 1.5GW when fleet level production is zero).

For each time period the model starts with the actual observed fleet error (eg. -1000MWh delta between the ID forecast, say 10,000MWh, and the actual generation, 9,000MWh), and creates a base asset error = fleet error / N. In the example this means that for the time period and N=1000, each asset has a base error of -1MWh, with an ID forecast of 10MWh and actual generation of 9MWh. The model then adds random deviations around this error for each N, with the size of the deviations governed mathematically by rho (strictly speaking we create a normal distribution of asset errors with an asset RMSE = fleet RMSE / sqrt(N * (1 + (N-1) * rho)). To further the example, this means Asset N1 may have a base error of -1.1MWh, N2 -0.9MWh, etc. These random deviations sum to zero across the aggregate of all assets to preserve the observed fleet error. **In other words, the model preserves the real historical fleet error while estimating how that error might have been distributed across individual assets.**

# 6. Results 

Tables extracted from excel, charts from Python script.

**Results for Germany:**

<img width="1649" height="749" alt="image" src="https://github.com/user-attachments/assets/de6efca6-a25a-47b1-9036-3856b4739fe7" />
-
<img width="889" height="490" alt="image" src="https://github.com/user-attachments/assets/31dff9b1-fd2b-4c4b-8c87-62b196e39cb9" />



**Results for Spain:**

<img width="1649" height="749" alt="image" src="https://github.com/user-attachments/assets/2c2e06a0-cf7e-4b5c-9945-98a051c25e59" />

<img width="889" height="490" alt="image" src="https://github.com/user-attachments/assets/046ed5e2-b9e2-4cd1-b5e4-f18d3fcb40d1" />



**Results for France:**

<img width="1649" height="749" alt="image" src="https://github.com/user-attachments/assets/cb68f7a9-6211-41f7-966d-362ac266f746" />

<img width="889" height="490" alt="image" src="https://github.com/user-attachments/assets/37e6e1ae-bd2a-42f4-a68c-c6670f94c618" />

# 7. Discussion

There are several points of interest here:

1. We can see the effect of the 2022 Energy crisis on solar capture prices. Solar assets selling power on DA markets in 2022 captured between €150-300MWh over the year. The shock reverberated into 2023, and we see high prices in 2021 due to a spike in gas prices post-lockdown. It would have been a very profitable time to be a solar producer selling on the merchant markets, however most new-build solar in Europe since 2010 has been on fixed-price government-backed contracts (eg. FiTs, CfDs, etc.), with only a  minority being able to achieve these high prices outside of the contracts.

2. 2021-2023 aside, day-ahead capture prices for solar sit in the €25-50/MWh range for Germany, €35-45/MWh range for Spain, €30-35/MWh range for France. Note that in 2020, 2024 and 2025 **these sit below the required LCOE (IRENA) for a solar asset to reach its required capital return**. Some further thoughts on this:
  - This is not immediately an issue for new-build utility-scale solar. As mentioned above, most new builds are backed by long (15-20yr) government contracts, which have clearing prices that by design meet the capital return requirements of asset owners (otherwise they would not build their assets). As such, solar plants will have little exposure to these very low prices until their contracts come to an end in the 2030s and 2040s.
  - **However**, capture prices for solar have shown no sign of slowing their decrease. Structurally, as Europe keeps adding solar power, the revenues achievable by solar assets on the power markets will keep decreasing due to cannibalisation. When solar assets currently under government contracts reach the end of their tenor (including dozens of GWs built pre-2020 with LCOEs far higher than today) they will face a serious shortfall in required revenues. Eg. a solar asset built with an LCOE of >€100/MWh in the early 2010s could face a massive shortfall in revenues if traded only on the markets where solar achieves capture prices of €30-40/MWh today. **LCOE is a lifetime levelised benchmark; if post-support merchant revenues are far below the levelised cost assumed at investment, lifetime economics can be impaired.**
  - <img width="1280" height="944" alt="image" src="https://github.com/user-attachments/assets/7ccf64aa-041a-47b6-bb62-f9fed6491b31" />
  - This could create a new generation of "stranded" assets without further government contracts/support: assets too uneconomic to justify continued function due to capital unable to achieve its required return.
  - In my view governments and grid operators have three choices here (ii. is my most likely and preferred option):
    - i. Do not renew government contracts and let these assets fail to meet capital return: very bad for continued investment, and unlikely if we want to pursue decarbonisation of our energy
    - ii. Offer lifetime or extended government contracts in exchange for grid operator control over solar production: this would take most solar assets into government hands, significantly lower risk and cost of capital, and allow grid operators to shutdown solar when there is excess production. However, this would be a net loss to the government as they are effectively subsidising solar at a power price much higher than its market capture price.
    - iii. Do not renew government contracts but mandate that all utilty-scale solar must install on-site battery storage to shift production to peak demand hours: this would require increased investment by "stranded asset" owners and does not immediatley fix the economics issue if Shifted Power Price < LCOE_solar + LCOS_BESS. For old solar plants with €100/MWh LCOE, adding BESS at €50/MWh LCOS requires shifted energy to be sold at >€150/MWh, possible but risky.
    
3. **Forecast errors have a non-neligeable cost effect on capture prices for solar.** Across the three geographies in question, costs due to forecasting errors range between 4-7% of the day-ahead solar capture price. This effectively reduces day-ahead revenues achievable by solar assets by 4-7%.
   - ID-Actual Fleet RMSE as a % of fleet size has been increasing since 2020: Spain's RMSE has risen from 3.9% to 7.4%, France from 9.3% to 9.6%, Germany from 2.6% to 3.0%
   - Assuming rho=0.75 (strongly correlated asset fleet), ID-Actual asset RMSEs are ~1.5x higher than at the fleet level.
   - As discussed above, increased asset-level RMSEs have no material impact for Germany which has a symmetric imbalance price. However for non-symmetric imbalance regimes (eg. France, Spain) this has a material on-linear impact on increasing error cost. In 2024, fleet-level ID-ACT error cost was €1.53/MWh & €0.56/MWh for Spain and France respectively (see results table). Asset-level error cost were calculated to be €2.80/MWh (+82%) & €1.01/MWh (+81%) - a ~80% increase in error costs from fleet to asset disaggregation.
   - For business case modelling, forecasting error costs can no longer be ignored as a long-term assumption, but also as a key area for further investment and research. Between 2020-2025, forecasting errors at the fleet level for this analysis have cost c.€2.2bn to asset operators, and likely more at the asset level. Even 10-20% reductions in RMSE could make a significant difference due to the non-linear nature of error costs.
  
# 8. Project limitations and improvements

The project is limited by the lacking of some key data:
- I have used the DA price as a proxy for the ID price. Whilst in practice these two tend to converge and provide a directionally accurate analysis here, they are not the same, and can meaningfully diverge during stressed times for the grid. A complete study of forecasting errors would incorporate ID prices.
- I have used the maximum observed generation as a proxy for fleet size. True fleet size may be higher than this due to grid connections being undersized compared to total asset power. A higher actual fleet size would lower the RMSE-% results shown in the analysis.
- Modelling for the imbalance price here is simplistic. In reality, balancing in the EU is achieved across a variety of different products rather than one single imbalance market (FCR, aFRR, mFRR). Nonetheless, I believe the prices downloaded on ENTSO-E reflect an aggregation of costs borne by the grid operator/operators across these different products.
- In real life, good asset operators will continuously update their production forecasts, rather than relying on the DA and then single ID forecast updates. Weather modelling has made vast speed and accuracy improvements in recent years. A good solar operator would seek to limit as much as possible their exposure to imbalance, and would regularly update their forecasts to avoid this unpredictable market. This would naturally depress the magnitude of errors and error costs analysed above.
  - A more rigorous analysis here would implement more refined and closer to time forecasts. That being said, I have anecdotally heard that even with very strong models high single-digit to low double-digit RMSEs for ID-Actual forecast errors are still common.
- However, I find confidence that the results of this analysis are directionally accurate base on experience from assumptions made in financial models. A good solar financial modeller will assume €2-3/MWh for "balancing costs", either incurred directly by the solar operator, or outsourced for a fee to a "route-to-market" provider who handles forecasting and balancing for the solar operator. From anecdote these tend to cost €1.5-3.0/MWh.
- **Regardless, my conclusion remains the same: solar forecasting errors are a net cost to solar assets in the region of low single digit €/MWh, and improving solar forecasts through AI or on-site technology (eg. all-sky cameras) would be a worthwhile investment: a €1/MWh improvement in revenue for the EU's entire ~400TWh production in 2025 is worth c.€400m to solar operators.**






