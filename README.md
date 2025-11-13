# 📦 Late Delivery Prediction POC

**Predict late deliveries before they happen using Microsoft Fabric, Semantic Link, AutoML, and Power BI**

---

## 📋 Overview

This Proof of Concept (POC) demonstrates a **complete end-to-end machine learning pipeline** for predicting late deliveries in global operations. Built entirely on Microsoft Fabric using AutoML, MLflow, Semantic Link, Lakehouse tables, and Power BI.

The solution identifies which **open deliveries will arrive late** vs the Customer Requested Delivery Date, enabling proactive intervention on high-risk orders with strategic account prioritization.

### 🚀 What This POC Delivers

✅ **End-to-end ML workflow** - Training → Registry → Scoring → BI reporting  
✅ **Predictive late delivery detection** - AutoML with classification + regression  
✅ **Risk scoring and bucketing** - 0-2, 3-5, 6-9, 10+ days late categories  
✅ **Strategic account prioritization** - High-priority flagging for critical customers  
✅ **Performance dashboards** - Interactive Power BI reports with DAX measures  
✅ **Fully repeatable pattern** - Scalable, Fabric-native architecture  

### Business Value

- **Proactive intervention** - Identify late deliveries before they occur
- **Strategic account protection** - Automatically flag high-priority customers
- **Root cause analysis** - Analyze patterns by carrier, plant, product, channel
- **Resource optimization** - Focus resources on highest-risk deliveries
- **Operational visibility** - Three analysis scenarios (historical, predictive, current day)
- **Unified platform** - All analytics workloads in one place (Fabric)

---

## 🎯 Use Case: Global Operations Insights - Order Delivery

**Business Problem**: 
Lack of visibility into which open deliveries will arrive late compared to the **Customer Requested Delivery Date**, preventing proactive actions to mitigate customer impact.

**Solution Scenarios**:
1. **Historical Performance** (Last 2 weeks): Closed deliveries - actual late vs on-time performance by carrier, plant, strategic account
2. **Predictive Insights** (Open deliveries): ML-scored risk predictions with lateness buckets (0-2, 3-5, 6-9, 10+ days)
3. **Current Day Monitoring**: Real-time tracking of deliveries at risk today

**Key Capabilities**:
- Daily predictions on all open deliveries
- Risk scores (0-1 normalized) and lateness buckets
- Strategic account + late delivery = high_priority flag
- Carrier and plant performance benchmarking
- Brand and channel late delivery patterns

---

## 🎯 Why Semantic Link?

**Semantic Link** bridges Power BI semantic models and Python notebooks in Fabric:

- **Single Source of Truth** - Use your existing "DLV Aging Columns & Measures" semantic model
- **Leverage Existing Work** - Reuse semantic models already created by your BI team
- **Always Current** - Live data access means you're always working with the latest information
- **Use DAX in Python** - Query your semantic model directly from notebooks
- **Full Circle** - Train on BI data → Score predictions → Power BI reports (all in Fabric)

```python
import sempy.fabric as fabric

# Query closed deliveries for training
dax_query = """
EVALUATE
FILTER(
    Aging,
    NOT(ISBLANK(Aging[GI Date]))
)
"""

df_closed = fabric.evaluate_dax(dataset="DLV Aging Columns & Measures", dax_string=dax_query)
```

---

## 🏗️ Architecture

### Workflow Overview

