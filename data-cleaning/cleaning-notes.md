<h1>Data Cleaning &amp; Preparation Notes</h1>

<p>The raw dataset was cleaned and enriched using Python (pandas) before loading into Excel via Power Query. Each step below documents what was done, what issues came up, and why specific decisions were made.</p>

<p>The full code is in <a href="data_cleaning_notebook.ipynb"><code>data_cleaning_notebook.ipynb</code></a>.</p>

---

<h2>The Raw Dataset</h2>

<p>The <a href="https://www.kaggle.com/datasets/ulrikthygepedersen/online-retail-dataset">Online Retail Dataset</a> from Kaggle. 541,909 rows, 8 columns. Each row is a line item on an invoice for a UK-based online retailer, covering December 2010 to December 2011.</p>

<p>Downloaded using the <code>kagglehub</code> Python package and loaded with pandas.</p>

---

<h2>Step 1 — Column Names</h2>

<p>The original columns used CamelCase (<code>StockCode</code>, <code>CustomerID</code>, <code>InvoiceDate</code>). These were converted to snake_case for readability and Python convention.</p>

<p>The first attempt used a simple regex that split on every uppercase letter. This turned <code>CustomerID</code> into <code>customer_i_d</code> because it treated each capital in the acronym "ID" as a separate word boundary. Fixed with a two-pass regex: one pass handles lowercase-to-uppercase transitions, the second handles consecutive uppercase letters followed by lowercase.</p>

```python
def clean_columns(df):
    def convert(name):
        name = re.sub(r'(?<=[a-z])(?=[A-Z])', '_', name)
        name = re.sub(r'(?<=[A-Z])(?=[A-Z][a-z])', '_', name)
        return name.lower().strip()
    df.columns = [convert(col) for col in df.columns]
    return df
```

<p>Result: <code>customer_id</code>, <code>stock_code</code>, <code>invoice_date</code>. Verified with <code>print(df.columns)</code> after running.</p>

---

<h2>Step 2 — Description Text Cleaning</h2>

<p>Product descriptions contained non-breaking space characters (<code>\xa0</code>) that rendered as small boxes, e.g. <code>PINK UNION JACK NBSP NBSP LUGGAGE TAG</code>. These came from the original web/Excel export.</p>

```python
df['description'] = (
    df['description']
    .str.replace('\xa0', ' ', regex=False)
    .str.replace(r'\s+', ' ', regex=True)
    .str.strip()
    .str.title()
)
```

<p>Replaced the non-breaking spaces with regular spaces, collapsed multiple spaces, stripped whitespace, and standardised to title case. Result: <code>Pink Union Jack Luggage Tag</code>.</p>

<p>About 1% of descriptions were null. These were kept in the dataset. Descriptions are not needed for revenue or customer analysis, so dropping rows over a missing text field would have been unnecessary.</p>

---

<h2>Step 3 — Customer ID</h2>

<p>Around 135,000 rows (~25% of transactions) had no customer ID. These represent guest checkouts or cash transactions. Dropping them would have removed a significant chunk of revenue.</p>

<p>Instead of deleting or filling with a placeholder, a boolean flag was created:</p>

```python
df['is_guest'] = df['customer_id'].isnull()
```

<p>This preserves the data for revenue analysis while clearly marking it as non-identifiable for customer-level metrics like repeat analysis.</p>

<p>The customer_id column was also converted from float to <code>Int64</code> (nullable integer type) to remove the trailing <code>.0</code> on every ID. Standard pandas converts numeric columns with nulls to float. <code>Int64</code> (capital I) supports both integers and <code>&lt;NA&gt;</code>.</p>

---

<h2>Step 4 — Data Types &amp; Memory</h2>

<p><code>stock_code</code> and <code>country</code> were converted to <code>category</code> type. Both columns have a limited set of repeated values across hundreds of thousands of rows. Memory usage was checked before and after with <code>df.memory_usage(deep=True)</code> to confirm the reduction.</p>

<p><code>invoice_date</code> was converted to datetime, and four time dimension columns were derived for pivot table grouping:</p>

```python
df['year'] = df['invoice_date'].dt.year
df['month'] = df['invoice_date'].dt.month
df['year_month'] = df['invoice_date'].dt.to_period('M').astype(str)
df['weekday'] = df['invoice_date'].dt.day_name()
```

---

<h2>Step 5 — Duplicates</h2>

