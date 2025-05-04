# ML Infrastructure:

I have chosen to do a Platform for ML/AI teams so that they can experiment & develop their projects.

## Functional Requirements:
* Support Notebook-based experimentation
* Support defining and running ML pipelines
* ML teams must be able to train models using both local and infrastructure compute.
* Model versioning and tracking
* Support deploying models to production as APIs/Services
* ML teams must have access to curated datasets from a feature store.
* User and access management
* The platform must track model performance and operational metrics.
* Handle users with no prior data

## Non-Functional Requirements:
* The system must support multiple teams and scale horizontally.
* Pipelines and services must recover gracefully from failures.
* Data must be encrypted in transit and at rest; access must be auditable.
* The system must offer a simple and unified UI for ML workflows.
* Should use GPU & embeddings to mitigate latency
* Logs and metrics must be centralized and queryable for debugging and analysis.

---

## Background & Assumptions

**Why I've chosen this topic**  
Before design started I have thought about what type of service am I building.
I've compiled this list of options:
1) Real-time w/ watch time as main metric - like YouTube
2) Not real-time w/ watch time as main metric - like Netflix
3) Real-time w/ click as main metric - like Google AD recommendations
4) Undefined about real-time w/ sells as main metric - like Amazon store
5) Real-time w/ watch (listen) time as main metric - like Yandex Music

Given how lots of big tech companies are developing some kind of ML related project, I've decided to try to design such system infra.


We will assume that there are other systems set in stone. Perhaps on cloud or local, so I have chosen only Open-Source technologies.

**Objectives:**  
- Provide integrated notebook environments for rapid prototyping  
- Automate data ingestion, feature engineering, training and deployment pipelines  
- Ensure robust versioning, tracking and audit trails for models and data  
- Enable low-latency access to curated feature sets and embeddings  
- Support secure, scalable model serving and observability  

**What systems I assume already there:**
- A datalake w/ S3 compatible storage.
- Kafka to pull events for event-driven ML services
- Version Control System ie Git
- Redis cache for real-time features (e.g. transaction counts, fraud scores) to support sub-100 ms inference for fraud/cancellation models
- Centralized Secret Store to get Certificates & Secrets with audits.
- Monitoring (e.g. Prometheus, Grafana, etc.) to send from stdout (& get alert if something is going south)
- Other company services/consumers for our models


## Design

---

