# Path DAX in Power BI

## Course 1: Power BI - DAX Contexts and Iteration

---

## 1. Data Model Overview

The project utilizes two separate datasets imported from **CSV** files. Before developing any DAX measures, a structural analysis of each table was conducted within the Data View:

### 1.1. Table: `Livros` (Dimension / Lookup Table)
Contains static, master data for each product.
* **ID:** Unique identifier for each book (used to establish relationships between tables).
* **Título:** Book title.
* **Preço Unitário:** Retail selling price per unit.
* **Preço de Custo:** Cost price (the amount the company paid to acquire or manufacture the book).
* **Estoque Livre:** Current available inventory.
* **Quantidade de Vendas:** Total sales volume recorded during the last month.

### 1.2. Table: `registro_vendas` (Fact / Transaction Table)
Contains the historical record of all sales transactions.
* **ID_Fatura:** Unique transaction identifier for each invoice.
* **ID_Produto:** The product code of the sold book (maps directly to the **ID** column in the `Livros` table).
* **Data_Compra / Data_Entrega:** Transaction dates (crucial for time-intelligence and logistical KPIs).
* **Metodo_Pagamento:** Payment method used (Cash, Credit Card, or PayPal).
* **Geographic Data:** Delivery address details including Address, City, and Postal Code.

### 1.3. DAX Business Insights & Application
Based on the data structure, the following areas have been identified for DAX implementation:
* **Total Revenue:** Multiplying the transaction quantities from the sales table by the *Preço Unitário* located in the books table (likely utilizing row context, relationships, and iterator functions).
* **Profit Margin:** Calculating the financial variance between *Preço Unitário* and *Preço de Custo*.
* **Lead Time:** Measuring the exact number of days elapsed between *Data_Compra* and *Data_Entrega*.

---

## 2. The Purpose of DAX

### 2.1. History and Evolution
* **Origin:** DAX (Data Analysis Expressions) is not exclusive to Power BI. It was created and released by Microsoft in 2010 for **Power Pivot** in Excel, five years before the launch of Power BI in 2015.
* **Popularity:** The language gained massive traction and evolved significantly alongside the rapid growth of Power BI and the broader adoption of Business Intelligence (BI) in the market.

### 2.2. Core Characteristics
* **Nature:** DAX is designed to be simple, direct, and highly efficient. However, its underlying logic operates differently from traditional programming languages, which may require a mindset shift for developers.
* **Functional Language:** Every calculation in DAX is driven by executing a **function**. Formulas are constructed by nesting functions within other functions to achieve complex logic.

### 2.3. The Three Pillars of DAX
To master DAX, it is essential to deeply understand its evaluation contexts and how rows are processed:
* **Filter Context (Contexto de Filtro):** The environment of filters applied to the data model *before* a calculation takes place (driven by slicers, visuals, or explicit formula coordinates). It restricts the data input to a specific subset.
* **Row Context (Contexto de Linha):** The capability of DAX to evaluate a formula on a row-by-row basis, considering the current value of each single row.
* **Iterations (Iterações):** Repetitive calculations where DAX scans a table to evaluate an expression line-by-line before aggregating the final output.

### 2.4. Excel vs. DAX Iterators
* **Traditional Spreadsheets (Excel):** Often require creating consecutive calculated columns step-by-step to reach a final result, expanding the physical size of the spreadsheet.
* **DAX Iterators:** Perform multi-step calculations dynamically within a single expression, saving memory and processing time. 
* **The Trade-off:** While highly efficient, this internal processing makes DAX **less visual**, requiring a strong understanding of evaluation contexts to ensure accurate results.

### 2.5. Calculated Columns: Introduction and Syntax
* **Access Path:** Data View > Table Tools > New Column.
* **The Formula Bar:** Everything written before the `=` sign defines the column name. Everything after contains the DAX expression.
* **Row Context in Action:** Calculated columns inherently operate under a **Row Context**. Any calculation or column reference is automatically computed row-by-row for the entire table.

### 2.6. Syntax Standards & Best Practices
* **Column References:** Columns are always enclosed in brackets: `[Column Name]`.
* **The Golden Rule for References:** Always precede a column name with its respective table name enclosed in single quotes (e.g., `'Table Name'[Column Name]`). This explicit definition prevents severe confusion and bugs in complex models.
* **IntelliSense:** Power BI provides real-time autocomplete suggestions. Use the keyboard arrows and press `Enter` to auto-fill names safely.

### 2.7. Arithmetic Operators & Code Comments
Calculated columns support standard operations evaluated line-by-line:
* **Operators:** Multiplication (`*`), Division (`/`), and Exponentiation (`^`).
* **Comments:** Use double dashes (`--`) or double forward slashes (`//`) for single lines. Use `/*` and `*/` for multi-line blocks.
* **Line Break Shortcut:** Press `Shift + Enter` to move to a new line inside the formula bar without executing the code.

### 2.8. Data Types and Quote Literals
* **Single Quotes (`' '`):** Exclusively reserved for referencing **Table Names**.
* **Double Quotes (`" "`):** Exclusively reserved for literal **Text Strings** (e.g., `"a"`). Writing raw text without double quotes triggers a compilation error (`#ERROR`).
* **Fixed Values:** Static numbers (e.g., `2000`) can be assigned directly and will replicate across every single row.

### 2.9. Measures: Introduction and Syntax
* **Access Path:** Data View > Right-click on the target table > New Measure.
* **The Core Rule of Measures:** Measures **cannot** reference a raw column directly (e.g., `Measure = 'Livros'[Preço Unitário]` triggers an error). 
* **The Need for Aggregation:** Because measures do not have an inherent Row Context, they require an aggregation function to collapse an entire column into a single scalar value.

### 2.10. Common Aggregation Functions
DAX functions accept a column reference as an argument:
* **`SUM(Table[Column])`:** Adds up all the values in the specified column.
  * *Example:* `Somatório dos preços = SUM(Livros[Preço Unitário])`
* **`MAX(Table[Column])`:** Returns the largest value found in the column.
  * *Example:* `Máximo da quantidade = MAX(Livros[Quantidade de vendas])`
* *Other standard aggregations include `MIN` and `AVERAGE`.* Inside standard aggregation functions, single quotes around the table name are optional if there are no spaces.

### 2.11. Visualizing Measures (The Canvas)
* **Invisible in Data View:** Measures do not create physical columns in tables; they live purely as mathematical instructions.
* **The Report View (Canvas):** To view a measure's output, it must be dragged onto the Canvas. 
* **The Card Visual:** Changing the visual type to a **Card** is the best practice to inspect a measure's aggregated single-value result.

### 2.12. Data Pane Icon Identification Guide
Visual indicators to audit fields in the Data pane:
* **Calculator Icon:** Standard DAX Measure.
* **Sigma Symbol (Σ):** Native numeric column.
* **Table with a Sigma Icon:** DAX Calculated Column.
* **No Icon:** Native text, date, or categorical string column.

---

