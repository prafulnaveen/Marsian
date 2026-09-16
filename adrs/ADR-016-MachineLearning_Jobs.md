# ADR-016 — Choice of Google Vertex AI for ML Model Training and Batch Predictions

## Context

The estate system requires machine learning for predictive analytics and optimization:

**ML Use Cases:**
- **Predictive Maintenance**: Predict ride equipment failure based on vibration/temperature trends
- **Anomaly Detection**: Identify unusual sensor readings (animal behavior deviation, ride performance drop)
- **Visitor Behavior Prediction**: Forecast peak hours, popular attractions to optimize staffing
- **Dynamic Pricing**: Adjust ticket prices based on demand, occupancy, seasonality
- **Animal Health Prediction**: Predict health issues before symptoms manifest (early intervention)
- **Churn Prediction**: Identify at-risk visitors, loyalty program members likely to cancel
- **Recommendation Engine**: Suggest rides/activities based on visitor preferences and history

**Model Types Required:**
- **Supervised Learning**: Regression (maintenance prediction, price optimization), Classification (anomaly detection, churn prediction)
- **Time-Series Models**: LSTM/Prophet for forecasting peak hours, visitor trends
- **AutoML**: Quick model development for non-critical use cases (AutoML classification for animal health)
- **Batch Predictions**: Score millions of historical records (visitors, rides) for analytics

**Training Data Sources:**
- **BigQuery**: Historical sensor data (temperature, vibration), visitor activity, transaction records
- **Cloud Storage**: Large datasets, model artifacts
- **Pub/Sub**: Real-time event streams for online learning

**Training Requirements:**
- **Scalable Training**: Models require GPU/TPU acceleration for large datasets (millions of rows)
- **Hyperparameter Tuning**: Automated tuning to find optimal model parameters
- **Experiment Tracking**: Track model versions, performance metrics, hyperparameters
- **Model Registry**: Store trained models, version control, easy deployment
- **Batch Predictions**: Process millions of records (weekly predictions on visitor/ride data)
- **Online Predictions**: Score individual requests in real-time (<100ms for visitor recommendations)

**Operational Requirements:**
- **Scheduled Retraining**: Retrain models weekly/monthly as new data arrives (model drift mitigation)
- **Model Monitoring**: Track model performance degradation over time
- **A/B Testing**: Compare model versions before full rollout
- **Audit Trail**: Track which model version predicted what (compliance, debugging)

**Scale Requirements:**
- **Training Dataset**: 10-50 GB (historical sensor data, visitor records, transaction history)
- **Prediction Volume**: 100-1,000 predictions per second (visitor recommendations, pricing decisions)
- **Model Frequency**: Weekly retraining, real-time scoring
- **Team Size**: 2-3 data scientists; need low operational burden

## Decision

**We will use Google Vertex AI as the primary ML platform for model training, hyperparameter tuning, batch/online predictions, and model deployment.**

Vertex AI provides a unified, managed ML environment that scales automatically, integrates natively with BigQuery and GCP services, handles model lifecycle management, and includes AutoML for rapid development—ideal for this workload.

## Key Differentiators

- **Unified ML Platform**  
  Vertex AI consolidates training, tuning, prediction, and monitoring in single platform. No multi-tool integration. Compared to legacy AI Platform (separated services), Vertex AI is modern, integrated solution.

- **BigQuery Integration**  
  Training data lives in BigQuery; Vertex AI reads directly without export. Predictions written back to BigQuery. Eliminates data movement, reduces latency.

- **AutoML Capabilities**  
  AutoML for tabular data (classification, regression), vision, NLP. Automatically trains multiple model architectures, selects best. Faster than manual model development.

- **Hyperparameter Tuning**  
  Automatic tuning finds optimal hyperparameters. Saves weeks of manual experimentation. Parallel trials test multiple configurations simultaneously.

- **GPUs and TPUs**  
  Support for GPUs (NVIDIA A100, V100) and TPUs for accelerated training. TensorFlow, PyTorch, scikit-learn all supported.

- **Custom Training**  
  Bring your own code (Python, R, anything in Docker). Vertex AI handles distributed training, resource allocation, logging.

- **Batch Predictions**  
  Process millions of records efficiently. Export predictions to BigQuery, Cloud Storage, Pub/Sub. Weekly model scoring of visitor/ride data.

- **Online Predictions**  
  Real-time model serving via REST/gRPC API. <100ms latency for individual predictions. Auto-scaling handles variable load.

- **Model Registry**  
  Centralized model storage, versioning, lineage tracking. Easy promotion (development → staging → production).

