# Integration Guide: ML Predictions → Power BI Reports

This guide outlines best practices for integrating the late delivery predictions into your Power BI semantic model and building effective reports.

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

After running the ML pipeline (Notebooks 01-03), you'll have a **`late_delivery_predictions`** table in your Lakehouse containing:

- **Delivery identifiers** (Delivery Document, Plant, etc.)
- **Predictions** (predicted_days_late, risk_score, lateness_bucket)
- **Business context** (STRATEGIC_ACCOUNT, high_priority flag)
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
   - Select table: `late_delivery_predictions`
   - Click **Load**

4. **Verify data loaded**
   - Check Data view for the new table
   - Verify columns: Delivery Document, predicted_days_late, risk_score, etc.

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
   - Select `late_delivery_predictions`
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
Aging[Delivery Document] ←→ late_delivery_predictions[Delivery Document]
```

**How to create:**

1. Go to **Model view** in Power BI Desktop
2. Drag from `Aging[Delivery Document]` to `late_delivery_predictions[Delivery Document]`
3. Set relationship properties:
   - **Cardinality:** One-to-Many (1:*)
   - **Cross-filter direction:** Both
   - **Make active:** Yes

### Optional Relationships (if using dimension tables)

If you have separate dimension tables:

```
DimPlant[Plant] ←→ late_delivery_predictions[Plant]
DimCarrier[Carrier Code] ←→ late_delivery_predictions[EWM Carrier Code]
DimCustomer[Sold To Key] ←→ late_delivery_predictions[Sold To - Key]
```

---

## Step 3: Build DAX Measures

### Key Performance Indicators (KPIs)

#### 1. Total Open Deliveries At Risk

```dax
Open Deliveries At Risk = 
CALCULATE(
    COUNTROWS(late_delivery_predictions),
    late_delivery_predictions[predicted_days_late] > 0
)
```

#### 2. High Priority At-Risk Count

```dax
High Priority At Risk = 
CALCULATE(
    COUNTROWS(late_delivery_predictions),
    late_delivery_predictions[high_priority] = "Yes"
)
```

#### 3. Strategic Accounts At Risk

```dax
Strategic At Risk = 
CALCULATE(
    COUNTROWS(late_delivery_predictions),
    late_delivery_predictions[STRATEGIC_ACCOUNT] = "Yes",
    late_delivery_predictions[predicted_days_late] > 0
)
```

#### 4. Average Predicted Lateness

```dax
Avg Predicted Lateness = 
AVERAGE(late_delivery_predictions[predicted_days_late])
```

#### 5. Total Value At Risk

```dax
Value At Risk = 
CALCULATE(
    SUM(late_delivery_predictions[DELIVERY_VALUE_USD]),
    late_delivery_predictions[predicted_days_late] > 5
)
```

### Conditional Formatting Measures

#### Risk Level Color

```dax
Risk Level Color = 
VAR PredictedDays = AVERAGE(late_delivery_predictions[predicted_days_late])
RETURN
    SWITCH(
        TRUE(),
        PredictedDays >= 10, "#D32F2F",  -- Red: 10+ days late
        PredictedDays >= 6, "#F57C00",   -- Orange: 6-9 days late
        PredictedDays >= 3, "#FBC02D",   -- Yellow: 3-5 days late
        PredictedDays > 0, "#388E3C",    -- Light Green: 0-2 days late
        "#1976D2"                         -- Blue: On-time/early
    )
```

#### Priority Icon

```dax
Priority Icon = 
IF(
    late_delivery_predictions[high_priority] = "Yes",
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
        late_delivery_predictions,
        "DeliveryDoc", late_delivery_predictions[Delivery Document],
        "Predicted", late_delivery_predictions[predicted_days_late]
    )
VAR ActualValues = 
    SELECTCOLUMNS(
        Aging,
        "DeliveryDoc", Aging[Delivery Document],
        "Actual", Aging[AGE_REQ_DATE]
    )
VAR Joined = 
    NATURALLEFTOUTERJOIN(PredictedValues, ActualValues)
RETURN
    AVERAGEX(
        Joined,
        ABS([Predicted] - [Actual])
    )
