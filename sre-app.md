
# The below is the detailed explanation of the steps followed to replicate the issue and the fixes applied.

---

## Issue 1: Nginx Not Proxying Requests to Flask `sre` Service

### Symptom

When running `docker compose up`, both services (nginx and sre) come up, but a request like:

```bash
curl localhost/sres
```

…returns an error or a **502 Bad Gateway** from nginx.

### Replicate the issue

```bash
git clone <repo>
cd devops-code-challenge-lt1002

docker-compose down
docker-compose up --build

curl -i localhost/sres

HTTP/1.1 502 Bad Gateway
Server: nginx/1.27.5
Date: Wed, 04 Jun 2025 16:11:00 GMT
Content-Type: text/html
Content-Length: 157
Connection: keep-alive

<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.27.5</center>
</body>
</html>
```

### Root Cause

In `sre.conf`, the reverse proxy is configured as:

```nginx
proxy_pass http://localhost:4567;
```

But inside Docker, `localhost` refers to the nginx container itself. Each service in Docker Compose should communicate via the service names. So according to the docker-compose.yml file, the service name of sre happens to be "app"

### Docker Compose YAML Reference

```yaml
services:
  nginx:
    image: nginx:1.27.5-alpine
    volumes:
      - ./nginx/sre.conf:/etc/nginx/conf.d/default.conf
    ports:
      - 80:80

  app:
    build: ./app
    ports:
      - 4567
```

So the proxy_pass value should be:

```nginx
proxy_pass http://app:4567;
```

### Fix

**Edit:** `nginx/sre.conf`

**Change:**
```nginx
proxy_pass http://localhost:4567;
```

**To:**
```nginx
proxy_pass http://app:4567;
```

### Reason

The nginx container must access the app via the internal Docker network by service name.

### Result

```bash
docker-compose down
docker-compose up --build

curl localhost/sres
{"sres":["bob","linda","tina","gene","louise"]}
```
---

## Fix 2: Sidecar Container for Health Polling & Metrics

### Step 1: Create Sidecar Script

```bash
cd devops-code-challenge-lt1002
mkdir sidecar
cd sidecar
```

**Create a Python File:** `sidecar/sidecar.py`
```python
import requests
import time
import statistics
import sys
import datetime

sys.stdout.reconfigure(line_buffering=True)

HEALTH_URL = "http://app:4567/health"
POLL_INTERVAL = 10
REPORT_INTERVAL = 60

metrics_history = {
    "requestLatency": [],
    "dbLatency": [],
    "cacheLatency": []
}

def fetch_health():
    try:
        response = requests.get(HEALTH_URL, timeout=3)
        if response.status_code != 200:
            print(f"[{datetime.datetime.now()}] ERROR: /health returned {response.status_code}", flush=True)
            sys.exit(1)
        return response.json()["metrics"]
    except Exception as e:
        print(f"[{datetime.datetime.now()}] Exception: {e}", flush=True)
        sys.exit(1)

def print_metrics():
    print(f"---- Metrics Report at {datetime.datetime.now()} ----")
    for metric, values in metrics_history.items():
        if values:
            avg = statistics.mean(values)
            min_val = min(values)
            max_val = max(values)
            print(f"{metric}: avg={avg:.6f} min={min_val:.6f} max={max_val:.6f}")
            metrics_history[metric] = []
    print("-" * 40)

def main():
    elapsed = 0
    while True:
        metrics = fetch_health()
        for key in metrics_history.keys():
            metrics_history[key].append(metrics[key])
        time.sleep(POLL_INTERVAL)
        elapsed += POLL_INTERVAL
        if elapsed >= REPORT_INTERVAL:
            print_metrics()
            elapsed = 0

if __name__ == "__main__":
    main()
```
### Purpose of the Script:

- Poll `/health` on `app:4567` every 10 seconds
- Track metrics: `requestLatency`, `dbLatency`, `cacheLatency`
- Print average, min, max every 60 seconds
- Exit non-zero if `/health` returns non-200

### Step 2: Dockerfile for Sidecar