![C1](https://raw.githubusercontent.com/sakosha/ml-system/refs/heads/master/C1vec.svg)

![C2](https://raw.githubusercontent.com/sakosha/ml-system/refs/heads/master/C2vec.svg)

**What isn't on the diagram?**
- Copying database files from Feast to DataLake for persistence and/or snapshotting it.
- Model promoting in MLFlow - it's via Jenkins & per service as some models value Recall other value Precision etc.

**Why Spark isn't on top of DataLake?**
- Feature engineering can require GPU based acceleration, and I don't want to have idle/unused GPUs on datalake for other services.

---

## Meeting Functional Requirements

| Requirement                                       | Component(s)                           | How we meet it                                                                                                                                                          |
|---------------------------------------------------|----------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Support notebook-based experimentation            | Jupyter                                | Data scientists run experiments in integrated Jupyter notebooks.                                                                                                        |
| Define and run end-to-end ML pipelines            | Jenkins + Metaflow + Spark             | Git commits trigger Jenkins to CD our pipeline, train & model logic.                                                                                                    |
| Train models locally and on shared infrastructure | Local Jupyter + Kubernetes runners     | Experiments can run locally or as containerized jobs on GPU-enabled k8s nodes.                                                                                          |
| Version and track models                          | MLFlow Tracking & Model Registry       | All runs, artifacts and model versions are stored, tracked and browsable in MLFlow.                                                                                     |
| Deploy models as APIs/services                    | MLFlow Model Serving on Kubernetes     | Models are containerized and exposed via RPC endpoints behind the ingress.                                                                                              |
| Provide curated datasets from a feature store     | Feast offline & online stores          | Data engineers populate Feast's online & offline stores. ML engineers take data for training/experimentation from Offline Store. Online Store is for Production Models. |
| Manage users and access control                   | Keycloak                               | Centralized authentication and role-based authorization for all users and services.                                                                                     |
| Track model performance and operational metrics   | Prometheus & Grafana                   | Connect to established monitoring mechanism.                                                                                                                            |
| Handle cold-start users with no prior data        | Redis streams + precomputed embeddings | TLDR: if it's recommendations of some kind -> give results based on embedding from all users in past 3 months. <br/>                                                    |

NOT TLDR: Here it's more about ML culture in company. For instance [Yandex](https://habr.com/ru/companies/yandex/articles/441586/) utilized embedding into vector spaces. And we can make an average (no topic affiliation for first visits) vector spaces to have good results, BUT some tasks usually do not use embedding like Time Series in Predictive Analytics.


## Meeting Non-Functional Requirements

| Requirement                                                       | Mechanism                                 | How we meet it                                                                          |
|-------------------------------------------------------------------|-------------------------------------------|-----------------------------------------------------------------------------------------|
| Support multiple teams and horizontal scaling                     | Kubernetes w/ Stateless-ish podman/docker | All platform components run in k8s with horizontal pod autoscaling (HPA).               |
| Graceful recovery from failures                                   | k8s health checks & retry policies        | Kubernetes restarts failed pods; pipelines auto-retry in Jenkins/Metaflow.              |
| Encrypt data in transit and at rest; audit all access             | TLS + Vault + S3-encryption               | TLS between services with certs from Vault; S3 buckets encrypted; audit logs kept.      |
| Offer a simple, unified UI for ML workflows                       | Ingress-based proxy                       | Single entry point routes to Jupyter, MLFlow UI, and Grafana.                           |
| Leverage GPUs and low-latency embeddings to reduce inference time | GPU nodes & Feast online store            | Jobs run on GPU nodes; Feast online store serves/streams embeddings with low latency.   |
| Logs and metrics                                                  | Prometheus & Grafana                      | All services export to Prometheus their stdout; Grafana provides dashboards and alerts. |

---

## Risks & Potential bottlenecks

| Risks                                    | Description                                                                | Mitigation Strategy                                                                                                                                                        |
|------------------------------------------|----------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Single Points of Failure of dependencies | Vault, GitLab, Keycloak, Data Lake Downtime                                | If these essential systems are down - it's poor luck, but if MLFlow system is alive it can pull a model from its registry to serve existing models until issues resolution |
| Notebook security                        | We expose train environment for faster dev cycle                           | We provide unprivileged images that will serve these notebooks.                                                                                                            |
| Model-serving bottleneck                 | MLFlow Serving pods overwhelmed for peak traffic                           | Configure HPA on CPU/GPU metrics, add request queueing                                                                                                                     |
| Feast pipeline lag                       | Batch feature pipelines on Spark may fall behind, delaying model training  | Enable incremental or streaming feature updates                                                                                                                            |
| Cold-start quality drop                  | Using generic embeddings for new users may reduce recommendation relevance | Continuously update and validate fallback embeddings, implement gradual user profiling to rapidly assign category to them, provide default popular/promoted content        |

## Exam Question Responses

### Scalability  
Our platform runs every component (Spark, Metaflow, MLFlow, Jupyter and serving) as containerized pods on Kubernetes with HPA. Spark executors and feature pipelines scale out based on workload, GPU nodes accommodate heavy training, and the feature store’s online layer serves embeddings in a timely manner.

### Security  
All service-to-service and client communications use TLS with certificates issued by Vault and rotated as per company security policy. Data at rest in S3; Keycloak provides centralized auth and role-based access, and Vault manages secrets with audit logging. <br/>This layered approach ensures end-to-end encryption, least-privilege access and full audit trails.

### Fault Tolerance
Kubernetes liveness/readiness probes automatically restart failed pods, while Jenkins and Metaflow pipelines retry on errors. Critical services like Keycloak and Vault are deployed in HA configurations, and MLFlow Serving pods are auto-scaled and wrapped in circuit breakers to isolate failures—ensuring that a single component outage doesn’t cascade through the system.

Thanks for reading!
<br/>[source](https://github.com/sakosha/ml-system/)