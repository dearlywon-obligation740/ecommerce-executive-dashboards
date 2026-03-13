<h1>Methodology</h1>

<h2>Approach</h2>

<p>This project follows a structured analytics workflow: clean the data, model it for analysis, build the metrics, then present them visually. Each stage is deliberately separated so changes in one layer do not break the others.</p>

---

<h2>Stage 1 — Data Cleaning</h2>

<p>The raw dataset had several issues. Cancellations and returns were mixed in with sales. Guest purchases had no customer ID. Non-product transactions (postage, bank charges, bad debt adjustments) were embedded throughout. And 5,268 rows were genuine full-row duplicates from what appears to be an export issue.</p>

<p>The guiding principle was <strong>flag, not delete</strong>. Every row is preserved. Boolean flag columns classify each transaction so the analysis layer can filter as needed. This means the same dataset supports revenue reporting (filter to sales), cancellation analysis (needs cancelled rows), return analysis (needs return rows), and customer segmentation (needs identifiable customers only).</p>

<p>The classification framework uses rule-based logic rather than manually hardcoded stock code lists. Description pattern matching and stock code format analysis identify non-product entries. This is transparent and reproducible. In a real business, the classification would be refined with domain input from the commercial team, but for a portfolio project the rule-based approach is defensible and clearly documented.</p>

<p>Full details and code in <a href="../data-cleaning/cleaning-notes.md">cleaning-notes.md</a> and <a href="../data-cleaning/data_cleaning_notebook.ipynb">the notebook</a>.</p>

---

<h2>Stage 2 — Data Modelling</h2>

<p>The cleaned data was loaded into Excel via Power Query as a single fact table in the Data Model. No star schema or dimension tables were needed for this scope. Time dimensions (year, month) are columns on the fact table itself.</p>

<p>Multiple pivot tables were built on a KPIs sheet, each filtered to different views: monthly revenue with calculated KPI columns, product revenue (top 10), customer segmentation (repeat vs one-time), and cancellation/return rates.</p>

<p>A separate <strong>CUBEVALUE reference table</strong> was added alongside the pivot outputs. This was necessary because the pivot tables are built on the Data Model and physically collapse when filtered by a slicer — rows disappear, the Grand Total shifts to a different row, and all the calculated formula columns (MoM %, vs Previous Month, etc.) break. CUBEVALUE reads the Data Model directly and is completely unaffected by slicer state, so it provides a stable source for charts, sparklines, and subtitle formulas.</p>

<p>The CUBEVALUE formulas use concatenated cell references to make the month dynamic (hardcoded in column F) while keeping the year dynamic from the pivot filter area cell. The filter criteria (<code>is_sale</code>, <code>is_cancelled</code>) are included in the CUBEVALUE arguments so the numbers match the pivot outputs exactly.</p>

---

<h2>Stage 3 — Dashboard Architecture</h2>

<h3>Dashboard 1 (Interactive)</h3>

<p>The data flow:</p>

<pre>
Slicers on Dashboard
  → Filter pivot tables on KPIs sheet
    → GETPIVOTDATA helper cells capture Grand Total values
      → Text boxes on Dashboard linked to helper cells
        → Charts reference CUBEVALUE column (slicer-independent)
          → Subtitle formulas use INDEX/MATCH against CUBEVALUE table
</pre>

<p>The critical design decision was using <strong>GETPIVOTDATA</strong> rather than direct cell references. Because the Data Model pivots physically collapse when filtered (rows disappear, the Grand Total moves to a different row number), fixed references like <code>=KPIs!B123</code> break. GETPIVOTDATA finds values by field name regardless of row position. The formulas are generated automatically by clicking on the Grand Total cell in the pivot and letting Excel write the reference.</p>

<p>For metrics that are not pivot fields (AOV, MoM %, vs Previous Month), these are calculated formula columns on the KPIs sheet that also collapse with the pivot. The CUBEVALUE reference table solves this: it always holds all 12 months of data, and INDEX/MATCH formulas look up the selected month against it to produce subtitle text and change indicators.</p>

<p>The change indicator logic compares <strong>current month's MoM % minus prior month's MoM %</strong>. This is an important distinction. If current month growth is -13% and last month's was +0.5%, showing last month's value as a green up-arrow would be misleading. The arrow needs to reflect whether things are getting better or worse, not the raw sign of the prior value.</p>

<h3>Dashboard 2 (Static)</h3>

<p>Deliberately isolated from Dashboard 1's slicers. It pulls from the Cohort Analysis sheet (which has no pivot tables and no slicer connections) and the product pivot. If the product pivot is connected to the month slicer, it can be disconnected via Report Connections.</p>

---

<h2>Visual Design</h2>

<p>Both dashboards use a consistent dark theme built with custom colours in Excel:</p>

<table>
  <tr><th>Colour</th><th>Hex</th><th>Usage</th></tr>
  <tr><td>Background</td><td>#0D0F14</td><td>Sheet background, all empty cells</td></tr>
  <tr><td>Surface</td><td>#13161E</td><td>KPI card fills</td></tr>
  <tr><td>Card</td><td>#181C27</td><td>Chart backgrounds, panels</td></tr>
  <tr><td>Border</td><td>#252A38</td><td>Shape outlines, chart gridlines</td></tr>
  <tr><td>Green</td><td>#4FFFB0</td><td>Revenue, positive changes, healthy indicators</td></tr>
  <tr><td>Blue</td><td>#5B8FFF</td><td>Targets, customer volume, product bars</td></tr>
  <tr><td>Red</td><td>#FF6B6B</td><td>Negative signals, leakage</td></tr>
  <tr><td>Gold</td><td>#FFD166</td><td>AOV, frequency, warnings</td></tr>
</table>

<p>Cards are rounded rectangle shapes with text boxes layered on top. This is cleaner and more repositionable than merged cells. Charts use the same palette with transparent backgrounds so they blend into the dashboard surface.</p>

---

<h2>Key Analytical Decisions</h2>

<p><strong>Revenue driver decomposition uses division from totals, not averaged averages.</strong> When showing the full year, AOV = Total Revenue / Total Invoices (not the average of 12 monthly AOVs). Averaging averages produces incorrect results when the underlying volumes differ month to month.</p>

<p><strong>Cancellation rate uses distinct invoice count, not row count.</strong> Each invoice can have multiple product lines. Counting rows would inflate the rate depending on how many products were on each order. Distinct count of invoice numbers gives the true order-level cancellation rate.</p>

<p><strong>Cancellation and return rates are calculated separately.</strong> They represent different operational issues. Cancellations happen before fulfilment (process or demand issue). Returns happen after delivery (product or quality issue). Combining them into a single leakage metric is useful for the headline number, but the breakdown matters for identifying where to act.</p>

<p><strong>December 2010 is excluded from seasonality analysis.</strong> The dataset contains only one month of 2010 data. Including it would distort the seasonal pattern. The data is retained in the full dataset but filtered out of time-series views.</p>

<p><strong>The 3-driver decomposition (Revenue = Customers × Frequency × Basket Size)</strong> was added after initial MoM analysis showed that revenue changes did not always align with customer count changes. March 2011 revenue jumped 36% while customers only grew 5.8% and AOV was flat, which did not reconcile until order frequency per customer was added as a third driver.</p>
