# Preparing Data for Analysis
* * *

## Types of Data
* Database Data Types(string,numeric,boolean,date time)
* Structured vs Unstructured
* Quantitative vs Qualitative Data
* First-, Second, and Third-Party Data
   * First-party data: collected by the organization itself.
   * Second-party data: which comes from vendors that provide a service or perform a business function on the organisation's behalf.
   * Third-party data: data that may be purchased or obtained from free sources such as those published by the government.
* Sparse data: A small amount of information within a larger set of empty or unimportant information.

## Profiling
* The first thing to do when starting working with any new data set.
* Checking what domain or domains are covered
* Having knowledge of how the data was generated
* Checking for how history is represented

### Distributions
Distribution allows us to understand the range of values that exist in the data and how often they occur whether there are nulls and whether negative values exist alongside positive ones.

#### Histograms and Frequencies
* Checking the frequency of values in each field (To know a data set, particular fields within the data set)
* Useful when having a question about
    1. Whether certain values are possible
    2. If spotting an unexpected value and want to know how commonly it occurs
* Frequency checks can be done on any data type(strings, numerics, dates, and booleans)
* Frequency queries are a great way to detect sparse data
* Frequency Query
  ```sql
select fruit,count(*) as quantity
from fruit_inventory
group by 1;
```
