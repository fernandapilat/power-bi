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

## 3. Calculated Columns vs. Measures (Efficiency & Use Cases)

### 3.1. Memory and Storage Performance
Choosing between a calculated column and a measure directly impacts report performance and file size:
* **Calculated Columns (Storage Heavy):** Computed during data refresh and stored physically inside the data model. 
  * *Impact:* They consume both **disk storage** and **RAM**. In tables with millions of rows, adding multiple calculated columns can significantly increase the `.pbix` file size and degrade performance.
* **Measures (Processing Heavy):** Do not occupy physical space or storage in the model. They are calculated dynamically on-the-fly.
  * *Impact:* They consume **CPU processing power** only when a visual on the canvas requests them. 

### 3.2. Evaluation Context Comparison
* **Calculated Columns ➔ Row Context:** They compute values line-by-line upon data load, completely static regarding user interactions on the report.
* **Measures ➔ Filter Context:** They evaluate data dynamically based on the current visual environment. They recalculate instantly whenever a user interacts with slicers, filters, or cross-filtering.

### 3.3. Best Practices for Model Optimization
* **The Golden Rule:** If a calculation can be achieved through either a calculated column or a measure, **always default to a measure** to preserve hardware resources.
* **Temporary Columns Strategy:** Creating step-by-step calculated columns can be a helpful intermediate strategy to understand a complex calculation. However, once the final logic is established, these redundant temporary columns should be deleted or converted into a single optimized measure to keep the model clean and fast.
* **Business Intent:** Calculated columns are ideal when you need to slice or group data (e.g., age groups, categories). Measures are ideal when you simply need to display a numeric KPI or scalar value (e.g., Total Sales, Profit Margin).

### 3.4. Practical Case: Building Gross Margin via Calculated Columns
To calculate the **Gross Margin %** of the last month, a step-by-step approach was tested by creating four sequential calculated columns in the `Livros` table:

1. **Faturamento Total (Total Revenue):** `Faturamento total = 'Livros'[Preço Unitário] * 'Livros'[Quantidade de vendas]`
2. **Custo Total (Total Cost):** `Custo Total = 'Livros'[Preço de custo] * 'Livros'[Quantidade de vendas]`
3. **Margem Bruta (Gross Margin):** `Margem bruta = 'Livros'[Faturamento total] - 'Livros'[Custo Total]`
4. **Margem Bruta % (Gross Margin %):** `Margem bruta % = 'Livros'[Margem bruta] / 'Livros'[Faturamento total]`

* **Formatting Tip:** To display decimals as percentages, select the column, go to **Column Tools** in the top menu, and click the **`%`** icon.

### 3.5. Visualizing the Result & The "Total Row" Trap
When testing these columns inside a Table Visual on the Canvas, the following configuration was used:
* **Fields:** `ID` (set to *"Don't Summarize"* to show individual books), `Faturamento total`, `Custo Total`, `Margem bruta`, and `Margem bruta %`.
* **The Bug:** While the percentage for each individual book row was mathematically correct, the **Total Row** displayed an impossible value of **4,029.48%**.

### 3.6. Why did the Total Row fail?
* **Column Summation Behavior:** By default, when a calculated column is dropped into a visual, Power BI applies a `SUM` aggregation to the total row. 
* **The Math Error:** Power BI literally added up the percentage of book 1 + book 2 + book 3... and so on. In business intelligence, **you can never sum ratios or percentages** to find a grand total; you must recalculate the ratio based on the aggregated totals.

### 3.7. Resolving the Total Bug with Measures
To fix the mathematical distortion in the total row, the percentage must be calculated using a **Measure** instead of a calculated column.

* **The Correct Formula:** `Margem bruta % correta = SUM('Livros'[Margem bruta]) / SUM('Livros'[Faturamento total])`
* **The Logic:** Instead of dividing row-by-row and then summing the results, the measure forces Power BI to sum the entire margin first, sum the entire revenue second, and then perform the division.
* **Formatting Tip:** Just like columns, select the measure and click the **`%`** icon under **Measure Tools** to format the decimals properly.

