# Brazilian E-Commerce Analysis Dashboard (Olist)

I built this Power BI dashboard to analyze sales trends, delivery performance, and customer satisfaction using the public Olist Brazilian E-commerce dataset. 

Most beginner projects use a single flat Excel sheet, but I wanted to challenge myself with a more realistic data engineering scenario. That's why I chose this dataset—it forced me to design a relational database model (Star Schema) and connect multiple tables.

---

## 📐 Data Model & Schema Design

The raw data is stored in the `data/` folder and was split across **8 separate CSV files** containing transaction details from 2016 to 2018. Before building any visuals, I had to clean the data and design a Star Schema to connect everything.

### The Tables:
*   `olist_customers_dataset.csv` (Demographics by city and state)
*   `olist_orders_dataset.csv` (Order tracking timestamps and statuses)
*   `olist_order_items_dataset.csv` (Item prices, shipping limits, and quantities)
*   `olist_order_payments_dataset.csv` (Payment values, installments, and payment types)
*   `olist_order_reviews_dataset.csv` (Review scores and timestamps)
*   `olist_products_dataset.csv` (Product specs, categories, and dimensions)
*   `olist_sellers_dataset.csv` (Seller locations)
*   `product_category_name_translation.csv` (English translations for Portuguese product categories)

### Connecting the Data (Star Schema)
I built a **Star Schema** centered around the transactional fact tables (`olist_order_items_dataset` and `olist_orders_dataset`), with one-to-many relationships pointing from the surrounding dimension tables:
*   `customers` ➔ `orders` (via `customer_id`)
*   `products` ➔ `order_items` (via `product_id`)
*   `sellers` ➔ `order_items` (via `seller_id`)
*   `payments` ➔ `orders` (via `order_id`)
*   `reviews` ➔ `orders` (via `order_id`)

---

## 📊 Core DAX Calculations

Here are some of the key DAX measures I wrote to calculate KPIs and track logistics performance:

*   **Total Revenue:**
    ```dax
    Total_Revenue = SUM(olist_order_items_dataset[price])
    ```
*   **Total Orders (Corrected to avoid relationship filter bugs):**
    ```dax
    Total_Orders = DISTINCTCOUNT(olist_order_items_dataset[order_id])
    ```
*   **True YoY Growth (Corrected to avoid "average of averages" skew):**
    ```dax
    Growth % = 
    VAR CurrentRev = [Total Revenue]
    VAR LastYearRev = [Revenue LY]
    VAR MaxYear = MAX('Date Table'[Year])
    VAR MaxYearSales = CALCULATE([Total Revenue], 'Date Table'[Year] = MaxYear)
    VAR PrevYearSales = CALCULATE([Total Revenue], 'Date Table'[Year] = MaxYear - 1)
    RETURN
    IF(
        ISFILTERED('Date Table'[Year]),
        IF(LastYearRev > 10000, DIVIDE(CurrentRev - LastYearRev, LastYearRev), BLANK()),
        DIVIDE(MaxYearSales - PrevYearSales, PrevYearSales)
    )
    ```
*   **Average Order Value (AOV):**
    ```dax
    Average_Order_Value = [Total_Revenue] / [Total_Orders]
    ```
*   **Geocoding Full Location (Calculated Column):**
    ```dax
    Full_Location = olist_customers_dataset[customer_state] & ", Brazil"
    ```
*   **Delivery Delay (Days):**
    This measure tracks how early or late orders arrived compared to their estimated delivery date:
    ```dax
    Actual_vs_Estimated_Delivery_Days = 
    AVERAGEX(
        FILTER(olist_orders_dataset, NOT(ISBLANK(olist_orders_dataset[order_delivered_customer_date]))),
        INT(olist_orders_dataset[order_delivered_customer_date] - olist_orders_dataset[order_estimated_delivery_date])
    )
    ```
    *(Negative numbers mean the shipping was faster than estimated; positive numbers show a delay).*

---

## 🖥️ What's Inside the Dashboard (Pages & Visuals)

I split the report into 5 pages to keep things clean and avoid cluttering a single screen with too many charts. Here is how I designed each page:

### 1. Executive Overview
This is the home page of the report, meant to give a quick snapshot of how the business is doing. I wanted a clean layout to show high-level numbers like total sales ($13.2M), overall order count (99K), customer count (96K), and Net Revenue Margin (83.0%). I also calculated the true year-over-year growth (which stands at 21.1%) and added a monthly revenue trend line and a map visual to quickly see which states in Brazil generate the most sales.

