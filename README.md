# Inventory & Supply Chain Analytics System

A full-stack inventory management system built with MySQL, Python, and Streamlit. The system combines a normalized relational database with an interactive web dashboard — enabling non-technical business users to monitor real-time inventory KPIs and execute operational workflows without writing a single SQL query.

---

## Business Problem

Most small and mid-size businesses manage inventory through Excel sheets or manual SQL queries that only technical teams can run. Operations managers, procurement heads, and warehouse staff have no real-time visibility into stock levels, supplier performance, or reorder needs without requesting help from IT.

This system solves that — the SQL handles all business logic, the Python connects it to a live interface, and the Streamlit dashboard puts operational control directly in the hands of non-technical users.

---

## System Architecture

```
MySQL Database (Backend)
        │
        │  mysql-connector-python
        │
Python Layer (p2dbfunctions.py)
  └── All SQL queries, stored procedure calls, db commits
        │
Streamlit UI (p2app.py)
  └── All user interaction, forms, dropdowns, display logic
```

**Separation of concerns:** Database logic lives entirely in `p2dbfunctions.py`. UI logic lives entirely in `p2app.py`. Changing a query never requires touching the UI, and changing the UI never requires touching the SQL.

---

## Database Schema

5 normalized tables connected through foreign key relationships:

```
suppliers (supplier_id PK)
    │
    ├──► products (supplier_id FK)
    │         │
    │         ├──► reorders (product_id FK)
    │         ├──► shipments (product_id FK, supplier_id FK)
    │         └──► stock_entries (product_id FK)
    │
    └──► shipments (supplier_id FK)
```

### Table Descriptions

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| suppliers | Vendor master data | supplier_id, supplier_name, contact_name, email, phone |
| products | Product catalog with live stock | product_id, product_name, category, price, stock_quantity, reorder_level, supplier_id |
| reorders | Purchase orders raised | reorder_id, product_id, reorder_quantity, reorder_date, status |
| shipments | Delivery records | shipment_id, product_id, supplier_id, quantity_received, shipment_date |
| stock_entries | Complete stock movement audit trail | entry_id, product_id, change_quantity, change_type (Sale/Restock), entry_date |

**Why normalized:** Supplier contact details live in one place — changing a supplier's email requires updating one row, not every product row. Product prices live in one place — recalculating sales value across 8 months always uses the current price. No redundant data, no inconsistency risk.

---

## Dataset

| Property | Value |
|----------|-------|
| Total Suppliers | 50 |
| Total Products | 206 |
| Total Categories | 5 |
| Products below reorder (no pending order) | 14 |
| 8-month Restock Value | $132,115.56 |

---

## Advanced SQL Used

### 1. Multi-table Joins
Used to calculate sales value — stock_entries knows quantity moved, products knows price, JOIN brings them together:

```sql
SELECT ROUND(SUM(ABS(se.change_quantity) * p.price), 2) AS total_sales_value
FROM stock_entries AS se
JOIN products AS p ON p.product_id = se.product_id
WHERE se.change_type = "Sale"
AND se.entry_date >= (SELECT DATE_SUB(MAX(entry_date), INTERVAL 8 MONTH) 
                      FROM stock_entries);
```

### 2. Correlated Subquery with NOT IN
Finds products below reorder level that do NOT already have a pending reorder — avoids false alerts for things already being handled:

```sql
SELECT COUNT(*) AS below_reorder_count
FROM products AS p
WHERE p.stock_quantity < p.reorder_level
AND p.product_id NOT IN (
    SELECT DISTINCT product_id
    FROM reorders
    WHERE status = 'Pending'
);
```

### 3. Dynamic Rolling Date Filter
8-month window calculated dynamically — always relative to the most recent transaction, not a hardcoded date:

```sql
AND entry_date >= (SELECT DATE_SUB(MAX(entry_date), INTERVAL 8 MONTH) 
                   FROM stock_entries)
```

### 4. Stored Procedure — AddNewProductManualID
Adds a new product and simultaneously creates the initial shipment and stock entry records across 3 tables in one call:

```sql
CREATE PROCEDURE AddNewProductManualID(
    IN p_name VARCHAR(225), IN p_category VARCHAR(100),
    IN p_price DECIMAL(10,2), IN p_stock INT,
    IN p_reorder INT, IN p_supplier INT
)
BEGIN
    -- Insert into products
    -- Insert into shipments  
    -- Insert into stock_entries
END
```

Called from Python:
```python
cursor.execute("CALL AddNewProductManualID(%s, %s, %s, %s, %s, %s)", params)
db.commit()
```

### 5. Stored Procedure with Transaction — MarkReorderAsReceived
The most critical procedure. Receiving a reorder must update 4 tables atomically — wrapped in START TRANSACTION / COMMIT so all 4 succeed together or all 4 roll back:

```sql
CREATE PROCEDURE MarkReorderAsReceived(IN in_reorder_id INT)
BEGIN
    START TRANSACTION;
    
    -- Update reorder status to 'Received'
    UPDATE reorders SET status = 'Received' WHERE reorder_id = in_reorder_id;
    
    -- Increase product stock quantity
    UPDATE products SET stock_quantity = stock_quantity + qty WHERE product_id = prod_id;
    
    -- Create new shipment record
    INSERT INTO shipments(shipment_id, product_id, supplier_id, quantity_received, shipment_date)
    VALUES (new_shipment_id, prod_id, sup_id, qty, CURDATE());
    
    -- Log restock in stock_entries
    INSERT INTO stock_entries(entry_id, product_id, change_quantity, change_type, entry_date)
    VALUES (new_entry_id, prod_id, qty, 'Restock', CURDATE());
    
    COMMIT;
END
```

