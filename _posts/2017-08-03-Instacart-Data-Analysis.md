# Instacart Analysis
In May, Instacart released a portion of their [order history](https://www.instacart.com/datasets/grocery-shopping-2017) for ML research and competitions.  Because I currently work in Consumer Goods, this data really interested me because it provided a look into online grocery shopping that I'm unable to obtain through our usual data sources in the office.

Fortunately, our team was recently able to hold a 3 hour 'hackathon' to work with this data &amp; learn about its contents.   Due to time constraints I pursued a more descriptive analysis; my findings from this analysis (and my one slide presentation) are shown below:

---
## Data Import + Wrangling

I used elements of the `tidyverse` to wrangle and visualize this data.

``` r
library(readr)
library(dplyr)
library(ggplot2)
```

Due to Instacart's data terms-of-service, I cannot include the data itself within my public repository. You will need to go out to the link above and download the data if you wish to replicate this analysis.

``` r
Aisles = read_csv("/Users/Matthew/GitHub/InstacartHackaton/Data/aisles.csv",progress = F)
Departments = read_csv("/Users/Matthew/GitHub/InstacartHackaton/Data/departments.csv",progress = F)
Orders = read_csv("/Users/Matthew/GitHub/InstacartHackaton/Data/orders.csv",progress = F)
Products = read_csv("/Users/Matthew/GitHub/InstacartHackaton/Data/products.csv",progress = F)
```

Prior orders were split into a test and training set for machine learning purposes. Since I was just conducting an exploratory analysis, I decided to get greedy and combine both into a complete order history dataframe.

``` r
Order_Products = read_csv("/Users/Matthew/GitHub/InstacartHackaton/Data/order_products__prior.csv",progress = F)
Order_Products_Train = read_csv("/Users/Matthew/GitHub/InstacartHackaton/Data/order_products__train.csv",progress = F)
Order_Products = bind_rows(Order_Products,Order_Products_Train)
```

To quickly get my data in a workable format for analysis, I joined all of my data frames together using the following `dplyr` chain.  As you might expect, this was an expensive operation, but the machine I used has 32GB of RAM, which prevented roadblocks under the time crunch.

``` r
Full_Dataset = Order_Products %>% 
  left_join(Products) %>% 
  left_join(Aisles) %>% 
  left_join(Departments) %>% 
  left_join(Orders)
```

Once I had a full dataset stored within a single dataframe, I was able to begin querying and investigating.

---
## Approach / Questions

Although I explored a few extra questions during the hackathon itself, this post focuses on two of the more interesting ones:

-   What's the relationship between repeat purchases (at the item level) and the order in which items were placed in the basket?
-   At what point is it nearly certain that a first-time item is in someone's basket?

## Considerations

Many users in the data were still fairly new, and that can interfere with repeat purchasing questions. In addition, some orders had hundreds of items in the basket, and we weren't interested in studying super-large baskets. As a result, we limited baskets to only include:

-   Baskets belonging to shoppers who had already conducted 9 or more orders with Instacart, and
-   We only focused on the first 25 items placed in each basket.

``` r
Reorders = Full_Dataset %>% 
  group_by(order_id) %>% 
  mutate(n_prods = max(add_to_cart_order)) %>% 
  filter(add_to_cart_order <= 25,order_number >= 10) %>% 
  group_by(add_to_cart_order) %>% 
  summarise(Reordered = mean(reordered)) 
```
## Some Preliminary Findings...

I created a plot to show the proportion of items that were reordered based on the 'position' of the item in all baskets. For qualifying Instacart shoppers, ~75% of all items ordered are reorders, shown using the horizontal line. The earlier an item is added to a basket, the more likely it has been ordered in the past.

I think this is probably driven by two things:

1.  Grocery staples like milk, soda, and produce are usually ordered early on in bigger transactions, and also
2.  I *believe* smaller orders (for use cases like late night ice cream, Sunday beer run, etc.) would generally contain previously purchased, indulgence-related items.

``` r
Mean = Full_Dataset %>% 
  filter(add_to_cart_order <= 25, order_number >= 10) %>% 
  ungroup()  %>% 
  summarise(mean(reordered))

ggplot() + 
  geom_point(data = Reorders, aes(x = add_to_cart_order,y = Reordered)) + 
  geom_hline(aes(yintercept = Mean)) + 
  labs(x = "Position of basket addition",
       y = 'Probability an item added at this \n point has been purchased before',
       title = "When do new purchases occur?") +
  scale_y_continuous(labels = scales::percent)
```

<img src="https://matthewbrower.github.io/img/Hackathon_Analysis_files/figure-markdown_github/unnamed-chunk-2-1.png">

Another way to look at this is to quantify the cumulative probability a shopper has added a 'new' item to their basket as it grows in size. The chart below highlights an interesting finding - by the time one of our repeat shoppers has placed 15 items in their basket, it's almost 100% probable that one of those 15 items is new to that shopper.  

The strength of this conclusion varies based on the filters applied above, but I still find this interesting and share some possible actions below.

``` r
Reorders$Prob = (1 - cumprod(Reorders$Reordered))

ggplot() + 
  geom_path(data = Reorders, aes(x = add_to_cart_order,y = Prob)) +  
  geom_point(data = Reorders, aes(x = add_to_cart_order,y = Prob)) +
  labs(x = "# of items in basket",
       y = 'Probability of at least\n one new item in the basket',
       title = "When are new items a near-certainty in the basket?") +
  scale_y_continuous(labels = scales::percent)
```
<img src="https://matthewbrower.github.io/img/Hackathon_Analysis_files/figure-markdown_github/unnamed-chunk-3-1.png">

## Possible Outcomes
I would take these learnings and recommend adjustments to Instacart's recommender algorithm:

-   For shoppers early on in an online trip, I believe shopper convenience can be maximized by recommending grocery staples and items that shopper has routinely purchased in the past.
-   Once a handful of items are in the basket, Instacart can then work to upsize orders by suggesting context-relevant (i.e. frequently cross-purchased with in-basket items) products that the shopper has NOT purchased in the past.
