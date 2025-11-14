# Integration Guide: ML Predictions → Power BI Reports

This guide outlines best practices for integrating the ship date predictions into your Power BI semantic model and building effective reports.

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Step 1: Add Predictions Table to Semantic Model](#step-1-add-predictions-table-to-semantic-model)
3. [Step 2: Create Relationships](#step-2-create-relationships)
4. [Step 3: Build DAX Measures](#step-3-build-dax-measures)
5. [Step 4: Create Effective Reports](#step-4-create-effective-reports)
6. [Step 5: Schedule & Refresh](#step-5-schedule--refresh)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)

---

## Overview

After running the ML pipeline (Notebooks 01-03), you'll have a **`ship_date_predictions`** table in your Lakehouse containing:

- **Delivery identifiers** (Delivery Document, Plant, etc.)
- **Predictions** (predicted_days_to_ship, predicted_ship_date, lead_time_bucket)
- **Business context** (STRATEGIC_ACCOUNT, extended_processing flag)
- **Metadata** (prediction_timestamp)

**Goal:** Integrate this table into your existing "DLV Aging Columns & Measures" semantic model for unified reporting.

---

## Step 1: Add Predictions Table to Semantic Model

### Option A: Power BI Desktop

1. **Open your semantic model** in Power BI Desktop
   - File → Open → Select "DLV Aging Columns & Measures.pbix"

2. **Add Lakehouse connection**
   - Home → Get Data → More
   - Search for "Lakehouse"
   - Select **Microsoft Fabric Lakehouse**
   - Click **Connect**

3. **Select predictions table**
   - Navigate to your Lakehouse
   - Select table: `ship_date_predictions`
   - Click **Load**

4. **Verify data loaded**
   - Check Data view for the new table
   - Verify columns: Delivery Document, predicted_days_to_ship, predicted_ship_date, etc.

### Option B: Fabric Portal (Direct Lake Mode)

1. **Open Fabric workspace**
   - Navigate to your workspace in Fabric portal

2. **Open semantic model**
   - Find "DLV Aging Columns & Measures"
   - Click **Open semantic model**

3. **Add table via Direct Lake**
   - Click **+ New data source**
   - Select **Lakehouse**
   - Choose your Lakehouse
   - Select `ship_date_predictions`
   - Click **Create**

4. **Advantages of Direct Lake:**
   - ✅ Near real-time updates (no import refresh needed)
   - ✅ Lower storage costs
   - ✅ Faster query performance for large datasets

---

## Step 2: Create Relationships

### Primary Relationship

**Connect predictions to historical data:**

```
Aging[Delivery Document] ←→ ship_date_predictions[Delivery Document]
```

**How to create:**

1. Go to **Model view** in Power BI Desktop
2. Drag from `Aging[Delivery Document]` to `ship_date_predictions[Delivery Document]`
3. Set relationship properties:
   - **Cardinality:** One-to-Many (1:*)
   - **Cross-filter direction:** Both
   - **Make active:** Yes

### Optional Relationships (if using dimension tables)

If you have separate dimension tables:

```
DimPlant[Plant] ←→ ship_date_predictions[Plant]
DimCarrier[Carrier Code] ←→ ship_date_predictions[EWM Carrier Code]
DimCustomer[Sold To Key] ←→ ship_date_predictions[Sold To - Key]
```

---

## Step 3: Build DAX Measures

### Key Performance Indicators (KPIs)

#### 1. Total Open Deliveries

```dax
Total Open Deliveries = 
COUNTROWS(ship_date_predictions)
```

#### 2. Extended Processing Count

```dax
Extended Processing Count = 
CALCULATE(
    COUNTROWS(ship_date_predictions),
    ship_date_predictions[extended_processing] = "Yes"
)
```

#### 3. Strategic Accounts with Extended Processing

```dax
Strategic Extended Processing = 
CALCULATE(
    COUNTROWS(ship_date_predictions),
    ship_date_predictions[STRATEGIC_ACCOUNT] = "Yes",
    ship_date_predictions[extended_processing] = "Yes"
)
```

#### 4. Average Predicted Lead Time

```dax
Avg Predicted Lead Time = 
AVERAGE(ship_date_predictions[predicted_days_to_ship])
```

#### 5. Total Value - Extended Processing

```dax
Value Extended Processing = 
CALCULATE(
    SUM(ship_date_predictions[DELIVERY_VALUE_USD]),
    ship_date_predictions[predicted_days_to_ship] > 7
)
```

### Conditional Formatting Measures

#### Lead Time Color

```dax
Lead Time Color = 
VAR PredictedDays = AVERAGE(ship_date_predictions[predicted_days_to_ship])
RETURN
    SWITCH(
        TRUE(),
        PredictedDays >= 10, "#D32F2F",  -- Red: 10+ days
        PredictedDays >= 6, "#F57C00",   -- Orange: 6-9 days
        PredictedDays >= 3, "#FBC02D",   -- Yellow: 3-5 days
        "#388E3C"                         -- Green: 0-2 days
    )
```

#### Priority Icon

```dax
Priority Icon = 
IF(
    ship_date_predictions[extended_processing] = "Yes",
    UNICHAR(9888),  -- Warning symbol
    ""
)
```

### Comparison Measures (Predicted vs Actual)

#### Prediction Accuracy (for closed deliveries)

```dax
Prediction Accuracy MAE = 
VAR PredictedValues = 
    SELECTCOLUMNS(
        ship_date_predictions,
        "DeliveryDoc", ship_date_predictions[Delivery Document],
        "Predicted", ship_date_predictions[predicted_days_to_ship]
    )
VAR ActualValues = 
    SELECTCOLUMNS(
        Aging,
        "DeliveryDoc", Aging[Delivery Document],
        "Actual", Aging[DAYS_TO_SHIP]
    )
VAR Joined = 
    NATURALLEFTOUTERJOIN(PredictedValues, ActualValues)
RETURN
    AVERAGEX(
        Joined,
        ABS([Predicted] - [Actual])
    )
```

**Note:** This measure requires calculating actual DAYS_TO_SHIP in the Aging table:
```dax
DAYS_TO_SHIP = DATEDIFF(Aging[Delivery Created On], Aging[GI Date], DAY)
```

---

## Step 4: Create Effective Reports

### Report 1: Executive Dashboard

**Purpose:** High-level overview of ship date forecasts

**Key Visuals:**

1. **KPI Cards** (top row)
   - Total Open Deliveries
   - Extended Processing Count
   - Strategic Extended Processing
   - Avg Predicted Lead Time (days)

2. **Donut Chart: Lead Time Distribution**
   - Axis: `lead_time_bucket`
   - Values: Count of deliveries
   - Legend: Color by lead time range

3. **Bar Chart: Top 10 Plants by Lead Time**
   - Axis: `Plant`
   - Values: Average predicted_days_to_ship
   - Sort: Descending

4. **Table: Extended Processing Alerts**
   - Columns: Delivery Document, Plant, Carrier, Predicted Ship Date, Lead Time, Value
   - Filter: `extended_processing = "Yes"`
   - Sort: Predicted Days To Ship (descending)

### Report 2: Operational Drill-Down

**Purpose:** Detailed view for operations team to plan logistics

**Key Visuals:**

1. **Matrix: Plant × Carrier Lead Times**
   - Rows: Plant
   - Columns: EWM Carrier Code
   - Values: Avg Predicted Days To Ship
   - Conditional formatting: Lead Time Color measure

2. **Table: Delivery Details**
   - Columns:
     - Delivery Document
     - Customer (Sold To - Key)
     - Plant
     - Brand
     - Channel
     - Created On
     - Predicted Ship Date
     - Lead Time (days)
     - Strategic Account (icon)
     - Value
   - Interactive filtering by Plant, Carrier, Brand

3. **Calendar Visual: Predicted Ship Dates**
   - Axis: `predicted_ship_date`
   - Values: Count of deliveries by date
   - Tooltip: Delivery details

4. **Scatter Plot: Lead Time vs Value**
   - X-axis: `predicted_days_to_ship`
   - Y-axis: `DELIVERY_VALUE_USD`
   - Color: `STRATEGIC_ACCOUNT`
   - Size: Delivery quantity

### Report 3: Model Performance Monitoring

**Purpose:** Track prediction accuracy and model drift

**Key Visuals:**

1. **KPI Cards**
   - Prediction Accuracy MAE
   - Model Version
   - Last Training Date
   - Records Scored Today

2. **Line Chart: Accuracy Over Time**
   - Track MAE week-over-week
   - Identify model degradation

3. **Histogram: Prediction Distribution**
   - X-axis: `predicted_days_to_ship`
   - Y-axis: Count
   - Compare to actual distribution

4. **Table: Top Prediction Errors**
   - Show deliveries where prediction was most off
   - Columns: Delivery, Predicted, Actual, Error

### Best Practices for Visuals

**✅ DO:**
- Use consistent color coding (red=urgent, yellow=caution, green=good)
- Add slicers for Plant, Carrier, Brand, Strategic Account
- Enable drill-through from summary to detail views
- Use tooltips to show additional context
- Add "Last Refreshed" timestamp

**❌ DON'T:**
- Overwhelm with too many visuals (max 6-8 per page)
- Use 3D charts (harder to read)
- Mix prediction and actual data without clear labels
- Hide important filters

---

## Step 5: Schedule & Refresh

### Automated Pipeline Schedule

**Recommended Schedule:**

```
Daily at 2:00 AM (off-peak hours)

Pipeline Flow:
1. Notebook 02: Train model (10 min)
2. Notebook 03: Score open deliveries (2 min)
3. Predictions table updated
4. Semantic model auto-refreshes (Direct Lake)
5. Reports show latest ship date forecasts
```

### Setting Up the Schedule

**In Fabric:**

1. **Create Data Pipeline**
   - Workspace → New → Data pipeline
   - Name: "Daily_Ship_Date_Predictions"

2. **Add Notebook Activities**
   ```
   Activity 1: Run Notebook
   - Notebook: 02_autoML_training_pipeline.ipynb
   - Timeout: 30 minutes
   
   Activity 2: Run Notebook (on success of Activity 1)
   - Notebook: 03_batch_scoring_pipeline.ipynb
   - Timeout: 15 minutes
   ```

3. **Schedule Pipeline**
   - Click "Schedule"
   - Frequency: Daily
   - Time: 2:00 AM
   - Time zone: Your local time zone
   - Enable: Yes

4. **Set Up Alerts**
   - On failure: Email admin
   - On success: Optional notification

### Refresh Strategy Options

| Strategy | Pros | Cons | When to Use |
|----------|------|------|-------------|
| **Direct Lake** | Real-time updates, no refresh needed | Requires Fabric F64+ | Default choice |
| **Import Mode** | Faster queries, works on any capacity | Manual refresh needed | Small datasets only |
| **Incremental Refresh** | Only refresh new predictions | Complex setup | Large historical datasets |

---

## Best Practices

### Data Governance

**✅ Row-Level Security (RLS)**

Apply the same RLS rules from the Aging table to predictions:

```dax
-- Example RLS rule for predictions table
[Plant] IN { 
    SELECTCOLUMNS(
        FILTER(PlantAccess, PlantAccess[User] = USERPRINCIPALNAME()),
        PlantAccess[Plant]
    )
}
```

**✅ Data Lineage Documentation**

Document in semantic model description:
```
Predictions Table: ship_date_predictions
Source: Fabric Lakehouse (ML Pipeline Output)
Update Frequency: Daily at 2:00 AM
Model: ship_date_predictor (AutoML Regression)
Accuracy: ~X days MAE (monitor weekly)
Owner: [Your Team]
```

### Performance Optimization

**1. Optimize Relationships**
- Use single-column keys where possible
- Avoid many-to-many relationships
- Set proper cardinality

**2. Create Aggregation Tables**

For large datasets, create aggregation tables:

```dax
PredictionsSummary = 
SUMMARIZE(
    ship_date_predictions,
    ship_date_predictions[Plant],
    ship_date_predictions[EWM Carrier Code],
    ship_date_predictions[lead_time_bucket],
    "AvgLeadTime", AVERAGE(ship_date_predictions[predicted_days_to_ship]),
    "TotalValue", SUM(ship_date_predictions[DELIVERY_VALUE_USD]),
    "DeliveryCount", COUNTROWS(ship_date_predictions)
)
```

**3. Use Variables in DAX**

```dax
-- Good: Uses variables for performance
Extended Processing Count = 
VAR ExtendedDeliveries = 
    FILTER(
        ship_date_predictions,
        ship_date_predictions[predicted_days_to_ship] > 7
    )
RETURN
    COUNTROWS(ExtendedDeliveries)

-- Bad: Recalculates filter multiple times
-- CALCULATE(COUNTROWS(...), FILTER(...)) -- repeated multiple times
```

### User Experience

**1. Add Report Navigation**
- Bookmark navigation between pages
- Breadcrumb trail (Executive → Operational → Detail)
- "Back" buttons on drill-through pages

**2. Contextual Help**
- Add info icons with tooltips explaining metrics
- Include "How to Use This Report" page
- Document what actions users should take

**3. Mobile-Friendly Layout**
- Create mobile view for executive dashboard
- Prioritize KPIs and high-priority alerts
- Use vertical layout for mobile

---

## Troubleshooting

### Issue: Predictions Table Not Showing in Semantic Model

**Solutions:**
1. Verify Lakehouse connection is active
2. Check table name matches exactly: `ship_date_predictions`
3. Refresh semantic model in Fabric portal
4. Check workspace permissions (need Contributor or higher)

### Issue: Relationship Not Working

**Solutions:**
1. Verify both columns have matching data types
2. Check for null values in join keys
3. Ensure cardinality is set correctly (usually 1:*)
4. Try "Both" cross-filter direction if filters not working

### Issue: DAX Measure Returning Blank

**Solutions:**
1. Check filter context (use DAX Studio to debug)
2. Verify column names match exactly (case-sensitive)
3. Test with simple COUNT first, then add complexity
4. Use CALCULATE to override filters if needed

### Issue: Report Performance is Slow

**Solutions:**
1. Switch to Direct Lake mode (if not already)
2. Reduce number of visuals per page (max 6-8)
3. Use aggregations for summary views
4. Remove unnecessary columns from predictions table
5. Check for circular relationships in model

### Issue: Predictions Seem Inaccurate

**Solutions:**
1. Check model performance metrics in Notebook 02 (MAE should be <3 days for lead time)
2. Verify feature columns match between training and scoring
3. Retrain model with more recent data
4. Check for data quality issues (nulls, incorrect dates)
5. Review prediction_timestamp to ensure using latest predictions

---

## Next Steps

After integration, consider:

1. **📊 Monitor Model Performance**
   - Track prediction accuracy weekly
   - Set up alerts if MAE degrades
   - Retrain model quarterly or when accuracy drops

2. **🔄 Iterate on Features**
   - Add new features (e.g., historical plant performance)
   - Test seasonal patterns
   - Incorporate order complexity metrics

3. **📈 Expand Use Cases**
   - Predict delivery costs
   - Forecast inventory needs based on ship dates
   - Optimize production scheduling

4. **👥 User Training**
   - Train operations team on using ship date forecasts
   - Document planning protocols for extended lead times
   - Gather feedback for improvements

---

## Additional Resources

- [Fabric Documentation: Semantic Models](https://learn.microsoft.com/en-us/fabric/data-warehouse/semantic-models)
- [Power BI: DAX Best Practices](https://learn.microsoft.com/en-us/power-bi/guidance/dax-best-practices)
- [Direct Lake Mode Guide](https://learn.microsoft.com/en-us/fabric/get-started/direct-lake-overview)
- [MLflow Model Registry](https://mlflow.org/docs/latest/model-registry.html)

---

## Support

For issues with:
- **Notebooks/ML Pipeline:** Check logs in Fabric workspace → Monitoring
- **Semantic Model:** Use DAX Studio or Performance Analyzer
- **Reports:** Check report settings and filters

**Last Updated:** November 14, 2025
