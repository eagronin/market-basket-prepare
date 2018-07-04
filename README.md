# Data Preparation

## Overview
This section summarizes the process of initial exploration and cleaning of the data from a UK-based online retailer.  The data are used in the [next section](https://eagronin.github.io/market-basket-prepare/) to discover co-occurrence relationships among customers’ purchase activities. Such an analysis is called market basket analysis or association analysis.  It is commonly used in marketing to increase the chance of cross-selling, provide recommendations to customers (e.g., based on their browsing history) and deliver targeted marketing (e.g., offer coupons to customers for products that are frequently purchased together with items that the customers recently bought).

The market basket analysis is discussed in the [next section](https://eagronin.github.io/market-basket-prepare/).

The analysis for this project was performed in R.

## Data Exploration and Cleaning

```R
items$Date = as.Date(items$InvoiceDate, "%m/%d/%y")               # create date variable from InvoiceDate, which also includes time
items$DateTime = strptime(items$InvoiceDate, format = "%m/%d/%y %H:%M")
items$Hour = as.numeric(substr(items$DateTime, 12, 13))           # create hour variable
items$Minute = as.numeric(substr(items$DateTime, 15, 16))         # create hour variable
items$HourMinute = items$Hour + items$Minute / 60
items$ID = paste(as.character(items$CustomerID), as.character(items$Date))   # new id variable based on (customer id, date) group
items$AmountSpent = items$Quantity * items$UnitPrice
```

output summary statistics

```R
dim(items)                    # dimensions
summary(items)                # summary stats
apply(is.na(items), 2, sum)   # missing values
```

check out mismatches between StockCode and Description
let's take a look, for example, at StockCode 22423, which has 3 mismatches with the description of "REGENCY CAKESTAND 3 TIER":

```R
temp = items[items$StockCode == 22423,]
unique(temp$Description)
temp[temp$Description != "REGENCY CAKESTAND 3 TIER",]
```

same issue appears to affect StockCode 84879:

```R
temp = items[items$StockCode == 84879,]
unique(temp$Description)
temp[temp$Description != "ASSORTED COLOUR BIRD ORNAMENT",]
```

let's take out returns, including items with zero quantity:

```R
nrow(items[items$Quantity <= 0,])   # returns
items = items[items$Quantity > 0,]
nrow(items)
```

let's now look at the remaining mismatches:

```R
temp = items[items$Description == "WHITE HANGING HEART T-LIGHT HOLDER",]
unique(temp$StockCode)
items$StockCode[items$StockCode == "85123a"] = "85123A"
unique(items$Description[items$StockCode == "85123A"])
items[items$Description == "CREAM HANGING HEART T-LIGHT HOLDER",]
items$Description[items$Description == "CREAM HANGING HEART T-LIGHT HOLDER"] = "WHITE HANGING HEART T-LIGHT HOLDER"

temp = items[items$StockCode == 20725,]
unique(temp$Description)
temp[temp$Description != "LUNCH BAG RED RETROSPOT",]
items[items$Description == "LUNCH BAG RED SPOTTY",]
items$Description[items$Description == "LUNCH BAG RED SPOTTY"] = "LUNCH BAG RED RETROSPOT"
```

remove items with negative or zero prices

```R
items[items$UnitPrice < 0,]
items[items$UnitPrice == 0,]
items = items[items$UnitPrice > 0,]
nrow(items)
```

remove items with missing or invalid descriptions

```R
items[grepl(x=items$Description, pattern="[\\?\\*]"),]     # descriptions with ? or *
items = items[!grepl(x=items$Description, pattern="[\\?\\*]"),]
nrow(items)

nrow(items[grepl(x=items$Description, pattern="^$"),])   # number of empty strings
items = items[!grepl(x=items$Description, pattern="^$"),]
nrow(items)

items = subset(items, !(Description %in% c("AMAZON FEE", "Adjust bad debt", "POSTAGE", "DOTCOM POSTAGE", "Manual")))
nrow(items)
```

now let's get rid of cancellations

```R
sum(grepl(x=items$InvoiceNo, pattern="^C.*"))       # cancelations
items = items[!grepl(x=items$InvoiceNo, pattern ="^C.*"),]
nrow(items)
```

output summary stats again; any remaining mismatches?

```R
summary(items)                # summary stats
```

there is a big chunk of missing customer IDs.  Should we use invoices instead?
let's understand better the difference between these two variables:

output unique IDs

```R
length(unique(items$CustomerID))
length(unique(items$InvoiceNo))
length(unique(items$InvoiceNo[!is.na(items$CustomerID)]))
length(unique(items$ID))
```

it appears that there are substantially more invoices than customers, which is consistent with uniquness of invoice number, per definition, while 
customers could have made purchases on different days.  In that case, purchases by the same customer will have the same cutomer ID but different
invoice numbers.
are there customers who made purchases on different days?

```R
by_customerID = tapply(items$Date, items$CustomerID, FUN = function(x) length(unique(x)))   # group unique dates by customer ID
temp = as.data.frame(by_customerID)     # convert vector to dateframe
temp$customerID = rownames(temp)        # convert rownmaes (customer ID) to a variable
by_no_days_customer = tapply(temp$customerID, temp$by_customerID, FUN = function(x) length(x))

barplot(by_no_days_customer, main = "Count of Customers by Number of Days\non Which They Made Purchases"
     , ylab = "Number of Customers", xlab = "Number of Days with Purchases"
     , xlim = c(0.75, 20), ylim = c(0, 1600)
     , frame.plot=FALSE, cex.main=1.5, cex.lab = .8, xaxt="n", yaxt="n", col="blue", pch = 20, type = "o")
axis(side=1, cex.axis = 0.8, lwd=2, tck=-.005)
axis(side=2, cex.axis = 0.8, lwd=2, tck=-.005)
box(which = "plot", bty = "l", lwd=2)
```

are there identical invoice numbers that appear on different days? I.e., are invoice numbers unique to each transaction?

```R
by_InvoiceNo = tapply(items$Date, items$InvoiceNo, FUN = function(x) length(unique(x)))   # group unique dates by InvoiceNo
temp = as.data.frame(by_InvoiceNo)     # convert vector to dateframe
temp$InvoiceNo = rownames(temp)        # convert rownmaes (InvoiceNo) to a variable
by_no_days_invoice = tapply(temp$InvoiceNo, temp$by_InvoiceNo, FUN = function(x) length(x))
```

how many invoices are there per (customer id, date) group? How many CustomerIDs are there with only one invoice per day? with 2 invoices per day? etc.

```R
invoices_per_ID = with(items[!is.na(items$CustomerID),], tapply(InvoiceNo, ID, FUN = function(x) length(unique(x))))
hist(invoices_per_ID)
temp = as.data.frame(invoices_per_ID)     # convert vector to dateframe
temp$ID = rownames(temp)        # convert rownmaes (customer ID) to a variable
items = merge(items, temp, by = "ID", all.x = TRUE)
IDs_per_numOfInvoices = tapply(temp$ID, temp$invoices_per_ID, FUN = function(x) length(x))

barplot(IDs_per_numOfInvoices, main = "Count of Customers by Number of Invoices\nIssued on the Same Day"
        , ylab = "Number of Customers", xlab = "Number of Invoices Issued on the Same Day to a Customer"
        , xlim = c(0.2, 5), ylim = c(0, 16000)
        , frame.plot=FALSE, cex.main=1.5, cex.lab = .8, xaxt="n", yaxt="n", col="blue", pch = 20, type = "o")
axis(side=1, cex.axis = 0.8, lwd=2, tck=-.005)
axis(side=2, cex.axis = 0.8, lwd=2, tck=-.005)
box(which = "plot", bty = "l", lwd=2)
```


By looking at the data it appears that invoices tend to be stamped within minutes from each other. We check this 
out for the group of customers with exactly 2 invoices per day.  According to the chart above, this is the second largest group 
after the group with 1 invoice per day.

```R
keeps = c("ID", "InvoiceNo", "DateTime")
invoices = unique(items[items$invoices_per_ID == 2, keeps])
invoices$InvoiceNo = sequence(rle(invoices$ID)$lengths)
invoices = subset(invoices, !(ID %in% c("14194 2011-03-18", "14911 2011-05-16", "15861 2011-11-13", "17758 2011-05-15")))
invoices$DateTime = as.numeric(invoices$DateTime)
wide = reshape(invoices, idvar = "ID", timevar = "InvoiceNo", direction = "wide")
wide$Diff = abs(wide$DateTime.2 - wide$DateTime.1)/60     # convert from seconds to minutes
median(wide$Diff, na.rm = TRUE)
```

Could it be that customers split their purchases into several invoices when they have too many items? 
To test this, we can comare the average number of distinct StockCodes per invoice between the group with one invoice per day and the group with 
multipe invoices per day.  If this reasoning holds, then it would make sense to consider 
multiple invoices paid on the same day for the same customer to be considered parts of the same shopping session. In that case, using customer ID
is preferrable to using InoiceNo, as InvoiceNo does not represent a separate shopping session. 
The t-test below shows that this is indeed the case, which suggests that we should use customer ID for identifying shopping sessions:

```R
keeps = c("ID", "StockCode")
one_invoice = items[(items$invoices_per_ID == 1) & !(is.na(items$CustomerID)), keeps]
one_invoice = with(one_invoice, tapply(StockCode, ID, FUN = function(x) length(unique(x))))
mean(one_invoice, na.rm = TRUE)
median(one_invoice, na.rm = TRUE)

many_invoices = items[(items$invoices_per_ID > 1) & !(is.na(items$CustomerID)), keeps]
many_invoices = with(many_invoices, tapply(StockCode, ID, FUN = function(x) length(unique(x))))
mean(many_invoices, na.rm = TRUE)
median(many_invoices, na.rm = TRUE)

t.test(one_invoice, many_invoices)
```

Would we have an issue if we removed the samples with missing customer IDs?  Let's compare the samples with and without 
customer ID:

split into two samples

```R
with_cust_id = items[!is.na(items$CustomerID),]
no_cust_id = items[is.na(items$CustomerID),]
```

number of distinct StockCodes per invoice

```R
dist_scode_with_id = tapply(with_cust_id$StockCode, with_cust_id$InvoiceNo, FUN = function(x) length(unique(x)))
dist_scode_no_id = tapply(no_cust_id$StockCode, no_cust_id$InvoiceNo, FUN = function(x) length(unique(x)))
t.test(dist_scode_with_id, dist_scode_no_id)
```

amount spent per invoice

```R
spent_with_id = tapply(with_cust_id$AmountSpent, with_cust_id$InvoiceNo, FUN = sum)
spent_no_id = tapply(no_cust_id$AmountSpent, no_cust_id$InvoiceNo, FUN = sum)
t.test(spent_with_id, spent_no_id)
```

hour of purchase (use mode function to determine the most frequent occurance of hour to prevent typos affecting results, 
as an invoice hour should be the same for all items contained in it)

```R
hour_with_id = tapply(
  with_cust_id$HourMinute, with_cust_id$InvoiceNo
  , FUN = function(x) {
    ux <- unique(x)
    ux[which.max(tabulate(match(x, ux)))]
  }
)
hour_no_id = tapply(
  no_cust_id$HourMinute, no_cust_id$InvoiceNo
  , FUN = function(x) {
    ux <- unique(x)
    ux[which.max(tabulate(match(x, ux)))]
  }
)
t.test(hour_with_id, hour_no_id)
```


We have an issue as the samples with and w/o CustomerID are quite different. Therefore, simply removing samples with missing CustomerID from the data
may introduce bias into our analysis. However, if InvoiceNo matches CustomerID closely, then we can use InvoiceNo when customer id is missing.

what is the fraction of CustomerIDs that are missing?

```R
sum(is.na(items$CustomerID))/nrow(items)
```

we find that about 24.93% are missing

what is the fraction of IDs with multiple invoices in the sample of non-missing CustomerIDs?  We can assume that this fraction is the same
in the sample of missing CustomerIDs.  This fraction is:

```R
1-IDs_per_numOfInvoices[1]/sum(IDs_per_numOfInvoices)
```

This result means that if we were to replace CustomerID with InvoiceNo in the sample with non-missing CustomerID, we would 
misclassify 8.20% of shopping trips as different shopping session, while they, in fact, were part of other shopping sessions.

what is the fraction of IDs with multiple invoices in the entire sample?

```R
(1-IDs_per_numOfInvoices[1]/sum(IDs_per_numOfInvoices)) * (sum(is.na(items$CustomerID))/nrow(items))
```

we end up with 2.04% of the shopping sessions that occured on different days misclassified as shopping trips that occured on the same day.  
for the subsequent analysis we will prefer this small amount of misclassification over losing 25% of observations and risking bias in the analysis
if the missing observations are not representative of the rest of the data.

let's replace the missing IDs with InvoiceNo / Date combinations

```R
items$ID[is.na(items$CustomerID)] =
          paste(as.character(items$InvoiceNo[is.na(items$CustomerID)])
              , as.character(items$Date[is.na(items$CustomerID)]))   # suppliment missing IDs with (invoice no, date) combinations
```

we end up with this number of shopping sessions:

```R
length(unique(items$ID))
```
