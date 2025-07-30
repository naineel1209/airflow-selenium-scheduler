# 🚀 TASKS.md - Complete Project Implementation Plan

## airflow-selenium-scheduler: End-to-End Development Tasks

This document outlines all tasks from basic setup to complete production deployment of the containerized Selenium web-scraping framework with Airflow orchestration and Kubernetes execution.

## 📋 Phase 1: Foundation & Basic Setup

### Task 1.1: Basic Selenium + Docker Integration
**Priority:** HIGH  
**Duration:** 2-3 days  
**Requirements:**
- Docker Desktop installed
- Python 3.11+ with selenium library
- Basic understanding of Docker concepts

**Deliverables:**
1. **Create a basic scraper script** (`project/basic/main.py`):
   ```python
   from selenium import webdriver
   from selenium.webdriver.common.by import By
   import time, os
   
   # Connect to selenium/standalone-chrome
   options = webdriver.ChromeOptions()
   options.add_argument("--headless")
   options.add_argument("--no-sandbox")
   
   driver = webdriver.Remote(
       command_executor="http://localhost:4444/wd/hub",
       options=options
   )
   
   try:
       driver.get("https://httpbin.org/get")
       title = driver.title
       with open("/data/output.txt", "w") as f:
           f.write(f"Title: {title}\nTimestamp: {time.time()}")
       print(f"✅ Scraped: {title}")
   finally:
       driver.quit()
   ```

2. **Test Docker connectivity**:
   ```bash
   # Terminal 1: Start selenium
   docker run -d -p 4444:4444 -v /tmp/selenium:/data selenium/standalone-chrome
   
   # Terminal 2: Run scraper
   python project/basic/main.py
   
   # Verify output
   cat /tmp/selenium/output.txt
   ```

3. **Create entrypoint script** (`docker/entrypoint.sh`):
   ```bash
   #!/bin/bash
   echo "Waiting for Selenium..."
   until curl -sSf http://localhost:4444/wd/hub/status; do sleep 2; done
   echo "✅ Selenium ready"
   python /project/main.py --job-id "$JOB_ID" --output "/data/output.json"
   ```

**Success Criteria:**
- Scraper successfully connects to Docker Selenium
- Output file created with scraped data
- No connection errors or timeouts

### Task 1.2: Multi-Stage Dockerfile Creation
**Priority:** HIGH  
**Duration:** 1-2 days  
**Dependencies:** Task 1.1

**Deliverables:**
1. **Multi-stage Dockerfile** (`docker/Dockerfile`):
   ```dockerfile
   # Stage 1: Python dependencies
   FROM python:3.11-slim AS builder
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY project /project
   
   # Stage 2: Selenium + Scraper
   FROM selenium/standalone-chrome:latest
   USER root
   
   # Install Python and copy from builder
   RUN apt-get update && apt-get install -y python3 python3-pip curl
   COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
   COPY --from=builder /project /project
   COPY docker/entrypoint.sh /entrypoint.sh
   RUN chmod +x /entrypoint.sh
   
   USER seluser
   ENTRYPOINT ["/entrypoint.sh"]
   ```

2. **Requirements file** (`requirements.txt`):
   ```
   selenium==4.15.0
   pyyaml==6.0.1
   requests==2.31.0
   ```

3. **Build and test**:
   ```bash
   docker build -t scraper-test .
   docker run --rm -e JOB_ID=test001 scraper-test
   ```

**Success Criteria:**
- Image builds without errors
- Selenium starts automatically
- Scraper connects and executes successfully
- Total image size  List[Dict]:
       with open(config_path) as f:
           return yaml.safe_load(f)
   
   def validate_job_config(job: Dict) -> bool:
       required_fields = ['job_id', 'project_path', 'browser']
       return all(field in job for field in required_fields)
   ```

**Success Criteria:**
- YAML loads and validates correctly
- Multiple job configurations supported
- Secrets configuration structure defined

### Task 1.4: Secrets Management Integration
**Priority:** MEDIUM  
**Duration:** 2-3 days  
**Dependencies:** Task 1.3

**Deliverables:**
1. **Kubernetes Secrets setup**:
   ```yaml
   # k8s/secrets-template.yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: scraper-secrets-{{ job_id }}
   type: Opaque
   data:
     api_key: {{ api_key_b64 }}
     db_password: {{ db_password_b64 }}
   ```

2. **Secret injection in Dockerfile**:
   ```dockerfile
   # Add to existing Dockerfile
   ENV SECRETS_PATH=/etc/secrets
   VOLUME ["/etc/secrets"]
   ```

3. **Scraper secret access** (`project/common/secrets.py`):
   ```python
   import os
   import json
   
   def get_secret(secret_name: str, default=None):
       """Load secret from mounted Kubernetes secret or env var"""
       # Try mounted secret first
       secret_path = f"/etc/secrets/{secret_name}"
       if os.path.exists(secret_path):
           with open(secret_path) as f:
               return f.read().strip()
       
       # Fallback to environment variable
       return os.getenv(secret_name.upper(), default)
   
   def load_all_secrets() -> dict:
       """Load all available secrets"""
       secrets = {}
       secrets_dir = "/etc/secrets"
       if os.path.exists(secrets_dir):
           for file in os.listdir(secrets_dir):
               with open(os.path.join(secrets_dir, file)) as f:
                   secrets[file] = f.read().strip()
       return secrets
   ```

4. **Test secret mounting**:
   ```bash
   # Create test secret
   kubectl create secret generic test-secret --from-literal=api_key=test123
   
   # Test in pod
   kubectl run test-pod --image=scraper-test --dry-run=client -o yaml > test-pod.yaml
   # Add secret volume mount to yaml
   kubectl apply -f test-pod.yaml
   ```

**Success Criteria:**
- Secrets correctly mounted in containers
- Scraper can access secrets via helper functions
- No secrets leaked in logs or environment

## 📋 Phase 2: Build System & Automation

### Task 2.1: Image Build Pipeline
**Priority:** HIGH  
**Duration:** 3-4 days  
**Dependencies:** Task 1.4

**Deliverables:**
1. **Build script** (`build/build_images.py`):
   ```python
   import subprocess, yaml, hashlib, datetime
   from pathlib import Path
   
   def get_source_hash(project_path: str) -> str:
       """Calculate hash of all Python files in project"""
       files = sorted(Path(project_path).rglob("*.py"))
       hash_content = ""
       for file in files:
           hash_content += f"{file}:{hashlib.sha256(file.read_bytes()).hexdigest()}\n"
       return hashlib.sha256(hash_content.encode()).hexdigest()[:12]
   
   def build_job_image(job: dict, registry: str = "localhost:5000"):
       job_id = job['job_id']
       project_path = job['project_path']
       source_hash = get_source_hash(project_path)
       timestamp = datetime.datetime.now().strftime('%Y%m%d_%H%M%S')
       
       image_tag = f"{registry}/scraper-{job_id}:{timestamp}_{source_hash}"
       
       build_cmd = [
           "docker", "build",
           "--build-arg", f"PROJECT_PATH={project_path}",
           "--build-arg", f"JOB_ID={job_id}",
           "-t", image_tag,
           "-f", "docker/Dockerfile", "."
       ]
       
       subprocess.run(build_cmd, check=True)
       subprocess.run(["docker", "push", image_tag], check=True)
       
       # Save image tag for DAG generator
       with open(f"build/tags/{job_id}.txt", "w") as f:
           f.write(image_tag)
       
       return image_tag
   ```

2. **Enhanced Dockerfile with build args**:
   ```dockerfile
   ARG PROJECT_PATH=/project/default
   ARG JOB_ID=unknown
   
   # Stage 1: Build dependencies
   FROM python:3.11-slim AS builder
   ARG PROJECT_PATH
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY ${PROJECT_PATH} /project
   
   # Stage 2: Runtime
   FROM selenium/standalone-chrome:latest
   ARG JOB_ID
   USER root
   
   # Install Python
   RUN apt-get update && apt-get install -y python3 python3-pip curl
   COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
   COPY --from=builder /project /project
   COPY docker/entrypoint.sh /entrypoint.sh
   RUN chmod +x /entrypoint.sh
   
   # Add labels for tracking
   LABEL job_id=${JOB_ID}
   LABEL build_date=$(date -u +"%Y%m%dT%H%M%SZ")
   
   USER seluser
   ENTRYPOINT ["/entrypoint.sh"]
   ```

3. **Build automation script** (`build/build_all.sh`):
   ```bash
   #!/bin/bash
   set -e
   
   echo "🔨 Building all job images..."
   mkdir -p build/tags
   
   # Build images for all jobs
   python build/build_images.py
   
   echo "✅ All images built successfully"
   ls -la build/tags/
   ```

**Success Criteria:**
- Images build for all jobs in config
- Source code changes trigger rebuilds
- Image tags saved for DAG generation
- Build process completes in  Optional[Dict]:
           """Retrieve job metadata"""
           data = self.redis.get(f"job:{job_id}")
           return json.loads(data) if data else None
       
       def get_all_active_jobs(self) -> Dict[str, Dict]:
           """Get all jobs with status=active"""
           jobs = {}
           for key in self.redis.keys("job:*"):
               job_id = key.decode().replace("job:", "")
               metadata = self.get_job_metadata(job_id)
               if metadata and metadata.get('status') == 'active':
                   jobs[job_id] = metadata
           return jobs
       
       def mark_job_building(self, job_id: str):
           """Mark job as currently building"""
           metadata = self.get_job_metadata(job_id) or {}
           metadata.update({'status': 'building', 'build_start': datetime.now().isoformat()})
           self.redis.set(f"job:{job_id}", json.dumps(metadata))
       
       def mark_job_failed(self, job_id: str, error: str):
           """Mark job build as failed"""
           metadata = self.get_job_metadata(job_id) or {}
           metadata.update({
               'status': 'failed',
               'error': error,
               'failed_at': datetime.now().isoformat()
           })
           self.redis.set(f"job:{job_id}", json.dumps(metadata))
   ```

