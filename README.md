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
*   **Total Orders:**
    ```dax
    Total_Orders = DISTINCTCOUNT(olist_orders_dataset[order_id])
    ```
*   **Average Order Value (AOV):**
    ```dax
    Average_Order_Value = [Total_Revenue] / [Total_Orders]
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

## 🖥️ Dashboard Layout & Features

I structured the report into three pages to answer different business questions:

1.  **Sales Performance:** Highlights revenue growth over time, top-selling product categories, and a map showing which Brazilian states generated the most sales.
2.  **Logistics & Delivery Speed:** Focuses on shipping performance, tracking which states suffer the longest delays, and analyzing carrier efficiency.
3.  **Customer Reviews & Payments:** Shows average review scores by product category and analyzes how customers prefer to pay (credit card installments vs. boleto/voucher).


---

## 🧠 Roadblocks & What I Learned

Since this was a learning project, I ran into a few tricky challenges:

*   **Active vs. Inactive Relationships:** The orders table has three different date columns: `purchase_timestamp`, `approved_at`, and `delivered_customer_date`. When I tried to link them to my calendar, Power BI blocked me. I learned that you can only have one active relationship at a time. I solved this by keeping the purchase date active and writing DAX measures utilizing `USERELATIONSHIP` to filter by delivery dates when calculating logistics metrics.
*   **Translating Portuguese Categories:** The raw products table lists categories in Portuguese. To make the dashboard readable for English speakers, I used Power Query to perform a left outer join with the translation lookup table, replacing the Portuguese names with their English equivalents before loading the data.
*   **Handling Raw Dates:** Some of the timestamp columns were loaded as text strings by default. I used Power Query to transform them into proper date/time types so that my time-intelligence calculations wouldn't break.

---

## ⚙️ How to Run This Locally

Because Power BI hardcodes local file paths, you will see a Data Source Error when you first open the dashboard. Here is how to fix it in 2 minutes:

1.  Open **Power BI Desktop**.
2.  Open `ecommerce.pbix` from this directory.
3.  Go to the Home tab, click the arrow under **Transform Data**, and select **Data source settings**.
4.  For each CSV file, click **Change Source...**, click **Browse**, and select the matching CSV file from the `Ecommerce/data/` folder on your computer.
5.  Click **Close** and then **Apply Changes**—the dashboard will load all your local data automatically!
