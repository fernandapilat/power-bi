# Path DAX in Power BI

## Course 2: Power BI - Deep Dive into DAX

## 1. Basic Table Functions & Data Modeling

Before developing advanced tabular expressions, a robust star/snowflake dimensional schema must be established. This module covers data ingestion, relationship cardinality mapping, and cross-table data retrieval using relationship-aware DAX functions.

### 1.1. Star Schema Modeling and Fact-Dimension Relationships
When combining disparate operational datasets (e.g., Marketing and Logistics), data must be structured into **Fact** and **Dimension** tables to leverage efficient column store compression and proper filter propagation.

* **Data Ingestion Profiling:** During CSV ingestion, the Power BI VertiPaq engine inspects a sample of the first 200 rows to automatically deduce file encoding (e.g., `Unicode UTF-8`), data types, and structural delimiters.
* **Relationship Cardinality (1:N):**
  * **Dimension Table (`registro_livros_marketing`):** Contains a unique, non-repeating primary key (`ID`). Marked with `1` in the model view.
  * **Fact Table (`registro_notas_logistica`):** Contains repeating foreign keys (`ID_Produto`) representing multiple historical transactional events (sales invoices). Marked with `*` (Many) in the model view.
  * **Filter Context Flow:** Filters flow downstream naturally from the **1-side** (Dimension) to the **Many-side** (Fact).

### 1.2. Cross-Table Data Retrieval: The `RELATED()` Function
To instantiate a Column Context Transition and physically pull a descriptive attribute from a Dimension table into a Fact table, DAX utilizes the `RELATED()` function.

* **Engine Mechanics:** `RELATED()` performs a virtual left outer join operation. It can **only** be executed from the "Many" side of a relationship to fetch a scalar value from the "One" side (following the existing relationship path upstream).
* **Implementation Syntax:**

```dax
  Categoria do livro = 
  RELATED(registros_livros_marketing[Categoria]) -- Fetches text-based dimension category into fact table
  ```

* **Architectural Advantage:** Clicking a visual checkbox only displays data dynamically. Generating a physical Calculated Column via `RELATED()` persists the data inside the data model's memory layout. This enables the newly generated attribute to be actively used inside downstream DAX calculations, conditional logic, and complex row-by-row fact evaluations.

### 1.3. Relational Architecture & Engine Constraints of `RELATED()`
To effectively utilize relational DAX functions, the underlying data model must leverage standardized relational database constraints, specifically Primary Keys (PK) and Foreign Keys (FK).

* **Relational Mapping:** 
  * **Primary Key (PK):** A column containing unique identifiers in the Dimension table (e.g., `ClienteID` in `Clientes`).
  * **Foreign Key (FK):** A column in the Fact table that references the Primary Key of a Dimension table to establish transactional mapping (e.g., `ClienteID` in `Pedidos`).
* **Syntax Blueprint:**

  Target_Column = RELATED(Related_Table[Related_Column])

* **Evaluation Context Requirement:** The `RELATED()` function strictly requires a **Row Context** to operate. It inspects the current row of the active table, identifies the foreign key, follows the relationship upstream to the dimension table, and extracts the corresponding scalar value. 
* **Measure Restriction:** A standalone measure cannot execute `RELATED()` directly (e.g., `Name = RELATED(Table[Column])`) because measures operate under a Filter Context by default, which lacks a row-by-row iteration mechanism. To use `RELATED()` within a measure, it must be wrapped inside an iterator function (such as `SUMX`, `MAXX`, or `FILTER`) that explicitly instantiates a Row Context.

### 1.4. Troubleshooting: Why `RELATED()` Fails to Autocomplete
If the Power BI IntelliSense (autocomplete menu) fails to display the target table or column names after typing `RELATED(`, check the following model architecture requirements:

* **Inverted Relationship Direction:** Verify that you are writing the expression inside the **Fact Table (Many-side)**. If you attempt to invoke `RELATED()` from a Dimension Table (1-side) targeting a Fact Table, the engine will hide the target table as it violates downstream filter propagation.
* **Missing Model Relationship:** The physical line connecting both tables must be active in the Model View. If the tables are disconnected, the engine cannot map the required cross-table row addresses.

### 1.5. Cross-Table Alignment & Multi-Function Table Filtering
In enterprise architectures, different departments often operate under distinct business rules for identical metrics (e.g., Marketing defines a sale upon checkout completion, while Logistics defines it upon physical delivery). Aligning these datasets requires programmatic cross-filtering checks using table-valued functions.

* **Scalar Value Constraint:** A Calculated Column operates under a Row Context and must strictly return a single scalar value (e.g., an integer, a string, or a decimal) for each row. Attempting to output an entire unfiltered or filtered table object directly into a column will trigger an engine compilation error: *"The expression refers to multiple columns. Multiple columns cannot be converted to a scalar value."*
* **Table-Valued Functions (`FILTER`):** The `FILTER()` function evaluates a specified table row-by-row and returns a dynamic, virtual sub-table containing only the rows that satisfy the boolean search predicate.
* **Row Count Aggregation (`COUNTROWS`):** To transform a virtual table object into a compliant scalar value, an aggregator like `COUNTROWS()` must be layered over the filtered table variables.