### 3.8. Core Concept: Context of Filter (Contexto de Filtro) vs. Row Context
This practical scenario showcases how evaluation contexts dictate the behavior of your data:
* **Calculated Column (Row Context):** It evaluated the division strictly on each physical row of the dataset during data load. When placed into the visual's Total row, it defaulted to a blind `SUM` of all those static percentages.
* **Measure (Filter Context / Visual Context):** It has no physical rows. It adapts dynamically to the visual's environment. 
  * On individual rows, the `ID` acts as a filter, forcing the measure to compute the math for that specific book.
  * On the Total row, there are no filters splitting the data, so the measure dynamically evaluates the formula against the aggregate totals of the entire model.

### 3.9. Measure Portability and Home Tables
* **Home Table Requirement:** Every DAX measure must be created inside or assigned to a specific table in the model.
* **Global Scope:** Unlike columns, a measure is not restricted to its "Home Table". Because it references explicit tables and columns within its code, you can move a measure to any other table in the Data pane, and it will compute exactly the same results without breaking the visuals.


### 3.10. Summary Cheat Sheet: Calculated Columns vs. Measures

The table below summarizes the architectural and operational differences between the two methods of implementation in DAX:

| Feature | Calculated Columns (`Calculated Columns`) | Measures (`Measures`) |
| :--- | :--- | :--- |
| **Calculation Level** | **Individual Values:** Computes a unique result for each single row. | **Aggregated Values:** Computes summaries (SUM, AVERAGE, MAX, etc.) over multiple rows. |
| **Storage & Memory** | **Physical:** Stored directly inside the data model, increasing file size (`.pbix`) and RAM usage. | **Virtual:** Calculated on-the-fly under demand; takes up zero storage space. |
| **Evaluation Context** | **Row Context:** Operates statically row-by-row during data refresh. | **Filter Context:** Operates dynamically, responding to slicers, filters, and visuals. |
| **Reusability** | **Restricted:** Specific to the table where they are created. | **Global:** Highly reusable across different visuals, tables, and reports. |
| **Primary Use Case** | **Slicing & Grouping:** Ideal for creating filters, categories, and age groups. | **KPIs & Metrics:** Ideal for values displayed inside charts, cards, and data matrices. |
| **Complexity** | Best suited for **simple, direct formulas** applied at a row level. | Designed for **complex calculations** involving filters and dynamic context shifts. |

---

## 4. DAX Iterators and Variables

### 4.1. Introduction to Variables
Variables act as dynamic containers within a specific DAX expression to store calculations, scalar values, or tables. 
* **Declaration:** Initiated by the keyword `VAR` followed by the variable name.
* **Execution:** Every variable block *must* end with the `RETURN` keyword, which tells DAX which final expression to evaluate and display.

### 4.2. Syntax Standards & Naming Conventions
* **CamelCase Pattern:** Variable names cannot contain spaces. The best practice is to capitalize the first letter of each word (e.g., `FaturamentoTotal`, `CustoTotal`).
* **Visual Identifiers (IntelliSense):** When typing code, the autocomplete menu displays an **`X`** icon next to variables, distinguishing them from functions (which display an **`fx`** icon).

### 4.3. Practical Case: Rewriting Gross Margin with Variables
Instead of creating multiple measures or columns, variables allow calculating the final ratio cleanly within a single measure. 

```dax
Margem bruta % VAR = 
VAR FaturamentoTotal = SUM('Livros'[Faturamento total])
VAR CustoTotal = SUM('Livros'[Custo total])
VAR MargemBruta = FaturamentoTotal - CustoTotal
RETURN
    MargemBruta / FaturamentoTotal
```

### 4.4. Key Advantages of Variables
1. **Readability (Less Verbose):** Avoids writing repetitive nested calculations. Code structure is organized and structured vertically using Shift + Enter.
2. **Performance (CPU Efficiency):** DAX evaluates the expression inside a variable only once. If that variable is called multiple times later in the code, DAX reuses the cached result instead of recalculating the formula from scratch.
3. **On-Demand Processing:** If you declare multiple variables but do not reference them after the RETURN keyword, DAX will completely ignore them and consume zero processing power.

### 4.5. Scope Limitation
* **Local Scope Only:** Variables in DAX are strictly local. A variable declared inside "Measure A" cannot be seen, called, or reused by "Measure B". 

