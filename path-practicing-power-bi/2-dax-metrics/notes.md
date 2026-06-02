# DAX Metrics & Calculations Notes

## Course Overview: Practicing Power BI: Creating Metrics with DAX

This repository stores notes, learning modules, and technical breakdowns for the second stage of the Power BI development cycle. Following the ETL process completed in Power Query, this phase focuses on implementing **DAX (Data Analysis Expressions)** to build dynamic business metrics, calculations, and analytical layers for Hermex.

---

## Module 1: Exploring DAX Commands & Optimization

### 1. The Revenue Challenge (Total Vendas)

#### Objective
The leadership team at Hermex needs to discover the company's total sales revenue. To achieve this, the model must multiply the ordered quantity (found in `Fato_Pedidos`) by the unit price. However, since the price lives in a separate dimension table (`Dim_Produtos`), a cross-table evaluation is required.

---

#### Approach 1: The Suboptimal Method (Calculated Column + SUM)

Initially, the problem was solved by adding a new physical column to the Fact table and then aggregating it.

##### Step 1: Creating the Calculated Column
Inside `Fato_Pedidos`, a new column was created using the `RELATED` function to fetch the price from the dimension table:

Total Vendas (Column) = Fato_Pedidos[Quantidade] * RELATED(Dim_Produtos[Preco])

* **`RELATED()`**: Scans the active relationship in the model to pull a value from the "One" side of a relationship (`Dim_Produtos`) into the "Many" side (`Fato_Pedidos`).

##### Step 2: Aggregating via Measure
Once the column existed, a standard measure was built inside the `_Measures` table to sum it up for report visuals (e.g., Card visual displaying **9 Billion**):

Total Vendas (Measure) = SUM(Fato_Pedidos[Total Vendas (Column)])

⚠️ **The Performance Bottleneck:** This approach generated over 147,000 new rows of hard data inside the Fact table. Expanding a Fact table with calculated columns eats up massive amounts of RAM and storage, creating a severe performance drag.

---

#### Approach 2: The Optimized Method (SUMX Iteration)

To fix the performance issue, the calculated column can be completely deleted from `Fato_Pedidos`. Instead, a single optimized measure computes the row-by-row multiplication virtually in memory.

##### Step-by-Step Implementation
1. Right-click the `_Measures` table and select **New Measure**.
2. Enter the following optimized DAX formula:

Total Vendas Otimizada = SUMX(Fato_Pedidos, Fato_Pedidos[Quantidade] * RELATED(Dim_Produtos[Preco]))

3. Delete the old `Total Vendas` column from `Fato_Pedidos` to reclaim memory space.

---

#### Technical Breakdown: How SUMX Works

Every DAX function ending with an **"X"** is an **Iterator**. 

* **`SUMX(Table, Expression)`**: It forces Power BI to create a temporary, virtual row context over the specified table.
* **Execution Flow:** It goes line-by-line through `Fato_Pedidos`, multiplies the quantity by the related price, stores the intermediate result in cache, and finally sums all the calculated values together into a single number.

| Feature | Approach 1 (SUM) | Approach 2 (SUMX) |
| :--- | :--- | :--- |
| **Storage Impact** | High (Adds physical rows to model). | None (Calculated virtually in RAM). |
| **Best Practice** | ❌ Avoid for Fact table metrics. |  Ideal for row-by-row Fact table math. |
| **Result** | 9 Billion | 9 Billion |

---

### 2. Regional Classification (SWITCH Function)

#### Objective
The logistics team needs to analyze order distribution by macro-region (Norte, Nordeste, Centro-Oeste, Sudeste, Sul) instead of just checking individual state codes (`UF`). The goal is to create a new grouping layer to allow regional slicing in dashboards.

#### Step-by-Step Implementation

1. Go to the **Data View** (Exibição de dados) and select the `Fato_Pedidos` table.
2. Click **New Column** (Nova Coluna) and apply the following DAX structure:

Regiao = 
SWITCH(
    'Fato_Pedidos'[UF],
    "BR-AC", "Norte", "BR-AP", "Norte", "BR-AM", "Norte", "BR-PA", "Norte", "BR-RO", "Norte", "BR-RR", "Norte", "BR-TO", "Norte",
    "BR-AL", "Nordeste", "BR-BA", "Nordeste", "BR-CE", "Nordeste", "BR-MA", "Nordeste", "BR-PB", "Nordeste", "BR-PE", "Nordeste", "BR-PI", "Nordeste", "BR-RN", "Nordeste", "BR-SE", "Nordeste",
    "BR-DF", "Centro-Oeste", "BR-GO", "Centro-Oeste", "BR-MT", "Centro-Oeste", "BR-MS", "Centro-Oeste",
    "BR-ES", "Sudeste", "BR-MG", "Sudeste", "BR-RJ", "Sudeste", "BR-SP", "Sudeste",
    "BR-PR", "Sul", "BR-RS", "Sul", "BR-SC", "Sul",
    "Desconhecido"
)

