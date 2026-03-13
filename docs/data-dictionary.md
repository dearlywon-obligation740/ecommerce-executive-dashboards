<h1>Data Dictionary</h1>

<h2>Source Dataset</h2>

<p>The <a href="https://www.kaggle.com/datasets/carrie1/ecommerce-data">Online Retail Dataset</a> from Kaggle. UK-based online retailer, December 2010 to December 2011. 541,909 rows of transactional line items.</p>

---

<h2>Cleaned Fact Table: <code>fact_transactions</code></h2>

<p>After cleaning in Python, each row is still a single line item on an invoice. No rows were deleted. Boolean flags were added so the data can be filtered dynamically in Excel.</p>

<h3>Original Fields</h3>

<table>
  <tr><th>Column</th><th>Type</th><th>Description</th></tr>
  <tr><td><code>invoice_no</code></td><td>Text</td><td>6-digit invoice number. Cancellations start with "C".</td></tr>
  <tr><td><code>stock_code</code></td><td>Text</td><td>Product code. Some are non-product codes (e.g. POST, DOT, M, BANK CHARGES).</td></tr>
  <tr><td><code>description</code></td><td>Text</td><td>Product name. Cleaned of non-breaking spaces and standardised to title case.</td></tr>
  <tr><td><code>quantity</code></td><td>Integer</td><td>Units per line item. Negative for returns and cancellations.</td></tr>
  <tr><td><code>invoice_date</code></td><td>DateTime</td><td>Date and time of the transaction.</td></tr>
  <tr><td><code>unit_price</code></td><td>Float</td><td>Price per unit in GBP. Can be negative for financial adjustments.</td></tr>
  <tr><td><code>customer_id</code></td><td>Int64</td><td>Unique customer identifier. Null for guest purchases (~25% of rows).</td></tr>
  <tr><td><code>country</code></td><td>Text</td><td>Country of the customer.</td></tr>
</table>

<h3>Derived Fields</h3>

<table>
  <tr><th>Column</th><th>Type</th><th>Description</th></tr>
  <tr><td><code>year</code></td><td>Integer</td><td>Extracted from invoice_date. Values: 2010, 2011.</td></tr>
  <tr><td><code>month</code></td><td>Integer</td><td>Extracted from invoice_date. Values: 1 to 12.</td></tr>
  <tr><td><code>year_month</code></td><td>Text</td><td>Period string for time-series grouping (e.g. "2011-03").</td></tr>
  <tr><td><code>weekday</code></td><td>Text</td><td>Day of the week (Monday to Sunday).</td></tr>
  <tr><td><code>is_guest</code></td><td>Boolean</td><td>TRUE if customer_id is null.</td></tr>
  <tr><td><code>net_revenue</code></td><td>Float</td><td>quantity × unit_price. Can be negative for returns.</td></tr>
  <tr><td><code>is_cancelled</code></td><td>Boolean</td><td>TRUE if invoice_no starts with "C".</td></tr>
  <tr><td><code>is_financial_adjustment</code></td><td>Boolean</td><td>TRUE if description matches financial adjustment patterns (adjust, debt, discount, bank, charge, manual, credit, sample).</td></tr>
  <tr><td><code>is_alpha_stock</code></td><td>Boolean</td><td>TRUE if stock_code is purely alphabetic (non-product codes).</td></tr>
  <tr><td><code>is_non_product</code></td><td>Boolean</td><td>TRUE if is_financial_adjustment OR is_alpha_stock.</td></tr>
  <tr><td><code>is_return</code></td><td>Boolean</td><td>TRUE if quantity is negative AND the row is not a non-product entry.</td></tr>
  <tr><td><code>is_sale</code></td><td>Boolean</td><td>TRUE if quantity is positive AND not non-product AND not cancelled. Primary filter for all dashboard pivots.</td></tr>
  <tr><td><code>return_value</code></td><td>Float</td><td>Absolute value of net_revenue for returns, 0 otherwise.</td></tr>
  <tr><td><code>gross_sale_value</code></td><td>Float</td><td>net_revenue for positive transactions, 0 otherwise.</td></tr>
</table>

---

<h2>Filtering Logic</h2>

<p>The dashboard pivot tables use two primary filters depending on the metric:</p>

<table>
  <tr><th>Metric Type</th><th>Filter</th><th>Reason</th></tr>
  <tr><td>Revenue performance (AOV, trends, products)</td><td><code>is_sale = TRUE</code></td><td>Isolates genuine completed sales</td></tr>
  <tr><td>Cancellation rate</td><td>No sale filter (needs cancelled rows)</td><td>Cancelled invoices must be included to calculate the rate</td></tr>
  <tr><td>Return rate</td><td>No sale filter (needs return rows)</td><td>Return rows must be included to calculate the rate</td></tr>
  <tr><td>Customer cohort analysis</td><td><code>is_sale = TRUE</code>, exclude null customer_id</td><td>Only identifiable customers with completed purchases</td></tr>
</table>

---

<h2>Cohort Analysis Table</h2>

<p>A customer-level summary derived from the filtered fact table. One row per customer. Built by pivoting the fact table by customer_id, then pasting values and adding the repeat flag.</p>

<table>
  <tr><th>Column</th><th>Type</th><th>Description</th></tr>
  <tr><td>Customer ID</td><td>Text</td><td>Unique customer identifier.</td></tr>
  <tr><td>Total Net Revenue</td><td>Float</td><td>Sum of net_revenue for that customer (sales only).</td></tr>
  <tr><td>Distinct Count of invoice_no</td><td>Integer</td><td>Number of unique orders placed.</td></tr>
  <tr><td>Repeat One-Time Flag</td><td>Text</td><td>"Repeat" if invoice count > 1, "One-Time" if invoice count = 1.</td></tr>
</table>

---

<h2>Key Calculated Metrics</h2>

<p>These are not stored columns. They are calculated in Excel using pivot tables and formulas.</p>

<table>
  <tr><th>Metric</th><th>Formula</th><th>Description</th></tr>
  <tr><td>AOV</td><td>Net Revenue / Distinct Invoice Count</td><td>Average spend per order.</td></tr>
  <tr><td>Revenue per Customer</td><td>Net Revenue / Distinct Customer Count</td><td>Average customer value in the period.</td></tr>
  <tr><td>Orders per Customer</td><td>Distinct Invoice Count / Distinct Customer Count</td><td>Purchase frequency.</td></tr>
  <tr><td>Revenue MoM %</td><td>(Current - Prior) / Prior</td><td>Month-on-month revenue growth rate.</td></tr>
  <tr><td>Cancellation Rate</td><td>Distinct Cancelled Invoices / Total Distinct Invoices</td><td>Proportion of orders cancelled. Uses distinct count, not row count.</td></tr>
  <tr><td>Return Rate</td><td>Total Return Value / Total Gross Sales</td><td>Revenue lost to returns as a percentage.</td></tr>
  <tr><td>Revenue Concentration</td><td>Top 10 Product Revenue / Total Revenue</td><td>How much revenue depends on top sellers.</td></tr>
</table>