### 4.6. Analytics: Calculating Lead Time with Dates
**Lead Time** represents the total time required to complete a specific process (e.g., the days elapsed between a customer buying a book and receiving it).

To calculate this in the `registro_vendas` table, a calculated column was created:
* **The Format Issue:** Subtracting two Datetime columns directly keeps the result formatted as a Date/Time string, which is unreadable for business metrics.
* **The Solution (`INT` Function):** Datetime values in Power BI are stored as numbers behind the scenes. Wrapping the subtraction in the `INT()` function forces the system to convert the time gap into a clean integer representing the total number of days.

Formula to copy and format manually:
Leadtime = INT('registro_vendas'[Data_Entrega] - 'registro_vendas'[Data_Compra])

### 4.7. Aggregating Lead Time: `AVERAGE`
When analyzing Lead Time in a report, summing the days makes no business sense. The true value lies in finding the average delivery speed.

* **Quick Visual Shortcut:** You can drop the `Leadtime` column into a table visual, click the dropdown arrow next to the field name, and switch the aggregation from **Sum** to **Average** (`Média`).
* **The Best Practice (Explicit Measure):** To keep the model clean and professional, it is always better to write an explicit DAX measure using the `AVERAGE` function.

Formula to copy and format manually:
Média do leadtime = AVERAGE('registro_vendas'[Leadtime])

### 4.8. DAX Iterators: The "X" Functions
An **Iterator** is a type of function that evaluates an expression row-by-row across a table (creating a temporary Row Context) and then applies an aggregation to the final results.

* **Sufix "X" Rule:** Most standard aggregation functions have an iterator version identified by an "X" at the end: `SUM` becomes `SUMX`, `AVERAGE` becomes `AVERAGEX`, `MIN` becomes `MINX`, and so on.
* **The Iterator Anatomy:** Every "X" function strictly requires at least two arguments:
  `FUNCTIONX( <Table>, <Expression> )`
  1. **Table:** The target table that DAX will scan line-by-line.
  2. **Expression:** The calculation/formula to execute on each individual row.

### 4.9. Practical Case: Replacing Physical Columns with `AVERAGEX`
Instead of storing a physical `Leadtime` column inside the model, `AVERAGEX` computes the time difference virtually in memory:

Formula to copy and format manually:
Média Leadtime Iterando = 
AVERAGEX(
    'registro_vendas', 
    INT('registro_vendas'[Data_Entrega] - 'registro_vendas'[Data_Compra])
)

* **The Result:** The measure scans the sales table, calculates the integer difference between delivery and purchase for each row, and finally calculates the grand average of those results.
* **Optimization Benefit:** This allowed deleting the physical `Leadtime` column from the model, instantly reducing file size (`.pbix`) and saving RAM.

### 4.10. Challenge Solved: Decoupling the Model from Calculated Columns
By refactoring the Gross Margin % calculation into a single standalone measure, the dataset was completely cleaned up, and all intermediate calculated columns were safely deleted.

* **Course Solution:** Used raw division (`MargemBruta / FaturamentoTotal`), which is prone to division-by-zero errors (`NaN` / `Infinity`).
* **My Optimized Solution:** Utilized the `DIVIDE()` function with an alternate result of `0`, ensuring total dashboard stability and bulletproof data integrity.

Formula applied successfully:
Margem Bruta % Final = 
VAR FaturamentoTotal = SUMX('livros', 'livros'[Preço Unitário] * 'livros'[Quantidade de vendas])
VAR CustoTotal = SUMX('livros', 'livros'[Preço de custo] * 'livros'[Quantidade de vendas])
RETURN
    DIVIDE(FaturamentoTotal - CustoTotal, FaturamentoTotal, 0)

---

## 5. DAX Contexts and Type Conversion

### 5.1. Data Type Conversion with `CONVERT`
The `CONVERT` function in DAX is a utility tool used to explicitly change a value from one data type to another. It ensures data consistency and compatibility inside measures or calculated columns.

* **Syntax Structure:**
  CONVERT(<Value>, <DataType>)