<p><code>df.duplicated().sum()</code> flagged 5,268 rows. Initial inspection was confusing because sorting by <code>invoice_no</code> grouped rows from the same invoice together, making it look like different products on the same order were being flagged as duplicates.</p>

<p>Closer inspection using <code>df.duplicated(keep=False)</code> confirmed these were genuine full-row copies: same invoice number, same stock code, same quantity, same price, same date, same customer. The same product line appears twice on the same invoice, identically. This is a known characteristic of this dataset, likely caused by export duplication.</p>

```python
df = df.drop_duplicates()
```

<p>Partial duplicates (same customer, same invoice, different products) were not affected. They represent legitimate multi-line orders.</p>

---

<h2>Step 6 — Transaction Classification</h2>

<p>This was the most important part. The raw data mixes sales, returns, cancellations, and financial adjustments together. Rather than deleting anything, a classification framework was built using boolean flags.</p>

<h3><code>is_cancelled</code></h3>
<p>Invoice numbers starting with "C" indicate cancelled orders. This is an invoice-level flag.</p>

```python
df['is_cancelled'] = df['invoice_no'].astype(str).str.startswith('C')
```

<p><em>Note:</em> There was an initial bug where this was accidentally applied to <code>invoice_date</code> instead of <code>invoice_no</code>. Caught during validation when the flag counts did not match expected patterns.</p>

<h3><code>is_financial_adjustment</code></h3>
<p>Some rows are accounting entries, not product transactions: bad debt adjustments, bank charges, discounts, manual credits, samples. These were identified by description pattern matching. The bad debt rows had unit prices of -£11,062, which would have severely distorted revenue metrics if left unclassified.</p>

```python
df['is_financial_adjustment'] = df['description'].str.contains(
    'adjust|debt|discount|bank|charge|manual|credit|sample',
    case=False, na=False
)
```

<h3><code>is_alpha_stock</code></h3>
<p>Stock codes that are purely alphabetic (e.g. <code>POST</code>, <code>D</code>, <code>M</code>, <code>BANK CHARGES</code>) are not real product SKUs. They represent postage, dotcom adjustments, manual entries, and other non-product transactions.</p>

```python
df['is_alpha_stock'] = df['stock_code'].astype(str).str.isalpha()
```

<h3><code>is_non_product</code></h3>
<p>Combines the two above into a single flag for anything that is not a genuine product transaction.</p>

```python
df['is_non_product'] = df['is_financial_adjustment'] | df['is_alpha_stock']
```

<h3><code>is_return</code></h3>
<p>Negative quantity on a row that is NOT a non-product entry. This separates actual product returns from negative accounting adjustments. Without this distinction, bad debt write-offs would be counted as "returns" and inflate the return rate.</p>

```python
df['is_return'] = (df['quantity'] < 0) & (~df['is_non_product'])
```

<h3><code>is_sale</code></h3>
<p>The master flag for clean commercial analysis. True only when quantity is positive, the row is not a non-product, and the invoice is not cancelled. This is the primary filter used across all dashboard pivot tables.</p>

```python
df['is_sale'] = (
    (df['quantity'] > 0) &
    (~df['is_non_product']) &
    (~df['is_cancelled'])
)
```

<p>There are dozens of different stock codes with different meanings. Rather than manually researching each one, a rule-based approach using description patterns and stock code format identifies non-product entries. This is transparent, reproducible, and would be refined with domain input from the commercial team in a real business setting.</p>

---

<h2>Step 7 — Revenue Columns</h2>

```python
df['revenue'] = df['quantity'] * df['unit_price']
df['return_value'] = df['revenue'].where(df['revenue'] < 0, 0).abs()
df['gross_sale_value'] = df['revenue'].where(df['revenue'] > 0, 0)
```

<p>Three columns engineered:</p>
<ul>
  <li><strong>revenue</strong> (later renamed to net_revenue in Excel) — quantity × unit_price. Can be negative for returns.</li>
  <li><strong>return_value</strong> — absolute value of revenue for returns, 0 otherwise.</li>
  <li><strong>gross_sale_value</strong> — revenue for positive transactions, 0 otherwise.</li>
</ul>

<p>These were created in Python rather than Excel because they are derived fields (data preparation), not reporting calculations. KPI formulas like return rate and AOV belong in the reporting layer.</p>

---

<h2>Step 8 — Validation</h2>

<p>Final checks before export:</p>

