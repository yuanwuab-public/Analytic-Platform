# Analytic-Platform（GCP, Snowflake, Databricks）

[English](README.md) | [日本語](README.ja.md) | [简体中文](README.zh-CN.md)

図のラベルは、すべての言語版で英語に統一しています。

## 分析目的

| 目的 | 必要なデータ | 手順・実装 | モデル種別 |
|---|---|---|---|
| **顧客セグメンテーション** | 顧客属性、取引履歴 | 1. 顧客の取引データをDWHへ取り込む。<br>2. RFM特徴量（最終購入からの経過期間・購入頻度・購入金額）を集計する。<br>3. K-Meansで顧客を分類する（Vertex AI / Spark MLlib / scikit-learn）。 | クラスタリング（教師なし学習） |
| **顧客生涯価値（CLV）予測** | 取引、購入日、顧客属性 | 1. 顧客別の過去売上を計算する。<br>2. 回帰モデル（線形回帰、XGBoost）で将来売上を予測する。<br>3. 高価値顧客のセグメントを分析層へ出力する。 | 回帰モデル |
| **キャンペーン効果分析** | キャンペーンデータ（表示、クリック、開封） | 1. キャンペーン別のクリック・開封・コンバージョンを集計する。<br>2. CTRとコンバージョン率を計算する。<br>3. セグメント別に効果を比較する。 | — |
| **売上予測** | 過去の売上、販促、季節性 | 1. 店舗・ブランド別、期間別に売上を集計する。<br>2. 販促、季節性、祝日、外部要因の特徴量を作成する。<br>3. 線形回帰または時系列モデル（ARIMA、Prophet、Vertex AI Forecasting）を学習する。<br>4. 予測を分析層へ出力し、ダッシュボードや計画に利用する。 | 回帰・時系列モデル |
| **コンバージョンファネル分析** | Webサイト・アプリの行動データ | 1. セッション、ページ閲覧、コンバージョンイベントを抽出する。<br>2. ページ閲覧 → カート追加 → 購入のファネルを作成する。<br>3. 各段階の離脱率を計算する。 | — |
| **マーケティング反応予測** | キャンペーンと顧客行動 | 1. キャンペーン・顧客属性・行動特徴量を結合する。<br>2. 反応の有無を予測する二値分類モデルを学習する。<br>3. 新しいキャンペーンの反応を予測する。 | 分類モデル |
| **顧客レビュー分析（GCP）** | レビュー本文、評価、商品メタデータ | 1. レビュー本文を収集してDWHへ取り込む。<br>2. テキストを前処理する（トークン化、ストップワード除去、見出し語化）。<br>3. 学習済みモデル（例：Vertex AI Text Sentiment API）で感情分析する。<br>4. 肯定・中立・否定に分類する。<br>5. LDAやBERTopicで「配送の問題」「品質への不満」「機能要望」などのテーマを抽出する。<br>6. 商品・ブランド・セグメント別に感情とテーマを集計し、改善箇所を把握する。<br>7. 分析層やBIダッシュボードへ結果を出力する。 | 自然言語処理（感情分析・トピックモデリング） |

## GCP上の分析基盤

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

## Snowflake上の分析基盤

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

## Databricks上の分析基盤

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