3. **Integration with build system**:
   ```python
   # Enhanced build_images.py
   from metadata_store import MetadataStore
   
   def build_job_with_metadata(job: dict):
       store = MetadataStore()
       job_id = job['job_id']
       
       try:
           store.mark_job_building(job_id)
           
           # Calculate hashes
           config_hash = hashlib.sha256(json.dumps(job, sort_keys=True).encode()).hexdigest()
           source_hash = get_source_hash(job['project_path'])
           
           # Build image
           image_tag = build_job_image(job)
           
           # Save metadata
           store.save_job_metadata(job_id, {
               'job_id': job_id,
               'image_tag': image_tag,
               'config_hash': config_hash,
               'source_hash': source_hash,
               'project_path': job['project_path'],
               'browser': job['browser'],
               'schedule': job['schedule']
           })
           
       except Exception as e:
           store.mark_job_failed(job_id, str(e))
           raise
   ```

**Success Criteria:**
- Redis deployed and accessible
- Metadata correctly stored and retrieved
- Build status tracking works
- DAG generator can read metadata

### Task 2.3: Git Integration & Version Control
**Priority:** MEDIUM  
**Duration:** 1-2 days  
**Dependencies:** Task 2.2

**Deliverables:**
1. **Repository structure**:
   ```
   airflow-selenium-scheduler/
   ├── .gitignore
   ├── .github/
   │   └── workflows/
   │       └── build-images.yml
   ├── README.md
   ├── config/
   │   └── jobs.yaml
   ├── build/
   │   ├── build_images.py
   │   ├── metadata_store.py
   │   └── tags/
   ├── dags/
   │   └── generate_dags.py
   ├── docker/
   │   ├── Dockerfile
   │   └── entrypoint.sh
   ├── k8s/
   │   ├── redis.yaml
   │   └── pvc-template.yaml
   └── project/
       ├── httpbin/
       └── news/
   ```

2. **Git ignore** (`.gitignore`):
   ```
   # Build artifacts
   build/tags/
   __pycache__/
   *.pyc
   
   # Secrets and env files
   .env
   secrets/
   
   # IDE files
   .vscode/
   .idea/
   
   # OS files
   .DS_Store
   Thumbs.db
   
   # Airflow
   airflow.db
   airflow-webserver.pid
   logs/
   ```

3. **GitHub Actions workflow** (`.github/workflows/build-images.yml`):
   ```yaml
   name: Build Docker Images
   
   on:
     push:
       paths:
         - 'config/jobs.yaml'
         - 'project/**'
         - 'docker/**'
       branches: [main]
   
   jobs:
     build:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v3
         
         - name: Set up Python
           uses: actions/setup-python@v3
           with:
             python-version: '3.11'
         
         - name: Install dependencies
           run: pip install pyyaml redis
         
         - name: Login to Registry
           run: echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
         
         - name: Build images
           env:
             REDIS_URL: ${{ secrets.REDIS_URL }}
           run: python build/build_images.py
   ```

**Success Criteria:**
- Repository properly structured
- Git hooks prevent sensitive data commits
- CI/CD triggers on relevant changes
- Build artifacts excluded from version control

## 📋 Phase 3: Airflow Integration

### Task 3.1: Dynamic DAG Generator
**Priority:** HIGH  
**Duration:** 3-4 days  
**Dependencies:** Task 2.2

**Deliverables:**
1. **DAG generator** (`dags/generate_dags.py`):
   ```python
   import yaml, json
   from airflow import DAG
   from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import KubernetesPodOperator
   from airflow.providers.amazon.aws.operators.s3 import S3CreateObjectOperator
   from datetime import datetime, timedelta
   import pendulum
   from build.metadata_store import MetadataStore
   
   # Initialize metadata store
   store = MetadataStore()
   active_jobs = store.get_all_active_jobs()
   
   def task_failure_alert(context):
       """Handle task failures"""
       ti = context['task_instance']
       job_id = context['dag'].tags[0] if context['dag'].tags else 'unknown'
       
       error_msg = f"""
       🚨 Scraping Job Failed
       Job ID: {job_id}
       Task: {ti.task_id}
       DAG: {ti.dag_id}
       Execution Date: {context['execution_date']}
       Log URL: {ti.log_url}
       """
       
       # Send to logging system or notification service
       print(error_msg)
   
   # Generate DAGs for all active jobs
   for job_id, metadata in active_jobs.items():
       if metadata.get('status') != 'active':
           continue
       
       # Load job config (could be cached in metadata)
       with open('config/jobs.yaml') as f:
           jobs_config = yaml.safe_load(f)
       
       job_config = next((j for j in jobs_config if j['job_id'] == job_id), None)
       if not job_config:
           continue
       
       dag_id = f"scrape_{job_id}"
       image_tag = metadata['image_tag']
       
       default_args = {
           'owner': 'scraper-team',
           'depends_on_past': False,
           'start_date': pendulum.datetime(2025, 1, 1, tz="UTC"),
           'email_on_failure': True,
           'email_on_retry': False,
           'retries': 1,
           'retry_delay': timedelta(minutes=5),
           'on_failure_callback': task_failure_alert
       }
       
       dag = DAG(
           dag_id=dag_id,
           default_args=default_args,
           description=f"Scraping job: {job_config.get('name', job_id)}",
           schedule_interval=job_config.get('schedule', '@daily'),
           max_active_runs=1,
           catchup=False,
           tags=[job_id, 'scraping']
       )
       
       # Create Kubernetes volumes
       volumes = [{
           'name': 'data',
           'persistentVolumeClaim': {'claimName': job_config['output_pvc']}
       }]
       
       volume_mounts = [{'name': 'data', 'mountPath': '/data'}]
       
       # Add secrets if specified
       if 'secrets' in job_config:
           for secret in job_config['secrets']:
               volumes.append({
                   'name': f"secret-{secret['name'].lower()}",
                   'secret': {'secretName': secret['secret_name']}
               })
               volume_mounts.append({
                   'name': f"secret-{secret['name'].lower()}",
                   'mountPath': f"/etc/secrets/{secret['name']}",
                   'readOnly': True
               })
       
       # Main scraping task
       scraper_task = KubernetesPodOperator(
           namespace='airflow',
           task_id='run_scraper',
           name=f'scraper-{job_id}-{{{{ ts_nodash }}}}',
           image=image_tag,
           cmds=['/entrypoint.sh'],
           env_vars={
               'JOB_ID': job_id,
               'BROWSER': job_config['browser'],
               'LOG_LEVEL': 'INFO'
           },
           volumes=volumes,
           volume_mounts=volume_mounts,
           resources={
               'limit_cpu': '1000m',
               'limit_memory': '2Gi',
               'request_cpu': '500m',
               'request_memory': '1Gi'
           },
           get_logs=True,
           pool='scraper_pool',
           in_cluster=True,
           is_delete_operator_pod=True,
           startup_timeout_seconds=300,
           dag=dag
       )
       
       # Optional S3 upload task
       if job_config.get('upload_to_s3'):
           upload_task = S3CreateObjectOperator(
               task_id='upload_to_s3',
               bucket_name=job_config['s3_bucket'],
               key=f"{job_id}/{{{{ ds }}}}/output.json",
               data="{{ ti.xcom_pull('run_scraper') }}",
               dag=dag
           )
           scraper_task >> upload_task
       
       # Register DAG globally
       globals()[dag_id] = dag
   
   print(f"Generated {len(active_jobs)} scraping DAGs")
   ```

