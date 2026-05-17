# Power Query Notes

## Hermex Database Overview

The initial data is stored in an Excel workbook (`.xlsx`) containing four distinct sheets. 

### Tables Structure

#### 1. Products (Produtos)
Contains details about the products handled by the logistics company.

| Column | Data Type / Description |
| :--- | :--- |
| id_produto | Product unique identifier |
| categoria_produto | Product category |
| preço | Unit price |

#### 2. Inventory (Estoque)
Tracks current product availability in stock.

| Column | Data Type / Description |
| :--- | :--- |
| ID Produto | Product unique identifier |
| Data atualização | Timestamp of the last update |
| Quantidade | Stock quantity available |

#### 3. Vehicles (Veículos)
Lists the fleet available for logistics operations.

| Column | Data Type / Description |
| :--- | :--- |
| ID veículos | Vehicle unique identifier |
| Tipo | Vehicle type (e.g., Car, Truck, Motorcycle) |
| Status | Current availability (e.g., Occupied, Available) |

#### 4. Orders (Pedidos)
The core transactional table containing order and delivery details.

| Column | Data Type / Description |
| :--- | :--- |
| ID Pedido | Order unique identifier |
| ID Produto | Product identifier |
| Quantidade | Quantity ordered |
| ID Veículo | Vehicle assigned to the delivery |
| Status do pedido | Delivery status (e.g., Delivered, In transit) |
| Data da compra | Purchase date |
| Data de entrega | Actual delivery date |
| Data previsão | Estimated delivery date |
| UF | State abbreviation code |
| ESTADO | Full state name |

---

---

## Step-by-Step Data Transformation (Data Cleaning)

### 1. Data Import
1. Open **Power BI Desktop**.
2. In the top ribbon, click **Get Data** > **Excel workbook**.
3. Select the `base-de-dados-hermex.xlsx` file and click **Open**.
4. In the Navigator window, check the boxes for **Pedidos**, **Estoque**, **Veículos**, and **Produtos**.
5. Click **Transform Data** to launch the Power Query Editor.

### 2. Removing Unnecessary Columns

| Query (Table) | Action Taken | Reason |
| :--- | :--- | :--- |
| **Produtos** | Removed `Column4` | Contained only null values. |
| **Pedidos** | Removed `Estados` | Contained only null values. |
| **Veículos** | Selected essential columns and applied **Remove Other Columns** | Kept only relevant columns, discarding multiple null columns at once. |

### 3. Handling Empty and Null Values

To ensure data quality across the dataset, the following validation steps were applied:

1. Enabled **Column Quality** under the **View** tab to check the percentage of valid, empty, and error values.
2. Switched the data profiling setting in the bottom-left status bar from *"Column profiling based on top 1000 rows"* to ***"Column profiling based on entire dataset"*** for complete accuracy.
3. Inspected the ID columns for **Estoque**, **Produtos**, and **Veículos**.
4. Filtered out blank rows by clicking the column header dropdown and selecting **Remove Empty**.

### 4. Advanced Value Replacement (Delivery Dates)

The `Data de entrega` (Delivery Date) column in the **Pedidos** table contained the text `"Não disponível"` for orders still in transit. Since these rows represent active orders, they could not be deleted. 

To convert the column type correctly later without generating errors, the text was replaced with blank values:

1. Right-clicked a cell containing `"Não disponível"` and selected **Replace Values...**.
2. Left the **Replace With** field completely empty.
3. Clicked **OK** to turn the text strings into standard blank values (`null`), preparing the column for proper date formatting.

> Data transformation is an iterative process. It may be necessary to return to Power Query to refine these steps as the analysis progresses.