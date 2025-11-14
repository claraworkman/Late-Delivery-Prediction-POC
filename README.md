# 📦 Ship Date Prediction POC

**Predict when orders will ship from distribution centers using Microsoft Fabric, Semantic Link, AutoML, and Power BI**

---

## 📋 Overview

This Proof of Concept (POC) demonstrates a **complete end-to-end machine learning pipeline** for predicting when open orders will ship from distribution centers. Built entirely on Microsoft Fabric using AutoML, MLflow, Semantic Link, Lakehouse tables, and Power BI.

The solution forecasts **GI Date (Goods Issue / Ship Date)** for open deliveries, enabling accurate promise dates to customers, better capacity planning, and proactive scheduling decisions.

### 🚀 What This POC Delivers

✅ **End-to-end ML workflow** - Training → Registry → Scoring → BI reporting  
✅ **Ship date forecasting** - AutoML regression predicting days until DC shipment  
✅ **Lead time bucketing** - 0-2, 3-5, 6-9, 10+ days to ship categories  
✅ **Strategic account prioritization** - High-priority flagging for critical customers  
✅ **Performance dashboards** - Interactive Power BI reports with DAX measures  
✅ **Fully repeatable pattern** - Scalable, Fabric-native architecture  

### Business Value

- **Accurate ship date promises** - Forecast when orders will leave DC for customer communication
- **Capacity planning** - Predict DC workload and shipping volume
- **Strategic account protection** - Prioritize expedited processing for critical customers
- **Root cause analysis** - Analyze lead time patterns by carrier, plant, product, channel
- **Resource optimization** - Schedule labor and logistics resources effectively
- **Operational visibility** - Historical performance vs predicted ship dates
- **Unified platform** - All analytics workloads in one place (Fabric)

---

## 🎯 Use Case: Distribution Center Ship Date Forecasting

**Business Problem**: 
Lack of visibility into when open orders will ship from distribution centers, making it difficult to provide accurate promise dates to customers and optimize DC operations.

**Key Context**:
- Actual customer delivery dates are **NOT tracked** in the system
- Only **GI Date (Goods Issue Date)** from DC is available
- Need to predict **when orders will ship from DC**, not when they'll arrive at customer

**Solution Scenarios**:
1. **Historical Performance** (Last 2-4 weeks): Closed deliveries - analyze actual lead times from order creation to ship
2. **Predictive Insights** (Open deliveries): ML-predicted ship dates with lead time buckets (0-2, 3-5, 6-9, 10+ days)
3. **Current Day Monitoring**: Track orders expected to ship today vs predicted dates

**Key Capabilities**:
- Daily predictions on all open deliveries (not yet shipped)
- Predicted ship date and days-to-ship forecasts
- Risk scores for orders with unusually long lead times
- Strategic account + long lead time = high_priority flag
- Carrier and plant lead time performance benchmarking
- Brand and channel shipping pattern analysis

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

## 🤖 Machine Learning Features

### Target Variable

**DAYS_TO_SHIP** - Days from order creation to actual ship date (GI Date)
- Calculated as: `GI Date - Delivery Created On`
- **Example values:** 3 days, 5 days, 10 days (lead time from creation to ship)
- **Type**: Continuous (regression)
- **Prediction output**: Days until ship → converted to predicted ship date

**Note:** We do NOT predict customer delivery dates (not tracked in system). We predict when orders will ship from the DC.

### Feature Categories

#### 🏭 Location Features
| Feature | Type | Description | Why It Matters |
|---------|------|-------------|----------------|
| **Plant** | Categorical | Manufacturing/shipping plant code | Different plants have different delivery performance |
| **Shipping Point** | Categorical | Distribution center | Proximity to customer, capacity constraints |

#### 🚚 Logistics Features
| Feature | Type | Description | Why It Matters |
|---------|------|-------------|----------------|
| **EWM Carrier Code** | Categorical | Carrier/shipping company | Carrier reliability varies significantly |
| **Shipping Condition** | Categorical | Shipping terms | Different conditions affect delivery speed |
| **EWM Shipping Condition** | Categorical | EWM-specific shipping condition | Warehouse-level shipping requirements |
| **Delivery Type** | Categorical | Type of delivery | Standard vs expedited delivery |

#### 👥 Customer Features  
| Feature | Type | Description | Why It Matters |
|---------|------|-------------|----------------|
| **STRATEGIC_ACCOUNT** | Binary | "Yes" or blank | Strategic customers get priority; used for high_priority flag |
| **Sold To - Key** | Categorical | Customer identifier | Customer-specific delivery patterns |
| **Ship To - Key** | Categorical | Shipping destination | Ship-to location affects delivery time |