```
┌──────────────────────────────────────┐
│   Fabric Semantic Model              │
│   "DLV Aging Columns & Measures"     │
│   (Aging Table)                      │
└────────────┬─────────────────────────┘
             │
             │ Semantic Link (DAX queries)
             │
    ┌────────┴────────┐
    │                 │
    ▼                 ▼
┌───────────┐    ┌─────────────┐
│  Closed   │    │    Open     │
│Deliveries │    │ Deliveries  │
│(Training) │    │  (Scoring)  │
└─────┬─────┘    └──────┬──────┘
      │                 │
      ▼                 │
┌─────────────────┐     │
│  Fabric Notebook│     │
│  Data Prep (01) │     │
└────────┬────────┘     │
         │              │
         ▼              │
┌─────────────────┐     │
│   AutoML        │     │
│  Training (02)  │     │
│  - Regression   │     │
│  - Classification│    │
└────────┬────────┘     │
         │              │
         ▼              │
┌─────────────────┐     │
│ MLflow Registry │     │
│  Model Storage  │     │
└────────┬────────┘     │
         │              │
         └──────┬───────┘
                │
                ▼
        ┌───────────────┐
        │Batch Scoring  │
        │Pipeline (03)  │
        │+ Risk Scores  │
        │+ Buckets      │
        │+ Priority     │
        └───────┬───────┘
                │
                ▼
        ┌───────────────┐
        │   Lakehouse   │
        │  Delta Table  │
        └───────┬───────┘
                │
                ▼
        ┌───────────────┐
        │   Power BI    │
        │    Report     │
        └───────────────┘
```

### How It Works

**Step 1: Data Preparation (Notebook 01)**
- Connect to "DLV Aging Columns & Measures" semantic model using Semantic Link
- Load **closed deliveries** (GI Date populated) for training
- Target variable: **AGE_REQ_DATE** (days late/early vs Customer Requested Date)
  - Positive values = late delivery
  - Negative values = early delivery
  - Zero = on-time delivery
- Explore strategic account performance, lateness distribution, feature correlations

**Step 2: Model Training (Notebook 02)**
- AutoML trains two model types:
  - **Regression**: Predict exact days late (AGE_REQ_DATE)
  - **Classification**: Predict late vs on-time (binary + multi-class buckets)
- Features: Plant, STRATEGIC_ACCOUNT, EWM_CARRIER_CODE, Shipping Point, Brand, Channel, Product Category
- Models registered in MLflow for tracking and versioning
- Training takes ~3-5 minutes with 180-second AutoML budget

**Step 3: Generate Predictions (Notebook 03)**
- Load trained regression model from MLflow registry
- Query **open deliveries** (GI Date blank) - these are awaiting completion
- Generate predictions with enhancements:
  - `predicted_days_late`: Regression output (exact days)
  - `is_late`: Binary classification (1 = late, 0 = on-time)
  - `lateness_bucket`: Category (0-2, 3-5, 6-9, 10+ days late)
  - `risk_score`: Normalized 0-1 score
  - `high_priority`: Flag for strategic accounts predicted late
- Write to Lakehouse table: **late_delivery_predictions**
- Predictions appear automatically in Power BI via Direct Lake

**Step 4: Power BI Dashboards**
- Import DAX measures from `powerbi/dax/measures_late_delivery.dax`
- Build visualizations:
  - **Executive Overview**: Late delivery rate, high-priority count, value at risk
  - **Lateness Buckets**: Distribution across 0-2, 3-5, 6-9, 10+ days
  - **Strategic Accounts**: Strategic vs non-strategic late rate comparison
  - **Carrier Performance**: Late rate by carrier with drill-through capabilities
  - **Plant Analysis**: Identify best/worst performing plants
  - **Current Day Monitoring**: Real-time tracking of today's at-risk deliveries

---

## 📂 Repository Structure

```
aging-prediction-poc/
│
├── 01_semantic_link_data_preparation.ipynb     # Load & explore closed deliveries
├── 02_autoML_training_pipeline.ipynb           # AutoML training (regression + classification)
├── 03_batch_scoring_pipeline.ipynb             # Batch scoring on open deliveries
│
├── notebooks/                                   # Utility modules
│   └── utils/
│       ├── __init__.py
│       ├── preprocessing.py                     # Data validation, cleaning
│       ├── feature_engineering.py               # Feature creation
│       └── model_utils.py                       # Evaluation, MLflow helpers
│
├── powerbi/                                     # Power BI artifacts
│   └── dax/
│       └── measures_late_delivery.dax           # DAX measures (buckets, strategic, KPIs)
│
├── data/                                        # Data documentation
│   ├── sample/                                  # Sample data files (if needed)
│   └── schema/
│       └── aging_schema.json                    # Aging table schema documentation
│
├── ml/                                          # ML documentation
│   └── models/                                  # Model artifacts (MLflow managed)
│
├── config/                                      # Configuration files
│   └── fabric_lakehouse_paths.yaml              # Fabric resource IDs and settings
│
├── diagrams/                                    # Architecture diagrams
│
└── README.md                                    # This file
```

