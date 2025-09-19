# 🧹 Sales Data Cleaning (Power BI)

This project shows how I cleaned a raw sales dataset in **Power BI** using both:
- **Power Query UI** (no code), and
- **M scripts** (Advanced Editor) for an automated, reproducible pipeline.

Dataset: **PriceCo_Sales_DataWrangling-2.xlsx**  
Goal: fix errors, standardize fields, remove bad rows, and document what changed so analysis and dashboards are reliable.

---

## 🎯 Objective
Apply **basic but essential data cleaning techniques** in Power BI so decision-makers and analysts can trust the numbers.

---


## ✅ What I Fixed (Problem → Action → Why it helps)


- **City / Membership / Payment (missing values)**  
  - *Problem:* Some rows had missing values in City, Membership, or Payment.  
  - *Action:* Removed those rows so all key fields are complete.  
  - *Why:* Prevents broken grouping and wrong averages in analysis.  

- **Membership (spelling)**  
  - *Problem:* Found “Nomal” instead of “Normal.”  
  - *Action:* Replaced all “Nomal” with “Normal.”  
  - *Why:* Keeps categories consistent and avoids splitting the same group.  

- **Date (format)**  
  - *Problem:* Dates were mixed (text vs. datetime with hours).  
  - *Action:* Converted all to standard Date (YYYY-MM-DD). Filtered or fixed errors if needed.  
  - *Why:* Enables proper time grouping (monthly or yearly trends).  

- **Numeric values (consistency)**  
  - *Problem:* Unit price, Quantity, Tax, Total, and Rating had mixed or invalid types.  
  - *Action:* Cast them to numeric; ensured Quantity is whole number; dropped rows missing critical numeric fields.  
  - *Why:* Keeps calculations consistent for totals, averages, and KPIs.  

- **Rating (outliers)**  
  - *Problem:* Found junk values like `999`, `9999`, `99999` in Rating.  
  - *Action:* Any `Rating ≥ 99` was reset to `9` (kept 1–10 scale).  
  - *Why:* Prevents distorted satisfaction results.  

- **Quantity (negative values)**  
  - *Problem:* Some rows had negative quantities like `-10`, `-100`.  
  - *Action:* Converted all negatives to positive numbers using `Number.Abs()` (e.g., `-10 → 10`).  
  - *Why:* Keeps all rows while avoiding negatives that would break totals and averages.  

- **Unit price (pattern errors)**  
  - *Problem:* Found placeholder values like `999`, `9999`, `99999`.  
  - *Action:* Removed rows with those exact unit prices.  
  - *Why:* Avoids inflated totals and misleading averages.  

---

## 🖼️ Cleaning Steps (Power Query UI, no code)

1. Fix Membership typo  
2. Standardize Date column  
3. Cast numeric columns correctly  
4. Remove incomplete rows  
5. Trim whitespace in text fields  
6. Add check column for Tax validation  

---

## 🔁 Two Ways to Handle Nulls

- **Approach A – Key fields only:** remove rows if key fields (`City`, `Membership`, `Payment`, `Unit price_mxp`, `Quantity`) are null.  
- **Approach B – Full row:** remove rows if **any column** is null (stricter).  

Both are implemented in the M scripts below.

---

## ⚡ M Script – Approach A (Key fields null removal, drop non-positive qty)

