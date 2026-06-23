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

## 3: Context Transition & Complex Filtering with CALCULATE()

### 3.1. Explicit Filter Overriding via `CALCULATE()`
The `CALCULATE()` function acts as the supreme orchestrator in DAX. It is the only function capable of modifying, destroying, or injecting new coordinates into the active Filter Context generated by the visual layout.

* **Sintaxe Mecânica:** `CALCULATE(<expression>, <filter1>, <filter2>, ...)`
  * **Expression:** The core arithmetic calculation or a pre-existing measure reference (e.g., `[Total de faturamento]`).
  * **Filter Arguments:** Complex predicates that evaluate independently and completely overwrite the active matrix visual filters for the specified columns.

* **The Verbose Pattern vs. Engine Optimization:**
  When a developer writes `FILTER(ALL(Column), Column = "Value")`, they are using an explicit table-iterator pattern. For simple scalar equality filters, the DAX engine internally translates this into an optimized, cleaner predicate.

* **Code Blueprint (Hardcoded Segment Filtering):**

  Fantasia Vendas = 
  CALCULATE(
      'Medidas'[Total de faturamento],
      'registro_livros_marketing'[Categoria] = "Fantasia"
  )

* **Visual Behavior (Filter Overwrite):** Because the filter parameter utilizes an absolute predicate, it violently destroys the visual table filters coming from the matrix headers (e.g., `Épico e Aventura` or `Mistério e Suspense`), forcing the engine to return the static `Fantasia` revenue ($5.715,00) across every single row cell.

### 3.2. Explicit vs. Implicit Filter Intersection (The FILTER Table Trap)
When nesting filters inside execution blocks, developers must strictly respect table versus column constraints to avoid evaluation errors and ensure predictable context interactions.

* **The Naked Column Error (Syntax Trap):** Passing a raw, naked column expression (e.g., `FILTER(Table[Column], ...)`) into a table-iterator function triggers a terminal engine crash. The `FILTER()` function strictly demands a complete **Table** reference as its primary evaluation field. To fix it, you must pass the entire table name or use optimized implicit predicates.

* **The Overwrite vs. Intersection Mechanism:**
  * **`Fantasia Vendas All` (Using `ALL`):** Forces the engine to completely wipe out the background matrix coordinates, evaluating the target predicate globally. Result: The exact same value ($5.715,00) is mirrored across all rows.
  * **`Fantasia Vendas` (Using Base Table/Implicit):** Intersects the background visual coordinate (e.g., Row context = `Épico e Aventura`) with the internal predicate (`Categoria = "Fantasia"`). Since a record cannot satisfy two distinct categorical coordinates simultaneously, the intersection results in a logical contradiction and returns an empty `BLANK`.

* **Production Code Optimization:**
  To achieve the intersection behavior without writing verbose, risk-prone `FILTER()` tables, use the optimized native DAX syntax:

```dax
  Fantasia Vendas = 
  CALCULATE(
      'Medidas'[Total de faturamento],
      'registro_livros_marketing'[Categoria] = "Fantasia"
  )
```

### 3.3. Cross-Dimensional Filtering & Logic Intersection
When a measure with an embedded hardcoded filter evaluates inside a visual mapped by a completely different dimension, the DAX engine performs a logical cross-dimensional intersection.

* **The Matrix Coordinate Merge (Dimension Cross-Join):** When evaluating `[Fantasia Vendas]` inside a table sliced by `Editora`:
  1. **Visual Filter Context:** The matrix row injects an active filter on a specific publisher (e.g., `Editora = "Alexandria"`).
  2. **Internal CALCULATE Modifier:** The measure injects its own hardcoded predicate (`Categoria = "Fantasia"`).
  3. **The Engine Intersection:** The engine combines both coordinates using an `AND` logical operator (`Editora = "Alexandria"` **AND** `Categoria = "Fantasia"`), evaluating the total revenue only for rows that satisfy both criteria simultaneously.

* **The BLANK Handling Reality:** If a specific coordinate pair yields no matching rows in the dataset (e.g., `Editora = "Povo do Livro"` **AND** `Categoria = "Fantasia"`), the engine evaluates to an empty state (`BLANK`). In DAX tables, rows resulting in a complete `BLANK` for a measure are implicitly hidden to protect layout cleanliness.

### 3.4. Overriding Visual Coordinates via CALCULATE & ALL
By default, DAX engine evaluations are bounded by the dimensional coordinates of the active visual matrix. To break this default boundary and perform macro-level comparisons, developers must force an explicit filter override.

* **Context Erasure with `ALL()`:** When embedded as a modifier inside `CALCULATE()`, the `ALL(Table[Column])` function acts as an engine instruction to selectively clear any active filters coming from that specific column in the visual header.
* **The Global Reference Pattern:** This combination forces the internal expression (e.g., `AVERAGE()`, `SUM()`) to evaluate against the entire data scope of that attribute, regardless of what the end-user is slicing or viewing in the report row.

## 4: Matrix Hierarchies & Advanced Context Evaluation

### 4.1. Matrix Drill-downs and Multi-Layered Filter Contexts
The Matrix visual introduction fundamentally reshapes data evaluation. Unlike flat tables, a Matrix establishes an implicit parent-child data hierarchy, dynamically layering multiple dimensional filters simultaneously across different row depths.

* **Hierarchical Coordinates:** When a user expands a parent node (`Categoria`) to reveal child rows (`Editora`), the execution block evaluates the DAX expressions under a compounding coordinate system (e.g., `Categoria = "Épico e Aventura"` **AND** `Editora = "Alexandria"`).
* **The Granular Calculation Trap (The Wrong Denominator):** Reusing simple filter-erasure functions inside matrix hierarchies often yields highly distorted percentages that defy mathematical expectations.

