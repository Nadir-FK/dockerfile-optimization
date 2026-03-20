# Dockerfile Optimization

A practical demonstration of Docker image optimization techniques — reducing a Python Flask image from **1.12 GB down to 131 MB** (88% reduction) using multi-stage builds, slim base images, and Docker best practices.

---

## Results

| | Base image | Final size | Build time |
|---|---|---|---|
| Before | `python:3.11` | **1.12 GB** | slow (no cache) |
| After | `python:3.11-slim` + multi-stage | **131 MB** | fast (layer cache) |

**88% smaller image** — faster pulls, lower cloud storage costs, reduced attack surface.

---

## What changed and why

### 1. Base image — `python:3.11` → `python:3.11-slim`

The full `python:3.11` image ships with compilers, build tools, and system packages your app will never use in production. The slim variant contains only what's needed to run Python.

```dockerfile
# Before — 1.12 GB
FROM python:3.11

# After — starts at ~130 MB
FROM python:3.11-slim
```

### 2. Multi-stage build

The builder stage installs dependencies. The final stage copies only the installed packages — leaving behind pip cache, build tools, and temporary files.

```dockerfile
# Stage 1 — install dependencies
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Stage 2 — copy only what's needed
FROM python:3.11-slim
COPY --from=builder /root/.local /root/.local
COPY . .
```

### 3. Layer caching — `requirements.txt` copied before source code

Docker caches each layer. By copying `requirements.txt` first and installing dependencies before copying the rest of the code, dependencies are only reinstalled when `requirements.txt` changes — not on every code change.

```dockerfile
# Bad — reinstalls everything on every code change
COPY . .
RUN pip install -r requirements.txt

# Good — dependencies cached as a separate layer
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

### 4. `.dockerignore`

Prevents unnecessary files (`.git`, `__pycache__`, `.env`) from being copied into the image, keeping the build context small and clean.

### 5. Production server

The bad Dockerfile uses Flask's built-in dev server. The optimized version uses Gunicorn — a production-grade WSGI server.

```dockerfile
# Before — dev server, not suitable for production
CMD ["python", "app.py"]

# After — production WSGI server
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

---

## Project structure

```
dockerfile-optimization/
├── app.py                  # Flask application
├── requirements.txt        # Python dependencies
├── Dockerfile.bad          # Unoptimized — for comparison
├── Dockerfile.optimized    # Production-ready
├── .dockerignore           # Excludes unnecessary files
└── README.md
```

---

## Run it yourself

**Build both images and compare:**

```bash
docker build -f Dockerfile.bad -t flask-bad .
docker build -f Dockerfile.optimized -t flask-optimized .
docker images | grep flask
```

**Run the optimized image:**

```bash
docker run -p 5000:5000 flask-optimized
```

App available at `http://localhost:5000`

---

## Key techniques demonstrated

- Multi-stage Docker builds
- Slim base images (`python:3.11-slim`)
- Dependency layer caching
- `.dockerignore` for clean build context
- Production WSGI server (Gunicorn)
- Environment variable best practices (`PYTHONDONTWRITEBYTECODE`, `PYTHONUNBUFFERED`)

---

## Why this matters for production

Smaller images mean faster deployments, lower Docker Hub / container registry storage costs, and a reduced attack surface — fewer packages means fewer potential vulnerabilities. These optimizations are standard practice in any professional DevOps workflow.

---

## Author

**Nadir** — DevOps Engineer specializing in Docker, CI/CD pipelines, and deployment automation.

[Upwork Profile](https://www.upwork.com) · [Docker Hub](https://hub.docker.com)
