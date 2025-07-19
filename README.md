# RFM-Analysis

# Python_RFM_Analysis: Customer segmentation
Please see the coding file attached or reach this link 
https://colab.research.google.com/drive/1gu1RJXNpoT_qajnI7ZYiWEX4dHvYRWBq


## I. Introduction
### 1. Business question
- SuperStore is a global retail company with a vast customer base.  

- On the occasion of Christmas and New Year, the Marketing department aims to launch marketing campaigns to appreciate customers who have supported the company over time, as well as to engage potential customers who could become loyal ones.  

- However, the Marketing team has not yet been able to segment this year's customers due to the large dataset, making manual processing—previously used in past years—impractical. Therefore, they are seeking assistance from the Data Analytics department to implement a customer segmentation model that will allow them to tailor marketing programs to different customer groups effectively.  

- The Marketing Director has proposed using the RFM model for segmentation. In the past, when the company was smaller, the team could calculate and classify customers using Excel. However, given the current data volume, they are requesting the Data Analytics department to develop a Python-based workflow to assess and implement customer segmentation efficiently.

### 2. DATASET
Dataset used (as attachment) is a transnational dataset which contains all the transactions occurring between 01/12/2010 and 09/12/2011 for a UK-based and registered non-store online retail.

|Field Name| Detail|
|---|---|
|InvoiceNo |Invoice number. Nominal, a 6-digit integral number uniquely assigned to each transaction. If this code starts with letter 'C', it indicates a cancellation.|
|StockCode |Product (item) code. Nominal, a 5-digit integral number uniquely assigned to each distinct product.|
|Description| Product (item) name. Nominal.|
|Quantity| The quantities of each product (item) per transaction. Numeric.|
|InvoiceDate| Invoice Date and time. Numeric, the day and time when each transaction was generated.|
|UnitPrice| Unit price. Numeric, Product price per unit in sterling.|
|CustomerID| Customer number. Nominal, a 5-digit integral number uniquely assigned to each customer.|
|Country| Country name. Nominal, the name of the country where each customer resides.|

### 3. ABOUT RFM ANALYSIS
***What is RFM Analysis?***
- RFM is a marketing analysis technique that stands for Recency, Frequency, and Monetary Value.
    - **Recency**: measures how recently a customer has made a purchase.
    - **Frequency**: measures how often a customer has made purchases.
    - **Monetary Value**: measures the total amount of money a customer has spent on purchases.

***Why RFM Analysis?***
- RFM is a part of Marketing Analysis and is used to evaluate customer value, helping businesses analyze and segment their customer base. This allows for tailored marketing campaigns or special customer care initiatives for each group.

***How does RFM Analysis work?***
- In RFM analysis, customers are scored based on three factors (Recency - how recently, Frequency - how often, Monetary - how much), then labeled (segmented) based on the combination of RFM scores

## **II. DATA PREPARATION (EDA)** 
In this data preparation stage, we will check for **_missing values, duplicates,_** and **_incorrect data types/data values_** to make sure the dataset is **_clean_** and **_ready_** for further analysis.
### 1. CHECKING
```Python
# Checking for Missing Values
print(ecommerce.isna().sum())

# Checking for Duplications
print(ecommerce.shape)
print(ecommerce.nunique())

# Checking data type
ecommerce.info()

# Checking data value
ecommerce.describe()
```
### 2. HANDLING
- **Missing values:**
  - 1454 rows in Description -> No info to fill in -> No action
  - 135080 rows in CustomerID -> depends on the RFM calculation -> Remove
- **Duplicates**: All columns have duplicates -> Table have no Primary Keys (Because 1 invoice having many purchased items, each line represents 1 item) -> No action
- **Data Type:**
  - InvoiceDate object -> datetime
  - UnitPrice: object with decimal values using commas -> float with decimal values using dots.
- **Data Values:**
  - Quantity < 0 -> Invoice refunded or canceled -> Remove
  - UnitPrice < 0 -> Assumption:Error -> Remove