* **Supported Destination Data Types:**
  1. **INTEGER:** Converts decimals to whole numbers (e.g., `CONVERT(3.14, INTEGER)` yields `3`).
  2. **DOUBLE:** Converts whole numbers to floating-point decimals (e.g., `CONVERT(42, DOUBLE)` yields `42.0`).
  3. **STRING:** Converts numbers or dates into standard text (e.g., `CONVERT(100, STRING)` yields `"100"`).
  4. **BOOLEAN:** Converts text strings or numbers into logical flags (e.g., `CONVERT("TRUE", BOOLEAN)` yields `TRUE`).
  5. **CURRENCY:** Converts values into fixed-point monetary values.
  6. **DATETIME:** Converts standard text representations of time into date/time objects.

* **Critical Constraint:** Not all types are cross-compatible. Attempting an illogical conversion (like converting words into an INTEGER) will result in computation errors. Always ensure the underlying text structure matches the target destination format.

### 5.2. Handling Missing Data: `BLANK()` and `ISBLANK()`
In DAX, missing values, empty cells, or data skips are categorized under a single unified concept called **BLANK**. 

* **Propagation Risk:** If you execute a mathematical operation (like multiplication) between a valid number and a `BLANK` value, DAX will naturally propagate the result as a `BLANK`.
* **The `BLANK()` Function:** Directly returns an empty value. While rarely used by itself, it is highly powerful when nested inside conditional logic to mask errors or hide empty rows.
  ColunaVazia = BLANK()

* **The `ISBLANK()` Function:** A logical checker that scans a value or column row-by-row and returns `TRUE` if the cell is empty, or `FALSE` if it contains any data.
  VerificaVazio = ISBLANK('registro_vendas'[Código_Postal_Entrega])

### 5.3. Conditional Logic: `IF()` and `IFERROR()`
Conditional functions allow the creation of dynamic data classifications and provide a vital mechanism for error handling inside the model.

* **The `IF()` Function:** Executes a logical test and branches the result based on whether the condition evaluates to `TRUE` or `FALSE`.
  
  Syntax Structure:
  Classificação da margem = 
  IF(
      'Livros'[Margem bruta %] > 0.4,
      "Margem alta",
      "Margem baixa"
  )

* **The `IFERROR()` Function:** Acts as a safety guardrail. It evaluates an expression, and if that expression produces a computation error (like dividing by zero or pulling mismatched types), it returns an alternate fallback value instead of breaking the visual or stopping execution.
  
```dax
  TratamentoErro = IFERROR('Livros'[Preço de custo] * 'Livros'[Quantidade de vendas], BLANK())
```

* **Best Practice with `BLANK()`:** Pairing `IFERROR` with `BLANK()` ensures that faulty rows simply remain empty and hidden, preserving dashboard presentation without halting the entire model's calculations.

### 5.4. Error Detection: The `ISERROR()` Function
Unlike `IFERROR`, which catches and resolves an error in a single step, `ISERROR()` is a pure logical checker. It evaluates an expression and returns `TRUE` if an error is detected, and `FALSE` if the calculation is successful.

* **Nesting with `IF()`:** To handle errors using `ISERROR`, you must manually nest it inside an `IF` statement.
  
  Syntax Structure:
  ManejoManualErro = 
  IF(
      ISERROR('Livros'[Preço de custo] * 'Livros'[Quantidade de vendas]),
      BLANK(), // What to do if TRUE (there is an error)
      'Livros'[Preço de custo] * 'Livros'[Quantidade de vendas] // What to do if FALSE (it is clean)
  )

* **`IFERROR` vs `IF(ISERROR())`:** * `IFERROR` is cleaner, faster to write, and optimized by the DAX engine.
  * `IF(ISERROR())` is legacy syntax but offers more flexibility if you need to run a completely different calculation when an error occurs.

  ### 5.5. Advanced Conditionals: The `SWITCH()` Function
When handling multiple evaluation branches (three or more outcomes), nesting multiple `IF` functions creates unreadable and complex code. The `SWITCH` function solves this by evaluating a single expression against an ordered list of conditions.

* **The `SWITCH(TRUE())` Pattern:** By passing `TRUE()` as the primary expression, you instruct DAX to scan down the list row-by-row and execute the very first condition that evaluates to true. Once a match is found, execution stops for that row.

