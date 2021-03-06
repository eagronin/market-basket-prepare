# Data Preparation

## Overview
This section summarizes the process of initial exploration and cleaning of the data from a UK-based online retailer.  The data are used in the [next section](https://eagronin.github.io/market-basket-analyze/) to discover the co-occurrence relationships among customers’ purchase activities, such as the likelihood to purchase a candle holder if the customer already has candles and matches in their shopping cart. Such an analysis is called market basket analysis or association analysis.  It is commonly used in marketing to increase the chance of cross-selling, provide recommendations to customers (e.g., based on their browsing history) and deliver targeted marketing (e.g., offer coupons to customers for products that are frequently purchased together with items that the customers recently bought).

Description and importing of the data are provided in the [previous section](https://eagronin.github.io/market-basket-acquire/).

The analysis is discussed in the [next section](https://eagronin.github.io/market-basket-analyze/).

The analysis for this project was performed in R.

## Data Exploration and Cleaning

We start by generating a few features that we will use in subsequent analysis.  These features include the date and time of the invoice ("Date" and "HourMinute", respectively), the total amount spent equal to the product of the item's price and quantity purchased ("AmountSpent"), and an ID variable identifying all purchases by a customer on a particular day ("ID"):

```R
items$Date = as.Date(items$InvoiceDate, "%m/%d/%y")               # create date variable from InvoiceDate, which also includes time
items$DateTime = strptime(items$InvoiceDate, format = "%m/%d/%y %H:%M")
items$Hour = as.numeric(substr(items$DateTime, 12, 13))           # create hour variable
items$Minute = as.numeric(substr(items$DateTime, 15, 16))         # create hour variable
items$HourMinute = items$Hour + items$Minute / 60
items$ID = paste(as.character(items$CustomerID), as.character(items$Date))   # new id variable based on (customer id, date) group
items$AmountSpent = items$Quantity * items$UnitPrice
items$InvoiceDate = NULL
```

With all the features in place, let's output summary statistics:

```R
dim(items)                    # dimensions
summary(items)                # summary stats
```

The summary statistics below show that the dataset has 541,909 samples and 14 features.  The customers reside predominantly in the UK, make approximately 50% of their purchases between 11:45am and 3:30pm (see HourMinute feature) and buy the median quantity of 3 units per item at the median price of 2.08 pounds per unit. We note and discuss further below that an invoice may include multiple items.

The summary statistics also show several patterns in the data that need to be addressed further.  Specifically, 135,080 CustomerIDs are missing, which is approximately 25% of the dataset.  Further, there are items with negative quantities (returns) and negative prices (see Panel A); mismatches between StockCode and Description counts (see Panel B), while the small magnitudes of mismatches in some instances (e.g., 2203 items for StockCode 22423 versus 2200 items for REGENCY CAKESTAND 3 TIER) and exact matches in other instances (e.g., 2159 items for StockCode 85099B versus the same number of items for JUMBO BAG RED RETROSPOT) suggest that StockCode and Description counts should match.

**Table 1**<br/>
Panel A. Numeric Features

Feature Name|Min.|1st Qu.|Median|Mean|3rd Qu.|Max|Missing
|:--- | ---:| ---:| ---:| ---:| ---:| ---:| ---:|
Quantity | -80,995 | 1 | 3 | 9.55 | 10 | 80,995	
UnitPrice | -11,062.06 | 1.25 | 2.08 | 4.61 | 4.13 | 38,970	
CustomerID | 12,346 | 13,953 | 15,152 | 15,288 | 16,791 | 18,287 | 135,080
Date | 12/1/10 | 3/28/11 | 7/19/11 | 7/4/11 | 10/19/11 | 12/9/11	
DateTime | 12/1/10 8:26 | 3/28/11 11:34 | 7/19/11 17:17 | 7/4/11 13:59 | 10/19/11 11:27 | 12/9/11 12:50	
Hour | 6 | 11 | 13 | 13.08 | 15 | 20	
Minute | 0 | 16 | 30 | 30.01 | 44 | 59	
HourMinute | 6.167 | 11.783 | 13.583 | 13.579 | 15.483 | 20.633	
AmountSpent | -168,469.6 | 3.4 | 9.75 | 17.99 | 17.4 | 168,469.6	

Panel B. Categorical Features

InvoiceNo | StockCode | Description | Country | ID
|:--- |:--- |:--- |:--- |:--- |
573585 :  1114 | 85123A :  2313 | WHITE HANGING HEART T-LIGHT HOLDER: 2369 | United Kingdom: 495478 | Length: 541909
581219:   749 | 22423:  2203 | REGENCY CAKESTAND 3 TIER: 2200 | Germany: 9495 | Class: character
581492:   731 | 85099B:  2159 | JUMBO BAG RED RETROSPOT: 2159 | France: 8557 | Mode: character
580729:   721 | 47566:  1727 | PARTY BUNTING: 1727 | EIRE:  8196
558475:   705 | 20725:  1639 | LUNCH BAG RED RETROSPOT: 1638 | Spain: 2533	
579777:   687 | 84879:  1502 | ASSORTED COLOUR BIRD ORNAMENT: 1501 | Netherlands: 2371	
(Other): 537202 | (Other): 530366 | (Other): 530315 | (Other): 537202 | (Other):   15279	

Further look at the data reveals that apart from irregularities observed from the summary statistics table above, there are a number of missing or invalid descriptions, such as "?display?", "?missing", "??", "historic computer difference?....se", "POSSIBLE DAMAGES OR LOST?", etc. Also, the highest prices in the data appear against such Description fields as "AMAZON FEE", "Adjust bad debt", "POSTAGE", "DOTCOM POSTAGE", "Manual", i.e., descriptions of various actions rather than names of items.  Finally, there are a number of cancellations that can be identified using letter "C" in the beginning of InvoiceNo.

Invalid samples should be removed from the data before we run the algorithm to determine associations between customer purchases.  We will deal with these samples first and look into handling missing CustomerIDs after that.

Let's check out mismatches between StockCode and Description.  Let's take a look, for example, at StockCode 22423, which has 2,203 instances versus 2,200 instances of "REGENCY CAKESTAND 3 TIER" (a difference of 3 instances):

```R
temp = items[items$StockCode == 22423,]
unique(temp$Description)
temp[temp$Description != "REGENCY CAKESTAND 3 TIER",]
```

This code generates the following output:

```
[1] REGENCY CAKESTAND 3 TIER faulty                   damages                 

       InvoiceNo StockCode Description Quantity UnitPrice CustomerID        Country       Date            DateTime Hour Minute HourMinute            ID AmountSpent
21339     538072     22423      faulty      -13         0         NA United Kingdom 2010-12-09 2010-12-09 14:10:00   14     10   14.16667 NA 2010-12-09           0
114429    546010     22423     damages      -19         0         NA United Kingdom 2011-03-08 2011-03-08 15:55:00   15     55   15.91667 NA 2011-03-08           0
192292    553396     22423     damages      -21         0         NA United Kingdom 2011-05-16 2011-05-16 16:48:00   16     48   16.80000 NA 2011-05-16           0
```

It appears that the 3 mismatches represent returns (negative Quantity for each) due to "damages" for two of the items and "faulty" for the third item.  Once we get rid of returned items, these mismatches then will be resolved.

The same issue appears to affect StockCode 84879 (1 mismatch as compared to the Description field):

```R
temp = items[items$StockCode == 84879,]
unique(temp$Description)
temp[temp$Description != "ASSORTED COLOUR BIRD ORNAMENT",]
```

This code generates the following output:

```
[1] ASSORTED COLOUR BIRD ORNAMENT damaged                      

       InvoiceNo StockCode Description Quantity UnitPrice CustomerID        Country       Date            DateTime Hour Minute HourMinute            ID AmountSpent
189300    553147     84879     damaged     -160         0         NA United Kingdom 2011-05-13 2011-05-13 13:58:00   13     58   13.96667 NA 2011-05-13           0
```

We can see that the mismatch resulted from one damaged item that has been returned.

Let's remove all the returned items (i.e., items with negative Quantity) from the data.  From looking further at the data, it  appears that there are also a number of items with zero quantity.  Let's remove these observations as well:

```R
nrow(items[items$Quantity <= 0,])   
items = items[items$Quantity > 0,]
nrow(items)
```

The above code removed `10,624` samples, thus reducing the total number of samples from `541,909` to `531,285`.

Now let's look at the remaining mismatches, i.e., StockCodes 85123A (56 mismatches) and 20725 (1 mismatch).  The code below identifies the reasons for the mismatches and resolve some of the mismatches:  

```R
temp = items[items$Description == "WHITE HANGING HEART T-LIGHT HOLDER",]
unique(temp$StockCode)
items$StockCode[items$StockCode == "85123a"] = "85123A"
unique(items$Description[items$StockCode == "85123A"])
items[items$Description == "CREAM HANGING HEART T-LIGHT HOLDER",]

temp = items[items$StockCode == "20725",]
unique(temp$Description)
temp[temp$Description != "LUNCH BAG RED RETROSPOT",]
items[items$Description == "LUNCH BAG RED SPOTTY",]
```

The output is the following:

```
[1] 85123A 85123a

[1] WHITE HANGING HEART T-LIGHT HOLDER ?                                  CREAM HANGING HEART T-LIGHT HOLDER

       InvoiceNo StockCode                        Description Quantity UnitPrice CustomerID        Country       Date            DateTime Hour Minute HourMinute               ID AmountSpent
537622    581334    85123A CREAM HANGING HEART T-LIGHT HOLDER        4      2.95      17841 United Kingdom 2011-12-08 2011-12-08 12:07:00   12      7   12.11667 17841 2011-12-08       11.80
538085    581395    85123A CREAM HANGING HEART T-LIGHT HOLDER        6      2.95      16892 United Kingdom 2011-12-08 2011-12-08 13:18:00   13     18   13.30000 16892 2011-12-08       17.70
538215    581402    85123A CREAM HANGING HEART T-LIGHT HOLDER        6      2.95      14653 United Kingdom 2011-12-08 2011-12-08 13:45:00   13     45   13.75000 14653 2011-12-08       17.70
538271    581404    85123A CREAM HANGING HEART T-LIGHT HOLDER        4      2.95      13680 United Kingdom 2011-12-08 2011-12-08 13:47:00   13     47   13.78333 13680 2011-12-08       11.80
538709    581412    85123A CREAM HANGING HEART T-LIGHT HOLDER        4      2.95      14415 United Kingdom 2011-12-08 2011-12-08 14:38:00   14     38   14.63333 14415 2011-12-08       11.80
539084    581432    85123A CREAM HANGING HEART T-LIGHT HOLDER       32      2.55      13798 United Kingdom 2011-12-08 2011-12-08 15:51:00   15     51   15.85000 13798 2011-12-08       81.60
539343    581439    85123A CREAM HANGING HEART T-LIGHT HOLDER        1      5.79         NA United Kingdom 2011-12-08 2011-12-08 16:30:00   16     30   16.50000    NA 2011-12-08        5.79
540838    581492    85123A CREAM HANGING HEART T-LIGHT HOLDER        3      5.79         NA United Kingdom 2011-12-09 2011-12-09 10:03:00   10      3   10.05000    NA 2011-12-09       17.37
541640    581538    85123A CREAM HANGING HEART T-LIGHT HOLDER        1      2.95      14446 United Kingdom 2011-12-09 2011-12-09 11:34:00   11     34   11.56667 14446 2011-12-09        2.95

[1] LUNCH BAG RED RETROSPOT LUNCH BAG RED SPOTTY   

      InvoiceNo StockCode          Description Quantity UnitPrice CustomerID Country       Date            DateTime Hour Minute HourMinute               ID AmountSpent
58133    541220     20725 LUNCH BAG RED SPOTTY      200      1.45      14156    EIRE 2011-01-14 2011-01-14 14:11:00   14     11   14.18333 14156 2011-01-14         290
```

The output shows that for StockCode 85123A most of the mismatches resulted from recording some of the StockCodes as 85123a ("a" instead of "A"). This appears to be a typo, as there is nothing else in the data suggesting that StockCodes for items 85123A and 85123a should be different.

After changing StockCodes 85123a to 85123A, we are still left with mismatches between StockCode and Description fields. While Description for almost all of the items with StockCode 85123A is "WHITE HANGING HEART T-LIGHT HOLDER", it appears that some of the items with this StockCode are named "CREAM HANGING HEART T-LIGHT HOLDER" and others have "?" in the description. Note that there is no overlap between the items that had StockCode 85123a (before we changed it to 85123A) and the items with "CREAM HANGING HEART T-LIGHT HOLDER" and "?" in their Description fields.

The items with "?" in the description will get dropped when we remove instances with invalid or missing descriptions.  We will not make changes to the 9 items with Description "CREAM HANGING HEART T-LIGHT HOLDER" as we are not sure whether the word "CREAM" instead of "WHITE" in these few purchases (all of which were made on the same date, December 8, 2011) was used intentionally or by mistake.

The mismatch between StockCode 20725 and Description amounts to one instance of "LUNCH BAG RED SPOTTY" in the description while all the other instances with this StockCode are "LUNCH BAG RED RETROSPOT".  As in the case of StockCode 85123A we will not make any changes as we are not sure whether the difference in the descriptions was made intentionally or by mistake.

Let's now move on to the other issues that we discovered by looking at the summary statistics.  As we pointed out above, there are items with negative prices.  The code below shows that there are only 2 such items. Similarly, the code shows that there are multiple instances with the price equal to zero. We remove items with negative or zero prices:

```R
items[items$UnitPrice < 0,]
items[items$UnitPrice == 0,]
items = items[items$UnitPrice > 0,]
nrow(items)
```

This code generates the following output:

```
       InvoiceNo StockCode     Description Quantity UnitPrice CustomerID        Country       Date            DateTime Hour Minute HourMinute            ID AmountSpent
299984   A563186         B Adjust bad debt        1 -11062.06         NA United Kingdom 2011-08-12 2011-08-12 14:51:00   14     51   14.85000 NA 2011-08-12   -11062.06
299985   A563187         B Adjust bad debt        1 -11062.06         NA United Kingdom 2011-08-12 2011-08-12 14:52:00   14     52   14.86667 NA 2011-08-12   -11062.06

[1] 1179

[1] 530104
```

We can see that the two negative prices correspond to adjustments for bad debt.  We remove the items with negative prices along with 1,179 items with zero prices.  We note that further exploration of data shows that many of the 1,179 instances with zero prices have missing or invalid Description fields).

The output above also shows that after the last round of cleaning the data, the number of observations was reduced from `531,285` to `530,104`.

As we have noted above, a number of items have missing or invalid Descriptions.  We remove these items:

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

As a result, the number of observations declines from `530,104` to `527,944`.

Finally, let's remove cancellations of purchases:

```R
sum(grepl(x=items$InvoiceNo, pattern="^C.*"))       # cancelations
items = items[!grepl(x=items$InvoiceNo, pattern ="^C.*"),]
nrow(items)
```

The number of observations remains at `527,944`, which means that all the cancellations were in the subset of items that we have already removed in prior rounds of data cleaning.

Let's output summary statistics again using `summary(items)`:

**Table 2**<br/>
Panel A. Numeric Features

Feature Name	 | 	Min	 | 	1st Qu.	 | 	Median	 | 	Mean	 | 	3rd Qu.	 | 	Max	 | 	Missing
:---	 | 	---:	 | 	---:	 | 	---:	 | 	---:	 | 	---:	 | 	---:	 | 	---:
Quantity	 | 	1	 | 	1	 | 	3	 | 	10.56	 | 	11	 | 	80,995	 | 	
UnitPrice	 | 	0.001	 | 	1.25	 | 	2.08	 | 	3.279	 | 	4.13	 | 	649.5	 | 	
CustomerID	 | 	12,346	 | 	13,975	 | 	15,159	 | 	15,301	 | 	16,801	 | 	18,287	 | 	131,459
Date	 | 	12/1/10	 | 	3/28/11	 | 	7/20/11	 | 	7/4/11	 | 	10/19/11	 | 	12/9/11	 | 	
DateTime	 | 	12/1/10 8:26	 | 	3/28/11 12:23	 | 	7/20/11 13:26	 | 	7/4/11 21:54	 | 	10/19/11 13:38	 | 	12/9/11 12:50	 | 	
Hour	 | 	6	 | 	11	 | 	13	 | 	13.08	 | 	15	 | 	20	 | 	
Minute	 | 	0	 | 	16	 | 	30	 | 	30.02	 | 	44	 | 	59	 | 	
HourMinute	 | 	6.333	 | 	11.783	 | 	13.583	 | 	13.577	 | 	15.483	 | 	20.3	 | 	
AmountSpent	 | 	0	 | 	3.75	 | 	9.9	 | 	19.47	 | 	17.46	 | 	168,469.6	 | 	

Panel B. Categorical Features

InvoiceNo	 | 	StockCode	 | 	Description	 | 	Country	 | 	ID
:---	 | 	:---	 | 	:---	 | 	:---	 | 	:---
573585  :  1113	 | 	85123A  :  2332	 | 	WHITE HANGING HEART T-LIGHT HOLDER  :  2323	 | 	United Kingdom:484079	 | 	Length. :  527944
581219  :   748	 | 	85099B  :  2112	 | 	JUMBO BAG RED RETROSPOT  :  2112	 | 	Germany  :  8658	 | 	Class  :  character
581492  :   730	 | 	22423  :  2017	 | 	REGENCY CAKESTAND 3 TIER   :  2017	 | 	France  :  8102	 | 	Mode  :  character
580729  :   720	 | 	47566  :  1706	 | 	PARTY BUNTING  :  1706	 | 	EIRE  :  7885	 | 	
558475  :   704	 | 	20725  :  1595	 | 	LUNCH BAG RED RETROSPOT  :  1594	 | 	Spain  :  2422	 | 	
579777  :   686	 | 	84879  :  1489	 | 	ASSORTED COLOUR BIRD ORNAMENT  :  1489	 | 	Netherlands  :  2322	 | 	
(Other)  :  523243	 | 	(Other)  :  516693	 | 	(Other)  :  516703	 | 	(Other)  :  14476	 | 	

The summary statistics look reasonable now, except we have not addressed the issue of missing CustomerIDs yet.  This feature is important for defining a shopping session.  Because our goal is to identify co-occurrence relationships among customers’ purchase activities, it is important to define a timeframe within which such purchase activities should be counted toward a co-occurrence relationship.  

If we were to analyze customer visits to a local supermarket, it would be intuitive to define a shopping session as one trip to the supermarket.  However, when online shopping is concerned, defining a shopping session is more subtle.  What if a customer has two invoices within the same day?  Should these invoices be considered as two separate shopping sessions?  What if the difference in the time stamps on these invoices is only one minute?  Maybe the only reason that we see two invoices instead of just one is because the customer made many purchases and decided to break them out into two invoices?  In that case it would be reasonable to consider the items on both invoices as parts of the same shopping session.  However, if the time difference between the two invoices is 10 hours, it might rather be reasonable to consider such invoices as separate shopping sessions (i.e., seeing items on the website during the earlier shopping session could affect purchasing behavior during that session but not during the later sessions). 

If we decide, for example, to define a shopping session for a customer as all the shopping that the customer did on the website within one day, then a shopping session can be uniquely identified by CustomerID and Date.  In that case availability of CustomerID is crucial.  However, if we decide to use invoices as proxies for shopping sessions, then we need only a unique invoice number (InvoiceNo) and do not need CustomerID.  

Let's understand better the difference between CustomerID and InvoiceNo. The code below outputs the number of unique values of CustomerID, InvoiceNo, and an ID variable that we defined earlier by concatenating CustomerID and Date (i.e., the date of the invoice).  

```R
length(unique(items$CustomerID))
length(unique(items$InvoiceNo[!is.na(items$CustomerID)]))
length(unique(items$ID[!is.na(items$CustomerID)]))
```

Based on the output from running the above code, we have 4,336 unique CustomerIDs, 18,416 unique values for InvoiceNo (when CustomerID is not missing) and 16,689 unique IDs (also when CustomerID is not missing).  We can see that the number of unique CustomerIDs is substantially lower than the number of unique values for InvoiceNo and ID.  This could be a result of customers making purchases on multiple dates. Hence, a few customers can generate a large number of unique invoices as well as a large number of unique IDs (because ID is defined as a CustomerID/Date combination).

In that case, purchases by the same customer will have the same customer ID but different invoice numbers.  Let's check if there are customers who made purchases on different days and plot the count of customers as a function of the number of days on which they made purchases:

```R
by_customerID = tapply(items$Date, items$CustomerID, FUN = function(x) length(unique(x)))   # group unique dates by customer ID
temp = as.data.frame(by_customerID)     # convert vector to dateframe
temp$customerID = rownames(temp)        # convert rownmaes (customer ID) to a variable
by_no_days_customer = tapply(temp$customerID, temp$by_customerID, FUN = function(x) length(x))

barplot(by_no_days_customer, main = "Figure 1. Count of Customers by Number of Days\non Which They Made Purchases"
     , ylab = "Number of Customers", xlab = "Number of Days with Purchases"
     , xlim = c(0.75, 20), ylim = c(0, 1600)
     , frame.plot=FALSE, cex.main=1.5, cex.lab = .8, xaxt="n", yaxt="n", col="blue", pch = 20, type = "o")
axis(side=1, cex.axis = 0.8, lwd=2, tck=-.005)
axis(side=2, cex.axis = 0.8, lwd=2, tck=-.005)
box(which = "plot", bty = "l", lwd=2)
```

The bar chart generated by the code above is shown below:

![](https://github.com/eagronin/market-basket-prepare/blob/master/figure-1.png?raw=true)

The chart shows that, while many customers made purchases on one day only, a large number of customers indeed shopped on multiple days over approximately one year period over which the data are available.

We now turn to the InvoiceNo attribute.  InvoiceNo values are unique as reported on the official website from which the data were downloaded, and as the code below verifies by finding that there are no invoices with the same InvoiceNo issued on different days:

```R
length(unique(items$InvoiceNo))
by_InvoiceNo = tapply(items$Date, items$InvoiceNo, FUN = function(x) length(unique(x)))   # group unique dates by InvoiceNo
temp = as.data.frame(by_InvoiceNo)     # convert vector to dateframe
temp$InvoiceNo = rownames(temp)        # convert rownmaes (InvoiceNo) to a variable
by_no_days_invoice = tapply(temp$InvoiceNo, temp$by_InvoiceNo, FUN = function(x) length(x))
```

How many invoices are there per ID (i.e., CustomerId/Date) group? How many IDs are there with only one invoice? with 2 invoices? etc.  The code below plots the count of IDs as a function of the number of invoices issued on the same day:

```R
invoices_per_ID = with(items[!is.na(items$CustomerID),], tapply(InvoiceNo, ID, FUN = function(x) length(unique(x))))
hist(invoices_per_ID)
temp = as.data.frame(invoices_per_ID)     # convert vector to dateframe
temp$ID = rownames(temp)        # convert rownmaes (customer ID) to a variable
items = merge(items, temp, by = "ID", all.x = TRUE)
IDs_per_numOfInvoices = tapply(temp$ID, temp$invoices_per_ID, FUN = function(x) length(x))

barplot(IDs_per_numOfInvoices, main = "Figure 2. Count of IDs by Number of Invoices\nIssued on the Same Day"
        , ylab = "Number of IDs", xlab = "Number of Invoices Issued on the Same Day to a Customer"
        , xlim = c(0.2, 5), ylim = c(0, 16000)
        , frame.plot=FALSE, cex.main=1.5, cex.lab = .8, xaxt="n", yaxt="n", col="blue", pch = 20, type = "o")
axis(side=1, cex.axis = 0.8, lwd=2, tck=-.005)
axis(side=2, cex.axis = 0.8, lwd=2, tck=-.005)
box(which = "plot", bty = "l", lwd=2)
```

The chart below shows that while majority of IDs correspond to one invoice only, a non-trivial number of IDs correspond to two or more invoices issued on the same date.  

![](https://github.com/eagronin/market-basket-prepare/blob/master/figure-2.png?raw=true)


As we pointed out above, if there are large time differences between such invoices (e.g., several hours) we may consider using InvoiceNo instead of ID to identify shopping sessions.   

However, by looking at the data it appears that invoices issued to a customer on the same day tend to be stamped within minutes from each other. We check this out for the group of customers with exactly 2 invoices per day.  According to the chart above, this is the second largest group following the group with 1 invoice per day.

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

It appears that the median time difference between two invoices issued on the same date is 2 minutes.  This amount of time is too short to consider the two invoices as proxies for two separate shopping sessions.  Rather, it is possible that customers who purchase more items tend to break out their purchases into multiple invoices.  In order to test this hypothesis, we compare the average number of distinct StockCodes per invoice between the group with one invoice per day and the group with multiple invoices per day.  If this hypothesis is true, then it would make sense to consider multiple invoices issued on the same day for the same customer as parts of the same shopping session. In that case, using CustomerID is preferable to using InvoiceNo. 

The code below performs a t-test to check the validity of this hypothesis:

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

The results of the t-test shown below suggest that indeed, customers who purchase more items tend to break out their purchases into multiple invoices.  The average numbers of distinct items in the group with one invoice per day and the group with multiple invoices per day are 22.1 and 32.7, respectively, and the difference is statistically significant.  At the same time, the average number of items per invoice is smaller in the group with multiple invoices per day. This suggests that the existence of multiple invoices for a customer on the same day could be a result of artificially splitting one large invoice into multiple smaller invoices, which does not justify using InvoiceNo as a proxy for a separate shopping session.

```
	Welch Two Sample t-test

data:  one_invoice and many_invoices
t = -12.685, df = 1533.9, p-value < 2.2e-16
alternative hypothesis: true difference in means is not equal to 0
95 percent confidence interval:
 -12.264783  -8.979764
sample estimates:
mean of x mean of y 
 22.14106  32.76333 
```


This, in turn, implies that we should use CustomerID for identifying shopping sessions.  Would we have an issue if we removed the samples with missing customer IDs?  Let's compare the samples with and without customer ID.  If the samples with CustomerID are representative of the samples without CustomerID, then removing samples without CustomerID will not bias the results. However, if there are systemic differences between the two groups, then simply removing samples without CustomerID may bias the results.  This would mean that the co-occurrence relationships among customers’ purchase activities that we will identify using the samples with CustomerID are going to generate incorrect recommendations for the customers represented by the samples without CustomerID.

We first split the dataset into two groups: with CustomerID available and with CustomerID missing:

```R
with_cust_id = items[!is.na(items$CustomerID),]
no_cust_id = items[is.na(items$CustomerID),]
```

Then we calculate the number of distinct StockCodes per invoice, amount spent per invoice and hour of purchase for each group and perform a means comparison test for the two groups.  The code is shown below:

  1. Calculate the number of distinct StockCodes per invoice for each group and compare the means using a t-test:

```R
dist_scode_with_id = tapply(with_cust_id$StockCode, with_cust_id$InvoiceNo, FUN = function(x) length(unique(x)))
dist_scode_no_id = tapply(no_cust_id$StockCode, no_cust_id$InvoiceNo, FUN = function(x) length(unique(x)))
t.test(dist_scode_with_id, dist_scode_no_id)
```

  2. Calculate the amount spent per invoice for each group and compare the means using a t-test:

```R
spent_with_id = tapply(with_cust_id$AmountSpent, with_cust_id$InvoiceNo, FUN = sum)
spent_no_id = tapply(no_cust_id$AmountSpent, no_cust_id$InvoiceNo, FUN = sum)
t.test(spent_with_id, spent_no_id)
```

  3. Select the time of purchase per invoice for each group (we calculate the most frequent occurrence of time across items that belong to the same invoice to prevent typos affecting results, as invoice time should be the same for all items contained in it), and compare the means using a t-test:

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

The results of the above tests are summarized in Table 3:

**Table 3**

Metric of Interest	 | 	Mean (CustomerID is Available)	 | 	Mean (CustomerID is Missing)	 | 	Difference	 | 	t-statistic	 | 	p-value
:---	 | 	---:	 | 	---:	 | 	---:	 | 	---:	 | 	---:
Number of distinct StockCodes per Invoice | 	20.99	 | 	95.41	 | 	-74.43	 | 	-19.93	 | 	2.20E-16
Amount spent per Invoice | 	476.10	 | 	1100.83	 | 	-624.73	 | 	-8.81	 | 	2.20E-16
Time of purchase	 | 	13.01	 | 	14.15	 | 	-1.14	 | 	-16.27	 | 	2.20E-16

It appears that the groups with and w/o CustomerIDs are statistically significantly different from one another along each dimension. Invoices in the group w/o CustomerIDs have substantially longer lists of items with larger amounts spent per invoice and are issued at a later time during the day.  Therefore, simply removing samples with missing CustomerID from the data may introduce bias into our analysis. 

How should we then define shopping sessions for the items with missing CustomerID?  As we have seen in Figure 2, each ID (i.e., CustomerID/Date combination) corresponds to precisely one InvoiceNo on most of the days, even though not on all days. What if we use InvoiceNo to define shopping sessions for the items with missing CustomerID?  

Let's see how many shopping sessions we are going to misclassify as multiple sessions while, in fact, they would be combined into a smaller number of shopping sessions if CustomerID was available.  This can be calculated using the following formula:

`(fraction of missing CustomerIDs) * (fraction of IDs with multiple invoices)`

We explain the logic of this formula in detail below.  

First let's check the fraction of CustomerIDs that are missing in the dataset:

```R
sum(is.na(items$CustomerID)) / nrow(items)
```

By executing the above line of code we can see that, as we pointed out earlier, 0.249 (or about 25%) of samples have missing CustomerIDs.

What is the fraction of IDs (CustomerID/Date combinations) with multiple invoices in the group with non-missing CustomerIDs?  This fraction is calculated as

```R
1 - IDs_per_numOfInvoices[1] / sum(IDs_per_numOfInvoices)
```

and equals to 0.082.  This result means that if we were to replace CustomerID/Date combinations with InvoiceNo in the group with non-missing CustomerID, we would misclassify 8.2% of shopping sessions as separate shopping sessions while, in fact, they were part of other shopping sessions. 

Let's assume that this fraction is the same in the group with missing CustomerIDs.  Then, if we misclassify 8.2% of shopping sessions in the group with missing CustomerIDs, how many shopping sessions do we misclassify as a fraction of the total number of shopping sessions in the entire dataset?  This can be calculated as the product of the fraction of IDs with multiple invoices in the group with missing CustomerIDs (8.2%) and the fraction of samples with missing CustomerIDs (24.9%):

```R
(1 - IDs_per_numOfInvoices[1] / sum(IDs_per_numOfInvoices)) * (sum(is.na(items$CustomerID)) / nrow(items))
```

which results in 2.04% of misclassified shopping sessions.  For the subsequent analysis we will prefer to have this small fraction of misclassified shopping sessions over losing 25% of observations and risking bias in the analysis that we discussed above.

The code below replaces the missing IDs using InvoiceNo / Date combinations:

```R
items$ID[is.na(items$CustomerID)] =
          paste(as.character(items$InvoiceNo[is.na(items$CustomerID)])
              , as.character(items$Date[is.na(items$CustomerID)]))   # suppliment missing IDs with (invoice no, date) combinations

length(unique(items$ID))
```

This results in the total of 18,062 shopping sessions.

Next step: [Analysis](https://eagronin.github.io/market-basket-analyze/)