3. Set the data type of the new column to **Text**.

---

#### Technical Breakdown: How SWITCH Works

The `SWITCH` function evaluates an expression against a list of values and returns one of multiple possible result expressions.

* **Expression / Target:** `'Fato_Pedidos'[UF]` — This is the column that the function examines row-by-row.
* **Value-Result Pairs:** The function checks if the row matches a specific state code (e.g., `"BR-SP"`) and assigns the corresponding region text (e.g., `"Sudeste"`).
* **The Default Value:** `"Desconhecido"` — The very last argument acts as an `ELSE` statement. If a row contains a state code that wasn't explicitly mapped in the list, it automatically falls into this fallback category, preventing blank anomalies.

> **Architecture Note:** While creating calculated columns in Fact tables should generally be minimized to save memory, columns used directly as **Slicers** (filtros de tela) or chart axes are valid use cases for calculated columns when the data is not available from the source Dimension tables.

---

### 3. Average Inventory Level (AVERAGE Function)

#### Objective
The operations team at Hermex needs to monitor the average quantity of products stored in warehouses over time to optimize replenishment cycles and prevent overstocking. The goal is to calculate a dynamic overall average of the available stock.

#### Step-by-Step Implementation

1. Right-click the dedicated `_Measures` table in the fields pane.
2. Select **New Measure** (instead of a column, ensuring the calculation responds dynamically to dashboard filters).
3. Enter the following DAX formula:

Média Total Estoque = 
AVERAGE(Dim_Estoque[Quantidade])

4. Format the measure as a **Whole Number** or **Decimal Number** in the measure tools tab.

---

#### Technical Breakdown: How AVERAGE Works

The `AVERAGE` function calculates the arithmetic mean of all numbers in a single column.

* **Context Evaluation:** Because this is a **Measure**, it evaluates inside the **Filter Context**. If a user slices the report by a specific year or product category, Power BI instantly filters the `Dim_Estoque[Quantidade]` column first, and then calculates the average of those remaining rows.
* **Performance Check:** It requires zero permanent storage or RAM footprint when the data model refreshes, as it executes purely on-the-fly when displayed in a visual.

---

### 4. Shipping Performance Metrics (FILTER & COUNTROWS Functions)

#### Objective
The logistics department at Hermex needs to track delivery performance by isolating shipping delays from on-time arrivals. The goal is to create two distinct measures to count the exact number of rows in the Fact table that meet these specific delivery criteria.

---

#### Part 1: Delayed Orders (Total pedidos atrasados)

##### Step-by-Step Implementation
1. Right-click the dedicated `_Measures` table and select **New Measure**.
2. Enter the following DAX formula:

Total pedidos atrasados = COUNTROWS(FILTER(Fato_Pedidos, Fato_Pedidos[Data de Entrega] > Fato_Pedidos[Data Previsao]))

3. Format the result as a **Whole Number** with thousands separators.

---

#### Part 2: On-Time Orders (Total pedidos no prazo)

##### Step-by-Step Implementation
1. Right-click the dedicated `_Measures` table and select **New Measure**.
2. Enter the following DAX formula:

Total pedidos no prazo = COUNTROWS(FILTER(Fato_Pedidos, Fato_Pedidos[Data de Entrega] <= Fato_Pedidos[Data Previsao]))

3. Format the result as a **Whole Number** with thousands separators.

---

#### Technical Breakdown: How FILTER and COUNTROWS Work Together

These metrics combine two fundamental DAX behaviors to evaluate data dynamically without adding physical rows to the model:

* **`FILTER(Table, Condition)`**: This function acts as a scanner. It creates a dynamic row context over `Fato_Pedidos`, goes line-by-line, and evaluates the conditional logic (e.g., is the actual delivery date greater than the forecast date?). It then returns a **virtual, temporary table** containing *only* the matching rows.
* **`COUNTROWS(VirtualTable)`**: Takes that temporary table generated by the `FILTER` function and simply counts how many rows are left inside it.

> **Performance Check:** By embedding this logic inside a virtual measure instead of creating physical calculated columns for filtering, the data model remains optimized and runs entirely in memory when loaded into visuals.

---

### 5. Delivery Efficiency Pipeline (AVERAGEX & DATEDIFF Functions)