**Why the transaction matters:** Without it, if step 3 (shipment insert) fails, steps 1 and 2 already committed — reorder shows "Received" and stock increased, but no shipment record exists. Data is now inconsistent. The transaction guarantees all-or-nothing execution.

### 6. VIEW — Unified Product History
UNION ALL combines shipment records and stock entry records into one unified timeline per product:

```sql
CREATE OR REPLACE VIEW product_inventory_history AS
SELECT product_id, 'Shipment' AS record_type, shipment_date AS record_date,
       quantity_received AS quantity, NULL AS change_type
FROM shipments

UNION ALL

SELECT product_id, 'Stock Entry' AS record_type, entry_date AS record_date,
       change_quantity AS quantity, change_type
FROM stock_entries;
```

Querying product history anywhere becomes one clean line:
```sql
SELECT * FROM product_inventory_history WHERE product_id = 101 ORDER BY record_date DESC;
```

---

## Dashboard Features

### Page 1 — Basic Information

**6 Real-time KPIs (live from MySQL on every load):**

| KPI | Value | Business Meaning |
|-----|-------|-----------------|
| Total Suppliers | 50 | Vendor base size |
| Total Products | 206 | Catalog size |
| Total Categories | 5 | Product diversity |
| 8-month Sales Value | — | Revenue from sales |
| 8-month Restock Value | $132,115.56 | Procurement spend |
| Below reorder, no pending | 14 | Products needing urgent action |

The last KPI is the most operationally critical — uses the NOT IN subquery to surface only products that need immediate attention, excluding those already being handled.

**3 Data Tables:**
- Supplier contact details
- All products with their supplier and current stock level
- Products that need to be reordered (stock < reorder_level)

### Page 2 — Operational Tasks

**4 Inventory Management Workflows:**

| Workflow | What happens in the database |
|----------|------------------------------|
| Add New Product | Calls `AddNewProductManualID` — inserts across products, shipments, stock_entries simultaneously |
| Product History | Queries `product_inventory_history` VIEW filtered by selected product |
| Place Reorder | Inserts new row into reorders table with status 'Pending' and CURDATE() |
| Receive Reorder | Calls `MarkReorderAsReceived` — transaction updates 4 tables atomically |

---

## Project Structure

```
Interactive-UI-for-SQL/
│
├── app/
│   ├── p2app.py             # Streamlit UI — all user interaction
│   └── p2dbfunctions.py     # Database layer — all SQL queries and procedure calls
│
├── database/
│   ├── schema.sql           # All CREATE TABLE, stored procedures, VIEW definitions
│   └── project2.sql         # Individual SQL queries for reference
│
├── data/
│   ├── products.csv
│   ├── suppliers.csv
│   ├── reorders.csv
│   ├── shipments.csv
│   └── stock_entries.csv
│
└── README.md
```

---

## Installation & Setup

```bash
# Clone the repository
git clone https://github.com/yourusername/inventory-supply-chain-system.git
cd inventory-supply-chain-system

# Install Python dependencies
pip install streamlit pandas mysql-connector-python

# Set up MySQL database
# 1. Open MySQL and create the database
mysql -u root -p
CREATE DATABASE dummy_project;
USE dummy_project;

# 2. Run the schema file to create tables and procedures
source database/schema.sql;

# 3. Load CSV data into tables
# (Import via MySQL Workbench or use LOAD DATA INFILE)

# 4. Update connection credentials in p2dbfunctions.py
# host, user, password, database

# Run the Streamlit dashboard
streamlit run app/p2app.py
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Database | MySQL 8.0 |
| DB Connection | mysql-connector-python |
| Backend Logic | Python 3, Pandas |
| Frontend UI | Streamlit |
| IDE | PyCharm |

---

## Key Design Decisions

**Why stored procedures over raw SQL in Python?**
Business logic (what tables to update, in what order, with what data) lives inside the database — not scattered across Python files. If the business rule changes (e.g. adding a fourth table to update on reorder receipt), you change the stored procedure in one place. All callers automatically get the new behavior.

**Why a transaction for receive reorder?**
Receiving a reorder is a multi-step operation that must succeed completely or not at all. Partial execution leaves the database in an inconsistent state — stock updated but no shipment record, or reorder marked received but stock not increased. The transaction guarantees atomicity.

**Why a VIEW for product history?**
The raw SQL combining shipments and stock_entries via UNION ALL is complex. Wrapping it in a VIEW means any future query needing product history writes one line instead of twenty. It's reusability and abstraction — the same principle as functions in Python.

**Why separate p2dbfunctions.py from p2app.py?**
Separation of concerns. Database logic is independently testable, independently modifiable, and completely decoupled from UI decisions. This mirrors professional backend/frontend separation in production web applications.

---

## What This Project Demonstrates

- Normalized relational database design from scratch
- Advanced SQL — joins, correlated subqueries, dynamic date filters, aggregations
- Stored procedures encapsulating multi-table business logic
- ACID-compliant transactions for data integrity
- SQL VIEWs for query abstraction and reusability
- Python-MySQL integration via mysql-connector
- Streamlit dashboard with forms, dropdowns, and real-time database reads
- Clean code architecture — database layer fully separated from UI layer

---

## Author

**Kanishk**
B.Com | Masters in Applied Statistics and Informatics
Gokhale Institute of Politics and Economics, Pune

[GitHub](https://github.com/kanishknarwani) | [LinkedIn](https://linkedin.com/in/kanishk-narwani)