2. **Pool configuration** (`k8s/airflow-pools.yaml`):
   ```yaml
   # This would be applied via Airflow CLI or API
   # airflow pools set scraper_pool 20 "Selenium scraping jobs pool"
   ```

**Success Criteria:**
- DAGs generated for all active jobs in metadata store
- Each DAG uses correct image tag from metadata
- Proper resource limits and pools configured
- Failure callbacks working

### Task 3.2: Kubernetes Pod Templates
**Priority:** MEDIUM  
**Duration:** 2 days  
**Dependencies:** Task 3.1

**Deliverables:**
1. **Pod template** (`k8s/scraper-pod-template.yaml`):
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: scraper-template
     labels:
       app: airflow-scraper
   spec:
     restartPolicy: Never
     containers:
     - name: scraper
       image: PLACEHOLDER
       resources:
         requests:
           cpu: 500m
           memory: 1Gi
         limits:
           cpu: 1000m
           memory: 2Gi
       volumeMounts:
       - name: data
         mountPath: /data
       - name: shm
         mountPath: /dev/shm
       env:
       - name: JOB_ID
         value: PLACEHOLDER
       - name: LOG_LEVEL
         value: INFO
     volumes:
     - name: data
       persistentVolumeClaim:
         claimName: PLACEHOLDER
     - name: shm
       emptyDir:
         medium: Memory
         sizeLimit: 1Gi
   ```

2. **Template integration in DAG generator**:
   ```python
   # Add to generate_dags.py
   from kubernetes import client as k8s
   
   # Load pod template
   with open('k8s/scraper-pod-template.yaml') as f:
       pod_template = yaml.safe_load(f)
   
   # Use in KubernetesPodOperator
   scraper_task = KubernetesPodOperator(
       # ... other parameters ...
       pod_template_file='k8s/scraper-pod-template.yaml',
       # OR
       full_pod_spec=k8s.V1Pod(**pod_template),
       dag=dag
   )
   ```

**Success Criteria:**
- Pod template properly configured with resource limits
- Shared memory volume for browser stability
- Template successfully used by DAG generator

### Task 3.3: PVC Management System
**Priority:** HIGH  
**Duration:** 2-3 days  
**Dependencies:** Task 3.1

**Deliverables:**
1. **PVC templates** (`k8s/pvc-template.yaml`):
   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: pvc-{{ job_id }}
     labels:
       app: airflow-scraper
       job_id: {{ job_id }}
   spec:
     accessModes:
       - ReadWriteOnce
     storageClassName: fast-io
     resources:
       requests:
         storage: 5Gi
   ```

2. **PVC management script** (`k8s/manage_pvcs.py`):
   ```python
   import yaml
   from kubernetes import client, config
   from build.metadata_store import MetadataStore
   
   def create_pvc_for_job(job_id: str, storage_size: str = "5Gi"):
       """Create PVC for a job"""
       config.load_incluster_config()  # or load_kube_config() for local
       v1 = client.CoreV1Api()
       
       # Load template
       with open('k8s/pvc-template.yaml') as f:
           template = f.read()
       
       # Replace placeholders
       pvc_yaml = template.replace('{{ job_id }}', job_id)
       pvc_manifest = yaml.safe_load(pvc_yaml)
       
       # Update storage size
       pvc_manifest['spec']['resources']['requests']['storage'] = storage_size
       
       try:
           v1.create_namespaced_persistent_volume_claim(
               namespace='airflow',
               body=pvc_manifest
           )
           print(f"✅ Created PVC for job {job_id}")
       except client.exceptions.ApiException as e:
           if e.status == 409:  # Already exists
               print(f"ℹ️  PVC for job {job_id} already exists")
           else:
               raise
   
   def cleanup_unused_pvcs():
       """Remove PVCs for jobs no longer in metadata store"""
       config.load_incluster_config()
       v1 = client.CoreV1Api()
       store = MetadataStore()
       
       active_jobs = set(store.get_all_active_jobs().keys())
       
       # List all scraper PVCs
       pvcs = v1.list_namespaced_persistent_volume_claim(
           namespace='airflow',
           label_selector='app=airflow-scraper'
       )
       
       for pvc in pvcs.items:
           job_id = pvc.metadata.labels.get('job_id')
           if job_id and job_id not in active_jobs:
               print(f"🗑️  Deleting unused PVC for job {job_id}")
               v1.delete_namespaced_persistent_volume_claim(
                   name=pvc.metadata.name,
                   namespace='airflow'
               )
   
   if __name__ == "__main__":
       # Create PVCs for all active jobs
       store = MetadataStore()
       for job_id in store.get_all_active_jobs():
           create_pvc_for_job(job_id)
       
       # Cleanup unused PVCs
       cleanup_unused_pvcs()
   ```

