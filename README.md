<h1 align="center">📊 ForecastPro: Scalable Sales Forecasting via MLOps 🚀</h1>

<p align="center"><strong>Production-grade, cloud-native MLOps system for time series forecasting at scale.</strong></p>

---

## 📚 Table of Contents

- [🔍 Overview](#-overview)
- [✨ Key Features](#-key-features)
- [🛠️ Tools & Technologies](#-tools--technologies)
- [⚙️ Development Environment](#️-development-environment)
- [📈 How It Works](#-how-it-works)
- [🚀 Setup Instructions](#-setup-instructions)
  - [🔧 Docker Compose Setup](#-docker-compose-setup)
  - [☸️ Kubernetes & Helm (Local)](#️-kubernetes--helm-local)
  - [☁️ Kubernetes & Helm (GCP)](#️-kubernetes--helm-gcp)
- [🧹 Cleanup Commands](#-cleanup-commands)
- [📦 MLflow Cloud Config](#-mlflow-cloud-config)
- [📚 References & Resources](#-references--resources)
- [📝 Developer Notes](#-developer-notes)
- [🙋‍♂️ Maintainer](#️-maintainer)
- [⭐ Support](#-support)

---

## 🔍 Overview

**ForecastPro** is a scalable, cloud-native MLOps platform for sales forecasting. Built on modern data and ML infrastructure, it automates ingestion, training, deployment, and monitoring across real-time and batch pipelines.

📦 **Dataset**: [Rossmann Sales Dataset](https://www.kaggle.com/datasets/pratyushakar/rossmann-store-sales)

![System Diagram](./files/sfmlops_software_diagram.png)

---

## ✨ Key Features

- ✅ Batch + real-time forecasting
- ⏱ Weekly auto-retraining using Airflow + Ray
- 🔁 Kafka stream for real-time updates
- ⚙️ Spark Streaming pipeline
- 🧠 MLflow experiment tracking
- 📈 Grafana + Prometheus dashboards
- 🖥️ Streamlit frontend with retrain buttons
- ☁️ Deployable via Docker, Kubernetes, Helm
- 🔄 GitHub Actions for CI/CD pipelines

---

## 🛠️ Tools & Technologies

| Layer             | Stack Components                                                                 |
|-------------------|-----------------------------------------------------------------------------------|
| Orchestration     | Apache Airflow, Spark Streaming                                                   |
| Model Training    | Prophet, Ray, MLflow                                                              |
| Streaming         | Apache Kafka                                                                      |
| Monitoring        | Prometheus, Grafana                                                               |
| Deployment        | FastAPI, Gunicorn, Uvicorn, Nginx                                                 |
| Data Storage      | PostgreSQL, pgAdmin                                                               |
| CI/CD             | GitHub Actions                                                                    |
| Cloud Infrastructure | Google Cloud Platform, Helm, Kubernetes                                       |
| Interface         | Streamlit                                                                         |

---

## ⚙️ Development Environment

| Tool         | Version              |
|--------------|----------------------|
| Docker       | 24.0.6               |
| Kubernetes   | v1.27.2 (Docker Desktop) |
| Helm         | v3.14.3              |

---

## 📈 How It Works

1. **Kafka producer** simulates new sales every 10 seconds.
2. **Airflow Daily DAG** consumes this → Spark cleans → inserts into Postgres.
3. **Airflow Weekly DAG**:
   - Pulls 4-month history
   - Trains 1,000+ Prophet models using Ray
   - Registers models in MLflow
   - Stores forecasts in Postgres
4. **Streamlit UI**:
   - Shows 7-day sales forecast
   - Allows one-click retraining
5. **Prometheus & Grafana** track pipeline and training metrics.
6. **MLflow UI** logs experiments with version control.

---

## 🚀 Setup Instructions

### 🔧 Docker Compose Setup

```bash
# 1. Clone the repo
git clone https://github.com/parthgawande/ForecastPro-Scalable-Sales-Forecasting-via-MLOps-.git
cd ForecastPro-Scalable-Sales-Forecasting-via-MLOps-

# 2. (Optional) Build images locally
docker-compose build

# 3. Start the full pipeline
docker-compose -f docker-compose.yml -f docker-compose-airflow.yml up -d

# 4. Access services:
#    - Streamlit: http://localhost:8501
#    - Airflow: http://localhost:8080
#    - MLflow: http://localhost:5000
#    - Grafana: http://localhost:3000
#    - pgAdmin: http://localhost:5050


# How to setup
Prerequisites: Docker, Kubernetes, and Helm

## With Docker Compose
1. *(Optional)* In case you want to build (not pulling images):
   ```
   docker-compose build
   ```
2. ```
   docker-compose -f docker-compose.yml -f docker-compose-airflow.yml up -d
   ```
3. Sometimes it can freeze or fail the first time, especially if your machine is not that high in spec (like mine T_T). But you can wait a second, try the last command again and it should start up fine.
4. That's it!

**Note:** Most of the services' restart is left unspecified, so they won't restart on failures (because sometimes it's quite resource-consuming during development, you see I have a poor laptop lol).

## With Kubernetes/Helm (Local cluster)
The system is quite large and heavy... I recommend running it locally just for setup testing purposes. Then if it works, just go off to the cloud if you want to play around longer OR stick with Docker Compose (it went smoother in my case)
1. Install Helm
   ```
   bash install-helm.sh
   ```
2. Create airflow namespace:
   ```
   kubectl create namespace airflow
   ```
3. Deploy the main chart:
   1. Fetch all dependencies
      ```
      cd sfmlops-helm
      helm dependency build
      ```
   2. ```
      helm -n mlops upgrade --install sfmlops-helm ./ --create-namespace -f values.yaml -f values-ray.yaml
      ```
4. Deploy Kafka:
   1. (1st time only)
      ```
      helm repo add bitnami https://charts.bitnami.com/bitnami
      ```
   2. ```
      helm -n kafka upgrade --install kafka-release oci://registry-1.docker.io/bitnamicharts/kafka --create-namespace --version 23.0.7 -f values-kafka.yaml
      ```
5. Deploy Airflow:
   1. (1st time only)
      ```
      helm repo add apache-airflow https://airflow.apache.org
      ```
   2. ```
      helm -n airflow upgrade --install airflow apache-airflow/airflow --create-namespace --version 1.13.1 -f values-airflow.yaml
      ```
   3. Sometimes, you might get a timeout error from this command (if you do, it means your machine spec is too poor for this system (like mine lol)). It's totally fine. Just keep checking the status with `kubectl`, if all resources start up correctly, go with it otherwise try running the command again.
6. Deploy Prometheus and Grafana:
   1. (1st time only)
      ```
      helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
      ```
   2. ```
      helm -n monitoring upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack  --create-namespace --version 57.2.0 -f values-kube-prometheus.yaml
      ```
   3. Forward port for Grafana:
      ```
      kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
      ```
      *OR* assign `grafana.service.type: LoadBalancer` in `values-kube-prometheus.yaml`
   4. One of the good things about kube-prometheus-stack is that it comes with many pre-installed/pre-configured dashboards for Kubernetes. Feel free to explore!
7. That's it! Enjoy your highly scalable Machine Learning system for Sales forecasting! ;)

**Note:** If you want to change namespace `kafka` and/or release name `kafka-release` of Kafka, please also change them in `values.yaml` and `KAFKA_BOOTSTRAP_SERVER` env var in `values-airflow.yaml`. They are also used in templating.

**Note 2:** In Docker Compose, Ray has already been configured to pull the embedded dashboards from Grafana. But in Kubernetes, this process involves a lot more manual steps. So, I intentionally left it undone for ease of setup of this project. You can follow the guide [here](https://docs.ray.io/en/latest/cluster/kubernetes/k8s-ecosystem/prometheus-grafana.html) if you want to anyway.

## With Kubernetes/Helm (on GCP)
Prerequisites: GKE Cluster (Standard cluster, *NOT* Autopilot), Artifact Registry, Service Usage API, gcloud cli
1. Follow this [Medium blog](https://medium.com/@gravish316/setup-ci-cd-using-github-actions-to-deploy-to-google-kubernetes-engine-ef465a482fd). Instead of using the default Service Account (as done in the blog), I recommend creating a new Service Account with Owner role for a quick and dirty run (but of course, please consult your cloud engineer if you have security concerns).
2. Download your Service Account's JSON key
3. Activate your Service Account:
   ```
   gcloud auth activate-service-account --key-file=<PATH_TO_JSON_KEY>
   ```
4. Connect local kubectl to cloud:
   ```
   gcloud container clusters get-credentials <GKE_CLUSTER_NAME> --zone <GKE_ZONE> --project <PROJECT_NAME>
   ```
5. Now `kubectl` (and `helm`) will work in the context of the GKE environment.
6. Follow the steps in [With Kubernetes/Helm (Local cluster)](#with-kuberneteshelm-local-cluster) section
7. If you face a timeout error when running helm commands for airflow or the system struggles to set up and work correctly, I recommend trying to upgrade your machine type in the cluster.

**Note:** For the machine type of node pool in the GKE cluster, from experiments, `e2-medium` (default) is not quite enough, especially for Airflow and Ray. In my case, I went for `e2-standard-8` with 1 node (explanation on why only 1 node is in [Important note on MLflow on Cloud](#important-note-on-mlflow-on-cloud) section). I also found myself the need to increase the quota for PVC in IAM too.

## Cleanup steps
```
helm uninstall sfmlops-helm -n mlops
helm uninstall kafka-release -n kafka
helm uninstall airflow -n airflow
helm uninstall kube-prometheus-stack -n monitoring
```

## Important note on MLflow on Cloud
In this setting, I set the MLflow's artifact path to point to a local path. Internally, MLflow expects this path to be accessible from both MLflow client and server (honestly, I'm not a fan of this model either). It is meant to be an object storage path like S3 (AWS) or Cloud Storage (GCP). For a full on-premises experience, we can create a Docker volume and mount it to the EXACT same path on both client and server to address this. In a local Kubernetes cluster, we can do the same thing by creating a PVC with `accessModes: ReadWriteOnce` (in `sfmlops-helm/templates/mlflow-pvc.yaml`).

**However** for on-cloud Kubernetes with a typical multi-node cluster, if we want the PVC to be able to read and write across nodes, we need to set `accessModes: ReadWriteMany`. Most cloud providers *DO NOT* support this type of PVC and recommend using centralized storage instead. Therefore, if you want to just try it out and run for fun, you can use this exact setting and create a single-node cluster (which will behave similarly to a local Kubernetes cluster, just on the cloud). For a real production environment, please create a cloud storage bucket, remove `mlflow-pvc.yaml` and its mount paths, and change the artifact path variable `MLFLOW_ARTIFACT_ROOT` in `sfmlops-helm/templates/global-configmap.yaml` to the cloud storage path. Here's the official [doc](https://mlflow.org/docs/latest/tracking/artifacts-stores.html) for more information.

# References / Useful resources
- Ray sample config: https://github.com/ray-project/kuberay/tree/master/ray-operator/config/samples
- Bitnami Kafka Helm: https://github.com/bitnami/charts/tree/main/bitnami/kafka
- Airflow Helm: https://airflow.apache.org/docs/helm-chart/stable/index.html
- Airflow Helm default values.yaml: https://github.com/apache/airflow/blob/main/chart/values.yaml
- dataset: https://www.kaggle.com/datasets/pratyushakar/rossmann-store-sales
- Original Airflow's docker-compose file: https://airflow.apache.org/docs/apache-airflow/2.8.3/docker-compose.yaml

