# Wine-Terroir
A quick scraping and analysis project to see whether there is a correlation between "good" weather and "good" vintages. We scrape the objectively *good* vintages by region from [wine-spectator](https://www.winespectator.com/vintage-charts) and then enrich it with weather data from meteostat. We try to find correlations between pre-defined weather metrics which lead to better wine quality and the score of the vintage rankings.

## Scraping
We scrape the website and its sub-sites with ```beautifulSoup``` and store the data in a JSON file. This approach is detailed in a [Jupyter Notebook](https://github.com/trashpanda-ai/Wine-Terroir/blob/main/1.%20Scraping.ipynb).

## Cleaning
We need to adjust some of the data, since it isn't coherent. The information is not structured identical for each sub-page, so we make sure the country, region and grape variety is consistent. Also some of the regions need to be cleaned. This approach is also detailed in a [Jupyter Notebook](https://github.com/trashpanda-ai/Wine-Terroir/blob/13844a79e441c1b0ee02f5d048f9470608d44dd8/2.%20Cleaning.ipynb).

## Enrichment
In order to enrich our scraped data, we obtain weather data based on the wines exact region and vintage. We follow a matryoshka approach of nesting an API in an API. The GPS location is extracted from the vivino.com region via Geopy. And then we can obtain a weather time series based on the GPS via meteostat. But a time series != feature. So we need to leverage expert domain knowledge to turn our raw time series in substantial features. Our feature engineering approach is based on [idealwine](https://www.idealwine.info/conditions-for-great-wine/) [.info](https://www.idealwine.info/conditions-necessary-great-wine-part-12/) for the growth period (March 11th -- September 20th) of the wines [Source](https://en.wikipedia.org/wiki/Harvest_(wine)):
> A fairly cold winter, a spring with alternating periods of sunshine and occasional rain, a fairly hot and dry summer with the odd shower to ward off water stress, and rain-free harvesting… conditions like these will virtually guarantee a good year.

> At temperatures below 10°C and above 35°C, photosynthesis will be disrupted, and vines will not grow properly. 

> Vines need between 400 and 600 mm of rain per year. […] A regular supply of water throughout the growth cycle is needed for a high-quality crop.

> Too much rain and damp during the May-July period can lead to the appearance of diseases such as mildew or oidium, which are caused by the growth of tiny fungi.

> If there is rain or […] strong wind during flowering, the pollen will be unable to achieve its task of pollinating the flowers. This is known as 'coulure'. 


For the desired weather data, we average the $n$ closest stations' data for a more robust and granular time series.

We obtain the (Region + Year)-Tuples from the wine data (which are unique, thus no *expensive* meteostat API call needs to be done twice) and for each of those we generate the following specifically engineered features for the growth period (March 11th -- September 20th) of the wines:
- ```Vola_Temp```: Volatility of temperature
- ```Vola_Rain```: Volatility of rain
- ```Longest_Dry```: Longest period without rain
- ```Longest_Wet```: Longest period with consecutive rain
- ```Avg_Rain```: Average rain fall
- ```Count_above35```: Number of days with temperature above 35 degrees
- ```Count_under10```: Number of days with temperature below 10 degrees
- ```Count_under0 ```: Number of days with temperature below 0 degrees
- ```Coulure_Wind```: Windspeed during the flowering
- ```June_Rain```: Rain during the May-July period leading to mildew or oidium

These $10$ features are then appended to the dataframe and exported as CSV file. This approach is also detailed in a [Jupyter Notebook](https://github.com/trashpanda-ai/Wine-Terroir/blob/13844a79e441c1b0ee02f5d048f9470608d44dd8/3.%20Enrichment.ipynb).

## Analysis
We quickly analyze the merged data after some more pre-processing and calculate the correlations in a heatmap:
![Heatmap](https://github.com/trashpanda-ai/Wine-Terroir/blob/main/plots/Heatmap%20Enrichment.png?raw=true)



We also conduct an ANOVA analysis to find the most influential weather metrics for our ```Score```, ```Vintage``` and ```Drinking Window```:
![ANOVA Score](https://github.com/trashpanda-ai/Wine-Terroir/blob/main/plots/ANOVA%20Score%20Enrichment.png?raw=true)

![ANOVA Drinking Window](https://github.com/trashpanda-ai/Wine-Terroir/blob/main/plots/ANOVA%20Drink%20Window%20Enrichment.png?raw=true)


This approach is also detailed in a [Jupyter Notebook](https://github.com/trashpanda-ai/Wine-Terroir/blob/f3966d60297df71535ad7c9b3694915caea79c1b/4.%20Analysis.ipynb).


## Modelling
This will be the most interesting aspect: Is it possible to predict the upcoming wine ratings solely based on the current weather data? 

Unfortunately, there is no predictive signal in the current weather data. Maybe a better weather API will render different results. Or the feature engineering based on expert knowledge doesn't fit this problem statement. The results of XGBoost:
| Metric | Value |
| --- | --- |
| R-square score | -0.152839 |
| Adjusted R-square score on test set | -0.154807 |
| Mean Squared Error (MSE) on test set | 17.229869 |
| Absolute Error (MAE) on test set | 3.019083 |

This approach is detailed in a [Jupyter Notebook](https://github.com/trashpanda-ai/Wine-Terroir/blob/main/5.%20Modelling.ipynb).

## Comparison between meteostat and open-meteo
Is there a better weather provider in terms of correlations? This means we do not care about accuracy of provided weather data, but only about the predictive quality our models can provide.
