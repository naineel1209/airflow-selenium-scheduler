<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" class="logo" width="120"/>

# airflow-selenium-scheduler

A production-grade, config-driven framework to run isolated, containerized Selenium web-scraping jobs in Kubernetes, orchestrated by Apache Airflow, with built-in persistence, concurrency control, and failure notifications.

## 🚀 System Overview

Each scraping job is:

- **Declaratively defined** in `jobs.yaml`
- **Automatically translated** into its own Airflow DAG
- **Executed** in a Kubernetes Pod with:
    - **Sidecar**: `selenium/standalone-chrome`
    - **Scraper container**: custom image built per job
- **Writes** output and logs to a PVC (or uploads to S3)
- **Managed** with pools, retries, and alerts


## 📦 Repository Structure

```
.
├── dags/
│   └── generate_dags.py       # Reads jobs.yaml & emits DAGs
├── config/
│   └── jobs.yaml              # Job definitions
├── docker/
│   ├── Dockerfile             # Multi-stage build template
│   └── entrypoint.sh          # Wait-for-Selenium + run scraper
├── build/
│   └── build_images.py        # Builds & pushes per-job images
├── project/
│   ├── main.py                # Scraper entrypoint
│   └── scrape.py              # Scraping logic
├── k8s/
│   ├── storageclass.yaml      # `fast-io` StorageClass
│   └── pvc-template.yaml      # PVC manifest template
└── README.md                  # This documentation
```


## ✅ 1. Prerequisites

- **Kubernetes cluster** (EKS/GKE/AKS or on-prem)
- **Airflow 2.5+** with the `cncf.kubernetes` provider
- **Docker registry** (ECR/GCR/Docker Hub)
- **S3 bucket** (for optional remote logs/artifacts)
- **SMTP / Slack / Teams webhook** (for failure notifications)


## 🧩 2. Job Configuration (`jobs.yaml`)

Define each job:

```yaml
- job_id: "job_001"
  name: "Scrape Example A"
  schedule: "0 * * * *"
  project_path: "/project/site_a"
  browser: "chrome"
  browser_config:
    headless: true
    window_size: "1920,1080"
  output_pvc: "pvc-job-001"
  upload_to_s3: true
  s3_bucket: "scraper-bucket"

- job_id: "job_002"
  name: "Scrape Example B"
  schedule: "@daily"
  project_path: "/project/site_b"
  browser: "edge"
  output_pvc: "pvc-job-002"
  upload_to_s3: false
```


## 🐳 3. Per-Job Build File Generator

Use **`build/build_images.py`** to build and push each job’s Docker image:

```python
import subprocess, yaml, datetime

with open('config/jobs.yaml') as f:
    jobs = yaml.safe_load(f)

for job in jobs:
    date_tag = datetime.date.today().strftime('%Y%m%d')
    image_tag = f"myregistry.com/scraper-{job['job_id']}:{date_tag}"

    subprocess.run([
        "docker", "build",
        "--build-arg", f"JOB_PATH={job['project_path']}",
        "-t", image_tag,
        "-f", "docker/Dockerfile", "."
    ], check=True)

    subprocess.run(["docker", "push", image_tag], check=True)

    with open(f"/tmp/image_tag_{job['job_id']}", "w") as out:
        out.write(image_tag)
```

**Outcome:**

- Custom image `myregistry.com/scraper-job_001:YYYYMMDD` built
- Contains only `/project/site_a` and dependencies
- Pushed to registry
- Tag saved to `/tmp/image_tag_job_001`


## 🐋 4. Dockerfile \& Entrypoint

### `docker/Dockerfile`

```dockerfile
# Stage 1: Install Python deps & copy code
FROM python:3.11-slim AS builder
WORKDIR /app
COPY project /project
RUN pip install --no-cache-dir selenium pyyaml

# Stage 2: Selenium + scraper
FROM selenium/standalone-chrome:latest
USER root

COPY --from=builder /project /project
COPY config/jobs.yaml /config/jobs.yaml
COPY docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

USER seluser
ENTRYPOINT ["/entrypoint.sh"]
```