---

## 🚀 Getting Started

### Prerequisites

- **Microsoft Fabric workspace** with:
  - Lakehouse (for storing predictions)
  - Power BI semantic model: **"DLV Aging Columns & Measures"**
  - MLflow experiment and model registry enabled
- **Python 3.10+** (automatically available in Fabric notebooks)
- **Semantic model access**: Permissions to query the Aging table

### Data Requirements

**Semantic Model Structure**:
- Table: **Aging** (single denormalized table with 79+ columns)
- Required columns:
  - **AGE_REQ_DATE**: Target variable (days late/early)
  - **GI Date**: Goods Issue date (null = open delivery)
  - **Plant**: Manufacturing/shipping plant
  - **STRATEGIC_ACCOUNT**: Strategic customer flag ("Yes" or blank)
  - **EWM_CARRIER_CODE**: Carrier identifier
  - **Shipping Point**: Distribution center
  - **Brand**, **Channel**, **Product Category**: Calculated DAX columns
  - **DELIVERY_VALUE_USD**: Shipment value

**Data Filtering Logic**:
- **Training data**: Closed deliveries where `NOT(ISBLANK([GI Date]))`
- **Scoring data**: Open deliveries where `ISBLANK([GI Date])`

### Setup Steps

#### 1. **Upload Notebooks to Fabric**

1. Navigate to your Fabric workspace (Data Engineering or Data Science experience)
2. Upload the three notebooks:
   - `01_semantic_link_data_preparation.ipynb`
   - `02_autoML_training_pipeline.ipynb`
   - `03_batch_scoring_pipeline.ipynb`
3. Attach notebooks to your default Lakehouse

#### 2. **Configure Fabric Resources**

Update `config/fabric_lakehouse_paths.yaml` with your Fabric workspace details:

```yaml
workspace_id: "your-workspace-id"
lakehouse_id: "your-lakehouse-id"
semantic_model_name: "DLV Aging Columns & Measures"
mlflow_experiment_name: "LateDeliveryPrediction"
model_registry_name_regression: "POC-LateDelivery-Regression-AutoML"
model_registry_name_classification: "POC-LateDelivery-Classification-AutoML"
output_table_name: "late_delivery_predictions"
target_column: "AGE_REQ_DATE"
```

#### 3. **Run the ML Pipeline**

Execute notebooks **in order**:

1. **Data Preparation** - `01_semantic_link_data_preparation.ipynb`
   - Install Semantic Link package (`sempy[fabric]`)
   - Connect to "DLV Aging Columns & Measures" semantic model
   - Load **closed deliveries** (GI Date populated) for training
   - Explore AGE_REQ_DATE distribution (target variable)
   - Analyze strategic account performance
   - Visualize late vs on-time deliveries

2. **Model Training** - `02_autoML_training_pipeline.ipynb` (~5 minutes)
   - Execute DAX query to load closed deliveries
   - Prepare features: Plant, STRATEGIC_ACCOUNT, EWM_CARRIER_CODE, Shipping Point, Brand, Channel
   - Target: AGE_REQ_DATE (days late/early)
   - Train regression model (predict exact days late)
   - Train classification model (binary late/on-time)
   - Evaluate performance (MAE, RMSE, R², accuracy, F1-score)
   - Register models in MLflow
   - Generate feature importance visualizations

