# Unified Real-Time Crypto Market Intelligence Pipeline

## The Problem

Investment firms operating in cryptocurrency markets face a structural data challenge that traditional financial analytics platforms were never designed to solve.

Price data alone tells you what happened. It does not tell you why, and it certainly does not tell you what is coming next.

When a portfolio manager at a crypto investment firm asks "should we increase our BTC position today?", the answer depends on at least three dimensions at once: the raw price mechanics (where is it trading relative to its moving averages, what is the volatility regime), the information environment (what is the news saying, is sentiment shifting), and the macro market mood (are participants collectively fearful or greedy). These three streams of signal exist in completely different formats, arrive at different frequencies, and have historically lived in different tools with no shared infrastructure connecting them.

The result is that analysts spend hours every morning manually pulling data from separate sources, reconciling date formats, and building one-off Excel models that are stale by the time they reach a decision-maker. By the time a signal is identified, the window to act on it has often already closed.

This project was built to eliminate that gap.

---

## What This Project Does

This pipeline treats a crypto investment firm's analytics stack the way a modern fintech would: as a real-time data product, not a batch reporting job.

We ingest three distinct data sources continuously, process them through a lakehouse medallion architecture, and surface a unified intelligence layer that combines price mechanics, news sentiment, and market psychology signals into a single queryable model. A machine learning layer sits on top to generate next-day price predictions. Everything flows live into Power BI dashboards that a portfolio team can act on the same morning the data arrives.

The architecture is built entirely on Azure and Databricks, designed to run on a student subscription using external locations rather than access keys, and is structured so that each processing layer is independently testable and extensible.

---

## Project Architecture

