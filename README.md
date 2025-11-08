#  Cilantro Analytical Engineering Case Study

## 📋 Project Overview
This project analyzes customer purchasing behavior to understand **visit frequency**, **spending patterns**, and **Customer Lifetime Value (CLTV)** for  *Cilantro* Cafes all over Egypt.  
Using transaction and customer data, we calculate heuristic CLTV, segment users via **RFM (Recency, Frequency, Monetary)** analysis, and summarize behavior by customer segment and acquisition channel.

---

## 🧱 Datasets

| Dataset | Description | Key Columns |
|----------|--------------|--------------|
| **customers.csv** | Contains customer profile and acquisition info | `customer_id`, `acquisition_channel`, `channels_used` |
| **transactions.csv** | Includes all purchase events | `transaction_id`, `customer_id`, `transaction_date`, `total_amount` |

---

## 🧹 1. Data Modeling & Cleaning

### **Joining & Preprocessing**
- Standardized `customer_id` across both datasets by removing prefixes (`USER_`, `APP_`, `WEB_`, `DEL_`).
- Removed walk-in orders (no customer linkage).
- Converted `transaction_date` to datetime.
- Flagged transactions with:
  - **Negative values:** converted to positive (assumed entry error).  
  - **Zero values:** dropped (invalid or promotional sales).
- One-hot encoded the `channels_used` field for richer channel analysis.

### **Join Logic**


```python
merged_df = transactions.merge(customers, left_on='customer_id', right_on='user_id', how='left')
```
Join Type: Left join (keeps all transactions, even if a customer record is missing incase of a walk-in).

Grain: One record per transaction.

After the merge, we aggregated transactions to the customer level using groupby operations.
## 💰 Customer Lifetime Value (CLTV)

### **Formula**

**CLTV = AOV × Normalized Frequency x lifespan**

### **Explanation**

- **AOV (Average Order Value)**  
  Calculated as the total transaction amount divided by the number of purchases per user.  
  ```python
  cust['AOV'] = cust['monetary_value'] / cust['frequency']


**2. Normalized Frequency**  
Normalized purchase frequency over the customer’s active duration.  

```python
cust['days_since_first_purchase'] = (analysis_date - cust['first_tx']).dt.days

def calc_purchase_rate(row):
    if row['frequency'] < 3 or row['days_since_first_purchase'] < 7:
        return row['frequency']
    else:
        return round((row['frequency'] / row['days_since_first_purchase']) * 365)
lifespan = 2
cust['norm_freq'] = cust.apply(calc_purchase_rate, axis=1)
```
### 🧭 Why Normalized Frequency?

Using raw purchase frequency alone can be misleading.  

For example:  
> A customer who made **12 purchases in one month** is far more engaged than another who made **12 purchases over an entire year**.

To address this, frequency is **normalized** by the customer’s active duration.  

- **`days_since_first_purchase`** measures the number of days between a customer’s first purchase and the analysis date (end of 2024).  
- The **first purchase date** is treated as the customer’s “signup date,” so earlier months are assumed to reflect consistent purchase behavior.  

This normalization provides a **more realistic view of customer engagement and spending intensity** over time, rather than just counting total transactions.

sources state that coffee shop customer lifetime often ranges 1–3 years for cafés so the safest option is to assume 2 years.

## 🎯 RFM Segmentation

Each customer was scored across **Recency**, **Frequency**, and **Monetary** metrics, then grouped into **5 segment levels** per metric.

### Bin Definitions

```python
recency_bins = [0, 7, 21, 55, 90, cust['recency'].max()]
recency_labels = [5, 4, 3, 2, 1]

freq_bins = [0, 4, 26, 52, 104, cust['norm_freq'].max()] 
freq_labels = [1, 2, 3, 4, 5]

