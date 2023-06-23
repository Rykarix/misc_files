# Pre analysis exploration
First I brought up the documentation from the following websites:
- [BigQuery Docs](https://cloud.google.com/bigquery/docs)
- [GA4 - BigQuery Export schema](https://support.google.com/analytics/answer/7029846#zippy=)

Then I broke down the event names in BigQuery by using the following SQL:
```sql
select
  event_name,
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
group by 1
order by 1
```
[Big Query - Exploratory code - 0 List event_name ](https://console.cloud.google.com/bigquery?sq=596530739646:facc2bec24c34a99a184a3f7ad809d18)
Output:

|event\_name|
|---|
|add_payment_info|
|add_shipping_info|
|add_to_cart|
|begin_checkout|
|click|
|first_visit|
|page_view|
|purchase|
|scroll|
|select_item|
|select_promotion|
|session_start|
|user_engagement|
|view_item|
|view_item_list|
|view_promotion|
|view_search_results|

# Analysis 1 - Traffic Mediums
```sql
  -- Purpose: Explore avenues to increase revenue by improving traffic flow through different mediums

   -- With more time:
   -- Further breakdown by continent/country/geolocation etc
   -- Deeper dive per medium. Exactly where is the traffic coming from?

   --format percentages to 3 sig fig
  create temp function toPercentage(number float64) as (round(100*number + 0.0005, 2));

   --format floats to 3 sig fig
  create temp function to3SF(number float64) as (round(number + 0.0005, 2));

  select
    -- Clean up
    case when traffic_source.medium = 'referral' then 'Referral' -- See https://support.google.com/analytics/answer/11080067
         when traffic_source.medium = 'organic' then 'Organic'
         when traffic_source.medium = 'cpc' then 'Paid search (CPC)'
         when traffic_source.medium = '<Other>' then 'UNKNOWN'
         when traffic_source.medium = '(none)' then 'Direct' -- compared to traffic_source.name to find this
         when traffic_source.medium = '(data deleted)' then 'UNKNOWN'
         else traffic_source.medium
    end as medium,

    -- Total views
    count(case when event_name = 'view_item' then 1 else null end) as total_views,

    -- Total revenue in USD
    sum(case when event_name = 'purchase' then items.item_revenue_in_usd else null end) as total_revenue_usd,

    -- Revenue per view
    to3SF(
    sum(case when event_name = 'purchase' then items.item_revenue else null end)/count(case when event_name = 'view_item' then 1 else null end)
    ) as rev_per_view,

    -- Percentage of views that resulted in a purchase
    toPercentage
    (
      count(distinct case when event_name = 'purchase' then user_pseudo_id else null end)/
      count(distinct case when event_name = 'view_item' then user_pseudo_id else null end)
    ) as percent_views2purchase,

    -- Percentage of views that resulted in items being added to cart
    toPercentage
    (
      count(distinct case when event_name = 'add_to_cart' then user_pseudo_id else null end)/
      count(distinct case when event_name = 'view_item' then user_pseudo_id else null end)
    ) as percent_views2cart,

    toPercentage
    (
      count(distinct case when event_name = 'purchase' then user_pseudo_id else null end)/
      count(distinct case when event_name = 'add_to_cart' then user_pseudo_id else null end)
    ) as percent_cart2purchase,

    -- Total USD of items added to cart but not purchased
    sum(case when event_name = 'add_to_cart' then items.price else null end) - sum(case when event_name = 'purchase' then items.item_revenue else null end) as add2cart_not_purchased,

  from `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`,
    unnest(items) as items
  -- where _table_suffix between '20210101' and '20210105' -- small processing size for testing
  group by medium
  order by rev_per_view DESC;

    -- ################
    -- # Redundant code
    -- ################

    -- -- Total number of items purchased
    -- sum(case when event_name = 'purchase' then items.quantity else null end) as items_purchased,
    -- -- Total number of transactions
    -- count(case when event_name = 'purchase' then ecommerce.transaction_id else null end) as total_transactions,
    -- -- Total number of times items were added to cart
    -- count(case when event_name = 'add_to_cart' then 1 else null end) as total_carts,

    -- -- Total revenue in USD
    -- sum(case when event_name = 'purchase' then items.item_revenue else null end) as total_revenue_usd,
```
[BigQuery - Analysis 1 - Traffic Mediums](https://console.cloud.google.com/bigquery?sq=596530739646:f9f9c76de60741acb6060f7cfc2b6dfc)

# Analysis 2 - View:Buy ratios
```sql
-- Purpose:
-- To determine which products have the highest view:buy ratio

-- Possible recommendations for high view:buy ratios:
-- MKTG/SALES: Ad campaigns for specific products
-- IT(Webdevs)/MKTG: More visibility for popular products on site

-- Possible recommendations for low view:buy ratios:
-- MKTG/SALES/Finance: Research product popularity vs production costs - Continued production worthit?

-- Improvements that could be made to the dataset:
-- Removal of discontinued products from table (as no relavence to purpose)

-- Ideas/additions for visualisation:
-- Add item categories, useful for interactive dashboards - "zoom" into different views, IE high level summary via categories vs individual breakdown of each item

-- format floats to 3 sig fig
create temp function to3SF(number float64) as (round(number + 0.0005, 2));

with table_view2buy as(
  select
    item_name,
    count(case when event_name = 'view_item' then 1 else null end) as unique_views,
    count(case when event_name = 'purchase' then 1 else null end) as unique_buys,

  -- total revenue
  to3SF(
    sum(case when event_name = 'purchase' then items.item_revenue_in_usd else null end)
    ) as total_revenue_usd,

  -- View to buy ratio
    to3SF(
    case when count(case when event_name = 'view_item' then user_pseudo_id else null end) = 0 then 0 -- prevent div by 0 error
    else
      count(distinct case when event_name = 'purchase' then user_pseudo_id else null end)/
      count(distinct case when event_name = 'view_item' then user_pseudo_id else null end)
    end
    ) as ratio_view2buy,

  from
    `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`,
    unnest (items) as items -- required for calling columns in the items record
  -- where _table_suffix between '20201101' and '20201225' -- XMAS Period -
--   where _table_suffix between '20201225' and '20210131' -- POST XMAS Period -
  group by
    item_name
  order by
    ratio_view2buy DESC
)

select
  item_name, unique_views, unique_buys, total_revenue_usd, ratio_view2buy
from
  table_view2buy
where
  item_name != "(not set)"
```
[BigQuery - Analysis 2 - View2Buy ratios](https://console.cloud.google.com/bigquery?sq=596530739646:1053368afd344a3bbb362c56cab80845)

# Analysis 3 - Transaction process
This analysis is comprised of 2 sql queries and some python code. The purpose of each piece of code has been included in the code

```sql
-- Purpose: To yeild statistical data per transaction. Ie average transaction size, average revenue per transaction etc

with transactions as(
  select
    ecommerce.transaction_id,
    sum(ecommerce.total_item_quantity) as total_item_quantity,
    sum(ecommerce.unique_items) as unique_items,
    sum(ecommerce.purchase_revenue_in_usd) as purchase_revenue_in_usd,
  from
    `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  where
    event_name = 'purchase'
  group by transaction_id)

select
  transaction_id, total_item_quantity, unique_items, purchase_revenue_in_usd
from
  transactions
where
  transaction_id != "(not set)"
```
[BigQuery - Analysis 3.1 - Transaction process](https://console.cloud.google.com/bigquery?sq=596530739646:eebbb05ce6154f34b0b1c43dd7c3c00c)

```sql
-- Purpose: To provide trend analysis over the course of the 3 months this dataset provided
-- Reasoning: Having previously recognised and commented on the volitiliy of this timeframe, XMAS + COVID, I thought I'd kill 2 birds with 1 stone by:
  -- 1 Proving it
  -- 2 Using it as part of the given task to increase top line revenue

with transactions as(
  select
    event_date,
    sum(ecommerce.total_item_quantity) as total_item_quantity,
    sum(ecommerce.unique_items) as unique_items,
    sum(ecommerce.purchase_revenue_in_usd) as purchase_revenue_in_usd,
  from
    `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  where
    event_name = 'purchase'
  group by event_date)

select
  event_date, total_item_quantity, unique_items, purchase_revenue_in_usd
from
  transactions
where
  event_date != "(not set)"
order by
  event_date
```
[BigQuery - Analysis 3.2 - Transaction process](https://console.cloud.google.com/bigquery?sq=596530739646:a2b9a0a0b49445e19a0fe8ececd233fc)

```py
!pip install pandas_bokeh # Required for pd.set_option below

import pandas as pd # Handle dataframes

from bokeh.plotting import figure, show, output_notebook # Beautiful & Interactive dashboards via bokeh, see https://bokeh.org if interested
import warnings
warnings.filterwarnings('ignore') # Suppress nuisance messages
output_notebook() # Required for .ipynb display
pd.set_option('plotting.backend', 'pandas_bokeh') # Change pandas plotting backend to use pandas_bokeh. No need to import library, just needs to be installed

# Create a cleaned dataframe for bokeh plot:
revenue_over_time = results.copy() # Preserve results data
revenue_over_time["event_date"] = pd.to_datetime(revenue_over_time["event_date"], format='%Y%m%d') # Reformat date column to date_time
revenue_over_time = revenue_over_time[["event_date","purchase_revenue_in_usd"]].set_index("event_date") # group by 2 columns of interest & set index to date
revenue_over_time.head(3) # Quick review of changes
```

|event\_date|purchase\_revenue\_in\_usd|
|---|---|
|2020-11-01 00:00:00|773\.0|
|2020-11-02 00:00:00|4789\.0|
|2020-11-03 00:00:00|3313\.0|
```py
# Create dashboard widget
fig = revenue_over_time.plot_bokeh.line(
    figsize=(1600,500),
    zooming=True, # Enable zoom interactivity by default
    title = "Trend analysis over Christmas period, 01 Nov 2020 to 31 Jan 2021",
    toolbar_location = "left",
    legend = "top_left",
    ylabel = "Revenue in USD",
    xlabel = "Event Date",
    vertical_xlabel=True # Set labels to vertical for cleaner graph
    );
```
![Output](../BigQuery%20Task/data/bokeh_plot.png "Output")
or [click here to run via google collab](https://colab.research.google.com/github/Rykarix/lhtask/blob/main/transactional_trend_analysis.ipynb)