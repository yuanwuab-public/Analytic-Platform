# Analytic-Platform（GCP, Snowflake, Databricks）

## Analytics Objectives
| Objective                                    | Data Needed                                   | Steps / Implementation                                                                                                                                                                                                                                                                                                   | Model Type                      |
| -------------------------------------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------- |
| **Customer Segmentation**                    | Customer demographics, transaction history    | 1. Load customer transaction data to DWH.<br>2. Aggregate RFM features (Recency, Frequency, Monetary).<br>3. Cluster customers using K-Means (Vertex AI / Spark MLlib / scikit-learn).                                                                                                                                   | clustering model (unsupervised) |
| **Customer Lifetime Value (CLV) Prediction** | Transactions, purchase dates, demographics    | 1. Compute historical revenue per customer.<br>2. Use regression model (linear regression, XGBoost) to predict future revenue.<br>3. Output high-value segments to analytic layer.                                                                                                                                       | regression model                |
| **Campaign Performance Analysis**            | Campaign dataset (impressions, clicks, opens) | 1. Aggregate clicks, opens, conversions per campaign.<br>2. Compute CTR, conversion rate.<br>3. Compare campaign effectiveness across segments.                                                                                                                                                                          | -                               |
| **Sales Prediction**          | Historical sales, promotions, seasonality     | 1. Aggregate historical sales by shop / brand and time periods.<br>2. Feature engineering: promotions, seasonality, holidays, external factors.<br>3. Train linear regression or time series models (ARIMA, Prophet, Vertex AI Forecasting).<br>4. Output predicted sales to analytic layer for dashboards and planning. | regression / time series model  |
| **Conversion Funnel Analysis**               | Website/app behavior data                     | 1. Extract sessions, page views, conversion events.<br>2. Build funnel: page view → add to cart → purchase.<br>3. Calculate drop-off rates at each stage.                                                                                                                                                                | -                               |
| **Marketing Response Prediction**            | Campaign + customer behavior                  | 1. Merge campaign, demographic, and behavioral features.<br>2. Train binary classification model (responded / not).<br>3. Predict for new campaigns.                                                                                                                                                                     | classification model            |

## Analytic-Platform on GCP
```mermaid
flowchart TB
    %% === Sources & DWH ===
    subgraph DWH_LAYER["Data Warehouse & Feature Engineering"]
        style DWH_LAYER fill:#e8f4ff,stroke:#333,stroke-width:1px

        DWH_BQ["BigQuery DWH"]
        FEATURE_BQ["Feature Store: Aggregate RFM, CLV features, campaign metrics"]
        DWH_BQ --> FEATURE_BQ

        CP["Campaign Performance\nSQL"]
        DWH_BQ --> CP
        CP --> ANALYTIC_LAYER
    end

    %% === ML Pipelines ===
    subgraph ML_PIPELINE["Containerized ML Pipeline (Vertex AI)"]
        style ML_PIPELINE fill:#fff3e6,stroke:#f3903d,stroke-width:1px

        CS["Customer Segmentation\nK-Means clustering"]
        CLV["CLV Prediction\nXGBoost Regression"]
        
        SP["Sales Prediction\nXGBoost Regression， Timeseries"]
        MARKETING["Marketing Response Prediction\nBinary classification"]
        FUNNEL["Conversion Funnel Analysis\nSQL aggregation"]

        FEATURE_BQ --> CS
        FEATURE_BQ --> CLV
        FEATURE_BQ --> SP
        FEATURE_BQ --> MARKETING
        FEATURE_BQ --> FUNNEL

        CS --> ANALYTIC_LAYER
        CLV --> ANALYTIC_LAYER
        sp --> ANALYTIC_LAYER
        MARKETING --> ANALYTIC_LAYER
        FUNNEL --> ANALYTIC_LAYER
    end

    %% === Analytic Layer ===
    subgraph ANALYTIC_LAYER["Analytic Layer / Dashboard"]
        style ANALYTIC_LAYER fill:#e6fff3,stroke:#33aa66,stroke-width:1px
        DASHBOARD["Visualize segments, CLV, campaign performance, funnel"]
    end

    %% === Containerization & CI/CD ===
    subgraph PIPELINE_INFRA["Pipeline Infra"]
        style PIPELINE_INFRA fill:#f0e8ff,stroke:#7b5fff,stroke-width:1px
        DOCKER["Docker Containers"]
        ARTIFACT["Artifact Registry\nStore container images"]
        CI_CD["Cloud Build"]
    end

    %% ==== MONITORING ==== 
    subgraph MONITOR["Monitoring & Automation"]
        style MONITOR fill:#e6fff3,stroke:#33aa66,stroke-width:1px
        SDK["Cloud SDK Collector"]
        PubSubMon["Pub/Sub Alerts"]
        CF["Cloud Function Alert Logic"]
        SNSNotify["SNS Notify Developers"]

        SDK --> PubSubMon --> CF --> SNSNotify
    end

    %% === Data Flows ===
    FEATURE_BQ --> SDK
    CS --> SDK
    CLV --> SDK
    CP --> SDK
    MARKETING --> SDK
    FUNNEL --> SDK

    %% === CI/CD Flows ===
    DOCKER --> ARTIFACT --> CI_CD --> ML_PIPELINE
```