```Python
# Remove missing values ​​in the CustomerID column
ecommerce.dropna(subset='CustomerID',inplace=True)

# Changing data type of InvoiceDate
ecommerce['InvoiceDate'] = pd.to_datetime(ecommerce['InvoiceDate'], format="%m/%d/%y %H:%M")

ecommerce['UnitPrice'] = ecommerce['UnitPrice'].astype(str)
ecommerce['UnitPrice'] = ecommerce['UnitPrice'].str.replace(',', '.')
ecommerce['UnitPrice'] = pd.to_numeric(ecommerce['UnitPrice'])

# Remove non-delivered invoices
rows_to_drop = ecommerce[(ecommerce['UnitPrice'] <= 0) | (ecommerce['Quantity'] <= 0)].index
ecommerce.drop(rows_to_drop, inplace = True)
```
## **III. RFM ANALYSIS & SEGMENTATION**
### 1. RFM CALCULATION
Calculating R,F,M respectively based on its definition:
  - **Recency**: number of days since last purchase of each customer.
  - **Frequency**: total number of transactions of each customer.
  - **Monetary Value**: total spending of each customer.
    
```Python
# Recency

# Find last purchase date of each customer
calculation = ecommerce.groupby("CustomerID").agg({"InvoiceDate":"max"})

# Calculate number of days from last purchase
calculation["CurrentDate"] = datetime(2011, 12, 31)
calculation["Recency"] = (calculation["InvoiceDate"] - calculation["CurrentDate"]).dt.days

# Frequency

# Find total purchase invoices per customer
calculation["Frequency"] = ecommerce.groupby("CustomerID")["InvoiceNo"].nunique()

# Monetory

# Calculate line total of each item in each invoice
ecommerce["LineTotal"] = ecommerce["Quantity"]*ecommerce["UnitPrice"]

# Calculate total purchase value per customer
calculation["Monetary"] = ecommerce.groupby("CustomerID")["LineTotal"].sum()
calculation.head()
```
|CustomerID |InvoiceDate	|CurrentDate	|Recency	|Frequency	|Monetary|
|---|---|---|---|---|---|
|12346.0	|2011-01-18 10:01:00	|2011-12-31	|-347|1	|77183.60|
|12347.0	|2011-12-07 15:52:00	|2011-12-31	|-24|7	|4310.00|
|12348.0	|2011-09-25 13:13:00	|2011-12-31	|-97|4	|1797.24|
|12349.0	|2011-11-21 09:51:00	|2011-12-31	|-40|1	|1757.55|
|12350.0	|2011-02-02 16:01:00	|2011-12-31	|-332|1	|334.40|

### 2. RANKING
**_Quintiles_** were used to **_assign scores_** to each RFM component of each customer. Then all separated R,F,M score were **_concated into 1 RFM score_**, which is foundation for segmentation stage and further analysis.
```Python
# Rank order of data to define cut point
orderFrequency = calculation["Frequency"].rank(method='first')

# Scoring R-F-M
calculation["F_score"] = pd.qcut(orderFrequency, 5, labels=["1", "2", "3", "4", "5"])
calculation[["R_score", "M_score"]] = calculation[["Recency", "Monetary"]].apply(lambda x: pd.qcut(x, 5, labels=["1", "2", "3", "4", "5"]))

# Concat R-F-M score
calculation["RFM_score"] = calculation.apply(lambda x:'%s%s%s' % (x["R_score"],x["F_score"],x["M_score"]),axis=1)
calculation["RFM_score"] = calculation["RFM_score"].astype(int)

calculation = calculation.reset_index()
calculation.head()
```
|CustomerID |InvoiceDate	|CurrentDate	|Recency	|Frequency	|Monetary|F_score	|R_score	|M_score	|RFM_score|
|---|---|---|---|---|---|---|---|---|---|
|12346.0	|2011-01-18 10:01:00	|2011-12-31	|-347|1	|77183.60|1|	1	|5	|115|
|12347.0	|2011-12-07 15:52:00	|2011-12-31	|-24|7	|4310.00|5|	5	|5	|555|
|12348.0	|2011-09-25 13:13:00	|2011-12-31	|-97|4	|1797.24|4|	2|	4|	244|
|12349.0	|2011-11-21 09:51:00	|2011-12-31	|-40|1	|1757.55|1|	4	|4|	414|
|12350.0	|2011-02-02 16:01:00	|2011-12-31	|-332|1	|334.40|1|	1|	2|	112|