monetary_bins = [0, 60, 120, 180, 250, cust['AOV'].max()]
monetary_labels = [1, 2, 3, 4, 5]
```

### 🧮 RFM Scoring Logic

- **Recency:** adjusted the margins to fit a coffee shop establishment.  
- **Frequency:** I classified the Customers into 5 catigories: twice a week | weekly | bi-weekly......  
- **Monetary:** Scanned the Cilantro app to be able to make reasonable ranges and used the AOV not the sum as i want to focus on the quality not quantity or orders.  

The **combined RFM score** (sum of Recency, Frequency, and Monetary scores) determines the **customer segment labels**, such as:

| **Segment**            | **Description**                            |
|-------------------------|--------------------------------------------|
| **Champions**           | Frequent, high-spending, recent buyers     |
| **Loyal Regulars**      | Consistent and valuable                    |
| **Potential Loyalists** | New but promising                          |
| **At Risk**             | Declining activity                         |
| **Inactive**            | Long inactive or churned                   |


## 🔮 Predictive CLTV

We trained ML models to estimate future CLTV using historical features: `recency`, `normalized frequency`, `AOV`, `lifetime_days`, `acquisition_channel`, and `segment`.
Horizon = 1 year

**Random forest Performance:**
- R² Score: 0.67
- MAE: 32.1
- 
**XGBoost Performance:**
- R² Score: 0.744
- MAE: 32.25

This model predicts each customer’s CLTV for the next 12 months.

# 📊 Generated Output Tables
## 1️⃣ User-Level Table

Contains all customer-level computed metrics:

| Column          | Description                                  |
|------------------|----------------------------------------------|
| `customer_id`    | Unique customer identifier                   |
| `recency`        | Days since last transaction                  |
| `frequency`      | Total number of transactions                 |
| `monetary_value` | Total spend by customer                      |
| `CLTV`           | Heuristic Lifetime Value                     |
| `segment`        | Assigned RFM-based segment                   |
| `lifetime_days`  | Days between first and last purchase         |
## 2️⃣ Segment Summary Table

Aggregates performance by RFM segment.

| Column              | Description                             |
|----------------------|-----------------------------------------|
| `segment`            | Segment label                           |
| `num_users`          | Number of customers in that segment      |
| `avg_frequency`      | Mean number of transactions              |
| `avg_monetary`       | Mean AOV or spending                     |
| `avg_CLTV`           | Average CLTV                             |
| `avg_lifetime_days`  | Average customer lifetime                |

## 3️⃣ Acquisition Channel Summary Table

Aggregates metrics by the marketing channel through which users signed up.

| Column               | Description                                         |
|-----------------------|-----------------------------------------------------|
| `acquisition_channel` | Channel name (e.g., Paid, Organic, Referral)        |
| `num_users`           | Number of users acquired through that channel       |
| `avg_frequency`       | Mean purchase rate                                  |
| `avg_monetary`        | Average spending per customer                       |
| `avg_CLTV`            | Average lifetime value                              |
| `avg_lifetime_days`   | Average retention duration                          |


## 3️⃣ Predictive Output Table

Holds results of the XGBoost model for the CLTV.

| Column               | Description                                         |
|-----------------------|-----------------------------------------------------|
| `Customer_id`         | Unique customer identifier                          |
| `Pred_CLTV`           | Predicted lifetime value                            |
| `Horizon`             | Prediction range                                    |


### 💾 Output Files

Each table is exported as a CSV file for reporting and further analysis:

```python
user_summary.to_csv("user_level_summary.csv", index=False)
segment_summary.to_csv("segment_summary.csv", index=False)
channel_summary.to_csv("acquisition_channel_summary.csv", index=False)
predictive_output.to_csv('predictive_output.csv', index=False)
```

## 💡 Key Insights

- **Champions** represent roughly **1% of customers** but contribute the **highest CLTV**, indicating strong loyalty and high-value behavior.  
- **Potential Loyalists** and **At Risk** segments make up the **majority of users**, making them ideal targets for **retention and reactivation campaigns**.  
- **Inactive users** form the **largest portion** of the customer base — implementing **personalized offers, loyalty rewards, or targeted discounts** could help bring them back and increase engagement.