<ul>
  <li><code>df.info()</code> — correct data types across all columns</li>
  <li><code>df.isnull().sum()</code> — nulls only where expected (customer_id for guests, description for ~1%)</li>
  <li><code>df.duplicated().sum()</code> — 0 remaining duplicates</li>
  <li><code>df[df['unit_price'] &lt; 0]</code> — only 2 rows (the bad debt adjustments, correctly flagged)</li>
  <li><code>df[df['quantity'] == 0]</code> — checked for zero-quantity rows</li>
  <li><code>(df['is_sale'] &amp; df['is_return']).sum()</code> — 0, confirming sales and returns do not overlap</li>
  <li><code>(df['is_non_product'] &amp; df['is_sale']).sum()</code> — 0, non-products not classified as sales</li>
  <li><code>(df['is_non_product'] &amp; df['is_return']).sum()</code> — 0, non-products not classified as returns</li>
  <li>Revenue by <code>is_non_product</code> — operational revenue ~£9.76M vs financial adjustments ~-£30K (0.3%)</li>
</ul>

---

<h2>Key Decisions</h2>

<p><strong>Flag instead of delete.</strong> Every row in the original dataset is preserved. Deleting cancellations would make it impossible to calculate the cancellation rate. Deleting returns would hide product quality signals. Deleting guest transactions would understate revenue by ~15%.</p>

<p><strong>Rule-based classification over manual lookup.</strong> Rather than hardcoding a list of stock codes, a rule-based approach using description patterns and stock code format identifies non-product entries. Transparent, reproducible, and defensible.</p>

<p><strong>Revenue engineering in Python, KPI calculation in Excel.</strong> Deliberately separating data preparation from reporting to demonstrate a multi-tool workflow and clear separation of concerns.</p>

<p><strong>December 2010 kept but excluded from seasonality.</strong> The dataset only contains one month of 2010 data. It was retained in the dataset but excluded from time-series analysis to avoid distortion.</p>

---

<h2>Output</h2>

<p>Exported as CSV and loaded into Excel's Data Model via Power Query as <code>fact_transactions</code>. Data types for ID columns were explicitly set to Text in Power Query to prevent Excel from auto-converting numeric-looking codes to numbers.</p>

---

<h2>Final Column List</h2>

<table>
  <tr><th>Column</th><th>Type</th><th>Source</th></tr>
  <tr><td>invoice_no</td><td>Text</td><td>Original</td></tr>
  <tr><td>stock_code</td><td>Category</td><td>Original</td></tr>
  <tr><td>description</td><td>Text</td><td>Original (cleaned)</td></tr>
  <tr><td>quantity</td><td>Integer</td><td>Original</td></tr>
  <tr><td>invoice_date</td><td>DateTime</td><td>Original (converted)</td></tr>
  <tr><td>unit_price</td><td>Float</td><td>Original</td></tr>
  <tr><td>customer_id</td><td>Int64 (nullable)</td><td>Original (converted)</td></tr>
  <tr><td>country</td><td>Category</td><td>Original</td></tr>
  <tr><td>year</td><td>Integer</td><td>Derived</td></tr>
  <tr><td>month</td><td>Integer</td><td>Derived</td></tr>
  <tr><td>year_month</td><td>Text</td><td>Derived</td></tr>
  <tr><td>weekday</td><td>Text</td><td>Derived</td></tr>
  <tr><td>is_guest</td><td>Boolean</td><td>Derived — customer_id null check</td></tr>
  <tr><td>is_cancelled</td><td>Boolean</td><td>Derived — invoice_no prefix "C"</td></tr>
  <tr><td>is_financial_adjustment</td><td>Boolean</td><td>Derived — description pattern match</td></tr>
  <tr><td>is_alpha_stock</td><td>Boolean</td><td>Derived — stock code format</td></tr>
  <tr><td>is_non_product</td><td>Boolean</td><td>Derived — financial adjustment OR alpha stock</td></tr>
  <tr><td>is_return</td><td>Boolean</td><td>Derived — negative qty AND not non-product</td></tr>
  <tr><td>is_sale</td><td>Boolean</td><td>Derived — positive qty, not non-product, not cancelled</td></tr>
  <tr><td>revenue</td><td>Float</td><td>Derived — quantity × unit_price</td></tr>
  <tr><td>return_value</td><td>Float</td><td>Derived — abs revenue for returns</td></tr>
  <tr><td>gross_sale_value</td><td>Float</td><td>Derived — revenue for sales</td></tr>
</table>