3. **Storage class** (`k8s/storageclass.yaml`):
   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: fast-io
   provisioner: kubernetes.io/aws-ebs  # or gce-pd, azure-disk
   parameters:
     type: gp3
     iops: "3000"
     throughput: "125"
   volumeBindingMode: WaitForFirstConsumer
   reclaimPolicy: Delete
   ```

**Success Criteria:**
- PVCs created automatically for new jobs
- Unused PVCs cleaned up properly
- Fast storage class configured
- Data persists across pod restarts

## 📋 Phase 4: Advanced Features

### Task 4.1: Comprehensive Logging System
**Priority:** HIGH  
**Duration:** 3-4 days  
**Dependencies:** Task 3.3

**Deliverables:**
1. **Enhanced entrypoint with logging** (`docker/entrypoint.sh`):
   ```bash
   #!/bin/bash
   set -e
   
   # Setup logging
   LOG_DIR="/data/logs"
   mkdir -p "$LOG_DIR"
   
   # Create log files
   SELENIUM_LOG="$LOG_DIR/selenium.log"
   SCRAPER_LOG="$LOG_DIR/scraper.log"
   SYSTEM_LOG="$LOG_DIR/system.log"
   
   exec 1> >(tee -a "$SYSTEM_LOG")
   exec 2> >(tee -a "$SYSTEM_LOG")
   
   echo "🚀 Starting scraper job: $JOB_ID at $(date)" | tee -a "$SYSTEM_LOG"
   
   # Wait for Selenium with detailed logging  
   echo "⏳ Waiting for Selenium Grid..." | tee -a "$SYSTEM_LOG"
   RETRY_COUNT=0
   MAX_RETRIES=30
   
   while ! curl -sSf http://localhost:4444/wd/hub/status > "$SELENIUM_LOG" 2>&1; do
       RETRY_COUNT=$((RETRY_COUNT + 1))
       if [ $RETRY_COUNT -ge $MAX_RETRIES ]; then
           echo "❌ Selenium failed to start after $MAX_RETRIES attempts" | tee -a "$SYSTEM_LOG"
           exit 1
       fi
       echo "⏳ Attempt $RETRY_COUNT/$MAX_RETRIES - Selenium not ready, waiting..." | tee -a "$SYSTEM_LOG"
       sleep 2
   done
   
   echo "✅ Selenium Grid is ready" | tee -a "$SYSTEM_LOG"
   
   # Run the scraper with logging
   echo "🕷️  Starting scraper script..." | tee -a "$SYSTEM_LOG"
   python /project/main.py \
       --job-id "$JOB_ID" \
       --config "/config/jobs.yaml" \
       --output "/data/output.json" \
       --log-file "$SCRAPER_LOG" 2>&1 | tee -a "$SCRAPER_LOG"
   
   SCRAPER_EXIT_CODE=$?
   
   if [ $SCRAPER_EXIT_CODE -eq 0 ]; then
       echo "✅ Scraper completed successfully" | tee -a "$SYSTEM_LOG"
   else
       echo "❌ Scraper failed with exit code $SCRAPER_EXIT_CODE" | tee -a "$SYSTEM_LOG"
   fi
   
   # Create summary
   echo "📊 Job Summary:" | tee -a "$SYSTEM_LOG"
   echo "  Job ID: $JOB_ID" | tee -a "$SYSTEM_LOG"
   echo "  Start Time: $(head -1 "$SYSTEM_LOG" | grep -o '[0-9][0-9]:[0-9][0-9]:[0-9][0-9]')" | tee -a "$SYSTEM_LOG"
   echo "  End Time: $(date '+%H:%M:%S')" | tee -a "$SYSTEM_LOG"
   echo "  Exit Code: $SCRAPER_EXIT_CODE" | tee -a "$SYSTEM_LOG"
   echo "  Output Files:" | tee -a "$SYSTEM_LOG"
   ls -la /data/*.json 2>/dev/null | tee -a "$SYSTEM_LOG" || echo "  No output files found" | tee -a "$SYSTEM_LOG"
   
   exit $SCRAPER_EXIT_CODE
   ```

2. **Enhanced failure callback** (`dags/failure_callbacks.py`):
   ```python
   import os
   from airflow.providers.smtp.operators.email import EmailOperator
   from airflow.providers.slack.operators.slack_webhook import SlackWebhookOperator
   
   def task_failure_alert(context):
       """Enhanced failure notification with logs"""
       ti = context['task_instance']
       dag_id = context['dag'].dag_id
       job_id = dag_id.replace('scrape_', '')
       
       # Try to read log snippet from PVC
       log_snippet = "Logs not available"
       try:
           # This would need to be adjusted based on actual log location
           with open(f"/data/logs/scraper.log") as f:
               lines = f.readlines()[-50:]  # Last 50 lines
               log_snippet = ''.join(lines)
       except Exception:
           pass
       
       error_message = f"""
   🚨 Scraper Job Failed
   
   **Job Details:**
   - Job ID: {job_id}
   - DAG: {dag_id}
   - Task: {ti.task_id}
   - Execution Date: {context['execution_date']}
   - Duration: {ti.duration}
   
   **Error Log (last 50 lines):**
   ```
   {log_snippet}
   ```
   
   **Actions:**
   - Check logs: {ti.log_url}
   - Review job config in jobs.yaml
   - Check if target website is accessible
   """
       
       # Send email notification
       email_task = EmailOperator(
           task_id='send_failure_email',
           to=['data-team@company.com'],
           subject=f'Scraper Job Failed: {job_id}',
           html_content=error_message.replace('\n', ''),
       )
       email_task.execute(context)
       
       # Send Slack notification
       slack_task = SlackWebhookOperator(
           task_id='send_slack_notification',
           http_conn_id='slack_default',
           message=error_message,
           channel='#data-alerts'
       )
       slack_task.execute(context)
   ```

3. **Log aggregation sidecar** (`k8s/log-sidecar.yaml`):
   ```yaml
   # Optional: Fluent Bit sidecar for log shipping
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: fluent-bit-config
   data:
     fluent-bit.conf: |
       [INPUT]
           Name tail
           Path /data/logs/*.log
           Tag scraper.*
           Refresh_Interval 5
       
       [OUTPUT]
           Name es
           Match scraper.*
           Host elasticsearch.logging.svc.cluster.local
           Index scraper-logs
   ```

**Success Criteria:**
- Comprehensive logs written to PVC
- Failure notifications include log snippets
- System logs show job lifecycle
- Logs accessible via Airflow UI

### Task 4.2: Health Checks & Monitoring
**Priority:** MEDIUM  
**Duration:** 2-3 days  
**Dependencies:** Task 4.1

**Deliverables:**
1. **Health check endpoints** (`project/common/health.py`):
   ```python
   import requests
   import time
   from selenium import webdriver
   from selenium.common.exceptions import WebDriverException
   
   def check_selenium_health(max_retries=5):
       """Check if Selenium Grid is healthy"""
       for attempt in range(max_retries):
           try:
               response = requests.get("http://localhost:4444/wd/hub/status", timeout=10)
               if response.status_code == 200:
                   status = response.json()
                   if status.get('value', {}).get('ready'):
                       return True, "Selenium Grid is ready"
               time.sleep(2)
           except Exception as e:
               if attempt == max_retries - 1:
                   return False, f"Selenium health check failed: {str(e)}"
               time.sleep(2)
       return False, "Selenium Grid not ready after retries"
   
   def test_browser_connection():
       """Test actual browser connection"""
       try:
           options = webdriver.ChromeOptions()
           options.add_argument("--headless")
           options.add_argument("--no-sandbox")
           
           driver = webdriver.Remote(
               command_executor="http://localhost:4444/wd/hub",
               options=options
           )
           
           # Test basic navigation
           driver.get("data:text/html,Health Check")
           title = driver.title
           driver.quit()
           
           return True, f"Browser test successful, title: {title}"
       except WebDriverException as e:
           return False, f"Browser connection failed: {str(e)}"
   
   def system_health_check():
       """Complete system health check"""
       checks = {}
       
       # Selenium health
       selenium_ok, selenium_msg = check_selenium_health()
       checks['selenium'] = {'status': selenium_ok, 'message': selenium_msg}
       
       # Browser test
       if selenium_ok:
           browser_ok, browser_msg = test_browser_connection()
           checks['browser'] = {'status': browser_ok, 'message': browser_msg}
       else:
           checks['browser'] = {'status': False, 'message': 'Skipped - Selenium not ready'}
       
       # File system checks
       import os
       data_dir_writable = os.access('/data', os.W_OK)
       checks['data_directory'] = {
           'status': data_dir_writable,
           'message': 'Data directory writable' if data_dir_writable else 'Data directory not writable'
       }
       
       overall_health = all(check['status'] for check in checks.values())
       
       return {
           'overall_healthy': overall_health,
           'checks': checks,
           'timestamp': time.time()
       }
   ```

2. **Pre-flight checks in entrypoint**:
   ```bash
   # Add to entrypoint.sh before main scraper execution
   echo "🔍 Running system health checks..." | tee -a "$SYSTEM_LOG"
   python -c "
   from project.common.health import system_health_check
   import json
   
   health = system_health_check()
   print(json.dumps(health, indent=2))
   
   if not health['overall_healthy']:
       print('❌ Health checks failed!')
       exit(1)
   else:
       print('✅ All health checks passed')
   " | tee -a "$SYSTEM_LOG"
   ```

3. **Monitoring dashboard** (`monitoring/prometheus-rules.yaml`):
   ```yaml
   groups:
   - name: scraper.rules
     rules:
     - alert: ScraperJobsFailing
       expr: rate(airflow_task_failures_total{dag_id=~"scrape_.*"}[5m]) > 0.1
       for: 2m
       labels:
         severity: warning
       annotations:
         summary: "High scraper job failure rate"
         description: "Scraper jobs are failing at rate {{ $value }} failures/minute"
     
     - alert: ScraperJobStuck
       expr: airflow_task_duration_seconds{dag_id=~"scrape_.*"} > 3600
       labels:
         severity: critical
       annotations:
         summary: "Scraper job running too long"
         description: "Job {{ $labels.dag_id }} has been running for {{ $value }} seconds"
   ```

**Success Criteria:**
- Health checks prevent unhealthy containers from starting
- System health status logged and monitored
- Alerts configured for common failure patterns
- Health check failures properly reported

### Task 4.3: S3 Integration & Artifact Management  
**Priority:** MEDIUM  
**Duration:** 2 days  
**Dependencies:** Task 4.1

**Deliverables:**
1. **S3 upload utility** (`project/common/s3_utils.py`):
   ```python
   import boto3
   import os
   import zipfile
   from datetime import datetime
   from botocore.exceptions import ClientError
   
   class S3Manager:
       def __init__(self, bucket_name: str):
           self.bucket_name = bucket_name
           self.s3_client = boto3.client('s3')
       
       def upload_job_artifacts(self, job_id: str, data_dir: str = "/data") -> dict:
           """Upload all job artifacts to S3"""
           timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
           s3_prefix = f"{job_id}/{timestamp}"
           
           uploaded_files = []
           
           # Create archive of all output files
           archive_path = f"{data_dir}/job_artifacts.zip"
           with zipfile.ZipFile(archive_path, 'w') as zipf:
               for root, dirs, files in os.walk(data_dir):
                   for file in files:
                       if file.endswith(('.json', '.csv', '.txt', '.log')):
                           file_path = os.path.join(root, file)
                           arcname = os.path.relpath(file_path, data_dir)
                           zipf.write(file_path, arcname)
           
           # Upload archive
           archive_key = f"{s3_prefix}/artifacts.zip"
           try:
               self.s3_client.upload_file(archive_path, self.bucket_name, archive_key)
               uploaded_files.append({
                   'type': 'archive',
                   'local_path': archive_path,
                   's3_key': archive_key,
                   'size': os.path.getsize(archive_path)
               })
           except ClientError as e:
               print(f"Failed to upload archive: {e}")
           
           # Upload individual log files for easy access
           logs_dir = f"{data_dir}/logs"
           if os.path.exists(logs_dir):
               for log_file in os.listdir(logs_dir):
                   if log_file.endswith('.log'):
                       local_path = os.path.join(logs_dir, log_file)
                       s3_key = f"{s3_prefix}/logs/{log_file}"
                       try:
                           self.s3_client.upload_file(local_path, self.bucket_name, s3_key)
                           uploaded_files.append({
                               'type': 'log',
                               'local_path': local_path,
                               's3_key': s3_key,
                               'size': os.path.getsize(local_path)
                           })
                       except ClientError as e:
                           print(f"Failed to upload {log_file}: {e}")
           
           return {
               'job_id': job_id,
               'timestamp': timestamp,
               'bucket': self.bucket_name,
               'uploaded_files': uploaded_files,
               'total_files': len(uploaded_files),
               'total_size': sum(f['size'] for f in uploaded_files)
           }
       
       def cleanup_local_files(self, data_dir: str = "/data", keep_logs: bool = True):
           """Clean up local files after S3 upload"""
           for root, dirs, files in os.walk(data_dir):
               for file in files:
                   file_path = os.path.join(root, file)
                   # Keep log files if requested
                   if keep_logs and '/logs/' in file_path:
                       continue
                   # Remove output files and temp files
                   if file.endswith(('.json', '.csv', '.txt', '.zip', '.tmp')):
                       try:
                           os.remove(file_path)
                           print(f"Cleaned up: {file_path}")
                       except OSError as e:
                           print(f"Failed to remove {file_path}: {e}")
   ```

2. **Enhanced DAG with S3 integration**:
   ```python
   # Add to generate_dags.py
   if job_config.get('upload_to_s3'):
       s3_upload = KubernetesPodOperator(
           namespace='airflow',
           task_id='upload_to_s3',
           name=f'upload-{job_id}-{{{{ ts_nodash }}}}',
           image='amazon/aws-cli:latest',
           cmds=['python', '-c'],
           arguments=[f"""
   import sys
   sys.path.append('/project')
   from common.s3_utils import S3Manager
   
   s3_manager = S3Manager('{job_config['s3_bucket']}')
   result = s3_manager.upload_job_artifacts('{job_id}')
   s3_manager.cleanup_local_files()
   
   print(f"Uploaded {{result['total_files']}} files, {{result['total_size']}} bytes")
   """],
           volumes=volumes,  # Same volumes as scraper task
           volume_mounts=volume_mounts,
           env_vars={
               'AWS_DEFAULT_REGION': 'us-east-1',
               'JOB_ID': job_id
           },
           dag=dag
       )
       
       scraper_task >> s3_upload
   ```

3. **S3 lifecycle policy** (`aws/s3-lifecycle-policy.json`):
   ```json
   {
       "Rules": [
           {
               "ID": "ScraperArtifactsLifecycle",
               "Status": "Enabled",
               "Filter": {
                   "Prefix": ""
               },
               "Transitions": [
                   {
                       "Days": 30,
                       "StorageClass": "STANDARD_IA"
                   },
                   {
                       "Days": 90,
                       "StorageClass": "GLACIER"
                   },
                   {
                       "Days": 365,
                       "StorageClass": "DEEP_ARCHIVE"
                   }
               ],
               "Expiration": {
                   "Days": 2555
               }
           }
       ]
   }
   ```

**Success Criteria:**
- Artifacts uploaded to S3 after successful runs
- Local files cleaned up after upload
- S3 lifecycle policies configured
- Upload failures don't fail the main scraper task

## 📋 Phase 5: Production Deployment

### Task 5.1: EKS Cluster Setup
**Priority:** HIGH  
**Duration:** 3-5 days  
**Dependencies:** All previous tasks

**Deliverables:**
1. **EKS cluster configuration** (`terraform/eks-cluster.tf`):
   ```hcl
   module "eks" {
     source = "terraform-aws-modules/eks/aws"
     
     cluster_name    = "airflow-scraper-cluster"
     cluster_version = "1.27"
     
     vpc_id     = module.vpc.vpc_id
     subnet_ids = module.vpc.private_subnets
     
     node_groups = {
       airflow = {
         desired_capacity = 2
         max_capacity     = 5
         min_capacity     = 2
         
         instance_types = ["t3.large"]
         
         k8s_labels = {
           role = "airflow"
         }
         
         taints = [
           {
             key    = "dedicated"
             value  = "airflow"
             effect = "NO_SCHEDULE"
           }
         ]
       }
       
       scrapers = {
         desired_capacity = 1
         max_capacity     = 20
         min_capacity     = 0
         
         instance_types = ["c5.xlarge"]
         
         k8s_labels = {
           role = "scraper"
         }
         
         scaling_config = {
           desired_size = 1
           max_size     = 20
           min_size     = 0
         }
       }
     }
     
     tags = {
       Environment = "production"
       Project     = "airflow-scraper"
     }
   }
   ```

2. **Cluster autoscaler** (`k8s/cluster-autoscaler.yaml`):
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: cluster-autoscaler
     namespace: kube-system
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: cluster-autoscaler
     template:
       metadata:
         labels:
           app: cluster-autoscaler
       spec:
         containers:
         - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.27.0
           name: cluster-autoscaler
           resources:
             limits:
               cpu: 100m
               memory: 300Mi
             requests:
               cpu: 100m
               memory: 300Mi
           command:
             - ./cluster-autoscaler
             - --v=4
             - --stderrthreshold=info
             - --cloud-provider=aws
             - --skip-nodes-with-local-storage=false
             - --expander=least-waste
             - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/airflow-scraper-cluster
   ```

3. **RBAC configuration** (`k8s/rbac.yaml`):
   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: airflow-sa
     namespace: airflow
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: airflow-cluster-role
   rules:
   - apiGroups: [""]
     resources: ["pods", "persistentvolumeclaims", "secrets", "configmaps"]
     verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
   - apiGroups: ["apps"]
     resources: ["deployments"]
     verbs: ["get", "list", "watch"]
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: airflow-cluster-role-binding
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: airflow-cluster-role
   subjects:
   - kind: ServiceAccount
     name: airflow-sa
     namespace: airflow
   ```

**Success Criteria:**
- EKS cluster deployed with proper node groups
- Cluster autoscaler configured and working
- RBAC properly configured for Airflow
- Networking and security groups configured

### Task 5.2: Airflow Production Deployment
**Priority:** HIGH  
**Duration:** 2-3 days  
**Dependencies:** Task 5.1

**Deliverables:**
1. **Airflow Helm values** (`helm/airflow-values.yaml`):
   ```yaml
   # Airflow Helm Chart Configuration
   airflow:
     image:
       repository: apache/airflow
       tag: 2.7.0-python3.11
     
     executor: KubernetesExecutor
     
     config:
       core:
         dags_folder: /opt/airflow/dags
         parallelism: 50
         max_active_tasks_per_dag: 20
         max_active_runs_per_dag: 1
       
       kubernetes_executor:
         namespace: airflow
         worker_container_repository: apache/airflow
         worker_container_tag: 2.7.0-python3.11
         delete_worker_pods: True
         delete_worker_pods_on_failure: False
       
       logging:
         remote_logging: True
         remote_base_log_folder: s3://airflow-scraper-logs
         remote_log_conn_id: aws_s3_default
       
       webserver:
         web_server_host: 0.0.0.0
         web_server_port: 8080
         workers: 4
         worker_refresh_batch_size: 1
         worker_refresh_interval: 6000
   
   postgresql:
     enabled: true
     persistence:
       enabled: true
       size: 20Gi
   
   redis:
     enabled: false  # Not needed with KubernetesExecutor
   
   # DAG and logs persistence
   dags:
     persistence:
       enabled: true
       size: 10Gi
       storageClassName: gp3
     gitSync:
       enabled: true
       repo: https://github.com/your-org/airflow-selenium-scheduler.git
       branch: main
       subPath: dags
   
   logs:
     persistence:
       enabled: true
       size: 50Gi
   
   # Resource limits
   webserver:
     resources:
       limits:
         cpu: 1000m
         memory: 2Gi
       requests:
         cpu: 500m
         memory: 1Gi
   
   scheduler:
     resources:
       limits:
         cpu: 1000m
         memory: 2Gi
       requests:
         cpu: 500m
         memory: 1Gi
   
   # Service account
   serviceAccount:
     create: false
     name: airflow-sa
   
   # Ingress
   ingress:
     enabled: true
     hosts:
       - name: airflow-scraper.company.com
         tls:
           enabled: true
           secretName: airflow-tls
   ```

2. **Deployment script** (`deploy/deploy-airflow.sh`):
   ```bash
   #!/bin/bash
   set -e
   
   echo "🚀 Deploying Airflow to EKS..."
   
   # Create namespace
   kubectl create namespace airflow --dry-run=client -o yaml | kubectl apply -f -
   
   # Apply RBAC
   kubectl apply -f k8s/rbac.yaml
   
   # Add Airflow Helm repo
   helm repo add apache-airflow https://airflow.apache.org
   helm repo update
   
   # Install or upgrade Airflow
   helm upgrade --install airflow apache-airflow/airflow \
     --namespace airflow \
     --values helm/airflow-values.yaml \
     --timeout 600s \
     --wait
   
   # Wait for deployment
   kubectl wait --for=condition=available --timeout=600s deployment/airflow-webserver -n airflow
   kubectl wait --for=condition=available --timeout=600s deployment/airflow-scheduler -n airflow
   
   echo "✅ Airflow deployed successfully!"
   echo "🌐 Access Airflow at: https://airflow-scraper.company.com"
   ```

3. **Connection setup** (`deploy/setup-connections.py`):
   ```python
   from airflow.models import Connection
   from airflow.utils.db import create_session
   
   def setup_connections():
       connections = [
           Connection(
               conn_id='kubernetes_default',
               conn_type='kubernetes',
               extra={
                   'in_cluster': True,
                   'namespace': 'airflow'
               }
           ),
           Connection(
               conn_id='aws_s3_default',
               conn_type='aws',
               extra={
                   'region_name': 'us-east-1'
               }
           ),
           Connection(
               conn_id='redis_default',
               conn_type='redis',
               host='redis-service.airflow.svc.cluster.local',
               port=6379,
               schema='0'
           )
       ]
       
       with create_session() as session:
           for conn in connections:
               existing = session.query(Connection).filter_by(conn_id=conn.conn_id).first()
               if not existing:
                   session.add(conn)
                   print(f"Created connection: {conn.conn_id}")
               else:
                   print(f"Connection already exists: {conn.conn_id}")
           session.commit()
   
   if __name__ == "__main__":
       setup_connections()
   ```

**Success Criteria:**
- Airflow deployed and accessible via ingress
- Scheduler and webserver running properly
- KubernetesExecutor configured correctly
- Connections and pools configured

### Task 5.3: CI/CD Pipeline Implementation
**Priority:** HIGH  
**Duration:** 3-4 days  
**Dependencies:** Task 5.2

**Deliverables:**
1. **Complete GitHub Actions workflow** (`.github/workflows/main.yml`):
   ```yaml
   name: Build and Deploy Scraper Framework
   
   on:
     push:
       branches: [main, develop]
     pull_request:
       branches: [main]
   
   env:
     AWS_REGION: us-east-1
     EKS_CLUSTER_NAME: airflow-scraper-cluster
     REGISTRY: ${{ secrets.ECR_REGISTRY }}
   
   jobs:
     changes:
       runs-on: ubuntu-latest
       outputs:
         config: ${{ steps.changes.outputs.config }}
         projects: ${{ steps.changes.outputs.projects }}
         docker: ${{ steps.changes.outputs.docker }}
         k8s: ${{ steps.changes.outputs.k8s }}
       steps:
         - uses: actions/checkout@v3
         - uses: dorny/paths-filter@v2
           id: changes
           with:
             filters: |
               config:
                 - 'config/jobs.yaml'
               projects:
                 - 'project/**'
               docker:
                 - 'docker/**'
               k8s:
                 - 'k8s/**'
     
     test:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v3
         
         - name: Set up Python
           uses: actions/setup-python@v3
           with:
             python-version: '3.11'
         
         - name: Install dependencies
           run: |
             pip install -r requirements.txt
             pip install pytest
         
         - name: Run tests
           run: |
             pytest tests/ -v
         
         - name: Validate jobs config
           run: |
             python -c "
             import yaml
             with open('config/jobs.yaml') as f:
                 jobs = yaml.safe_load(f)
             print(f'Validated {len(jobs)} job configurations')
             "
     
     build-images:
       needs: [changes, test]
       if: needs.changes.outputs.config == 'true' || needs.changes.outputs.projects == 'true' || needs.changes.outputs.docker == 'true'
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v3
         
         - name: Configure AWS credentials
           uses: aws-actions/configure-aws-credentials@v1
           with:
             aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
             aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
             aws-region: ${{ env.AWS_REGION }}
         
         - name: Login to ECR
           run: |
             aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $REGISTRY
         
         - name: Set up Redis connection
           run: |
             # Port forward to Redis in EKS cluster
             aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME
             kubectl port-forward service/redis-service 6379:6379 -n airflow &
             sleep 5
         
         - name: Build and push images
           env:
             REDIS_URL: redis://localhost:6379/0
             DOCKER_REGISTRY: ${{ env.REGISTRY }}
           run: |
             python build/enhanced_incremental_build.py
         
         - name: Update deployment
           run: |
             # Trigger Airflow DAG refresh
             kubectl exec deployment/airflow-scheduler -n airflow -- airflow dags reserialize
     
     deploy-k8s:
       needs: [changes, test]
       if: needs.changes.outputs.k8s == 'true' && github.ref == 'refs/heads/main'
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v3
         
         - name: Configure AWS credentials
           uses: aws-actions/configure-aws-credentials@v1
           with:
             aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
             aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
             aws-region: ${{ env.AWS_REGION }}
         
         - name: Update kubeconfig
           run: |
             aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME
         
         - name: Apply Kubernetes manifests
           run: |
             kubectl apply -f k8s/ -n airflow
         
         - name: Manage PVCs
           run: |
             python k8s/manage_pvcs.py
   ```

2. **Pre-commit hooks** (`.pre-commit-config.yaml`):
   ```yaml
   repos:
     - repo: https://github.com/pre-commit/pre-commit-hooks
       rev: v4.4.0
       hooks:
         - id: trailing-whitespace
         - id: end-of-file-fixer
         - id: check-yaml
         - id: check-added-large-files
         - id: detect-private-key
     
     - repo: https://github.com/psf/black
       rev: 23.1.0
       hooks:
         - id: black
           language_version: python3.11
     
     - repo: https://github.com/PyCQA/flake8
       rev: 6.0.0
       hooks:
         - id: flake8
   ```

3. **Testing framework** (`tests/test_scraper_framework.py`):
   ```python
   import pytest
   import yaml
   from build.config_parser import load_jobs_config, validate_job_config
   from build.metadata_store import MetadataStore
   
   def test_jobs_config_validity():
       """Test that jobs.yaml is valid"""
       jobs = load_jobs_config()
       assert isinstance(jobs, list)
       assert len(jobs) > 0
       
       for job in jobs:
           assert validate_job_config(job), f"Invalid job config: {job}"
   
   def test_unique_job_ids():
       """Test that all job IDs are unique"""
       jobs = load_jobs_config()
       job_ids = [job['job_id'] for job in jobs]
       assert len(job_ids) == len(set(job_ids)), "Duplicate job IDs found"
   
   def test_required_fields():
       """Test that all jobs have required fields"""
       jobs = load_jobs_config()
       required_fields = ['job_id', 'name', 'project_path', 'browser', 'output_pvc']
       
       for job in jobs:
           for field in required_fields:
               assert field in job, f"Missing required field '{field}' in job {job.get('job_id', 'unknown')}"
   
   @pytest.mark.integration
   def test_metadata_store_connection():
       """Test Redis connection"""
       try:
           store = MetadataStore()
           store.redis.ping()
       except Exception as e:
           pytest.skip(f"Redis not available: {e}")
   ```

**Success Criteria:**
- CI/CD pipeline runs on code changes
- Images built and pushed to ECR
- Tests pass before deployment
- Kubernetes manifests applied automatically
- Airflow DAGs refresh after image updates

## 📋 Phase 6: Final Testing & Documentation

### Task 6.1: End-to-End Testing
**Priority:** CRITICAL  
**Duration:** 3-5 days  
**Dependencies:** Task 5.3

**Deliverables:**
1. **Test job configurations** (`config/test-jobs.yaml`):
   ```yaml
   - job_id: "e2e_test_001"
     name: "E2E Test - Basic HTTP"
     schedule: "@once"
     project_path: "/project/test/basic_http"
     browser: "chrome"
     browser_config:
       headless: true
     output_pvc: "pvc-e2e-test-001"
     upload_to_s3: false
   
   - job_id: "e2e_test_002"
     name: "E2E Test - JavaScript Heavy"
     schedule: "@once"
     project_path: "/project/test/javascript_heavy"
     browser: "chrome"
     browser_config:
       headless: true
       wait_timeout: 30
     output_pvc: "pvc-e2e-test-002"
     upload_to_s3: true
     s3_bucket: "scraper-test-outputs"
   ```

2. **Test scrapers** (`project/test/basic_http/main.py`):
   ```python
   from selenium import webdriver
   from selenium.webdriver.common.by import By
   import json, time, sys
   import argparse
   
   def main():
       parser = argparse.ArgumentParser()
       parser.add_argument('--job-id', required=True)
       parser.add_argument('--output', required=True)
       parser.add_argument('--log-file')
       args = parser.parse_args()
       
       print(f"🧪 Starting E2E test for job: {args.job_id}")
       
       options = webdriver.ChromeOptions()
       options.add_argument("--headless")
       options.add_argument("--no-sandbox")
       options.add_argument("--disable-dev-shm-usage")
       
       driver = webdriver.Remote(
           command_executor="http://localhost:4444/wd/hub",
           options=options
       )
       
       try:
           # Test 1: Basic page load
           print("Test 1: Loading httpbin.org/get")
           driver.get("https://httpbin.org/get")
           assert "httpbin" in driver.title.lower()
           
           # Test 2: JSON response parsing
           print("Test 2: Parsing JSON response")
           pre_element = driver.find_element(By.TAG_NAME, "pre")
           response_data = json.loads(pre_element.text)
           assert "origin" in response_data
           
           # Test 3: User agent check
           print("Test 3: Checking user agent")
           assert "Chrome" in response_data["headers"]["User-Agent"]
           
           # Save results
           results = {
               "job_id": args.job_id,
               "timestamp": time.time(),
               "tests_passed": 3,
               "tests_failed": 0,
               "data": response_data,
               "status": "success"
           }
           
           with open(args.output, 'w') as f:
               json.dump(results, f, indent=2)
           
           print(f"✅ All tests passed! Results saved to {args.output}")
           
       except Exception as e:
           error_results = {
               "job_id": args.job_id,
               "timestamp": time.time(),
               "error": str(e),
               "status": "failed"
           }
           
           with open(args.output, 'w') as f:
               json.dump(error_results, f, indent=2)
           
           print(f"❌ Test failed: {e}")
           sys.exit(1)
           
       finally:
           driver.quit()
   
   if __name__ == "__main__":
       main()
   ```

3. **Comprehensive test suite** (`tests/test_e2e.py`):
   ```python
   import pytest
   import time
   import subprocess
   import yaml
   from kubernetes import client, config
   
   @pytest.fixture(scope="session")
   def k8s_client():
       config.load_kube_config()
       return client.CoreV1Api()
   
   @pytest.fixture(scope="session")
   def airflow_api():
       # Setup Airflow API client
       pass
   
   def test_full_workflow():
       """Test complete workflow from config to execution"""
       
       # 1. Load test configuration
       with open('config/test-jobs.yaml') as f:
           test_jobs = yaml.safe_load(f)
       
       # 2. Build test images
       print("Building test images...")
       result = subprocess.run([
           'python', 'build/enhanced_incremental_build.py',
           '--config', 'config/test-jobs.yaml'
       ], capture_output=True, text=True)
       assert result.returncode == 0, f"Image build failed: {result.stderr}"
       
       # 3. Create test PVCs
       print("Creating test PVCs...")
       for job in test_jobs:
           subprocess.run([
               'kubectl', 'apply', '-f', f"k8s/pvc-{job['job_id']}.yaml"
           ])
       
       # 4. Trigger DAG runs via Airflow API
       print("Triggering test DAG runs...")
       # Implementation depends on Airflow API setup
       
       # 5. Wait for completion and verify results
       print("Waiting for test completion...")
       time.sleep(300)  # 5 minutes max wait
       
       # 6. Verify outputs
       for job in test_jobs:
           # Check if output files exist in PVC
           # Verify content is correct
           pass
   
   def test_scaling_behavior():
       """Test that system can handle multiple concurrent jobs"""
       # Create multiple test jobs
       # Trigger them simultaneously
       # Verify all complete successfully
       # Check resource utilization
       pass
   
   def test_failure_scenarios():
       """Test failure handling and recovery"""
       # Test with invalid target URLs
       # Test with network timeouts
       # Test with browser crashes
       # Verify proper error handling and notifications
       pass
   ```

**Success Criteria:**
- All E2E tests pass consistently
- System handles concurrent job execution
- Failure scenarios handled gracefully
- Performance meets requirements (jobs complete within expected timeframes)

### Task 6.2: Production Documentation
**Priority:** HIGH  
**Duration:** 2-3 days  
**Dependencies:** Task 6.1

**Deliverables:**
1. **Complete README.md** with:
   - System overview and architecture
   - Prerequisites and setup instructions
   - Configuration guide for jobs.yaml
   - Deployment instructions
   - Troubleshooting guide
   - API documentation

2. **Operations runbook** (`docs/operations.md`):
   ```markdown
   # Airflow Selenium Scraper Operations Guide
   
   ## Daily Operations
   
   ### Monitoring Checklist
   - [ ] Check Airflow UI for failed DAGs
   - [ ] Monitor cluster resource utilization
   - [ ] Verify S3 uploads are working
   - [ ] Check for stuck pods
   
   ### Common Issues
   
   #### Pods Stuck in Pending
   **Symptoms:** Scraper pods show "Pending" status
   **Causes:** 
   - Insufficient cluster resources
   - Node selector/affinity issues
   - PVC binding problems
   
   **Resolution:**
   ```
   # Check cluster capacity
   kubectl top nodes
   kubectl describe pod  -n airflow
   
   # Scale cluster if needed
   aws eks update-nodegroup-config --cluster-name airflow-scraper-cluster --nodegroup-name scrapers --scaling-config minSize=2,maxSize=25,desiredSize=5
   ```
   
   #### Selenium Connection Timeouts
   **Symptoms:** Scraper logs show "Selenium not ready" errors
   **Causes:**
   - Chrome process crashes
   - Memory limits too low
   - Network issues
   
   **Resolution:**
   - Increase memory limits in DAG
   - Check /dev/shm size
   - Verify Docker image health
   ```

3. **Developer guide** (`docs/developer.md`):
   ```markdown
   # Developer Guide
   
   ## Adding a New Scraping Job
   
   1. Create project directory:
   ```
   mkdir -p project/my_new_scraper
   ```
   
   2. Implement main.py:
   ```
   # Follow the template in project/template/
   ```
   
   3. Add to jobs.yaml:
   ```
   - job_id: "my_new_job"
     name: "My New Scraper"
     schedule: "0 9 * * *"
     project_path: "/project/my_new_scraper"
     # ... other config
   ```
   
   4. Test locally:
   ```
   docker run -p 4444:4444 selenium/standalone-chrome &
   python project/my_new_scraper/main.py --job-id test
   ```
   
   5. Deploy:
   ```
   git add .
   git commit -m "Add new scraper job"
   git push origin main
   ```
   ```

4. **API documentation** using tools like Sphinx or MkDocs

**Success Criteria:**
- Documentation is comprehensive and accurate
- New developers can onboard using the guides
- Operations team can troubleshoot issues
- All APIs are documented

This comprehensive task breakdown provides a complete roadmap for building the airflow-selenium-scheduler framework from basic setup through production deployment. Each task includes specific deliverables, success criteria, and code examples to guide implementation.

[1] https://hub.docker.com/r/selenium/standalone-chrome
[2] https://github.com/SeleniumHQ/docker-selenium
[3] https://stackoverflow.com/questions/61237299/how-to-avoid-authentication-when-connecting-to-standalone-chrome-debug-container
[4] https://www.browserstack.com/guide/run-selenium-tests-in-docker
[5] https://stackoverflow.com/questions/79158316/how-can-i-correctly-run-a-selenium-standalone-chrome-container-with-a-different
[6] https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/
[7] https://www.sparkcodehub.com/airflow/best-practices/project-structure
[8] https://github.com/SeleniumHQ/docker-selenium/issues/1772
[9] https://www.baeldung.com/ops/kubernetes-equivalent-of-env-file-docker
[10] https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/overview.html
[11] https://www.youtube.com/watch?v=7FUBk12Zoc4
[12] https://kubernetes.io/docs/concepts/configuration/secret/
[13] https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html
[14] https://hub.docker.com/r/selenium/standalone-chromium
[15] https://spacelift.io/blog/kubernetes-environment-variables
[16] https://airflow.apache.org/docs/apache-airflow/2.3.0/core-concepts/tasks.html
[17] https://ikalamtech.com/selenium-with-docker/
[18] https://www.mirantis.com/cloud-native-concepts/getting-started-with-kubernetes/what-are-kubernetes-secrets/
[19] https://stackoverflow.com/questions/44424473/airflow-structure-organization-of-dags-and-tasks
[20] https://www.mirantis.com/blog/cloud-native-5-minutes-at-a-time-using-kubernetes-secrets-with-environment-variables-and-volume-mounts/
[21] https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html
[22] https://www.astronomer.io/docs/learn/dag-best-practices/
[23] https://www.astronomer.io/docs/learn/managing-airflow-code/
[24] https://reintech.io/blog/organizing-structuring-airflow-projects
[25] https://www.scraperapi.com/web-scraping/github/
[26] https://octopus.com/devops/ci-cd/ci-cd-with-docker/
[27] https://crawlbase.com/blog/scraping-github-repositories-and-profiles/
[28] https://dev.to/itsahsanmangal/building-a-robust-cicd-pipeline-with-docker-a-comprehensive-guide-4k8b
[29] https://airflow.apache.org/docs/apache-airflow/2.1.4/best-practices.html
[30] https://brightdata.com/blog/how-tos/how-to-scrape-github-repositories-in-python
[31] https://spacelift.io/blog/docker-ci-cd
[32] https://hevodata.com/learn/airflow-dags/
[33] https://github.com/DaveSimoes/Web-Scraping
[34] https://circleci.com/blog/build-cicd-pipelines-using-docker/
[35] https://stackoverflow.com/questions/69268969/which-project-structure-approach-should-i-choose-for-my-web-scraping-project
[36] https://docs.docker.com/build/ci/
[37] https://dev.to/ken_mwaura1/getting-started-with-a-web-scraping-project-10ej
[38] https://www.docker.com/blog/docker-and-jenkins-build-robust-ci-cd-pipelines/
