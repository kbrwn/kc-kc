AWS Node Price Reconciliation Methodology
=========================================

Kubecost is capable of aggregating the costs of EC2 compute resources over a given timeframe with a specified duration step size. To achieve this, Kubecost uses Athena queries to gather usage data points with differing price models. The result of this process is a list resources with their cost by timeframe.

## Athena Queries

The reconciliation process makes two queries to Athena, one to gather resources that are paid for with either the on-demand model or a savings plan and one query for resources on the reservation price model. The First query includes resources given at a blended rate, which could be on-demand usage or resources that have exceeded the limits of a savings plan. It will also include resourses which are part of a savings plan which will have a savings plan effective cost. The second query only includes reserved resourse and the cost which reflects the rate they were reserved at.

The Queries make use of the folloing columns from Athena:

- `line_item_usage_start_date` The beginning timestamp of the line item usage. Used to filter resource usage within a date range and to aggregate on usage window.

- `line_item_usage_end_date` The ending timestamp of the line item usage. Used to filter resource usage within a date range and to aggregate on usage window.

- `line_item_resource_id` An id, also called the provider id, given to line items that are instantiated resources.

- `line_item_line_item_type` The type of a line item, used to determine if the resource usage is covered by a savings plan and has a discounted price.

- `line_item_usage_type` What is being used in a line item, for the purposes of a compute resource this, is the type of VM and where it is running

- `line_item_product_code` The service that a line item is from. Used to filter out items not from EC2.

- `reservation_reservation_a_r_n` Amazon Resource Name for reservation of line item, the presense of this value is used to identify a resource as being part of a reservation plan.

- `line_item_unblended_cost` The undiscounted cost of a resource.

- `savings_plan_savings_plan_effective_cost` The cost of a resource discounted by a savings plan

- `reservation_effective_cost` The cost of a resource discounted by a reservation



#### On-Demand/Savings Plan Query
This query is grouped by six columns: `line_item_usage_start_date`, `line_item_usage_end_date`, `line_item_resource_id`, `line_item_line_item_type`, `line_item_usage_type` and `line_item_product_code`. The columns `line_item_unblended_cost` and `savings_plan_savings_plan_effective_cost` are summed on this grouping. Finally the query filters out rows that are not within a given date range, have a missing `line_item_resource_id` and have a `line_item_product_code` not equal to "AmazonEC2". The grouping has three important aspects, the timeframe of the line items, the resource as defined by the resource id and the usage type, which is later used to determine the proper cost of the resources as it was used. This means that line items are grouped according to the resource, the time frame of the usage, and the rate at which the usage was charged.

#### Reservation Query
The reservation query is grouped on five columns: `line_item_usage_start_date`, `line_item_usage_end_date`, `reservation_reservation_a_r_n`, `line_item_resource_id` and `line_item_product_code`. The query is summed on the `reservation_effective_cost` and filtered by the date window, for missing `reservation_reservation_a_r_n` values and also removes line items with `line_item_product_code` not equal to "AmazonEC2". This grouping is affectively on resource id by timeframe removing all non-reservation line items

## Processing Query results

The on-demand Query is categorized into different resource types: compute, network, storage and other. Network is identified by the presence of the "byte" in the `line_item_usage_type`. Compute and storage are identified by the presence of "i-" and "vol-" prefixes in `line_item_resource_id` respectively. Non compute values are removed from the results. Out of the two costs aggregated by this query the correct one to use is determined by the `line_item_line_item_type`, if it has a value of "SavingsPlanCoveredUsage", then the `savings_plan_savings_plan_effective_cost` is used as the cost and if not then the `line_item_unblended_cost` is used.

In the reservation query all of the results are of the compute category and there is only the `reservation_effective_cost` to use as cost

These results are then merged into one set, with the provider id used to associate the cost with other information about the resource.

## Why doesn't this match the AWS Cost Explorer?

There's a number of different ways to look at your node cost data. The default for the cost explorer is "unblended" but it makes the most sense from an allocation perspective to use the "amortized" rates. Be sure "amortized" is selected when looking at cost data. Here's an example of how they can vary dramatically on our test cluster.

The t2-mediums here are covered by a savings plan. Unblended, we're only being charged $0.06/day for two of them.
<img width="1047" alt="Screen Shot 2021-04-22 at 11 00 09 PM" src="https://user-images.githubusercontent.com/453512/115825827-d229ca80-a3be-11eb-961f-6ae66f56c605.png">

When amortized costs are selected, the price jumps to $1.50/day
<img width="1427" alt="Screen Shot 2021-04-22 at 11 05 28 PM" src="https://user-images.githubusercontent.com/453512/115826171-52e8c680-a3bf-11eb-8e47-fd62a2e9a44b.png">

Which should closely match our data on the Assets page, for days where we have adjustments come in from the pricing CUR.
<img width="1792" alt="Screen Shot 2021-04-22 at 11 03 45 PM" src="https://user-images.githubusercontent.com/453512/115825960-08ffe080-a3bf-11eb-9968-881f70d11e3c.png">

Edit this doc on [Github](https://github.com/kubecost/docs/blob/main/aws-node-price-reconcilitation-methodology.md)

<!--- {"article":"4407596809111","section":"4402829036567","permissiongroup":"1500001277122"} --->