- **Managed Model Monitoring**  
  Track prediction accuracy, data drift, model drift over time. Automatic alerts on degradation.

- **Explainability**  
  Understand why model made prediction (feature importance, Shapley values). Critical for business decisions (pricing, maintenance decisions).

- **Cost-Effective Autoscaling**  
  Serverless prediction endpoints scale from zero to thousands of requests/second. Pay only for resources used.

## Alternatives Considered

- **Self-Managed TensorFlow / PyTorch on GKE**  
  Running ML training and serving on Kubernetes cluster:
  - **Infrastructure Complexity**: Requires managing Kubernetes resources, GPU allocation, service scaling. Operational burden.
  - **Hyperparameter Tuning Manual**: No built-in distributed tuning; requires writing custom code. Time-consuming.
  - **Model Serving Complexity**: Deploying trained models to production requires custom serving infrastructure (TensorFlow Serving, Seldon, etc).
  - **Monitoring Absent**: No built-in model monitoring; must implement custom dashboards. Hard to detect model drift.
  - **Not Recommended**: Managed service (Vertex AI) eliminates infrastructure complexity.

- **Amazon SageMaker**  
  AWS's managed ML platform:
  - **Cloud Lock-in**: Contradicts GCP platform choice (GKE, BigQuery, Cloud SQL). Multi-cloud complexity without benefit.
  - **Data Pipeline**: BigQuery data must export to SageMaker; SageMaker predictions import back to BigQuery. Extra steps, latency.
  - **Feature Store Separate**: SageMaker Feature Store separate from BigQuery; feature definition duplication.
  - **Not GCP Native**: GCP services (Dataflow, BigQuery) not integrated with SageMaker. Custom bridges required.
  - **Not Recommended**: Vertex AI better integrated with GCP ecosystem.

- **Databricks**  
  Unified data and ML platform:
  - **Data Lake Focused**: Databricks designed for data lakes (Lakehouse). Not ideal for transactional data in BigQuery.
  - **Cost**: Databricks pricing based on compute hours; can exceed budget for small models.
  - **Separation from GCP**: Databricks separate platform; additional vendor relationship, additional tool to manage.
  - **Feature Duplication**: Feature definitions must be replicated in Databricks and BigQuery. Complex sync.
  - **Not Recommended**: Vertex AI more directly integrated with BigQuery.

- **Kubeflow**  
  Open-source ML workflows on Kubernetes:
  - **Self-Managed Complexity**: Kubeflow requires Kubernetes cluster management; operational burden similar to self-managed TensorFlow.
  - **Setup Complexity**: Installing, configuring, scaling Kubeflow requires significant engineering effort.
  - **Smaller Ecosystem**: Fewer pre-built components compared to Vertex AI. More custom code required.
  - **Monitoring Limited**: Kubeflow includes basic monitoring; not as comprehensive as Vertex AI's managed monitoring.
  - **Not Recommended for Simplicity**: Vertex AI managed service eliminates infrastructure complexity.

- **Cloud Run + Custom Python Scripts**  
  Scheduling Python ML scripts via Cloud Run + Cloud Scheduler:
  - **Limited Scalability**: Cloud Run has 60-minute timeout; long training jobs cannot complete. Only suitable for quick predictions.
  - **No Model Management**: No centralized model registry, versioning, or lineage tracking. Models scattered across storage.
  - **No Hyperparameter Tuning**: Tuning requires manual script modifications. Time-consuming.
  - **Not Suitable for Training**: Cloud Run designed for short-lived requests; not ML training platform.
  - **Suitable for Simple Predictions**: Cloud Run OK for real-time scoring; not suitable for model training.

- **Spark MLlib (on Dataproc)**  
  Apache Spark's ML library on managed Spark cluster:
  - **Micro-Batch Training**: Spark micro-batching introduces latency; not optimal for real-time ML.
  - **Model Serving Absent**: Spark MLlib trains models; serving models requires separate tool (MLflow, custom serving). Not unified.
  - **Infrastructure Management**: Dataproc cluster requires sizing, scaling; operational overhead vs. Vertex AI's serverless.
  - **Feature Engineering Limited**: Spark SQL/DataFrames OK but not as optimized as BigQuery's columnar analysis.
  - **Not Recommended**: Vertex AI more integrated, simpler than Spark MLlib.

- **Google Cloud AI Platform (Legacy)**  
  Older version of Vertex AI:
  - **Deprecated**: Cloud AI Platform being sunset in favor of Vertex AI. New features only in Vertex AI.
  - **Less Integrated**: Older platform less integrated with BigQuery, Cloud Functions.
  - **Not Recommended**: Use Vertex AI (newer, better integrated).