#### 📦 Product Features
| Feature | Type | Description | Why It Matters |
|---------|------|-------------|----------------|
| **Brand** | Categorical | Product brand (DAX calculated)<br>• Callaway/Odyssey<br>• Jack Wolfskin<br>• TravisMathew/Cuater<br>• Topgolf<br>• Ogio | Some brands may have longer lead times |
| **Channel** | Categorical | Sales channel (DAX calculated)<br>• E-commerce<br>• Inter-company<br>• Other | Different channels have different SLAs |
| **Product Category** | Categorical | Product classification | Product complexity affects delivery time |
| **Product Type** | Categorical | Type of product | Custom vs standard products |
| **Standard Or Custom** | Categorical | Standard or customized product | Custom products require longer fulfillment |

#### 💰 Financial Features
| Feature | Type | Description | Why It Matters |
|---------|------|-------------|----------------|
| **DELIVERY_VALUE_USD** | Numeric | Shipment value in USD | Higher-value shipments may get priority handling |
| **DELIVERY_QTY** | Numeric | Delivery quantity | Large orders may take longer to fulfill |

#### 📋 Status & Processing Features
| Feature | Type | Description | Why It Matters |
|---------|------|-------------|----------------|
| **Credit Status** | Categorical (DAX calculated)<br>• Credit checked, ok<br>• Credit checked, not ok<br>• Document Released<br>• No Credit check yet | Categorical | Credit holds can delay shipments |
| **Distribution Status** | Categorical (DAX calculated)<br>• Confirmed<br>• Distributed<br>• Planned for Distribution<br>• Relevant | Categorical | Processing stage affects delivery timeline |
| **STATUS** | Categorical | Delivery status | Current delivery state |
| **Delivery Priority** | Categorical | Priority level | Higher priority may ship faster |

#### 📅 Temporal Features
| Feature | Type | Description | Why It Matters |
|---------|------|-------------|----------------|
| **Delivery Created On** | Date | Order creation date | Age of order affects urgency |
| **Req. Date Header** | Date | Customer requested delivery date | Target delivery date |
| **AGE_CREATEDON** | Numeric | Days since order creation | Older orders may have delays |

### Feature Engineering

**Encoding Categorical Variables:**
- **Label Encoding**: Used for high-cardinality features (Plant, Shipping Point, Carrier, Customer IDs)
- **Category Codes**: Convert categorical strings to numeric codes for ML algorithms
- **Handling Missing Values**: Fill with "Unknown" before encoding for categorical, median for numeric

**Temporal Features Created:**
```python
df['created_dayofweek'] = pd.to_datetime(df['Delivery Created On']).dt.dayofweek
df['created_month'] = pd.to_datetime(df['Delivery Created On']).dt.month
```

**Features Actually Used in Notebooks** (from `02_autoML_training_pipeline.ipynb`):
- Plant, Shipping Point, EWM Carrier Code
- Brand, Channel, Product Category, Product Type, Standard Or Custom
- STRATEGIC_ACCOUNT, Sold To - Key
- Delivery Type, DELIVERY_QTY, DELIVERY_VALUE_USD, Delivery Priority, Shipping Condition
- Credit Status, Distribution Status, STATUS
- Temporal: created_dayofweek, created_month (if available)

**Potential Enhancements** (see Next Steps):
- Historical carrier performance (30-day late rate by carrier)
- Customer-level historical late delivery rate
- Distance calculations (ship-from to ship-to)
- Holiday proximity indicators
- Seasonal patterns (Q4 peak season)

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
- Load closed deliveries (GI Date populated = already shipped)
- Calculate target variable: DAYS_TO_SHIP = GI Date - Delivery Created On
- Validate data quality (check for required fields: GI Date, Delivery Created On)

**Step 2: Model Training (Notebook 02)**
- Train AutoML regression model to predict days-to-ship
- Features: Plant, Carrier, Brand, Channel, Product, Customer, Created Day/Month, etc.
- Target: DAYS_TO_SHIP (lead time from creation to ship)
- Register model in MLflow registry: "ship_date_predictor"

**Step 3: Batch Scoring (Notebook 03)**
- Load open deliveries (GI Date blank = not yet shipped)
- Load trained model from MLflow
- Predict days-to-ship for each open delivery
- Convert to predicted ship date: Delivery Created On + predicted_days_to_ship
- Create lead time buckets (0-2, 3-5, 6-9, 10+ days)
- Flag high-priority: Strategic accounts with unusually long lead times
- Save predictions to Lakehouse delta table: `ship_date_predictions`