3. **Batch Scoring** - `03_batch_scoring_pipeline.ipynb`
   - Load regression model from MLflow
   - Query **open deliveries** (GI Date blank) for scoring
   - Generate predictions with enhancements:
     - `predicted_days_late`: Exact days
     - `is_late`: Binary flag (1=late, 0=on-time)
     - `lateness_bucket`: 0-2, 3-5, 6-9, 10+ categories
     - `risk_score`: Normalized 0-1 score
     - `high_priority`: Strategic + late flag
   - Analyze high-risk deliveries (top 10, strategic accounts, brand/channel breakdown)
   - Save to `late_delivery_predictions` table
   - Display summary statistics

#### 4. **Connect Power BI Report**

1. Open your Power BI workspace
2. Connect to the `late_delivery_predictions` table in your Lakehouse (Direct Lake mode)
3. Import DAX measures:
   - Open `powerbi/dax/measures_late_delivery.dax`
   - Copy measures into Power BI Desktop
   - Publish to Fabric workspace
4. Build dashboards using recommended visuals (see below)

---

## 🔄 Retraining the Model

To retrain with new data:

1. **Refresh semantic model**: Ensure "DLV Aging Columns & Measures" has latest data
2. **Run training notebook**: Execute `02_autoML_training_pipeline.ipynb`
3. **Run scoring notebook**: Execute `03_batch_scoring_pipeline.ipynb` for updated predictions
4. **Validate predictions**: Check `late_delivery_predictions` table in Lakehouse

**When to retrain:**
- **Weekly** (recommended for production)
- When adding new plants, carriers, or product lines
- If prediction accuracy drops below acceptable threshold
- After significant process changes (new fulfillment centers, carrier changes)
- When late delivery rate changes significantly

**Monitoring Model Performance**:
- Track MAE (Mean Absolute Error) over time
- Monitor late delivery rate accuracy (predicted vs actual)
- Analyze residuals for systematic bias
- Review high-priority delivery accuracy (strategic accounts)

---

## 📊 Power BI Dashboards

### DAX Measures Provided

Import from `powerbi/dax/measures_late_delivery.dax`:

**Basic Metrics**:
- Total Deliveries, Predicted Late Deliveries, Late Delivery Rate
- Bucket counts (0-2, 3-5, 6-9, 10+ days)
- Strategic vs Non-Strategic late counts
- High Priority Deliveries

**Value Impact**:
- Late Delivery Value ($), On-Time Delivery Value ($)
- Value at Risk - Strategic ($)
- Average Late Delivery Value

**Performance Metrics**:
- Avg Predicted Days Late, Avg Risk Score
- Carrier Late Rate, Plant Late Rate
- Historical Late Rate (actual performance)

**Trend Analysis**:
- Closed Last 14 Days, Late Last 14 Days, Late Rate Last 14 Days

### Recommended Visuals

**Page 1 - Executive Overview**
- **KPI Cards**: Late Delivery Rate, High Priority Deliveries, Value at Risk - Strategic
- **Donut Chart**: Late vs On-Time distribution
- **Column Chart**: Late deliveries by lateness bucket (0-2, 3-5, 6-9, 10+)
- **Line Chart**: Late Rate Last 14 Days (trend)

**Page 2 - Strategic Account Focus**
- **KPI Cards**: Strategic Late Deliveries, Strategic Late Rate
- **Table**: Top 10 high-priority deliveries (strategic + late) with predicted days, value, carrier
- **Bar Chart**: Strategic vs Non-Strategic late rate comparison
- **Scatter Plot**: Predicted Days Late vs Delivery Value (color by strategic account)

**Page 3 - Carrier & Plant Performance**
- **Matrix**: Carrier Late Rate by Plant
- **Bar Chart**: Late deliveries by carrier (sorted by count)
- **Bar Chart**: Late deliveries by plant (sorted by late rate)
- **Slicer**: Filter by carrier, plant, shipping point

**Page 4 - Lateness Bucket Analysis**
- **Stacked Bar Chart**: Lateness buckets by carrier
- **Pie Chart**: % distribution across buckets (Pct Bucket measures)
- **Table**: Deliveries grouped by bucket with avg risk score
- **Filters**: Brand, Channel, Product Category