```m
let
    // 1) Load Excel and the "Sample" sheet  ← change the path
    Source = Excel.Workbook(File.Contents("C:\\Data\\PriceCo_Sales_DataWrangling-2.xlsx"), null, true),
    Sample_Sheet = Source{[Item="Sample",Kind="Sheet"]}[Data],
    PromotedHeaders = Table.PromoteHeaders(Sample_Sheet, [PromoteAllScalars=true]),

    // 2) Trim text columns
    Trimmed =
        Table.TransformColumns(
            PromotedHeaders,
            {
                {"City", Text.Trim, type text},
                {"Membership", Text.Trim, type text},
                {"Gender", Text.Trim, type text},
                {"Product line", Text.Trim, type text},
                {"Payment", Text.Trim, type text},
                {"Invoice ID", Text.Trim, type text}
            }
        ),

    // 3) Fix the typo in Membership
    FixedMembership = Table.ReplaceValue(Trimmed, "Nomal", "Normal", Replacer.ReplaceText, {"Membership"}),

    // 4) Set data types
    Typed =
        Table.TransformColumnTypes(
            FixedMembership,
            {
                {"Invoice ID", type text},
                {"City", type text},
                {"Membership", type text},
                {"Gender", type text},
                {"Product line", type text},
                {"Unit price_mxp", type number},
                {"Quantity", Int64.Type},
                {"Tax 15%", type number},
                {"Total_mxp", type number},
                {"Date", type date},
                {"Payment", type text},
                {"Rating", type number}
            }
        ),

    // 5) Clean Rating outliers: if Rating >= 99 → set to 9
    CleanedRating =
        Table.TransformColumns(
            Typed,
            {{"Rating", each if _ = null then null else if _ >= 99 then 9 else _, type number}}
        ),


    // Fix Quantity: convert negatives to positive (absolute value)
    FixedQty =
        Table.TransformColumns(
            CleanedRating,
            {{"Quantity", each if _ = null then null else Number.Abs(_), Int64.Type}}
        ),

    // 7) Remove invalid Unit prices (999 / 9999 / 99999)
    CleanedPrice =
        Table.SelectRows(
            PositiveQty,
            each not List.Contains({999, 9999, 99999}, [Unit price_mxp])
        ),

    // 8) Remove rows where Unit price_mxp equals 999, 9999, or 99999
    CleanedPrice =
        Table.SelectRows(
            CleanedRating,
            each not List.Contains({999, 9999, 99999}, [Unit price_mxp])
        )

    // 9) Remove rows with nulls in IMPORTANT fields only
    NoNulls_KeyFields =
        Table.SelectRows(
            CleanedPrice,
            each [City] <> null
                and [Membership] <> null
                and [Payment] <> null
                and [Unit price_mxp] <> null
                and [Quantity] <> null
        )
in
    NoNulls_KeyFields
## ⚡ M Script – Approach B (auto drom any null value row acrossall columns)


let
    // 1) Load Excel and the "Sample" sheet  ← change the path
    Source = Excel.Workbook(File.Contents("C:\\Data\\PriceCo_Sales_DataWrangling-2.xlsx"), null, true),
    Sample_Sheet = Source{[Item="Sample",Kind="Sheet"]}[Data],
    PromotedHeaders = Table.PromoteHeaders(Sample_Sheet, [PromoteAllScalars=true]),

    // 2) Trim text columns
    Trimmed =
        Table.TransformColumns(
            PromotedHeaders,
            {
                {"City", Text.Trim, type text},
                {"Membership", Text.Trim, type text},
                {"Gender", Text.Trim, type text},
                {"Product line", Text.Trim, type text},
                {"Payment", Text.Trim, type text},
                {"Invoice ID", Text.Trim, type text}
            }
        ),

    // 3) Fix the typo in Membership
    FixedMembership = Table.ReplaceValue(Trimmed, "Nomal", "Normal", Replacer.ReplaceText, {"Membership"}),

    // 4) Set data types
    Typed =
        Table.TransformColumnTypes(
            FixedMembership,
            {
                {"Invoice ID", type text},
                {"City", type text},
                {"Membership", type text},
                {"Gender", type text},
                {"Product line", type text},
                {"Unit price_mxp", type number},
                {"Quantity", Int64.Type},
                {"Tax 15%", type number},
                {"Total_mxp", type number},
                {"Date", type date},
                {"Payment", type text},
                {"Rating", type number}
            }
        ),

    // 5) Clean Rating outliers: if Rating >= 99 → set to 9
    CleanedRating =
        Table.TransformColumns(
            Typed,
            {{"Rating", each if _ = null then null else if _ >= 99 then 9 else _, type number}}
        ),

    // 6) Add return flag, normalize Quantity to positive for analysis
    AddedReturnFlag = Table.AddColumn(CleanedRating, "IsReturn", each [Quantity] <> null and [Quantity] < 0, type logical),
    FixedQty =
        Table.TransformColumns(
            AddedReturnFlag,
            {{"Quantity", each if _ = null then null else if _ < 0 then Number.Abs(_) else _, Int64.Type}}
        ),

    // NOTE: If you want financials to reflect returns too, also flip Total/Tax when IsReturn = true.
    // Example (uncomment if needed):
    // FixedTotals =
    //     Table.TransformColumns(
    //         FixedQty,
    //         {
    //             {"Total_mxp", each if [IsReturn] then -_ else _, type number},
    //             {"Tax 15%",   each if [IsReturn] then -_ else _, type number}
    //         }
    //     ),

     // 7) Remove rows where Unit price_mxp equals 999, 9999, or 99999
    CleanedPrice =
        Table.SelectRows(
            CleanedRating,
            each not List.Contains({999, 9999, 99999}, [Unit price_mxp])
        )

    // 8) Remove rows with ANY null across the whole row (automatic)
    NoNulls_AllColumns =
        Table.SelectRows(
            FixedQty,
            each List.NonNullCount(Record.ToList(_)) = Table.ColumnCount(FixedQty)
        )
in
    NoNulls_AllColumns




📊 Before vs After Ratings

| Customer ID | Original Rating | Cleaned Rating |
| ----------- | --------------- | -------------- |
| C001        | 7               | 7              |
| C002        | 999             | 9              |
| C003        | 5               | 5              |
| C004        | 9999            | 9              |
| C005        | 99999           | 9              |
| C006        | 10              | 10             |
| C007        | 3               | 3              |


🔄 Data Cleaning Workflow (Mermaid Diagram)

flowchart TD
    A[Load Excel File] --> B[Promote Headers]
    B --> C[Trim Text Columns]
    C --> D[Fix Membership Typo (Nomal→Normal)]
    D --> E[Set Data Types]
    E --> F[Clean Rating Outliers (≥99 → 9)]
    F --> G[Handle Quantity]
    G -->|Option 1| H[Remove Rows with Quantity ≤ 0]
    G -->|Option 2| I[Add IsReturn Flag + Normalize Quantity]
    H --> J[Remove Invalid Unit Prices (999, 9999, 99999)]
    I --> J
    J --> K{Null Handling}
    K -->|Approach A| L[Remove Nulls in Key Fields]
    K -->|Approach B| M[Remove Nulls in Any Column]
    L --> N[Final Clean Dataset]
    M --> N

📂 Repo Structure
    sales-data-cleaning/
    ├── PriceCo_Sales_DataWrangling-2.xlsx    # Raw data
    ├── Cleaned_Report.pbix                   # Power BI file with steps applied
    ├── README.md
    └── .gitignore