## Analytic-Platform on Snowflake
```mermaid
flowchart LR

%% ===== SOURCE =====
subgraph SRC["SOURCE (GCP)"]
    BQ_DWH["BQ DWH"]
    GCS_RAW["GCS Raw"]
end

%% ===== INGEST =====
subgraph ETL["INGEST (GCP ETL / PIPELINE)"]
    Composer["Composer"]
    Dataflow["Dataflow"]
    SF_LOAD["SF Loader (Snowpipe/Connector)"]
end

BQ_DWH --> Composer --> SF_LOAD
GCS_RAW  --> Dataflow  --> SF_LOAD

%% ===== SNOWFLAKE DWH =====
subgraph SF["SNOWFLAKE DWH"]
    RAW_SF["RAW"]
    FEATURE_SF["FEATURES"]
    ANALYTIC_SF["ANALYTIC"]
end

SF_LOAD --> RAW_SF --> FEATURE_SF

%% ===== SNOWPARK ML =====
subgraph ML["ML (Snowpark)"]
    CS["Segmentation"]
    CLV["CLV"]
    SP["Sales Forecast"]
    MKT["Marketing Response"]
    FUNNEL["Funnel SQL"]
end

FEATURE_SF --> CS --> ANALYTIC_SF
FEATURE_SF --> CLV --> ANALYTIC_SF
FEATURE_SF --> SP --> ANALYTIC_SF
FEATURE_SF --> MKT --> ANALYTIC_SF
FEATURE_SF --> FUNNEL --> ANALYTIC_SF

%% ===== BI =====
subgraph BI["BI"]
    DASH["Tableau / Looker"]
end
ANALYTIC_SF --> DASH

%% ===== CI/CD =====
subgraph CICD["CI/CD"]
    DKR["Docker"]
    REG["Artifact Registry"]
    DEP["Build / GitHub Actions"]
end
DKR --> REG --> DEP --> ML

%% ===== MONITORING =====
subgraph MON["MONITORING"]
    COL["Collector"]
    ERR["Error DB"]
    NOTIFY["Notify"]
end
ML --> COL
FEATURE_SF --> COL
COL --> ERR
COL --> NOTIFY

%% COLORS
style SRC fill:#E3F2FD,stroke:#2196F3
style ETL fill:#E8F5E9,stroke:#4CAF50
style SF fill:#FFF8E1,stroke:#FFC107
style ML fill:#F3E5F5,stroke:#9C27B0
style BI fill:#E0F7FA,stroke:#00ACC1
style CICD fill:#FBE9E7,stroke:#FF7043
style MON fill:#FFEBEE,stroke:#E53935
```

## Analytic-Platform on Databricks
```mermaid
flowchart TB

%% === SOURCE -> INGEST ===
subgraph INGEST["Ingestion / Pipeline to Databricks"]
style INGEST fill:#e1f5fe,stroke:#0288d1,stroke-width:1px,font-size:14px

    BQ["BigQuery DWH on GCP"]
    GCS["GCS Raw Files"]
    DF["Dataflow / Composer ETL"]
    DBXLOAD["Databricks Auto Loader\n(ingest to Delta)"]

    BQ --> DF --> DBXLOAD
    GCS --> DF
end

%% === TABLE LAYERS ===
subgraph DATABRICKS_DWH["Databricks Lakehouse\nBronze / Silver / Gold"]
style DATABRICKS_DWH fill:#fff8e1,stroke:#ffb300,stroke-width:1px,font-size:14px

    BRONZE["Bronze (raw imported)"]
    SILVER["Silver (clean + standard schema)"]
    FEATURES["Gold / Features\nRFM, CLV aggregates,\ncampaign metrics"]

    DBXLOAD --> BRONZE --> SILVER --> FEATURES
end

%% === ML ===
subgraph ML_PIPELINE["ML Pipeline on Databricks (Spark + MLflow)"]
style ML_PIPELINE fill:#efebff,stroke:#5e35b1,stroke-width:1px,font-size:14px

    SEG["Segmentation\nK-Means"]
    CLV["CLV Regression\nXGBoost or Linear Regression"]
    SALES["Sales Prediction\nTimeSeries or Regression"]
    RESP["Marketing Response\nBinary Classification"]
    FUNNEL["Funnel Analysis\nSpark SQL"]

    FEATURES --> SEG --> ANALYTIC
    FEATURES --> CLV --> ANALYTIC
    FEATURES --> SALES --> ANALYTIC
    FEATURES --> RESP --> ANALYTIC
    FEATURES --> FUNNEL --> ANALYTIC
end

%% === OUTPUT ===
subgraph ANALYTIC["Analytic Layer / Dashboard"]
style ANALYTIC fill:#e8f5e9,stroke:#388e3c,stroke-width:1px,font-size:14px

    BI["BI / Databricks SQL Dashboards\n(or PowerBI / Tableau)"]
end

%% === CI/CD ===
subgraph INFRA["Pipeline Infra / Deployment"]
style INFRA fill:#fbe9e7,stroke:#e64a19,stroke-width:1px,font-size:14px

    DOCKER["Docker Images"]
    REG["Artifact Registry"]
    CI_CD["GitHub Actions / Cloud Build\nTrigger Databricks Jobs"]
end

DOCKER --> REG --> CI_CD --> ML_PIPELINE

%% === MONITORING ===
subgraph MON["Monitoring & Alerting"]
style MON fill:#ffebee,stroke:#d32f2f,stroke-width:1px,font-size:14px

    LOGS["Databricks Audit Logs"]
    MLFLOW["MLflow Metrics"]
    ALERT["Alert Rules → Notify (Slack / Teams)"]

    LOGS --> ALERT
    MLFLOW --> ALERT
end

```