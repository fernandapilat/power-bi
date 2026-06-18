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