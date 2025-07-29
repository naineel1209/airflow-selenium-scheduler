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