- **MLflow + Artifact Registry**  
  Open-source experiment tracking + GCP artifact storage:
  - **Training Infrastructure Required**: MLflow only tracks experiments; still need training infrastructure (GKE, Dataproc, etc).
  - **No Hyperparameter Tuning**: MLflow tracks experiments but doesn't automate tuning.
  - **No Managed Serving**: MLflow doesn't manage prediction serving; requires separate deployment.
  - **Complementary Tool**: MLflow can supplement Vertex AI (model registry alternative) but doesn't replace Vertex AI's full capabilities.
  - **Not Recommended as Primary**: Vertex AI more comprehensive.

## Why Vertex AI is Better Than Alternatives

| Criterion | Vertex AI | SageMaker | Databricks | Kubeflow | Self-Managed TF | Cloud Run |
|-----------|-----------|-----------|-----------|----------|-----------------|-----------|
| **Training** | Excellent | Excellent | Very Good | Good | Complex | Not suitable |
| **Hyperparameter Tuning** | Automatic | Automatic | Automatic | Manual | Manual | N/A |
| **Model Serving** | Built-in | Built-in | Limited | Requires tool | Requires tool | Limited (timeout) |
| **BigQuery Integration** | Native | Via export/import | Limited | Via plugin | Via plugin | Via API |
| **AutoML Support** | Excellent | Good | Limited | Limited | Manual | N/A |
| **Batch Predictions** | Excellent | Excellent | Good | Requires code | Requires code | Not suitable |
| **Online Predictions** | Excellent (<100ms) | Excellent | Limited | Requires tool | Requires tool | Acceptable (timeout limit) |
| **Model Monitoring** | Built-in | Built-in | Limited | Limited | Manual | N/A |
| **Explainability** | Built-in | Built-in | Limited | Limited | Manual | N/A |
| **Operational Burden** | Minimal | Minimal | Medium | High | Very High | Low (but limited) |
| **Setup Time** | Hours | Hours | Days | Days | Weeks | Hours |
| **Cost Predictability** | Excellent (pay-per-use) | Excellent (per-use) | Variable | Fixed (infrastructure) | Fixed (infrastructure) | Excellent (per-use) |
| **GCP Integration** | Native | N/A (AWS) | Limited | Via K8s | Limited | Native |

**Why Vertex AI wins:**

1. **Unified Platform**: Single tool for training, tuning, batch/online predictions, monitoring. No multi-tool integration.

2. **BigQuery Native**: Training data in BigQuery; predictions output to BigQuery. No data movement, low latency.

3. **Automatic Tuning**: Hyperparameter tuning automated. Weeks of manual experimentation eliminated.

4. **Serverless Scaling**: Online predictions scale from zero to 1,000+ requests/second automatically. Pay only for predictions served.

5. **AutoML**: Rapidly develop models without data science expertise. Classification, regression, NLP, vision all AutoML-enabled.

6. **Model Monitoring**: Track accuracy, drift, degradation automatically. Alerts on performance regression.

7. **Explainability**: Understand model predictions (feature importance, Shapley values). Essential for business decisions.

8. **GCP Ecosystem**: Seamless integration with Dataflow (training data pipeline), Cloud Functions (trigger predictions), Pub/Sub (events).

## Risks & Trade-offs