**Step 4: Power BI Reporting**
- Add `ship_date_predictions` table to semantic model
- Build dashboards showing:
  - Predicted ship dates for open orders
  - Lead time distribution by plant/carrier
  - Orders expected to ship today/this week
  - Historical lead time performance vs predictions

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
│       └── measures_late_delivery.dax           # Legacy DAX measures (archived - no longer used)
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
  - **GI Date**: Goods Issue date (ship date from DC) - null = open delivery
  - **Delivery Created On**: Order creation date - required for calculating DAYS_TO_SHIP
  - **Plant**: Manufacturing/shipping plant
  - **STRATEGIC_ACCOUNT**: Strategic customer flag ("Yes" or blank)
  - **EWM_CARRIER_CODE**: Carrier identifier
  - **Shipping Point**: Distribution center
  - **Brand**, **Channel**, **Product Category**: Calculated DAX columns
  - **DELIVERY_VALUE_USD**: Shipment value

**Data Filtering Logic**:
- **Training data**: Closed deliveries where `NOT(ISBLANK([GI Date])) AND NOT(ISBLANK([Delivery Created On]))`
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
mlflow_experiment_name: "ShipDatePrediction"
model_registry_name: "ship_date_predictor"
output_table_name: "ship_date_predictions"
target_column: "DAYS_TO_SHIP"
```

#### 3. **Run the ML Pipeline**

Execute notebooks **in order**:

1. **Data Preparation** - `01_semantic_link_data_preparation.ipynb`
   - Install Semantic Link package (`sempy[fabric]`)
   - Connect to "DLV Aging Columns & Measures" semantic model
   - Load **closed deliveries** (GI Date and Delivery Created On populated)
   - Calculate DAYS_TO_SHIP target variable
   - Explore lead time distribution
   - Validate required date columns

2. **Model Training** - `02_autoML_training_pipeline.ipynb` (~10 minutes)
   - Execute DAX query to load closed deliveries
   - Calculate DAYS_TO_SHIP = GI Date - Delivery Created On
   - Prepare features: Plant, Carrier, Brand, Channel, Product, Customer
   - Train regression model to predict days from creation to ship
   - Evaluate performance (MAE, RMSE, R²)
   - Register model in MLflow as "ship_date_predictor"
   - Generate feature importance visualizations

3. **Batch Scoring** - `03_batch_scoring_pipeline.ipynb`
   - Load trained model from MLflow
   - Query **open deliveries** (GI Date blank) for scoring
   - Generate predictions:
     - `predicted_days_to_ship`: Predicted lead time in days
     - `predicted_ship_date`: Delivery Created On + predicted days
     - `lead_time_bucket`: 0-2, 3-5, 6-9, 10+ days categories
     - `extended_processing`: Strategic accounts with long lead times
   - Analyze deliveries with extended processing times
   - Save to `ship_date_predictions` table
   - Display summary statistics

#### 4. **Connect Power BI Report**

1. Open your Power BI workspace
2. Connect to the `ship_date_predictions` table in your Lakehouse (Direct Lake mode)
3. Build dashboards for:
   - Predicted ship dates for open orders
   - Lead time distribution by plant/carrier
   - Orders expected to ship today/this week
4. See INTEGRATION_GUIDE.md for detailed Power BI setup

---

## 🔄 Retraining the Model

To retrain with new data:

1. **Refresh semantic model**: Ensure "DLV Aging Columns & Measures" has latest data
2. **Run training notebook**: Execute `02_autoML_training_pipeline.ipynb`
3. **Run scoring notebook**: Execute `03_batch_scoring_pipeline.ipynb` for updated predictions
4. **Validate predictions**: Check `ship_date_predictions` table in Lakehouse

**When to retrain:**
- **Weekly** (recommended for production)
- When adding new plants, carriers, or product lines
- If prediction accuracy drops below acceptable threshold (MAE > 3 days)
- After significant process changes (new fulfillment centers, carrier changes)
- When lead time patterns change significantly

**Monitoring Model Performance**:
- Track MAE (Mean Absolute Error) over time - target < 3 days
- Monitor predicted vs actual ship dates for closed deliveries
- Analyze residuals for systematic bias
- Review prediction accuracy for strategic accounts

---

## 📊 Power BI Integration

See **INTEGRATION_GUIDE.md** for complete Power BI setup instructions, including:
- How to add `ship_date_predictions` table to your semantic model
- DAX measures for ship date forecasting KPIs
- Report templates and best practices
- Refresh scheduling and optimization

**Quick Start - Key Metrics**:
- Total Open Deliveries
- Extended Processing Count (deliveries with long lead times)
- Average Predicted Lead Time
- Strategic Accounts with Extended Processing
- Predicted Ship Date Distribution---

## 📈 Model Performance

Expected performance (will vary based on your data):

**Regression Model (Predict days to ship)**:
- **MAE:** ~2-3 days (mean absolute error)
- **RMSE:** ~3-5 days (root mean squared error)
- **R² Score:** ~0.70-0.85 (variance explained)

Performance improves with:
- More historical data (6+ months recommended)
- Consistent carrier and plant performance
- Clean feature engineering (no null values in key fields)
- Regular retraining (weekly or monthly)

---
- **RMSE:** ~3-5 days (root mean squared error)
- **R² Score:** ~0.65-0.80 (variance explained)

**Business Impact Metrics**:
- **Lead Time Accuracy**: Within ±3 days for 70-80% of deliveries
- **Strategic Account Accuracy**: Typically 5-10% better than average
- **Forecasting Window**: Average 3-10 days advance notice of ship date
- **Extended Processing Detection**: 75-85% accuracy for orders with long lead times

*Performance depends on data quality, feature availability, and shipping pattern consistency*

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
- Verify column names: `Aging[GI Date]`, `Aging[Delivery Created On]`
- Check for calculated columns: Brand, Channel, Credit Status may be DAX calculated

**Target variable (DAYS_TO_SHIP) calculation fails**
- DAYS_TO_SHIP is calculated in notebook: `GI Date - Delivery Created On`
- Verify both date columns exist and are populated in closed deliveries
- Check date format: Should be datetime, not string
- For training, filter requires BOTH dates: `NOT(ISBLANK([GI Date])) AND NOT(ISBLANK([Delivery Created On]))`

**Feature columns missing**
- Verify column names match exactly (case-sensitive)
- Use: `df.columns.tolist()` to see available columns
- Update feature list in Notebook 02 if columns are different
- For calculated columns (Brand, Channel), ensure semantic model refreshed

**MLflow model not found**
- Check model registry name: `ship_date_predictor`
- Verify you've run training notebook (`02_autoML_training_pipeline.ipynb`) first
- Check MLflow experiments in Fabric workspace: View → MLflow
- Model version may be `1` or higher - check notebook output

**Power BI shows no predictions**
- Verify `ship_date_predictions` table exists in Lakehouse (Tables section)
- Check table was written successfully (notebook output should show row count)
- Refresh Power BI semantic model: Right-click lakehouse → Refresh
- Verify Direct Lake mode: Table should appear instantly without manual import

**Spark write fails with schema mismatch**
- Use `.option("overwriteSchema", "true")` when writing to Lakehouse
- Error often caused by column type changes or renamed columns
- Solution in notebook: `df_predictions.write.mode("overwrite").option("overwriteSchema", "true").saveAsTable("ship_date_predictions")`

**Regression model accuracy is low**
- Check data quality: Ensure ship dates (GI Date) and created dates are valid
- Verify feature engineering: Categorical encoding, temporal features properly calculated
- Consider stratified train/test split by ship date range
- Review feature importance: Remove low-value features that add noise

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
- Business requirement: Predict ship date lead time from distribution center
- Key constraint: Customer delivery dates NOT available - only DC ship dates (GI Date) tracked
- Three scenarios: Historical (closed deliveries), Predictive (open orders), Current day

---

## 🎯 Next Steps

Ideas for extending this POC:

**Enhanced Features**:
- Add historical carrier performance metrics (30-day avg ship time)
- Include order quantity, weight, and distance features
- Add temporal features: day of week, month, holiday proximity
- Customer-level historical ship time averages

**Advanced Modeling**:
- **Multi-class classification**: Predict ship time buckets (1-2 days, 3-5 days, 6-10 days, 10+ days)
- **Time-series forecasting**: Predict ship time trends by week/month/season
- **Model explanations**: Add SHAP values to understand lead time drivers for each prediction
- **Ensemble models**: Combine multiple regression models for confidence intervals

**Production Deployment**:
- **Automated pipeline**: Schedule daily scoring via Fabric pipeline orchestration
- **Real-time predictions**: Deploy as API endpoint for live scoring during order creation
- **Alerting**: Email/Teams notifications for orders with long predicted ship times
- **Feedback loop**: Track actual ship dates, retrain weekly with new data

**Business Integration**:
- **Action recommendations**: Suggest carrier/plant changes for orders with long predicted ship times
- **Cost-benefit analysis**: Calculate expediting costs vs standard shipping
- **SLA monitoring**: Flag orders at risk of missing target ship dates
- **Root cause analysis**: Dashboard showing drivers of long ship times (carrier, plant, product mix, seasonality)

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
- **Target**: DAYS_TO_SHIP (creation to DC ship date) - only metric available in data
- **Data Split**: Closed deliveries (GI Date populated) for training, Open deliveries (GI Date blank) for scoring
- **Regression Model**: Predict exact days to ship from order creation
- **Business Constraint**: Customer delivery dates NOT tracked - only DC ship dates (GI Date) available

---

## 📄 License

This POC is provided as-is for demonstration and educational purposes.

---

**Built with Microsoft Fabric, Semantic Link, FLAML AutoML, and Power BI** 🚀

**Use Case**: Global Operations Insights - Order Delivery Late Prediction
