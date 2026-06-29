# Course 3: Power BI - Applying DAX to Business Scenarios

## 1: Project Alignment and Advanced Scope

### 1.1. Core Deliverables & Analytical Architecture
The primary objective of this project is to transition from basic row/filter context manipulation into advanced business intelligence frameworks using dynamic DAX expressions.

* **Key Deliverables Matrix:**
  | Analytical Framework | Business Value | Core DAX Mechanics |
  | :--- | :--- | :--- |
  | **ABC / Pareto Analysis** | Classification of revenue drivers (80/20 rule) to optimize inventory/investments. | Virtual tables (`SUMX`), dynamic ranking, and running totals. |
  | **Moving Averages & Totals** | Smoothing out seasonality to identify true macro-economic trends over time. | Time Intelligence modification and window functions. |
  | **Scenario Analysis (What-If)** | Simulating economic shocks, price elasticity, or marketing campaign impacts. | Parameter harvesting via disconnected tables. |

* **Architectural Strategy:** All business logic will be solved dynamically inside the DAX engine using optimized measures, completely avoiding hardcoded columns to maintain data compression and memory performance.

### 2: Time Intelligence and Trend Smoothing

### 2.1. Moving Totals & Volatility Reduction
When handling daily sales data, variance often introduces noise (spikes and drops) that masks the actual business trajectory. Implementing a dynamic moving window allows the model to smooth out these fluctuations for cleaner trend line comparison.

* **Variables (`VAR / RETURN`):** Instantiating `MAX(Tb_Calendario[Date])` inside a variable captures the specific date coordinate currently being evaluated by the chart axis. This preserves the current date context before recalculating the timeline.
* **The `DATESBETWEEN` Engine:** This function builds a customized data table on the fly. It accepts a date column, a dynamic starting boundary ($CurrentDate - 30$ days), and a dynamic ending boundary ($CurrentDate$), forcing `CALCULATE` to sum data inside this shifting 31-day window.

* **Production Code (Rolling 30-Day Totals and Last Year Comparison):**

```dax  
  total movel 30 dias = 
  VAR DataAtual = MAX(Tb_Calendario[Date])
  RETURN
  CALCULATE(
      [Total Vendas],
      DATESBETWEEN(
          Tb_Calendario[Date],
          DataAtual - 30,
          DataAtual
      )
  )

  total movel 30 dias LY = 
  VAR DataAtual = MAX(Tb_Calendario[Date])
  RETURN
  CALCULATE(
      [Total Vendas Ano Anterior],
      DATESBETWEEN(
          Tb_Calendario[Date],
          DataAtual - 30,
          DataAtual
      )
  )
```

### 2.2. Moving Averages & Iteration Engines (`AVERAGEX`)
While a Moving Total shows the absolute accumulated volume, a Moving Average divides that volume by the temporal window size to calculate the standardized daily performance. This normalizes comparisons across months with different lengths (e.g., February vs. March).

* **The Iteration Architecture (`AVERAGEX`):** Unlike standard aggregation functions, `AVERAGEX` is an iterator. It takes an temporary table as its first argument (generated dynamically by `DATESBETWEEN`) and loops through it day by day, evaluating the measure (second argument) for each row before calculating the final arithmetic mean.

* **Production Code (Rolling 30-Day Averages and Last Year Comparison):**

```dax
  media movel 30 dias = 
  VAR DataAtual = MAX(Tb_Calendario[Date])
  RETURN
  AVERAGEX(
      DATESBETWEEN(
          Tb_Calendario[Date],
          DataAtual - 30,
          DataAtual
      ),
      [Total Vendas]
  )

  media movel 30 dias LY = 
  VAR DataAtual = MAX(Tb_Calendario[Date])
  RETURN
  AVERAGEX(
      DATESBETWEEN(
          Tb_Calendario[Date],
          DataAtual - 30,
          DataAtual
      ),
      [Total Vendas Ano Anterior]
  )
```

### 2.3. Context Filtering & Row Context Validation (`COUNTROWS`)
When utilizing time intelligence over a continuous calendar table, the engine may evaluate periods where no underlying transactional data exists (e.g., displaying blank or repeated values at the boundaries of the historical data). To prevent visual distortion and enforce strict alignment with active sales periods, a logical short-circuit pattern using `IF` and `COUNTROWS` must be implemented.

* **The Validation Guard Pattern:** By nesting the core calculation inside an `IF(COUNTROWS(Fact_Table) > 0, ...)` statement, the DAX engine evaluates the rolling window only for dates containing actual transaction history. If no rows exist in the fact table for that context, it returns a blank, cleaning up charts and tables dynamically.

* **Production Code (Protected Measures):**

```dax
total movel 30 dias = 
VAR DataAtual = MAX(Tb_Calendario[Date])
RETURN
IF(
    COUNTROWS(Tb_ItensNotas) > 0,
    CALCULATE(
        [Total Vendas],
        DATESBETWEEN(
            Tb_Calendario[Date],
            DataAtual - 30,
            DataAtual
        )
    )
)

media movel 30 dias = 
VAR DataAtual = MAX(Tb_Calendario[Date])
RETURN
IF(
    COUNTROWS(Tb_ItensNotas) > 0,
    AVERAGEX(
        DATESBETWEEN(
            Tb_Calendario[Date],
            DataAtual - 30,
            DataAtual
        ),
        [Total Vendas]
    )
)
```

### 2.4. Performance Optimization via Short-Circuiting (`Performance Analyzer`)
Adding a logical validation before executing complex time intelligence calculations does not just clean up the visual aspect of the report; it radically optimizes query performance. 

When displaying continuous dates in a table or visual, the DAX engine naturally attempts to calculate the rolling measures for every single calendar row, even if no underlying transaction data exists for that specific period (generating repetitive or blank rows).

* **The Performance Bottleneck:** Without a validation guard, filtering a specific year forces the engine to iterate over thousands of empty rows from adjacent years, drastically increasing the **DAX Query** time inside the **Performance Analyzer**.
* **The Short-Circuit Solution:** By using `IF(COUNTROWS(Fact_Table) > 0, ...)`, the engine performs a lightweight count first. If the count is zero, it immediately skips the expensive `AVERAGEX` or `CALCULATE` logic (short-circuiting).
* **Benchmark Results:** In production scenarios, filtering out these empty context evaluations can reduce the DAX Query execution time from **897 ms to 26 ms** — a **97% increase in performance**, ensuring instantaneous dashboard interactions.