### 3. SEGMENTATION
Segment customers of SuperStore based on **_11 pre-defined groups._**
```Python
# Convert comma-separated string to a list of RFM scores
segmentation["RFM Score"] = segmentation["RFM Score"].str.split(",")

# Transform each element of a list-like to a row
segmentation = segmentation.explode("RFM Score").reset_index(drop=True)
segmentation["RFM Score"] = segmentation["RFM Score"].astype(int)

# Merge segmentation with calculation df to show Segment name
rfm = calculation.merge(segmentation, how="left", left_on="RFM_score", right_on="RFM Score")
rfm.head()
```
|CustomerID |InvoiceDate	|CurrentDate	|Recency	|Frequency	|Monetary|F_score	|R_score	|M_score	|RFM_score| Segment|
|---|---|---|---|---|---|---|---|---|---|---|
|12346.0	|2011-01-18 10:01:00	|2011-12-31	|-347|1	|77183.60|1|	1	|5	|115| Cannot Lose Them|
|12347.0	|2011-12-07 15:52:00	|2011-12-31	|-24|7	|4310.00|5|	5	|5	|555| Champions|
|12348.0	|2011-09-25 13:13:00	|2011-12-31	|-97|4	|1797.24|4|	2|	4|	244| At Risk|
|12349.0	|2011-11-21 09:51:00	|2011-12-31	|-40|1	|1757.55|1|	4	|4|	414| Promising|
|12350.0	|2011-02-02 16:01:00	|2011-12-31	|-332|1	|334.40|1|	1|	2|	112| Lost customers|

## **IV. VISUALIZATION**
### 1. DISTRIBUTION OF R,F,M
```Python
colnames = ["Recency", "Frequency", "Monetary"]

for col in colnames:
    sns.displot(rfm[col], kde=True, height=3, aspect=4)
    plt.title(f"Distribution of {col}")
    plt.show()
```
<img width="600" alt="Dist of RFM" src="https://github.com/user-attachments/assets/27879450-6acc-4b6d-8267-0d8e1a6b58f4">

### 2. SEGMENT BY CUSTOMER COUNT
```Python
# Count number of customers per Segment
grp = rfm.groupby('Segment').agg({'CustomerID':'count'})
grp = grp.reset_index()
grp["Percent"] = grp["CustomerID"]/ (grp["CustomerID"].sum())

# Define colors
colors = sns.color_palette('GnBu',11)

# Draw treemap
fig,ax = plt.subplots(1, figsize=(10,5))
sq.plot(sizes=grp["CustomerID"],
              label=grp["Segment"],
              value=[f'{x*100:.2f}%' for x in grp["Percent"]],
              alpha=.8,
              color=colors,
              bar_kwargs=dict(linewidth=1.5, edgecolor="white")
              )
plt.title("Customer Size by Segment", fontsize=13)
plt.axis("off")
plt.show()
```
<img width="600" alt="Seg by cust count" src="https://github.com/user-attachments/assets/860c5988-f214-47e9-a4a4-152b269ea740">