* **The Broken Matrix Share Code (Educational Specimen):**

```dax
  Porcentagem = 
  VAR TotalDeFaturamentoEditora = 'Medidas'[Total de faturamento]
  VAR TotalDeVendasCategoria = CALCULATE('Medidas'[total de faturamento], ALL(registro_livros_marketing[Categoria]))
  VAR porcentagem = DIVIDE(TotalDeFaturamentoEditora, TotalDeVendasCategoria)
  RETURN
      porcentagem
```

* **Contextual Breakdown:** Inside the `TotalDeVendasCategoria` variable, using `ALL(Categoria)` destroys the category coordinate but *leaves the active visual publisher coordinate untouched*. As a result, the calculation evaluates the share of a single publisher against that same publisher globally, instead of tracking the publisher's share strictly within that specific category subset. This produces conflicting values like 71.89% inside sub-rows that cannot be rolled up to 100%.

### 4.4. The Two DAX Contexts & CALCULATE Evaluation Engine
To build scalable data models, developers must isolate Row Context (scanning engine) from Filter Context (coordination engine), leveraging CALCULATE to bridge or modify them.

* **Row Context (The Scanner):** Evaluates expressions line-by-line. 
  * Active inside **Calculated Columns** or triggered explicitly by **Iterator Functions** (`SUMX`, `AVERAGEX`). It does not inherently compute aggregates or cross-filter dimensions.
* **Filter Context (The Slicer):** The global coordinates active at evaluation time, dictated by visual headers, slicers, or explicit engine instructions.

* **CALCULATE Filter Modification Paths:**
  1. **Filter Overwrite (Context Cleansing):** Erases incoming matrix coordinates using `ALL` to inject a clean global predicate.
  2. **Filter Intersection (Context Aggregation):** Retains active background visual coordinates and introduces an additional constraint via logical `AND`.

* **Anti-Pattern Warning (The FILTER Table Trap):** Passing a full `FILTER(ALL(Table), ...)` block for simple column predicates forces an expensive row-by-row table scan, killing engine performance. High-performance DAX mandates native implicit predicates.

### 4.5. Multi-Value Predicates & Context Preservation (IN & KEEPFILTERS)
When expanding a business rule to target multiple categorical values simultaneously without destroying background matrix coordinates, DAX requires multi-value list evaluation paired with explicit filter preservation.

* **The `IN` Operator (List Syntax):** Instead of nesting messy, low-performance logical `OR` (`||`) statements, DAX allows the use of curly braces `{}` to construct an inline list. The `IN` operator performs a highly optimized membership check against this discrete data array.

* **The `KEEPFILTERS` Modifier:** By default, when `CALCULATE` injects a filter on a column, it overwrites existing filters on that same column. Wrapping the predicate in `KEEPFILTERS()` forces the engine into an **intersection** instead of an overwrite, merging the visual's coordinates with the measure's internal list via a logical `AND`.

* **Production Code (Multi-Category Evaluation):**

```dax
  Total de vendas Fantasia = 
  CALCULATE(
      'Medidas'[Total de faturamento],
      KEEPFILTERS('registro_livros_marketing'[Categoria] IN {"Fantasia", "Mitologia e Fantasia"})
  )
```

## 5: Advanced DAX Engine Review & Architectural Q&A

### 5.1. Cross-Table Navigation via RELATED()
* **Question:** Explain the operational mechanics of the `RELATED` function and its performance implications within a calculated column.
* **Technical Architecture:** The `RELATED` function operates strictly across active relationships, pulling data from the **"One"** side (Lookup/Dimension table) into the **"Many"** side (Fact table) of a $1 \rightarrow *$ model topology. It requires an active row context to perform the row-by-row lookup mismatch prevention.
* **Performance Assessment:** Reusing `RELATED` inside Calculated Columns is an anti-pattern for large datasets. Because Calculated Columns are computed during model refresh and stored statically in RAM, they bypass the VertiPaq column store optimization, increasing file size and slowing down memory processing compared to dynamic measures.

### 5.2. Evaluation Context Mechanics
* **Question:** Define Filter Context within Power BI, map its types, and provide operational examples.
* **The Twin Engine Contexts:**
  1. **Filter Context (The Where Clause):** The dynamic environment established by external visual elements (matrix headers, slicers, report filters). It filters the underlying data tables *before* any calculation begins.
  2. **Row Context (The Iterator):** The line-by-line scanning state triggered by calculated columns or iterative X-functions (`SUMX`, `FILTER`). It recognizes individual row slots but *does not* inherently filter other tables or perform aggregations.
* **Application Example:** When placing `Total de Faturamento` into a Matrix rowed by `Categoria`, the *Filter Context* isolates rows matching that specific category. If a `SUMX` is called inside that measure, it opens a *Row Context* to scan the isolated rows one by one.

### 5.3. CALCULATE Execution and Predicate Logic
* **Question:** Explain the internal mechanics of `CALCULATE(<expression>, <filters>)` within a corporate business scenario.
* **Engine Execution Order:** `CALCULATE` is the only function capable of destroying, modifying, or creating evaluation contexts. It evaluates its filter arguments in a clean background state, applies them to the current Filter Context (overwriting or intersecting), and then executes the core scalar expression.
* **Predicate Capacity:** There is no hard limit to the number of filter predicates passed inside a single `CALCULATE` block. Multiple predicates separated by commas are natively evaluated using an implicit logical **`AND`** (intersection).
* **Corporate Use Case (Market Share):** In a retail dashboard, calculating a brand's market share requires dividing its sales by the entire category's total. `CALCULATE` achieves this by injecting an `ALL(Brand)` predicate, forcing the denominator to ignore visual rows and capture the global category baseline.