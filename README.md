# Music Separation as a Service (MSaaS)

![Music separation](images/music_separation.png)

A small “music separation” microservice system built around **[Demucs](https://github.com/facebookresearch/demucs)** that:
1) accepts an MP3 via a REST API,  
2) queues a job in Redis,  
3) runs Demucs in a worker, and  
4) stores the separated stems in an S3-compatible object store (MinIO).

> **What you get:** vocals / drums / bass / other as downloadable MP3s.

---

## Architecture

**Components**
- **REST API (`rest/rest-server.py`)**: Flask server that receives MP3 (Base64), writes it to MinIO, and enqueues a job in Redis.
- **Redis (`redis/`)**: simple list queue (`toWorkers`) used by the worker.
- **Worker (`worker/worker-server.py`)**: pulls job IDs from Redis, downloads the MP3 from MinIO, runs Demucs, and uploads stems to MinIO.
- **MinIO (`minio/`)**: S3-compatible storage for input/output tracks.
- **Logs (`logs/`)**: a small pod that can subscribe to Redis and help with debugging.

**Data flow**
1. Client → `POST /apiv1/separate` with `{ mp3: <base64> }`
2. REST server stores `UUID.mp3` to MinIO bucket `input-tracks` and pushes `UUID` to Redis list `toWorkers`
3. Worker pops `UUID`, downloads `UUID.mp3`, runs Demucs, uploads:
   - `UUID-vocals.mp3`
   - `UUID-drums.mp3`
   - `UUID-bass.mp3`
   - `UUID-other.mp3`
   into MinIO bucket `output-tracks`

---

## API

Base URL is the REST service (default `http://localhost:5000` when port-forwarded).

### `POST /apiv1/separate`
**Body (JSON)**
```json
{
  "mp3": "<base64-encoded-mp3-bytes>",
  "callback": { "url": "http://example.com", "data": { "anything": "optional" } }
}
```

**Response**
```json
{ "songhash": "<uuid>", "message": "File uploaded to MinIO and task queued for processing" }
```

> Note: the current code accepts a `callback` field but does not execute callbacks yet.

### `GET /apiv1/queue`
Returns the current Redis queue contents (`toWorkers`).

### `GET /apiv1/bucket`
Lists MinIO buckets and object names (useful for debugging).

### `GET /apiv1/track/<track>`
Downloads an output MP3 from `output-tracks`.

In this project, the worker uploads stems as:
- `<songhash>-vocals.mp3`
- `<songhash>-drums.mp3`
- `<songhash>-bass.mp3`
- `<songhash>-other.mp3`

So to download vocals you would call:
- `GET /apiv1/track/<songhash>-vocals`

### `DELETE /apiv1/remove/<track>`
Deletes `<track>.mp3` from `output-tracks`.

### `GET /apiv1/delete_contents`
Deletes **all** objects in all buckets (debug endpoint — use carefully).

---

## Quickstart (Kubernetes)

### Prerequisites
- Kubernetes cluster (minikube / kind / Docker Desktop Kubernetes)
- `kubectl`
- Docker (only needed if you rebuild/push images)

### 1) Deploy everything
```bash
./deploy-all.sh
```

This applies:
- Redis
- REST server
- Worker
- Logs
- MinIO service

### 2) Port-forward the REST API
```bash
kubectl port-forward service/rest-server 5000:5000
```

Now the API is reachable at `http://localhost:5000`.

### 3) Send a sample request
From the repo root:
```bash
pip install -r rest/requirements.txt
python3 sample-requests.py
```

The response includes a `songhash`. After the worker finishes, download stems:
```bash
curl -L -o vocals.mp3 "http://localhost:5000/apiv1/track/<songhash>-vocals"
curl -L -o drums.mp3  "http://localhost:5000/apiv1/track/<songhash>-drums"
curl -L -o bass.mp3   "http://localhost:5000/apiv1/track/<songhash>-bass"
curl -L -o other.mp3  "http://localhost:5000/apiv1/track/<songhash>-other"
```

---

## Local development (edit code without rebuilding images)

There’s a helper script that deploys Redis/MinIO on Kubernetes and forwards ports to your laptop so you can run `rest-server.py` and/or `worker-server.py` locally:

```bash
./deploy-local-dev.sh
```

It forwards:
- Redis → `localhost:6379`
- MinIO → `localhost:9000` (API) and `localhost:9001` (console)

Then, in separate terminals you can run:
```bash
# REST server locally
cd rest
pip install -r requirements.txt
python3 rest-server.py
```

```bash
# Worker locally
cd worker
pip install -r requirements.txt
python3 worker-server.py
```

> Tip: when running locally, ensure your `MINIO_HOST` / `REDIS_HOST` environment variables match the forwarded endpoints.

---

## Project layout

```
.
├── rest/            # Flask REST API + Kubernetes manifests
├── worker/          # Demucs worker + Kubernetes deployment
├── redis/           # Redis deployment/service manifests
├── minio/           # MinIO config + service manifests
├── logs/            # optional logging pod
├── data/            # sample mp3 files (client tests)
└── sample-requests.py
```

---

## Troubleshooting

### MinIO namespace mismatch (`minio-ns`)
Some manifests reference `minio-ns` (e.g., the service name `minio-proj.minio-ns.svc.cluster.local`), but `minio/minio-deployment.yaml` is set to `namespace: default`.

If your cluster does not have the `minio-ns` namespace, create it and ensure MinIO runs there, **or** update manifests to keep everything in `default`.

Example to create the namespace:
```bash
kubectl create namespace minio-ns
```

### Ingress manifest port
`rest/rest-ingress.yaml` routes to service `rest-server` port `80`, but the REST service is defined on port **5000**.
If you plan to use Ingress, update the service/ingress ports so they match.

### Worker is slow / no GPU
The worker uses:
```py
device = 'cuda' if torch.cuda.is_available() else 'cpu'
```
On CPU-only clusters, separation can be slow (especially for long tracks).

---

## Notes
- This repo currently stores tracks in MinIO buckets `input-tracks` and `output-tracks`.
- The worker uses the **htdemucs** model preset.