* **Syntax Structure:**
  Classificação do faturamento = 
  SWITCH(
      TRUE(),
      'Livros'[Faturamento total] > 20000, "Faturamento alto",
      'Livros'[Faturamento total] > 15000, "Faturamento médio",
      "Faturamento baixo" // The default fallback (Else)
  )

* **Key Execution Rule (Order Matters):** Since DAX scans from top to bottom, broader conditions must sit below restrictive ones. In the example above, placing `> 15000` before `> 20000` would trap a 25,000 value inside the "Médio" category incorrectly.
* **The Fallback Argument:** The final string (`"Faturamento baixo"`) acts as the standard `ELSE`. If none of the conditions above it are met, DAX automatically applies this value.

### 5.6. Implementing `SWITCH()` Inside Measures (Context Transition)
When writing a **Measure** instead of a Calculated Column, DAX prevents you from referencing raw table columns (e.g., `'Table'[Column]`) directly inside conditional logic. This occurs because measures operate under Filter Context and require a single, aggregated scalar value.

* **The Solution (Variable Aggregation):** To evaluate column values inside a measure, you must first "wrap" the column inside an aggregation function (like `SUM`, `MAX`, or `SELECTEDVALUE`) using a variable (`VAR`).

```dax
  Categoria de vendas = 
  VAR QntdVendas = SUM('livros'[Quantidade de vendas])
  RETURN
      SWITCH(
          TRUE(),
          QntdVendas > 240, "Alta",
          QntdVendas > 100, "Média",
          "Baixa"
      )
```

* **How it Behaves:** Even though the measure uses `SUM()`, it dynamically calculates the sales volume row-by-row inside a table visual due to the active **Filter Context** of that specific row, applying the correct conditional tag ("Alta", "Média", "Baixa") automatically.

### 5.7. Practical Challenges: Advanced SWITCH and Iterators (X-Functions)
Documenting the resolution of practical structural challenges regarding conditional boundaries and row-by-row iteration.

#### Challenge 1: Expanded Pricing Tiers with `SWITCH()`
* **Objective:** Create a calculated column separating unit costs into three distinct commercial tiers (Low, Medium, High).
* **Implementation:**
  Classificação Preço = 
  SWITCH(
      TRUE(),
      'livros'[Preço de custo] >= 40, "Custo Alto",
      'livros'[Preço de custo] > 30, "Custo Médio",
      "Custo Baixo"
  )
* **Key Concept:** Leveraged the top-to-bottom evaluation behavior of `SWITCH(TRUE())` to avoid writing complex `AND()` or `&&` range boundaries.

#### Challenge 2: Dynamic Row-by-Row Minimum with `MINX()`
* **Objective:** Calculate the lowest total revenue (Faturamento) snapshot by multiplying pricing and volume before executing the minimum evaluation.
* **Implementation:**
  FaturamentoMin = MINX('livros', 'livros'[Preço Unitário] * 'livros'[Quantidade de vendas])
* **Key Concept:** Used an **Iterator Function (`MINX`)** to inject a virtual Row Context over the `'livros'` table, executing the scalar multiplication line-by-line before retrieving the absolute minimum value for the Filter Context.

## References & Study Materials

To sustain the technical depth of this repository, the learning path is backed by the official documentation and leading literature in the Business Intelligence industry:

* **Official Documentation:** [Microsoft DAX Reference Guide](https://learn.microsoft.com/pt-br/dax/) — Used for syntax validation, data type conversion rules, and engine constraints.
* **Literature - Fundamentals & Strategy:** *Business Intelligence: Implementar do jeito certo e a custo zero* (Ronaldo Braghittoni) — Focused on architecture, modeling strategies, and delivering zero-cost corporate value.
* **Literature - Advanced Engine & Performance:** *The Definitive Guide to DAX: Business Intelligence for Microsoft Power BI, SQL Server Analysis Services, and Excel* (Marco Russo & Alberto Ferrari, 2019) — The global authority blueprint used to master complex Filter/Row Contexts, Context Transition, and VertiPaq engine performance optimization.

---