![Project Architecture](https://github.com/YoussefHamedddd/Depi-Final-Project---Unified-Crypto-Intelligence-Lakehouse/blob/main/docs/Project%20Architecture.jpg?raw=true)

The pipeline runs on Azure Data Lake Storage Gen2 as the storage layer, with Databricks handling all compute, orchestration, and transformation. GitHub is integrated directly with Databricks Repos for version control. Power BI connects via a live DirectQuery stream to the Gold layer.

---

## Data Sources

**Source 1 — Historical Crypto Prices (Batch)**

23 CSV files covering daily OHLCV data (Open, High, Low, Close, Volume, Market Cap) for 23 coins from 2013 through 2021. This is the foundational price dataset that everything else is joined against. Loaded via Spark batch ingestion into the Bronze layer.

**Source 2 — Crypto News with Sentiment (JSON Streaming)**

Unstructured news articles and headlines from sources including CoinDesk, CoinTelegraph, and CryptoNews. Each record contains the article text, publication timestamp, referenced coin symbol, and a pre-labeled sentiment score. Loaded via Autoloader streaming. Natural language processing runs in the Silver transformation to enrich records with sentiment category classification before joining against the price timeline.

**Source 3 — Fear and Greed Index (CSV Streaming)**

Daily market psychology scores (0-100) across five sub-components: volatility, momentum, volume, social media, and market trend. This index captures the collective emotional state of the crypto market on any given day and serves as a macro-level feature in both the Gold unified model and the ML prediction layer. Also loaded via Autoloader streaming.

---

## Pipeline Orchestration

![Databricks Workflow](https://github.com/YoussefHamedddd/Depi-Final-Project---Unified-Crypto-Intelligence-Lakehouse/blob/main/docs/Databricks%20Workflow.png?raw=true)

The full pipeline runs as a single Databricks Workflow job with nine tasks executing in dependency order. The three data sources are ingested in parallel (batch and two streaming paths), transformed independently through their respective Silver notebooks, then converged into the Gold Build step before the ML prediction notebook runs as the final task.

All tasks succeeded in the run shown above. The longest step is ML Price Prediction at 1 minute 49 seconds. Total pipeline runtime is under 5 minutes end to end.

---

## Medallion Architecture

**Bronze Layer** — Raw ingestion with no transformations. Every record is written exactly as received with ingestion timestamps and source file metadata attached. Schema is enforced at read time via explicit schema definitions rather than inference.

**Silver Layer** — Cleaning, type casting, normalization, and enrichment. This is where NLP sentiment classification runs on the news stream, where the Fear and Greed index numeric encoding is generated, and where coin symbols are normalized to match across all three sources. Quality filters drop records with null primary keys or invalid date ranges.

**Gold Layer** — A unified Delta table joining all three Silver tables on trade date and coin symbol. Derived features including daily return percentage, daily price range, unified market signal score, and composite sentiment index are computed here. This table is the single source of truth for both Power BI and the ML model.

---

## Data Model

![Galaxy Schema](https://github.com/YoussefHamedddd/Depi-Final-Project---Unified-Crypto-Intelligence-Lakehouse/blob/main/docs/Galaxy%20Schema.png?raw=true)

The semantic model in Power BI follows a galaxy schema with two fact tables sharing conformed dimensions.

`fact_crypto_daily` is the core analytical fact table. It holds one row per coin per day and carries price metrics, technical indicators (MA7, MA14, MA30), sentiment aggregations, and market flow features.

`fact_price_predictions` holds the ML model output — one prediction per coin per day — with actual versus predicted close price, prediction error percentage, and model name for model comparison.

The two facts share `dim_date`, `dim_coin`, `dim_sentiment`, `dim_ma_signal`, and `dim_flow_signal`. This structure allows cross-fact analysis, for example comparing how well the model predicted prices on days when the Fear and Greed index was in the Extreme Fear zone.

---

## Power BI Dashboards

The report has four pages navigable from the left sidebar.

**Market Overview**

![Market Overview](https://github.com/YoussefHamedddd/Depi-Final-Project---Unified-Crypto-Intelligence-Lakehouse/blob/main/docs/Power%20Bi%20Dashboard%20Pages/1-Market%20Overview.jpg?raw=true)

Top-level summary of the dataset: 23 coins, 112 trillion in total trading volume, average market cap of $15.43B, and a highest recorded price of $64.86K. The bar chart shows market cap concentration — Tether dominates at $121B — and the line chart shows the average close price trend from 2013 through 2021, capturing both the 2018 peak and the 2021 bull run.

**Technical Analysis**

![Technical Analysis](https://github.com/YoussefHamedddd/Depi-Final-Project---Unified-Crypto-Intelligence-Lakehouse/blob/main/docs/Power%20Bi%20Dashboard%20Pages/2-Technical%20Analysis.jpg?raw=true)

Price mechanics for each coin filtered by month. The main chart overlays Close Price against MA7, MA14, and MA30 moving averages across the full date range. Below it, a 7-day volatility chart shows how risk regime varied over time — the 2021 period shows dramatically higher volatility than any prior cycle. The MA7 Signal Distribution bar chart on the right shows that prices spent nearly equal time above and below the 7-day moving average (18.5K vs 18.2K records), with very few days precisely at the average.

**Sentiment and Market Signals**

![Sentiment and Market Signals](https://github.com/YoussefHamedddd/Depi-Final-Project---Unified-Crypto-Intelligence-Lakehouse/blob/main/docs/Power%20Bi%20Dashboard%20Pages/3-Sentiment%20&%20Market%20Signals.jpg?raw=true)

The sentiment page shows the output of the NLP pipeline running on Source 2. Average sentiment score across all articles is 0.09 (slightly positive overall). The pie chart breaks the distribution into 44% neutral, 38% positive, and 18% negative. The Net Exchange Flow Trend line chart at the bottom runs from October through November and shows net inflow/outflow movement around the zero axis, useful for identifying accumulation versus distribution phases.

**ML Price Predictions**

![ML Price Predictions](https://github.com/YoussefHamedddd/Depi-Final-Project---Unified-Crypto-Intelligence-Lakehouse/blob/main/docs/Power%20Bi%20Dashboard%20Pages/4-ML%20Price%20Predictions.jpg?raw=true)

Model performance summary: 96.1% accuracy, 3.9% MAPE, and a mean absolute error of $31.12 across 37K total predictions. The top-20 recent forecasts table shows AAVE predictions for the June 2021 period with prediction errors consistently under 0.30. The scatter plot on the right shows predicted versus actual next-close prices for five coins with strong diagonal alignment indicating model calibration. The area chart at the bottom overlays average actual versus predicted price across the full year, showing the model tracked the price curve closely even through the significant drawdown in the second half of 2021.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Cloud Platform | Microsoft Azure (Student Subscription) |
| Storage | Azure Data Lake Storage Gen2 — External Locations |
| Compute and Orchestration | Azure Databricks (Workflows, Jobs) |
| Batch Ingestion | Apache Spark |
| Streaming Ingestion | Spark Structured Streaming + Autoloader (cloudFiles) |
| Data Format | Delta Lake |
| NLP | PySpark MLlib / custom sentiment classification |
| Machine Learning | Databricks ML (price prediction model) |
| Version Control | GitHub integrated via Databricks Repos |
| Visualization | Power BI (Live Connection to Gold layer) |

---

## Repository Structure