![Executive Overview](data/Screenshot%202026-06-20%20223729.png)

---

### 2. Revenue & Margin Drivers
This page is all about the money. I built this to figure out where our profit margins are actually coming from. It lets you compare gross sales directly against net revenue after shipping/freight costs. I also added handy filters for years and months so you can drill down into specific periods, and mapped out revenue by seller city to spot where the top merchants are located.

![Revenue & Margin Drivers](data/Screenshot%202026-06-20%20223800.png)

---

### 3. Customer Segmentation (RFM)
This was one of the most interesting parts to build. I wanted to group our customer base using the RFM model (Recency, Frequency, Monetary). I wrote some DAX to calculate how recently a customer bought, how often they buy, and how much they spend, and then classified them into groups like "At Risk", "Loyal", or "New". I put a donut chart showing the split and a scatter plot comparing **Recency vs. Monetary** to visualize the customer value distribution.

![Customer Segmentation](data/Screenshot%202026-06-20%20223824.png)

---

### 4. Product & Category Performance
Here, I focused purely on inventory and product catalog analytics. Since Olist has over 30,000 unique products, I wanted to see which categories make up the bulk of our sales. I used a treemap for the top 10 categories by revenue and added metrics to show how many unique products we have per category. It's really helpful for identifying which items are high-volume but low-margin.

![Product & Category Performance](data/Screenshot%202026-06-20%20223852.png)

---

### 5. Delivery, Reviews & Operations
Lastly, I built an operations page to track logistics and customer satisfaction. This is where I calculated the actual vs. estimated delivery gap and on-time delivery rates. I wanted to see if shipping speed directly affected customer review scores. I put in a trend line for late deliveries and a breakdown comparing review ratings for on-time vs. late shipments, which clearly shows that late orders get way worse reviews.

![Delivery, Reviews & Operations](data/Screenshot%202026-06-20%20223920.png)

---

## 🧠 Roadblocks & What I Learned

Since this was a learning project, I ran into a few tricky challenges that took me some time to figure out:

*   **Active vs. Inactive Relationships:** This was a major headache. The orders table has three different date columns: purchase timestamp, approved date, and delivered customer date. When I tried to link all of them to my calendar table, Power BI threw errors because you can't have multiple active relationships between the same two tables. I learned about inactive relationships and how to use the `USERELATIONSHIP()` function in DAX to activate them on the fly for specific logistics calculations.
*   **The "Average of Averages" Growth Trap:** Initially, the overall YoY growth KPI showed 208.2% because the card visual was mathematically averaging monthly YoY percentages. I rewrote the DAX using year filtering conditions to ensure it divides the absolute aggregated values, showing the true overall growth of 21.1%.
*   **Cross-Filter Direction modeling:** The category charts initially displayed exactly 96K orders for all rows because the category filter from the products dimension could not travel upstream to the orders table. I fixed this by redefining the `[Total Orders]` measure to perform a distinct count on the downstream `olist_order_items_dataset[order_id]` table.
*   **Geocoding with State Abbreviations:** Bing Maps plotted Brazilian state abbreviations (like SP, AL, AM) all over the globe (Spain, Alabama, Armenia). I created a calculated column concatenating `", Brazil"` and categorized the column as a "State or Province" geographical type to force the map engine to zoom in and align correctly inside Brazil.
*   **Translating Portuguese Categories:** Since Olist is a Brazilian dataset, the product categories were in Portuguese. I didn't want the dashboard to have mixed languages. I found a translation table in the dataset and used Power Query to do a left outer join. Now all category visuals use the English names automatically.
*   **Handling Raw Dates:** Some date columns were imported as text strings, which broke my time-intelligence calculations. I had to go back to Power Query and change the column types to Date/Time before loading them. It was a good reminder to always clean and verify data types first.

---

## ⚙️ How to Run This on Your Computer

Because Power BI hardcodes absolute local file paths, you will see a Data Source Error when you first open the dashboard. Here is how to fix it in 2 minutes:

1.  Open **Power BI Desktop**.
2.  Open `ecommerce.pbix` from this directory.
3.  Go to the Home tab, click the arrow under **Transform Data**, and select **Data source settings**.
4.  For each CSV file, click **Change Source...**, click **Browse**, and select the matching CSV file from the `Ecommerce/data/` folder on your computer.
5.  Click **Close** and then **Apply Changes**—the dashboard will load all your local data automatically!

