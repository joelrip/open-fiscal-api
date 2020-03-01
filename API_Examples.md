---
title: "API_Examples"
output: html_document
---
  
```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

This document contains embedded R code that demonstrates how to access the demo Open FI$Cal API, located at https://catalog.ogopendata.com/.

This version, the .md file, can be viewed more easily on Github. The code can be copied and pasted into R to run and see the resutls. The other version of this file, the .Rmd file, can be knit within RStudio to produce an HTML document that runs all of the embedded code and displays the results.

See also the documentation for the [CKAN API](https://docs.ckan.org/en/ckan-2.7.3/api/) and the [CKAN Datastore API](https://docs.ckan.org/en/ckan-2.7.3/maintaining/datastore.html#the-datastore-api).

## Load Necessary Packages
                                                                                                                     
If you don't have these packages installed, install them with `install.packages("packagename")`.

```{r load, results = "hide"}
library("tidyverse") #a set of packages that make working in R more efficient
library("httr") #tools for working with URLs
library("jsonlite") #JSON parser and generator
library("data.table") #more efficient extention of R's base data.frames
```
## Get Resource IDs

This script uses the CKAN API to get all the information about the `california-expenditure-data` package (i.e. dataset) and extract the IDs for each of the resources that can be searched with the datastore API. For this particular dataset, there is one resource (i.e. file) for each month of data.

This is an important step, as the resulting list of resource IDs will allow future scripts to loop through all of the resources to get information on all months.

```{r get_resources}
pkg_show <- GET("https://catalog.ogopendata.com/api/3/action/package_show?id=california-expenditure-data") #gets info on the "california-expenditure-data" package
pkg_show_text <- content(pkg_show, "text") #extracts the text content
pkg_show_JSON <- fromJSON(pkg_show_text, flatten = T) #turns the JSON response into a usable format
pkg_show_resources <- pkg_show_JSON$result$resources #extracts just the results$resources table from the response
resource_ids <- pkg_show_resources$id[which(pkg_show_resources$datastore_active)] #creates a vector of just the resource IDs that have indexed, searchable data (are "datastore active") - WILL BE USED LATER
resource_ids
```

## Code Demos Using One Resource

### Get the first 5 rows of data, skipping the first, from a specific resource.

NOTE: On this demo server, this request sometimes results in a server error. If this happens, choose a different resource, paste it into the appropriate place in the URL, and try again.

```{r five_rows}
five_rows <- GET("https://catalog.ogopendata.com/datastore/odata3.0/3e5b9f4e-c1dc-4752-bb38-096d480ca3f1?$top=5&$skip=1&$format=json")
five_rows_text <- content(five_rows, "text")
five_rows_JSON <- fromJSON(five_rows_text, flatten = T)
five_rows_JSON
```

### Use SQL to get all vendors like `%HARLEY%` from a specific resource.

```{r harley}
harley <- GET("https://catalog.ogopendata.com/api/3/action/datastore_search_sql?sql=SELECT%20*%20from%20%22cc491680-1eb0-42d2-8dac-47a2f1204d3e%22%20WHERE%20%22Vendor%20Name%22%20LIKE%20%27%HARLEY%%27")
harley_text <- content(harley, "text")
harley_JSON <- fromJSON(harley_text, flatten = T)
harley_records <- harley_JSON$result$records
harley_records <- harley_records[ , -24] #remove "_full_text" field for easier display
head(harley_records)
```

### Use SQL to get all business units like `2720` from a specific resource.

Each department has a four-digit business unit code. 2720 is the California Highway Patrol.

```{r test_dept}
test_dept <- GET("https://catalog.ogopendata.com/api/3/action/datastore_search_sql?sql=SELECT%20*%20from%20%22cc491680-1eb0-42d2-8dac-47a2f1204d3e%22%20WHERE%20%22Business%20Unit%22%20LIKE%20%272720%27")
test_dept_text <- content(test_dept, "text")
test_dept_JSON <- fromJSON(test_dept_text, flatten = T)
test_dept_records <- test_dept_JSON$result$records
test_dept_records <- test_dept_records[ , -24]
head(test_dept_records)
```

## Use a loop to get data from all resources

Using the `resource_ids` vector we created earlier, this script loops through all of the resource IDs and gets information for the specified business unit. The example uses BU 8880, which is the Department of FISCal.

```{r all_resources}
FISCal_list <- vector(mode = "list", length = length(resource_ids)) #creates a list to drop the retrieved data into; in this case, it will be a list of data frames
BU = 8880 #can be any business unit you want
count = 0
for (item in resource_ids) {
  count = count + 1
  temp <- GET(paste0("https://catalog.ogopendata.com/api/3/action/datastore_search_sql?sql=SELECT%20*%20from%20%22", item, "%22%20WHERE%20%22Business%20Unit%22%20LIKE%20%27", BU , "%27")) #gets the data for the appropriate resource and BU
  temp_text <- content(temp, "text")
  temp_JSON <- fromJSON(temp_text, flatten = T)
  temp_records <- temp_JSON$result$records
  FISCal_list[[count]] <- temp_records #after previous steps format the data correctly, this drops it into the appropriate spot in the list
}
FISCal <- rbindlist(FISCal_list) #data.table method to quickly combine all list items into one data frame
FISCal <- FISCal[ , -24]

###Plot number of transactions by month
FISCal_month_plot <- FISCal %>%
  ggplot(aes(x = as.Date(`Accounting Date`))) +
  geom_histogram(fill = "steelblue", color = "white", bins = 41) +
  scale_x_date() +
  theme_minimal() +
  labs(title = "Number of Transactions Per Month")
FISCal_month_plot
```