* **Implementation Syntax (Self-Referential Fact Validation):**

  Quantidade vendida Logística = 
  VAR ID_ATUAL = 'registro_notas_logistica'[ID_Produto]
  VAR TABELA_IDS = FILTER('registro_notas_logistica', 'registro_notas_logistica'[ID_Produto] = ID_ATUAL)
  RETURN
      COUNTROWS(TABELA_IDS)

* **Engine Mechanics (Override via Variables):** By capturing the current row's product identifier into the `ID_ATUAL` variable, the `FILTER()` function scans the *entire* logistics fact table, isolates all historical rows matching that specific product ID, and passes that sub-set to `COUNTROWS()`. This effectively calculates the total volume of delivery receipts per product directly within the fact table layer.

### 1.6. Table-Valued Functions: `VALUES()` vs. `DISTINCT()`
DAX utilizes table-valued functions to extract unique arrays from a specific column layout under the current filter context. While highly similar, their behavioral mechanics differ when encountering data integrity violations.

* **The Blank Row Phenomenon:** If a Fact table contains foreign keys that do not exist in the corresponding Dimension table (a referential integrity violation), the Power BI VertiPaq engine automatically injects an invisible **Blank Row** into the Dimension table to capture these orphaned records.
* **Functional Divergence:**
  * **`VALUES()`:** Returns all unique values from the column *plus* the engine's invisible Blank Row if a referential integrity violation is detected in the model.
  * **`DISTINCT()`:** Returns strictly the unique values physically present in the column, completely ignoring the engine's automatically generated Blank Row.

* **Syntax Blueprint:**
```dax
  Table_Output = VALUES(TableName[ColumnName])
  Table_Output = DISTINCT(TableName[ColumnName])
```

## 2. Combining Functions & Iterative DAX Contexts

To deliver advanced financial and operational metrics, standalone aggregations are insufficient. This module covers data model optimization through dedicated measure tables and dynamic row-by-row matrix evaluations using iterative functions.

### 2.1. Model Architecture: Disconnected Measure Containers
As a governance and scalability best practice in enterprise Power BI deployments, measures should never be scattered across transactional fact or dimension tables. 

* **The Measure Table Pattern:** A completely blank, disconnected table is instantiated solely to act as a logical folder for DAX measures. This optimizes the "Data" pane layout by separating calculated metrics from physical columns.
* **Implementation Steps:** Formulated via `Modeling > New Table`, generating a blank semantic entity. Once measures are assigned to it, the default empty column can be hidden, turning the table icon into a calculator symbol at the top of the pane.

### 2.2. Row-by-Row Iteration in Measures: The `SUMX()` Function
While a Calculated Column inherently operates within a Row Context, a standard Measure operates under a Filter Context and cannot reference naked columns without an explicit aggregation. To force a row-by-row calculation inside a measure, an **Iterator (X-Function)** must be used.

* **Engine Mechanics (`SUMX`):** `SUMX()` instantiates a temporary, in-memory Row Context over a specified table. It loops through that table row-by-row, computes the row-level expression (multiplication, division, etc.), caches the result, and finally performs a global standard `SUM` aggregation over the entire stack.
* **Syntax Blueprint:**

```dax
  Total de faturamento = 
  SUMX(
      'registro_livros_marketing', 
      'registro_livros_marketing'[Quantidade Vendas] * 'registro_livros_marketing'[Preço Unitário]
  )
```

* **Filter Context Interaction:** When placed into a Visual Matrix or Table, the measure evaluates the `SUMX()` expression within the active coordinate filters of that specific row (e.g., executing the multiplication only for the specific `ID` or `Título` visible in that row segment), naturally rolling up to the total line.

### 2.3. The Structural Mechanics of Iterator (X-Functions)
Iterator functions execute a programmatic loop, scanning an explicit target table row-by-row to compute complex mathematical expressions at a granular level before executing a final aggregation roll-up.

* **Granular Resource Cost:** Iterators force the VertiPaq engine to drop from high-speed columnar processing down to row-by-row linear evaluation. If nested or applied over millions of records unnecessarily, they cause substantial memory consumption and query latency.
* **Core Standard Iterators:**
  * **`SUMX()` / `AVERAGEX()`:** Computes a row-level scalar expression, then calculates the global Sum or Arithmetic Mean of those evaluated results.
  * **`MINX()` / `MAXX()`:** Evaluates a row-level expression and extracts the absolute lowest or highest single scalar value from the resulting temporary column array.
  * **`COUNTAX()`:** Scans the row-by-row expression and counts the rows where the resulting scalar output is non-blank (evaluates numbers, dates, or strings).

* **The Non-Iterator Classification Exception:** Functions that modify or return complex filter context architectures—such as table manipulation modifiers—do not fall under the standard transactional row-by-row arithmetic iterator classification.