**Create a Dockerfile:** `sidecar/Dockerfile`

```Dockerfile
FROM python:3.11-alpine
RUN pip install requests
COPY sidecar.py /sidecar.py
CMD ["python", "-u", "/sidecar.py"]
```

### Step 3: Update `docker-compose.yml` as below

```yaml
services:
  nginx:
    image: nginx:1.27.5-alpine
    volumes:
      - ./nginx/sre.conf:/etc/nginx/conf.d/default.conf
    ports:
      - "80:80"
    depends_on:
      - app
      - sidecar

  app:
    build: ./app
    ports:
      - "4567:4567"

  sidecar:
    build: ./sidecar
    depends_on:
      - app
```

### Step 4: Run Compose

```bash
docker compose down
docker compose up --build
```

### Output: Run the below command to check the metrics output
```
docker compose logs -f sidecar
```

### Command Output:
```text
sidecar-1  | ----------------------------------------
sidecar-1  | ---- Metrics Report at 2025-06-04 23:34:54.477214 ----
sidecar-1  | requestLatency: avg=0.751795 min=0.313127 max=0.965783
sidecar-1  | dbLatency: avg=0.601748 min=0.122320 max=0.931448
sidecar-1  | cacheLatency: avg=0.597802 min=0.431672 max=0.780374
sidecar-1  | ----------------------------------------
sidecar-1  | ---- Metrics Report at 2025-06-04 23:35:54.505710 ----
sidecar-1  | requestLatency: avg=0.518775 min=0.012321 max=0.815564
sidecar-1  | dbLatency: avg=0.534917 min=0.137244 max=0.908015
sidecar-1  | cacheLatency: avg=0.349327 min=0.144121 max=0.812436
sidecar-1  | ----------------------------------------
sidecar-1  | ---- Metrics Report at 2025-06-04 23:36:54.529879 ----
sidecar-1  | requestLatency: avg=0.448524 min=0.072682 max=0.995410
sidecar-1  | dbLatency: avg=0.321049 min=0.032527 max=0.533437
sidecar-1  | cacheLatency: avg=0.510771 min=0.141873 max=0.766533
sidecar-1  | ----------------------------------------
```

### Step 5: Test Failure Mode

```bash
docker compose logs -f sidecar
docker stop devops-code-challenge-lt1002-app-1 # To stop the sre app container
```

**Expected:**

Sidecar exits with code `1` if `/health` fails.

**Actual Output:**
```
sidecar-1  | ----------------------------------------
sidecar-1  | ---- Metrics Report at 2025-06-04 23:35:54.505710 ----
sidecar-1  | requestLatency: avg=0.518775 min=0.012321 max=0.815564
sidecar-1  | dbLatency: avg=0.534917 min=0.137244 max=0.908015
sidecar-1  | cacheLatency: avg=0.349327 min=0.144121 max=0.812436
sidecar-1  | ----------------------------------------
sidecar-1  | ---- Metrics Report at 2025-06-04 23:36:54.529879 ----
sidecar-1  | requestLatency: avg=0.448524 min=0.072682 max=0.995410
sidecar-1  | dbLatency: avg=0.321049 min=0.032527 max=0.533437
sidecar-1  | cacheLatency: avg=0.510771 min=0.141873 max=0.766533
sidecar-1  | ----------------------------------------
sidecar-1  | [2025-06-04 23:37:44.579931] Exception: HTTPConnectionPool(host='app', port=4567): Max retries exceeded with url: /health (Caused by NameResolutionError("<urllib3.connection.HTTPConnection object at 0x7f652ff4fa10>: Failed to resolve 'app' ([Errno -5] Name has no usable address)"))
sidecar-1 exited with code 1
```

** From the above output, it is clearly evident that the sidecar is handling all of the below requirements:
- Poll `/health` on `app:4567` every 10 seconds
- Track metrics: `requestLatency`, `dbLatency`, `cacheLatency`
- Print average, min, max every 60 seconds
- Exit non-zero if `/health` returns non-200