```

---

## Step 4: Create Effective Reports

### Report 1: Executive Dashboard

**Purpose:** High-level overview of at-risk deliveries

**Key Visuals:**

1. **KPI Cards** (top row)
   - Total Open Deliveries
   - High Priority At Risk
   - Strategic Accounts At Risk
   - Avg Predicted Lateness

2. **Donut Chart: Risk Distribution**
   - Axis: `lateness_bucket`
   - Values: Count of deliveries
   - Legend: Color by risk level

3. **Bar Chart: Top 10 At-Risk Plants**
   - Axis: `Plant`
   - Values: Count of deliveries
   - Filter: `predicted_days_late > 5`

4. **Table: High Priority Alerts**
   - Columns: Delivery Document, Plant, Carrier, Predicted Days Late, Value
   - Filter: `high_priority = "Yes"`
   - Sort: Predicted Days Late (descending)

### Report 2: Operational Drill-Down

**Purpose:** Detailed view for operations team to take action

**Key Visuals:**

1. **Matrix: Plant × Carrier Performance**
   - Rows: Plant
   - Columns: EWM Carrier Code
   - Values: Avg Predicted Days Late
   - Conditional formatting: Risk Level Color measure

2. **Table: Delivery Details**
   - Columns:
     - Delivery Document
     - Customer (Sold To - Key)
     - Plant
     - Brand
     - Channel
     - Predicted Days Late
     - Risk Score
     - Strategic Account (icon)
     - Value
   - Interactive filtering by Plant, Carrier, Brand

3. **Line Chart: Predictions Over Time**
   - Axis: `prediction_timestamp`
   - Values: Count of deliveries by lateness bucket
   - Legend: lateness_bucket

4. **Scatter Plot: Risk vs Value**
   - X-axis: `predicted_days_late`
   - Y-axis: `DELIVERY_VALUE_USD`
   - Size: `risk_score`
   - Color: `STRATEGIC_ACCOUNT`

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
   - X-axis: `predicted_days_late`
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
5. Reports show latest predictions
```

### Setting Up the Schedule

**In Fabric:**

1. **Create Data Pipeline**
   - Workspace → New → Data pipeline
   - Name: "Daily_Late_Delivery_Predictions"

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
Predictions Table: late_delivery_predictions
Source: Fabric Lakehouse (ML Pipeline Output)
Update Frequency: Daily at 2:00 AM
Model: late_delivery_predictor (AutoML Regression)
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
    late_delivery_predictions,
    late_delivery_predictions[Plant],
    late_delivery_predictions[EWM Carrier Code],
    late_delivery_predictions[lateness_bucket],
    "AvgPredicted", AVERAGE(late_delivery_predictions[predicted_days_late]),
    "TotalValue", SUM(late_delivery_predictions[DELIVERY_VALUE_USD]),
    "DeliveryCount", COUNTROWS(late_delivery_predictions)
)
```

**3. Use Variables in DAX**

```dax
-- Good: Uses variables for performance
Open At Risk = 
VAR AtRiskDeliveries = 
    FILTER(
        late_delivery_predictions,
        late_delivery_predictions[predicted_days_late] > 0
    )
RETURN
    COUNTROWS(AtRiskDeliveries)

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
2. Check table name matches exactly: `late_delivery_predictions`
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
1. Check model performance metrics in Notebook 02 (MAE should be <5 days)
2. Verify feature columns match between training and scoring
3. Retrain model with more recent data
4. Check for data quality issues (nulls, incorrect values)
5. Review prediction_timestamp to ensure using latest predictions

---

## Next Steps

After integration, consider:

1. **📊 Monitor Model Performance**
   - Track prediction accuracy weekly
   - Set up alerts if MAE degrades
   - Retrain model quarterly or when accuracy drops

2. **🔄 Iterate on Features**
   - Add new features (e.g., historical carrier performance)
   - Test seasonal patterns
   - Incorporate external data (weather, holidays)

3. **📈 Expand Use Cases**
   - Predict delivery costs
   - Forecast inventory needs
   - Optimize carrier selection

4. **👥 User Training**
   - Train operations team on using predictions
   - Document action protocols for at-risk deliveries
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

**Last Updated:** November 13, 2025