**Page 5 - Current Day Monitoring**
- **KPI Card**: Today Late Deliveries (filtered by prediction_date = TODAY())
- **Table**: Today's at-risk deliveries with delivery number, customer, carrier, predicted days late
- **Real-time refresh**: Auto-refresh every hour during business hours

---

## 📈 Model Performance

Expected performance (will vary based on your data):

**Regression Model (Predict exact days late)**:
- **MAE:** ~2-4 days (mean absolute error)
- **RMSE:** ~3-6 days (root mean squared error)
- **R² Score:** ~0.65-0.80 (variance explained)

**Classification Model (Predict late vs on-time)**:
- **Accuracy:** ~80-90%
- **Precision (late):** ~75-85% (when predicting late, how often correct)
- **Recall (late):** ~70-85% (of actual late deliveries, how many caught)
- **F1-Score:** ~0.75-0.85

**Business Impact Metrics**:
- **High-Priority Accuracy**: Strategic account late prediction accuracy typically 5-10% higher
- **Value Protected**: Focus on high-risk deliveries can prevent 60-80% of value at risk
- **Proactive Intervention Window**: Average 5-7 days advance notice for late deliveries

*Performance depends on data quality, feature availability, and delivery pattern consistency*

---

## 🔍 Troubleshooting

**Semantic Link connection fails**
- Verify semantic model name: `"DLV Aging Columns & Measures"`
- Check workspace permissions (Contributor or Admin role required)
- Ensure you're running in a Fabric notebook (not local Jupyter)
- Try: `fabric.list_datasets()` to see available models

**DAX query returns no data or errors**
- Check Aging table exists: Open semantic model in Power BI, verify table name
- Test simple query first: `EVALUATE TOPN(10, Aging)`
- Verify GI Date column name: `Aging[GI Date]` vs `Aging[GIDate]`
- Check for calculated columns: Brand, Channel, Credit Status may be DAX calculated

**Target variable (AGE_REQ_DATE) is blank or null**
- AGE_REQ_DATE should be pre-calculated in semantic model (DAX column)
- Verify calculation: `DATEDIFF([Req. Date Header], [GI Date], DAY)`
- For training, filter to closed deliveries: `NOT(ISBLANK(Aging[GI Date]))`
- Check date columns are populated: Req. Date Header and GI Date must exist

**Feature columns missing**
- Verify column names match exactly (case-sensitive)
- Use: `df.columns.tolist()` to see available columns
- Update feature list in `config/fabric_lakehouse_paths.yaml`
- For calculated columns (Brand, Channel), ensure semantic model refreshed

**MLflow model not found**
- Check model registry name: `POC-LateDelivery-Regression-AutoML`
- Verify you've run training notebook (`02_autoML_training_pipeline.ipynb`) first
- Check MLflow experiments in Fabric workspace: View → MLflow
- Model version may be `1` or higher - check notebook output

**Power BI shows no predictions**
- Verify `late_delivery_predictions` table exists in Lakehouse (Tables section)
- Check table was written successfully (notebook output should show row count)
- Refresh Power BI semantic model: Right-click lakehouse → Refresh
- Verify Direct Lake mode: Table should appear instantly without manual import

**Spark write fails with schema mismatch**
- Use `.option("overwriteSchema", "true")` when writing to Lakehouse
- Error often caused by column type changes or renamed columns
- Solution in notebook: `df_predictions.write.mode("overwrite").option("overwriteSchema", "true").saveAsTable("late_delivery_predictions")`

**Classification model accuracy is low**
- Check class imbalance: If 90% deliveries on-time, model may always predict on-time
- Use SMOTE or class weighting in FLAML AutoML settings
- Consider stratified train/test split
- Adjust classification threshold (default 0.5) based on business requirements

**Predictions are all the same value**
- Features may have low variance or be mostly null
- Check feature engineering: Ensure categorical features are encoded
- Verify train/test data comes from same distribution
- Inspect feature importance: Low-importance features may dominate