### 2.4. Overriding Filter Context for Market Share Analytics (`ALL` & `DIVIDE`)
To perform comparative percentages (e.g., Pareto Analysis or Total Share Contribution), DAX metrics must calculate granular cell coordinates against a static grand total. This requires programmatic manipulation of the filter coordinates via the `ALL()` modifier.

* **The Filter Context Obstacle:** By default, when a measure is evaluated inside a visual table matrix, the structural rows inject an active filter context (e.g., isolating a single `ID_Produto`). This prevents standard calculations from accessing data outside that specific coordinate boundaries.
* **The Absolute Filter Removal Modifier (`ALL`):** When wrapped around a table or column reference, the `ALL()` function serves as an engine instruction to ignore and destroy any active filters originating from the visual matrix elements. This forces the engine to aggregate across the full dataset table structure, producing identical grand total numbers on every single visual row.

* **Code Blueprint (Grand Total Locking):**

  Total de Faturamento ALL = 
  SUMX(
      ALL('registro_livros_marketing'), 
      'registro_livros_marketing'[Quantidade Vendas] * 'registro_livros_marketing'[Preço Unitário]
  )

* **Safe Division Implementation (`DIVIDE`):** In standard computing architectures, dividing by zero or by a null field (`BLANK`) throws a terminal exception. DAX resolves this via the `DIVIDE()` function, which evaluates numerator and denominator expressions and automatically returns a clean `BLANK` if the denominator equates to zero or empty.

* **Code Blueprint (Market Share %):**

```dax
  Porcentagem vendas = 
  DIVIDE('Medidas'[Total de faturamento], 'Medidas'[Total de Faturamento ALL])
```

### 2.5. Filter Context Modifier Functions
Filter modifiers programmatically alter, destroy, or preserve the active filter coordinates injected by visual elements (matrices, slicers, charts) inside the data model engine during a measure evaluation.

* **Core Modifiers & Engine Behaviors:**
  * **`ALL()`:** Acts as an absolute filter destroyer. It completely clears the filter context from specified columns or entire tables, forcing the engine to see the global dataset regardless of user visual selections.
  * **`ALLEXCEPT()`:** Clears filters from all columns of a specified table *except* for the columns explicitly passed as arguments. This is highly useful for locking aggregations at a specific dimension attribute level (e.g., keeping the 'Product Category' filter while wiping out individual 'Product IDs').
  * **`FILTER()`:** A table-returning iterator that scans an existing table row-by-row, tests a logical condition, and returns a narrower table subset. It restricts or injects new filters into the context, rather than just clearing them.

* **The Referential Integrity & Evaluation Context Reality:**
  * **`VALUES()`:** Unlike modifiers that destroy filters, `VALUES()` strictly **preserves** the active filter context. It returns a single-column table containing the unique (distinct) visible values currently active in that cell's coordinate block.

  ### 2.6. Dynamic Context Re-evaluation & Intro to `CALCULATE()`
One of the core architectural advantages of DAX Measures over Calculated Columns is their absolute versatility and sensitivity to the active Filter Context. A single measure can seamlessly adapt its scalar output depending on the dimensional attributes mapping the visual grid.

* **Dimensional Roll-up Pattern:** When the visual matrix shifting granular elements (e.g., individual `Título do Livro`) is replaced by a higher-level dimension category (e.g., `Categoria`), the measure automatically discards the row-level filter and computes the aggregation over the broader category intersection.
* **The Fragmented Data Solution:** Moving the market share percentages from 100+ unique products to 4 core macro-categories transforms low-value noise into actionable corporate insights (e.g., identifying underperforming categories like 'Mistério e Suspense' at 9.12% for strategic marketing reinvestment).

* **The Evolution to Complex Semantic Filtering:** Up to this stage, filters have been injected implicitly by the visual structural headers (Rows, Columns, Slicers). To decouple the arithmetic calculation from the visual layout and enforce explicit, hardcoded data boundaries, developers must leverage the premier context-modifying function: **`CALCULATE()`**.

### 2.7. Business Intelligence Frameworks: Pareto Analysis (The 80/20 Rule)
Data aggregation is not just about computing mathematical absolute values; it is about driving corporate resource allocation. Pareto Analysis is a foundational business framework used to distinguish the "vital few" from the "trivial many."

* **The Core Principle:** Approximately 80% of corporate outcomes (revenue, support tickets, product defects) are driven by 20% of the underlying causes or operational entities.
* **Operational Application (Buscante Case):** * By grouping data by `Categoria`, the team discovered that "Épico e Aventura" alone drives **47.77%** of total revenue.
  * Combined with "Mitologia e Fantasia" (32.40%), just **two categories account for over 80% (exactly 80.17%) of the entire company's revenue**.
* **Strategic Action:** Instead of distributing marketing budgets equally across all books (which wastes cash on low-yield assets), leadership can now surgically reinvest in underperforming sectors like "Mistério e Suspense" (9.12%) or double down on their primary cash cows.