### 3. SEGMENT BY TOTAL SALES
```Python
# Calculate Total Sales per Segment
grpM = rfm.groupby("Segment").agg({"Monetary":"sum"})
grpM = grpM.reset_index()
grpM["Percent"] = grpM["Monetary"]/ (grpM["Monetary"].sum())
grpM.head()

fig,ax = plt.subplots(1, figsize=(10,5))
sq.plot(sizes=grpM["Monetary"],
              label=grpM["Segment"],
              value=[f'{x*100:.2f}%' for x in grpM["Percent"]],
              alpha=.8,
              color=colors,
              bar_kwargs=dict(linewidth=1.5, edgecolor="white")
              )
plt.title("Total Sales by Segment", fontsize=13)
plt.axis("off")
plt.show()
```
<img width="600" alt="Seg by sales" src="https://github.com/user-attachments/assets/3309fde1-6d91-400a-be0b-774405a275e7">

### 4. ORDER RECENCY BY SEGMENT
```Python
# Order Recency by Segment
ax = rfm.groupby('Segment').Recency.mean().plot.bar(figsize=(10, 5), color=colors, width=0.8)
plt.title('Order Recency by Segment')

for p in ax.patches:
    height = p.get_height()
    # Place the label below the bar
    ax.annotate(f'{height:.2f}',
                (p.get_x() + p.get_width() / 2., height-11),
                ha='center', va='bottom', fontsize=9)

plt.show()
```
<img width="600" alt="Seg by sales" src="https://github.com/user-attachments/assets/0cbf8025-4f77-44d1-9239-bf9a50cb2683">

```Python
ax = rfm.groupby('Segment').Frequency.mean().plot.bar(figsize=(10, 5), color=colors, width=0.8)
plt.title('Order Frequency by Segment')

for p in ax.patches:
    height = p.get_height()
    # Place the label above the bar
    ax.annotate(f'{height:.2f}',
                (p.get_x() + p.get_width() / 2., height),
                ha='center', va='bottom', fontsize=9)

plt.show()
```
<img width="600" alt="Seg by sales" src="https://github.com/user-attachments/assets/f8d44811-e7e1-477b-9795-2c58f7e875b2">

## **V. INSIGHTS & RECOMMENDATIONS**
### 1. Characteristics of the 11 Customer Segments:
- **Champions:** The **best customers**—they shop frequently, spend a lot, and have **high engagement**. They are the **most valuable group** and should be retained.  
- **Loyal:** **Repeat customers** who regularly return and provide high value to the business.  
- **Potential Loyalists:** Customers with the potential to become **loyal**—they have made recent purchases and show **stable spending behavior**.  
- **Promising:** **New customers** who have just started shopping and **show growth potential**.  
- **New Customers:** Customers who have **just made their first transaction**. The focus should be on **building relationships** to develop long-term value.  
- **Cannot Lose Them:** **Important customers** who used to spend significantly but have **recently become inactive**. Immediate action is needed to **retain them**.  
- **At Risk:** Customers who **used to shop frequently** but have **reduced interactions** recently. They need **re-engagement campaigns** to bring them back.  
- **Need Attention:** Customers who **are not highly active** but still hold **some value**. They need **additional motivation** to return.  
- **About to Sleep:** Customers with **low engagement** and **few recent transactions**. With the right activation strategies, they could be **converted into loyal customers**.  
- **Hibernating Customers:** Customers who have **made purchases in the past** but have **not been active recently**.  
- **Lost Customers:** Customers who **have not interacted** for a **long time**. **Reactivating them will be difficult** and they have **low potential**.  

### 2. Customer Segments at SuperStore:

Approximately **19% of customers** belong to the **Champion** segment—representing the **best customers**. Two other **high-potential** segments, **Loyal** and **Potential Loyalist**, together account for nearly **20%** of the customer base. **Champions and Loyal customers** also contribute the **highest revenue**, with **Champions generating around 62% of total revenue**.  
➡ **Strategy:** Offer **exclusive perks, loyalty programs, or premium (VIP) services** to enhance engagement and retention.  