---

## 📚 Resources

**Microsoft Fabric Documentation**:
- [Semantic Link Overview](https://learn.microsoft.com/fabric/data-science/semantic-link-overview)
- [MLflow in Fabric](https://learn.microsoft.com/fabric/data-science/mlflow-autologging)
- [Direct Lake Mode](https://learn.microsoft.com/power-bi/enterprise/directlake-overview)
- [Fabric Notebooks](https://learn.microsoft.com/fabric/data-engineering/how-to-use-notebook)

**FLAML AutoML**:
- [FLAML Documentation](https://microsoft.github.io/FLAML/)
- [AutoML Examples](https://microsoft.github.io/FLAML/docs/Examples/AutoML-for-XGBoost/)

**Use Case Reference**:
- Original use case document: "Global Operations Insights - Order delivery"
- Business requirement: Predict late deliveries vs Customer Requested Date
- Three scenarios: Historical (2 weeks closed), Predictive (open), Current day

---

## 🎯 Next Steps

Ideas for extending this POC:

**Enhanced Features**:
- Add historical carrier performance metrics (30-day late rate)
- Include order quantity, weight, and distance features
- Add temporal features: day of week, month, holiday proximity
- Customer-level historical late delivery rate

**Advanced Modeling**:
- **Multi-class classification**: Direct prediction of lateness buckets (0-2, 3-5, 6-9, 10+)
- **Time-series forecasting**: Predict late delivery trends by week/month
- **Model explanations**: Add SHAP values to understand drivers for each prediction
- **Ensemble models**: Combine regression + classification for confidence intervals

**Production Deployment**:
- **Automated pipeline**: Schedule daily scoring via Fabric pipeline orchestration
- **Real-time predictions**: Deploy as API endpoint for live scoring during order creation
- **Alerting**: Email/Teams notifications for high-priority late deliveries
- **Feedback loop**: Track actual outcomes, retrain weekly with new data

**Business Integration**:
- **Action recommendations**: Suggest carrier changes for high-risk deliveries
- **Cost-benefit analysis**: Calculate intervention costs vs late delivery penalties
- **SLA monitoring**: Flag deliveries at risk of SLA breach
- **Root cause analysis**: Dashboard showing why deliveries are predicted late (carrier, plant, product mix)

**Power BI Enhancements**:
- **What-if parameters**: Simulate impact of carrier/plant changes
- **Drill-through pages**: Detailed delivery-level analysis
- **Mobile app**: On-the-go monitoring for operations teams
- **Embedded analytics**: Embed predictions in customer portals

---

## 💡 Key Learnings

This POC demonstrates:

✅ **DAX + Python Integration** - Query Power BI semantic models directly from Python  
✅ **Semantic Link Value** - Leverage existing BI models for ML without data duplication  
✅ **Fabric Ecosystem** - Seamless integration of notebooks, Lakehouse, MLflow, and Power BI  
✅ **AutoML Efficiency** - Production-ready models in minutes with FLAML  
✅ **BI Developer Friendly** - Familiar DAX patterns make ML accessible to BI teams  
✅ **Single Table Simplicity** - No complex joins needed with denormalized Aging table  
✅ **Business-Focused ML** - Risk scores, buckets, and priority flags align with operational needs  
✅ **Proactive Operations** - Predict issues before they happen, not just analyze past performance

**Key Technical Decisions**:
- **Target**: AGE_REQ_DATE (customer expectations) vs AGE_CREATEDON (internal metric)
- **Data Split**: Closed deliveries (GI Date populated) for training, Open deliveries (GI Date blank) for scoring
- **Dual Models**: Regression for exact days + Classification for late/on-time binary
- **Strategic Prioritization**: high_priority flag ensures critical customers get attention

---

## 📄 License

This POC is provided as-is for demonstration and educational purposes.

---

**Built with Microsoft Fabric, Semantic Link, FLAML AutoML, and Power BI** 🚀

**Use Case**: Global Operations Insights - Order Delivery Late Prediction