#### Objective
The leadership team at Hermex requires a key performance indicator (KPI) named **Ship to Door** to evaluate operational logistics. The goal is to calculate the dynamic average number of days that elapse between the customer's purchase date and the actual delivery date across all orders.

#### Step-by-Step Implementation
1. Right-click the dedicated `_Measures` table and select **New Measure**.
2. Enter the following DAX formula:

Ship to Door = AVERAGEX(Fato_Pedidos, DATEDIFF(Fato_Pedidos[Data da Compra], Fato_Pedidos[Data de Entrega], DAY))

3. Format the measure as a **Decimal Number** (e.g., with 1 or 2 decimal places) to track precise fractional day shifts in delivery performance.

---

#### Technical Breakdown: How AVERAGEX and DATEDIFF Work

This metric uses an advanced iterator to execute row-level date math across the entire Fact table dynamically:

* **DATEDIFF(Start_Date, End_Date, Interval)**: Compares two date columns line-by-line and calculates the exact elapsed time between them based on the specified interval (`DAY`).
* **AVERAGEX(Table, Expression)**: Acts as an engine that forces a virtual row context over `Fato_Pedidos`. It first computes the `DATEDIFF` result for every individual row, stores those numbers temporarily in cache memory, and finally calculates the global arithmetic mean of all those results combined.

> **Performance Check:** Calculating date differences dynamically inside a measure keeps the overall file size compact compared to generating heavy, physical calculated columns for day counts in a 147k-row Fact table.

---

### 6. Dynamic Filter Modification (CALCULATE Function)

#### Objective
The logistics analysts at Hermex need to zoom into specific regional performance to compare operational bottlenecks. The goal is to isolate the average delivery time (the **Ship to Door** metric) exclusively for the **Sudeste** region, ensuring this specific calculation ignores or overrides conflicting external slicers.

#### Step-by-Step Implementation
1. Right-click the dedicated `_Measures` table and select **New Measure**.
2. Enter the following DAX formula:

Tempo Médio de Entrega (Sudeste) = CALCULATE([Ship to Door], Fato_Pedidos[Regiao] = "Sudeste")

3. Format the measure as a **Decimal Number** with 1 or 2 decimal places.

---

#### Technical Breakdown: The Power of CALCULATE

The `CALCULATE` function is the single most powerful engine in DAX because it is the only function capable of modifying the **Filter Context** of a report.

* **Order of Execution:** Unlike traditional functions, `CALCULATE` executes its filter arguments (the second parameter onwards) *before* evaluating the initial mathematical expression.
* **Filter Context Modification:** It forces the data model to apply a strict virtual filter (`Fato_Pedidos[Regiao] = "Sudeste"`) across the tables. Once the dataset is narrowed down to Sudeste rows only, it applies the pre-existing `[Ship to Door]` measure over that specific subset.
* **Measure Reusability:** This demonstrates a core DAX best practice—instead of rewriting the complex `AVERAGEX` and `DATEDIFF` logic all over again, you simply wrap the original `[Ship to Door]` measure inside `CALCULATE`, keeping the code clean, modular, and easy to maintain.

---

### 7. Revenue Share & Variables (VAR, RETURN, ALL & DIVIDE Functions)

#### Objective
The strategic planning team at Hermex needs to determine the revenue contribution (market share) of the **Norte** region against the company's total global turnover. The goal is to calculate a dynamic percentage that stays accurate even when external filters are applied.

#### Step-by-Step Implementation
1. Right-click the dedicated `_Measures` table and select **New Measure**.
2. Enter the following DAX formula utilizing variables for performance and readability:

% Receita de Pedidos da Região Norte = 
VAR ReceitaConcluida = 
    CALCULATE([Total Vendas Otimizada], Fato_Pedidos[Regiao] = "Norte")
 
VAR ReceitaTotal = 
    CALCULATE([Total Vendas Otimizada], ALL(Fato_Pedidos))
 
RETURN 
    DIVIDE(ReceitaConcluida, ReceitaTotal, 0)

3. Format the measure strictly as a **Percentage (%)** with 1 or 2 decimal places.

---

#### Technical Breakdown: Variables and Context Cleaners

This metric introduces advanced DAX architecture patterns resembling traditional functional programming blocks:

* **VAR / RETURN Blocks:** Variables (`VAR`) act as temporary storage containers inside memory. They evaluate their assignments immediately and lock the value in cache, preventing the engine from recalculating the same sub-expression multiple times, which drastically improves report speed. The `RETURN` keyword signals the end of declarations and outputs the final result.
* **ALL(Table_or_Column)**: This is a powerful validation controller. It acts as a filter remover. In `VAR ReceitaTotal`, `ALL(Fato_Pedidos)` commands the engine to completely ignore any current user selections on charts or slicers, guaranteeing that the denominator always holds the absolute, global company revenue.
* **DIVIDE(Numerator, Denominator, AlternateResult)**: Best practice standard for division in DAX. It safe-guards the model against mathematically illegal operations. If the denominator evaluates to blank or zero, it seamlessly outputs `0` instead of crashing the report visual.

