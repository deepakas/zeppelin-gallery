%md 
# Zeppelin example on Graph Visualisation using wikipedia webanalytics data - D3 Force Layout and Sankey Chart 

### The example is based on the databricks example from this link. 
https://docs.cloud.databricks.com/docs/latest/featured_notebooks/Wikipedia%20Clickstream%20Data.html

### The wikipedia web analytics data used is downloaded from the following website
https://datahub.io/dataset/wikipedia-clickstream

## Here are the steps used in the example

1. Add dependencies for databricks csv package. 
2. Download the wikipedia file and unzip it .
3. Read the file using spark dataframes
4. Write it to parquet format for fast reload on restarting Zeppelin. 
5. Load parquet file and register as a table
6. Summarise using Spark SQL
7. Create the reusable graph function 
8. Create the reusable displayForceLayout function to display in Force Layout format 
9. Create the reusable displaySankeyLayout
10. Run Example  on big cities

## 1. Add dependencies for databricks csv package. 
```
    %dep
    z.reset()

// Add spark-csv package
z.load("com.databricks:spark-csv_2.11:1.4.0") 
```