| Risk Area | Description | Mitigation |
|-----------|-------------|-----------|
| **Vendor Lock-in (GCP)** | Selecting Vertex AI tightly couples to GCP; migrating to another cloud difficult | Accept GCP commitment (already using GKE, BigQuery, Cloud SQL); GCP market leader in ML, unlikely to change, evaluate multi-cloud only if requirements change |
| **Training Cost Unpredictability** | GPU/TPU training costs variable; large datasets could incur unexpected expenses | Monitor training costs via Cloud Monitoring, set training quotas, use spot instances for non-critical training, estimate costs before tuning |
| **Model Serving Latency** | Online predictions via REST API add <50ms network latency; <100ms target tight for real-time scoring | Accept latency for recommendations/pricing (batch predictions sufficient), test latency in production, use gRPC for lower latency if needed |
| **Data Privacy (BigQuery)** | Training data in BigQuery accessible to Vertex AI; data governance important for sensitive visitor data | Implement BigQuery access controls (IAM roles), encrypt sensitive columns (PII), implement data masking for non-prod training |
| **Model Drift Over Time** | Models degrade as data distribution changes; retraining frequency must be optimized | Implement automatic model monitoring, set up drift detection alerts, retrain models weekly/monthly, A/B test new models before production |
| **AutoML Model Quality** | AutoML generates models automatically but might not be optimal; hand-tuned models sometimes better | Use AutoML for baseline; consider custom TensorFlow models for critical use cases, compare AutoML vs. custom via holdout test set |
| **Hyperparameter Tuning Cost** | Tuning trials run on expensive hardware (GPUs); tuning too many hyperparameters could increase costs | Limit tuning to key hyperparameters, use bayesian search (efficient), set max trials limit, monitor tuning cost budget |
| **Feature Store Complexity** | Managing features (input data for models) separate from BigQuery could create confusion | Use Vertex AI Feature Store integrated with BigQuery, document feature definitions, implement feature versioning |
| **Model Explainability Limitations** | Explainability (Shapley values) requires additional computation; might be too slow for high-throughput prediction | Cache explanations for common inputs, use explainability for offline analysis (not per-request), accept unexplainable predictions for non-critical models |
| **Prediction Endpoint Cost** | Running prediction endpoints 24/7 incurs cost even with zero traffic | Use autoscaling with min replicas = 0 (coldstart risk), batch predictions for non-urgent scoring, implement cost alerts |

## Conclusion

Vertex AI is the optimal ML platform for the estate system because:

1. **Unified ML Environment**: Single platform for training, tuning, batch/online predictions, monitoring. Eliminates multi-tool complexity.

2. **BigQuery Native**: Training data lives in BigQuery; no export/import. Predictions output to BigQuery for analytics. Native integration eliminates data movement.

3. **Automatic Hyperparameter Tuning**: Finds optimal hyperparameters automatically. Weeks of manual experimentation eliminated. Parallel trials test multiple configurations simultaneously.

4. **Serverless Scaling**: Online predictions scale from zero to 1,000+ requests/second. Pay only for predictions served; no idle capacity cost.

5. **AutoML for Rapid Development**: AutoML classification/regression enables data scientists to develop models in days vs. weeks. Non-data-science teams can develop simple models.

6. **Built-In Model Monitoring**: Track accuracy, drift, degradation automatically. Alerts on performance regression trigger retraining.

7. **Explainability & Compliance**: Feature importance, Shapley values enable understanding model decisions. Critical for business decisions (pricing changes, maintenance decisions).

8. **GCP Ecosystem Integration**: Seamless integration with Dataflow, BigQuery, Cloud Functions, Pub/Sub. No custom bridges or data pipeline complexity.

**Recommendation**:
- Deploy Vertex AI for each ML use case:
  - **Predictive Maintenance**: Custom TensorFlow model (vibration/temperature → failure probability) trained on 2-year ride sensor history
  - **Anomaly Detection**: AutoML classification model (normal vs. anomalous sensor readings) trained on labeled sensor data
  - **Visitor Peak Hour Prediction**: Time-series model (LSTM or Prophet) forecasting hourly visitors
  - **Dynamic Pricing**: Regression model (occupancy + demand + date → optimal price) trained on historical transaction data
  - **Churn Prediction**: Classification model (visitor activity → churn probability) trained on loyalty program data
  - **Animal Health**: Time-series model predicting health issues from temperature/activity trends
- Use Vertex AI AutoML for rapid MVP models; upgrade to custom TensorFlow for critical production models
- Implement weekly retraining via Cloud Scheduler + Cloud Functions (triggers Vertex AI retraining job)
- Use BigQuery for feature store; track feature definitions, lineage, versioning
- Deploy prediction endpoints with autoscaling (min replicas = 1 for availability; scale to 10+ during peak)
- Enable model monitoring; set alerts on accuracy <90%, data drift, prediction latency >100ms
- Implement A/B testing for model rollouts (route 10% traffic to new model; monitor performance 1 week before full rollout)
- Use Vertex AI Feature Store to manage training features; document feature definitions, transformations
- For batch predictions (weekly scoring of visitor/ride data): use Vertex AI batch prediction service
- For real-time predictions (pricing decisions, recommendations): use Vertex AI online prediction endpoints
- Monitor prediction costs via Cloud Monitoring; implement cost budgets (max $XXX/day for training, $XXX/day for predictions)
- Document model decisions (why this model, not that one); maintain model registry with deployment dates, performance metrics
- Implement MLOps pipeline: data pipeline → model training → evaluation → deployment → monitoring → retraining
- Test model updates in staging environment; validate accuracy, latency, cost before production deployment