More than **one-third of the customers** are at risk of **not returning**, belonging to the **Hibernating Customers, Lost Customers, and About to Sleep** segments. Notably, **Hibernating Customers** make up the **second-largest** customer group at **16.32%**.  
➡ **Strategy:** **Identify the reasons** why these customers are not returning and **address** the underlying issues.  

Approximately **18% of customers** fall into the **Need Attention, At Risk, and Cannot Lose Them** categories, meaning they are also at risk of **churning** and need **re-engagement efforts**.  
➡ **Strategy:** **Provide attractive incentives** such as **discounts, free gifts, or special promotions** to encourage their return.

<img width="600" alt="Seg by sales" src="https://github.com/user-attachments/assets/fe065410-fc2b-4d61-856c-7dfc74c68792">

### 3. Proposed Solutions for High-Value Customer Segments (Champion, Loyal, and Potential Loyalist) 
#### **1. Champions**  

- **Enhance customer service** through **loyalty programs**, including **special promotions and exclusive offers** to show appreciation while allowing customers to accumulate reward points.  
- **Focus on efficient customer support**, ensuring quick handling of **returns, exchanges, and warranties**.  
- **Increase revenue** by recommending **higher-value products or product bundles** based on the following criteria:  
  - **Itemlist1:** A list of products priced **above the average price** of items previously purchased by **Champions**.  
  - **Itemlist2:** Products that are among the **most frequently purchased items** by Champions.
```Python
# Find Champion customers
champions = rfm[rfm["Segment"] == "Champions"]
champions = champions[["CustomerID"]]

# Find items purchased by Champions + their unit price
item = champions.merge(ecommerce, on="CustomerID", how="left")
item_price = item[["Description","UnitPrice"]]
item_price = item_price.groupby("Description").agg({"UnitPrice":"mean"})
item_price = item_price.reset_index()

# Find avg price of items purchased by Champions
mean_price = item_price["UnitPrice"].mean()

# List of items purchased by Champions, having price > avg price of all items purchased by Champions
itemlist1 = item_price.loc[item_price["UnitPrice"] > mean_price, ["Description", "UnitPrice"]]

print(itemlist1)
```
<img width="600" alt="Seg by sales" src="https://github.com/user-attachments/assets/a15b347a-7f1c-4f69-b8ce-6afe038df9ed">

#### **2. Loyal & Potential Loyalist**  

Provide **targeted promotions and offers** tied to **specific spending thresholds** to **increase engagement and spending** within this group.  

##### **Example 1**  

- **Offer:** Free gifts to encourage transactions that **exceed the group's average order value**.  
- **Eligibility Condition:** Transactions must have a value **higher than the Average Order Value** of **Loyal and Potential Loyalist customers**.  
- **Applicable Customer Group:** Only for **Loyal and Potential Loyalist** customers.  
- **Group’s Average Order Value:** **~$379**.  

```Python
# Find customers in Loyal & Potential Loyalist segment
loyal = rfm[(rfm["Segment"] == "Loyal") | (rfm["Segment"] == "Potential Loyalist")]
loyal = loyal[["CustomerID"]]

# Filter invoices of Loyal & Potential Loyalist customers
loyal_iv = loyal.merge(ecommerce, on="CustomerID", how="left")

# Find the invoice value
loyal_value = loyal_iv.groupby("InvoiceNo").sum("LineTotal")

# Find the average invoice value
loyal_avg = loyal_value['LineTotal'].mean()

loyal_avg

Ouput: 378.9460510510511
```

##### **Example 2**  

- **Offer:** Higher-value **free gifts** to encourage **larger transactions**.  
- **Eligibility Condition:** Transactions must have a value **higher than the Company's Average Order Value**.  
- **Applicable Customer Group:** **All customers**.  
- **Company's Average Order Value:** **~$480**.

```Python
# Find the value of all delivered invoices
totalvalue = ecommerce.groupby("InvoiceNo").sum("LineTotal")

# Find the average value of all delivered invoices
avgvalue = totalvalue['LineTotal'].mean()

avgvalue

Output: 480.86595639974104
```
