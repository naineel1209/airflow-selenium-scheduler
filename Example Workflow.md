# 🚀 Quick Overview: How the Selenium + Airflow + Kubernetes Scraper System Works (Single Job Example)

Let’s walk through the **full lifecycle of one job**, say `job_001`, and watch it travel from a YAML config to a Kubernetes pod that scrapes the web like a scalpel.

---

## 🔖 1. job_001: Defined in `jobs.yaml`

```yaml
- job_id: "job_001"
  name: "Scrape Example Site A"
  schedule: "0 * * * *"
  project_path: "site_a"
  browser: "chrome"
  browser_config:
    headless: true
    window_size: "1920,1080"
  output_pvc: "pvc-job-001"
  upload_to_s3: true
  s3_bucket: "scraper-bucket"
````

---

## 🧱 2. Scraper Code Lives in: `project/site_a/`

Contains:

```bash
project/site_a/
├── main.py         # Entry point
└── scrape.py       # Contains actual Selenium scraping logic
```

`main.py` accepts CLI args like `--job-id`, `--browser`, etc., and calls `scrape.py`.

---

## 🐳 3. You Run `build_images.py` to Build the Image

Script reads `jobs.yaml` → loops over each job:

```python
build_cmd = [
    "docker", "build",
    "--build-arg", f"JOB_PATH={project_path}",  # "site_a"
    "-t", image_tag,
    "-f", "docker/Dockerfile",
    "."
]
subprocess.run(build_cmd)
```

### Result:

✅ Custom Docker image `myregistry.com/scraper-job_001:20250729` built
✅ It only contains `/project/site_a`
✅ Image pushed to Docker/ECR/GCR registry
✅ Tag saved in `/tmp/image_tag_job_001`

---

## 🧠 4. Airflow DAG is Auto-Generated for job\_001

The `dags/generate_dags.py` script reads `jobs.yaml` and creates:

```python
dag = DAG("scrape_job_001", schedule_interval="0 * * * *")

task = KubernetesPodOperator(
    task_id='run_scraper',
    image='myregistry.com/scraper-job_001:20250729',
    cmds=["/entrypoint.sh"],
    env_vars={
        'JOB_ID': 'job_001',
        'BROWSER': 'chrome'
    },
    volumes=[{
        'name': 'data',
        'persistentVolumeClaim': {'claimName': 'pvc-job-001'}
    }],
    volume_mounts=[{
        'name': 'data',
        'mountPath': '/data'
    }],
    get_logs=True,
    pool='scraper_pool'
)
```

✅ This DAG appears in Airflow UI
✅ It’s scheduled to run hourly

---

## 🚀 5. Job Runs → Kubernetes Pod is Spawned

Airflow triggers the DAG on schedule:

* Spawns a K8s pod with:

  * **Container A**: `selenium/standalone-chrome`
  * **Container B**: your `scraper-job_001` image
* Both containers share:

  * `/data` via PVC: for artifacts/logs
  * `/dev/shm`: via `emptyDir`, for headless Chrome stability

---

## 🧪 6. Scraper Executes in Container

Inside container B:

```bash
/entrypoint.sh
```

Script waits for Chrome at `localhost:4444`, then runs:

```bash
python /project/main.py \
  --job-id job_001 \
  --config /config/jobs.yaml \
  --browser chrome \
  --output /data/output.json
```

✅ Browser automates
✅ Output stored in `/data/output.json`
✅ Logs are streamed back to Airflow UI

---

## ☁️ 7. Upload to S3 (if enabled)

Airflow task optionally uploads output:

```python
upload = S3CreateObjectOperator(
    task_id='upload_artifacts',
    bucket_name='scraper-bucket',
    key='job_001/{{ ts }}/output.json',
    data="{{ ti.xcom_pull('run_scraper', 'output_json') }}"
)
```

✅ File now stored in S3
✅ Organized by timestamp and job ID

---

## 📬 8. Failure Notifications

If anything fails:

* `task_failure_alert` reads last 50 log lines
* Sends email / Microsoft Teams alert

---

## 📂 9. Logs & Artifacts

* In PVC: `/data/`
* On Airflow host: `<AIRFLOW_HOME>/jobs/job_001/<timestamp>/`
* In S3: `s3://scraper-bucket/job_001/<timestamp>/`

---

## 🧬 SUMMARY — job\_001 Flow

| Step                    | Component                  | Result                                          |
| ----------------------- | -------------------------- | ----------------------------------------------- |
| Define job              | `jobs.yaml`                | Declarative config                              |
| Scraper code            | `project/site_a`           | Modular logic for job                           |
| Build image             | Docker + `build_images.py` | Custom container with only required code        |
| Generate DAG            | `generate_dags.py`         | Dynamically creates Airflow DAG                 |
| Run DAG                 | Airflow                    | Launches Kubernetes pod with Selenium + scraper |
| Execute scraper         | K8s Pod + Chrome           | Fetches data, stores to `/data/output.json`     |
| Upload artifacts (opt.) | S3                         | Archives files for later analysis               |
| Notify on failure       | Airflow callback           | Emails / Teams alerts with log context          |

---

## 🔥 That's the system. One job. One YAML line. One Docker build. One containerized Selenium run. All managed by Airflow + K8s.

Tell me if you want this flow visualized as a Mermaid diagram or turned into a dev onboarding doc with real-time triggers.