---

---

### 8. Creating an Automated Calendar Table (Dim_Calendario)

#### Objective
The logistics model at Hermex contains multiple date fields scattered across different tables (such as purchase, delivery, and inventory update dates). To establish valid structural relationships and allow comprehensive time-intelligence slicing, a central, continuous **Calendar Dimension Table** must be generated using automated DAX functions.

#### Step-by-Step Implementation

1. In the **Home** or **Table Tools** tab of Power BI Desktop, click **New Table** (Nova Tabela).
2. Enter the following DAX script to generate the entire calendar structure dynamically:

Dim_Calendario =  
ADDCOLUMNS(
    CALENDARAUTO(),
    "Dia num", DAY([Date]),
    "Dia nome", FORMAT([Date], "dddd"),
    "Dia Semana", WEEKDAY([Date]),
    "Semana Num", WEEKNUM([Date]),
    "Mês Num", MONTH([Date]),
    "Mês Nome", FORMAT([Date], "mmm"),
    "Trimestre", QUARTER([Date]),
    "Ano", YEAR([Date])
)

3. Navigate to the Model View and establish the active relationship between `Dim_Calendario[Date]` and `Fato_Pedidos[Data da Compra]`.
4. Right-click the `Dim_Calendario` table in the fields pane, select **Mark as date table** (Marcar como tabela de data), choose the `Date` column, and confirm.

---

#### Technical Breakdown: The Automated Calendar Engine

This script leverages a dynamic column-building approach to structure time attributes without hardcoded dates:

* **CALENDARAUTO()**: Scans the entire data model automatically to find the minimum and maximum dates across all tables. It then returns a single-column table named `[Date]` containing a continuous, unbroken list of daily timestamps spanning from the first day of the earliest year to the last day of the latest year found.
* **ADDCOLUMNS(Table, Name, Expression, ...)**: Takes the generated calendar column and dynamically appends new structural attributes row-by-row based on the value of `[Date]`.
* **QUARTER([Date])**: A native DAX function that calculates and outputs the financial or calendar quarter as an integer (1 through 4).
* **FORMAT([Date], "mmm") / "dddd"**: Formats dates into localized text blocks. Using `"mmm"` extracts a compact, three-letter abbreviated month name (e.g., "jan", "fev"), which is ideal for optimizing chart space, while `"dddd"` extracts the full weekday name (e.g., "segunda-feira").

---

### 9. Matrix Cross-Filtering & Segmented Revenue Share (CALCULATE & ALL on Dimension)

#### Objective
The logistics directors at Hermex need to analyze the percentage distribution of sales by vehicle type within each geographical region. The goal is to build a Matrix visual where the rows represent the `Regiao`, columns represent the vehicle `Tipo`, and the values show the internal market share of each vehicle per region, summing up to exactly 100% horizontally.

#### Step-by-Step Implementation
1. Right-click the dedicated `_Measures` table and select **New Measure**.
2. Enter the following DAX formula:

% Vendas por Veículo e Região = 
VAR VendasContexto = [Total Vendas Otimizada]
VAR VendasDaRegiao = CALCULATE([Total Vendas Otimizada], ALL(Dim_Veículos))
RETURN
    DIVIDE(VendasContexto, VendasDaRegiao, 0)

3. Format the measure strictly as a **Percentage (%)** with 1 or 2 decimal places.
4. **Visual Setup:** Create a **Matrix** visual, drag `Fato_Pedidos[Regiao]` to Rows, `Dim_Veículos[Tipo]` to Columns, and this new measure to the Values area.

---

#### Technical Breakdown: Advanced Filter Context Isolation

This solution uses a targeted filter clearing strategy to calculate partial totals inside a multi-axis visual:

* **Targeted ALL(Dimension_Table)**: Instead of clearing all report filters with `ALL(Fato_Pedidos)`, passing `ALL(Dim_Veículos)` directs the engine to selectively drop filters coming *only* from the vehicle fleet attributes. 
* **Context Preservation**: When evaluating a specific matrix cell (e.g., "Norte" rows and "Moto" columns), the denominator overrides the "Moto" filter to calculate the sum of *all* vehicles, but preserves the "Norte" filter intact. This ensures the division calculates the vehicle's share relative to that specific region, rather than the global company turnover.