### `docker/entrypoint.sh`

```bash
#!/usr/bin/env bash

# Wait for Selenium to be ready
until curl -sSf http://localhost:4444/wd/hub/status; do
  sleep 1
done

# Run the scraper
python /project/main.py \
  --job-id "$JOB_ID" \
  --config "/config/jobs.yaml" \
  --browser "$BROWSER" \
  --output "/data/output.json"
```


## 🧠 5. Airflow DAG Generator

### `dags/generate_dags.py`

```python
import yaml
import pendulum
from airflow import DAG
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import KubernetesPodOperator

# Load job configs
with open('/opt/airflow/config/jobs.yaml') as f:
    jobs = yaml.safe_load(f)

for job in jobs:
    # Read the image tag from /tmp
    with open(f"/tmp/image_tag_{job['job_id']}") as f:
        image_tag = f.read().strip()

    dag = DAG(
        dag_id=f"scrape_{job['job_id']}",
        schedule_interval=job['schedule'],
        start_date=pendulum.datetime(2025, 1, 1, tz="UTC"),
        catchup=False,
        max_active_runs=1,
        default_args={'on_failure_callback': task_failure_alert}
    )

    KubernetesPodOperator(
        namespace='airflow',
        task_id='run_scraper',
        image=image_tag,
        cmds=['/entrypoint.sh'],
        env_vars={
            'JOB_ID': job['job_id'],
            'BROWSER': job['browser']
        },
        volumes=[{
            'name': 'data',
            'persistentVolumeClaim': {'claimName': job['output_pvc']}
        }],
        volume_mounts=[{'name': 'data', 'mountPath': '/data'}],
        get_logs=True,
        pool='scraper_pool',
        in_cluster=True,
        is_delete_operator_pod=True,
        dag=dag
    )

    globals()[dag.dag_id] = dag
```


## 📦 6. PVC \& Storage Setup

### `k8s/pvc-template.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-{{ job_id }}
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-io
  resources:
    requests:
      storage: 5Gi
```

- **StorageClass**: SSD or `emptyDir.medium: "Memory"` for `/dev/shm`
- **One PVC per job** ensures isolation


## 📂 7. Artifact \& Log Paths

- **Inside Pod**:
    - `/data/` → mounted PVC for outputs \& logs
- **On Host (Airflow)**:

```
<AIRFLOW_HOME>/jobs/<job_id>/<execution_ts>/
```

- **On S3 (optional)**:

```
s3://<bucket>/<job_id>/<execution_ts>/output.json
```


**Cleanup:** Post-task script removes raw `/data` files, retains logs for upload.

## 🔄 8. Concurrency \& Pools

- **Airflow Pool**: `scraper_pool` with **20 slots**
- Assign `pool='scraper_pool'` in `KubernetesPodOperator` to throttle concurrent pods


## 💥 9. Failure Handling \& Notifications

- **`get_logs=True`** streams stdout/stderr to Airflow logs
- **`task_failure_alert`** callback:

1. Reads last 50 lines from `context['task_instance'].log_filepath`
2. Sends Email (**SMTP**) and Teams (**Webhook**) alerts
- **Retries**: `retries=1` with exponential backoff


## 🛠️ 10. Deployment Steps

1. **Define job** in `config/jobs.yaml`
2. **Implement scraper** in `project/<job_id>/main.py`
3. **Build \& push images**

```bash
python build/build_images.py
```

4. **Create PVCs** in Kubernetes

```bash
kubectl apply -f k8s/pvc-template.yaml
```

5. **Deploy Airflow** (with KubernetesExecutor \& dynamic DAGs)
6. **Monitor** in Airflow UI—DAGs auto-appear and run

## 📎 Final Word

Add a new scraper by updating **`jobs.yaml`** and providing a Python project folder. The framework handles image builds, DAG creation, container orchestration, persistence, and notifications—scaling effortlessly to thousands of jobs.

