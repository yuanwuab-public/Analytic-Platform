# Analytic-Platform（GCP, Snowflake, Databricks）

[English](README.md) | [日本語](README.ja.md) | [简体中文](README.zh-CN.md)

所有语言版本的图表均使用英文标签。

## 分析目标

| 目标 | 所需数据 | 步骤与实现 | 模型类型 |
|---|---|---|---|
| **客户分群** | 客户属性、交易历史 | 1. 将客户交易数据加载到 DWH。<br>2. 聚合 RFM 特征（最近购买间隔、购买频率、购买金额）。<br>3. 使用 K-Means 对客户分群（Vertex AI / Spark MLlib / scikit-learn）。 | 聚类模型（无监督学习） |
| **客户生命周期价值（CLV）预测** | 交易、购买日期、客户属性 | 1. 计算每位客户的历史收入。<br>2. 使用回归模型（线性回归、XGBoost）预测未来收入。<br>3. 将高价值客户分群输出到分析层。 | 回归模型 |
| **营销活动效果分析** | 营销活动数据（曝光、点击、打开） | 1. 按活动聚合点击、打开与转化。<br>2. 计算 CTR 与转化率。<br>3. 比较各分群中的活动效果。 | — |
| **销售预测** | 历史销售、促销、季节性 | 1. 按门店、品牌与时间段聚合销售。<br>2. 构建促销、季节性、节假日与外部因素特征。<br>3. 训练线性回归或时间序列模型（ARIMA、Prophet、Vertex AI Forecasting）。<br>4. 将预测输出到分析层，用于仪表板与业务规划。 | 回归 / 时间序列模型 |
| **转化漏斗分析** | 网站 / 应用行为数据 | 1. 提取会话、页面浏览与转化事件。<br>2. 构建页面浏览 → 加入购物车 → 购买的漏斗。<br>3. 计算各阶段的流失率。 | — |
| **营销响应预测** | 营销活动与客户行为 | 1. 合并活动、客户属性与行为特征。<br>2. 训练二分类模型，预测是否响应。<br>3. 预测新活动的客户响应。 | 分类模型 |
| **客户评论分析（GCP）** | 评论文本、评分、商品元数据 | 1. 收集客户评论并加载到 DWH。<br>2. 进行文本预处理（分词、去停用词、词形还原）。<br>3. 使用预训练模型（如 Vertex AI Text Sentiment API）进行情感分析。<br>4. 将评论分为正面、中性、负面。<br>5. 使用 LDA、BERTopic 提取“配送问题”“质量投诉”“功能需求”等主题。<br>6. 按商品、品牌、客户群聚合情感与主题，识别改进方向。<br>7. 将结果输出到分析层或 BI 仪表板。 | NLP（情感分析与主题建模） |

## GCP 分析平台

```mermaid
flowchart TB
    %% === Sources & DWH ===
    subgraph DWH_LAYER["Data Warehouse & Feature Engineering"]
        style DWH_LAYER fill:#e8f4ff,stroke:#333,stroke-width:1px

        DWH_BQ["BigQuery DWH"]
        FEATURE_BQ["Feature Store: Aggregate RFM, CLV features, campaign metrics"]
        REVIEW_TABLE["Customer Review Text Table"]

        DWH_BQ --> FEATURE_BQ
        DWH_BQ --> REVIEW_TABLE

        CP["Campaign Performance\nSQL"]
        DWH_BQ --> CP
        CP --> ANALYTIC_LAYER
    end

    %% === ML Pipelines ===
    subgraph ML_PIPELINE["Containerized ML Pipeline (Vertex AI)"]
        style ML_PIPELINE fill:#fff3e6,stroke:#f3903d,stroke-width:1px

        CS["Customer Segmentation\nK-Means clustering"]
        CLV["CLV Prediction\nXGBoost Regression"]
        SP["Sales Prediction\nXGBoost Regression / Time Series"]
        MARKETING["Marketing Response Prediction\nBinary classification"]
        FUNNEL["Conversion Funnel Analysis\nSQL aggregation"]

        REVIEW_ANALYSIS["Customer Review Comment Analysis\n(Vertex AI Text API: Sentiment + Topics)"]

        FEATURE_BQ --> CS
        FEATURE_BQ --> CLV
        FEATURE_BQ --> SP
        FEATURE_BQ --> MARKETING
        FEATURE_BQ --> FUNNEL

        REVIEW_TABLE --> REVIEW_ANALYSIS

        CS --> ANALYTIC_LAYER
        CLV --> ANALYTIC_LAYER
        SP --> ANALYTIC_LAYER
        MARKETING --> ANALYTIC_LAYER
        FUNNEL --> ANALYTIC_LAYER
        REVIEW_ANALYSIS --> ANALYTIC_LAYER
    end

    %% === Analytic Layer ===
    subgraph ANALYTIC_LAYER["Analytic Layer / Dashboard"]
        style ANALYTIC_LAYER fill:#e6fff3,stroke:#33aa66,stroke-width:1px
        DASHBOARD["Visualize segments, CLV, campaign performance, funnel, review sentiment & topics"]
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
    REVIEW_ANALYSIS --> SDK

    %% === CI/CD Flows ===
    DOCKER --> ARTIFACT --> CI_CD --> ML_PIPELINE
```

## Snowflake 分析平台

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

## Databricks 分析平台

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
