# Quote Board — PA2577 "Build Something"

**Quote Board** is a small Kubernetes application for collecting short quotes and presenting statistics from the collected data. quotes-api owns the quote data and is responsible for persistence. stats-api performs statistical analysis through REST calls to quotes-api.

## 2. Architecture
| Component | Responsibility |
|---|---|
| **gateway** | Single external HTTP entry point (NodePort 30080). Routes /api/stats to stats-api and all other traffic to quotes-api |
| **quotes-api** | The sole owner of persistent quote data. Serves the web UI and provides the quote REST API. Exposed internally via ClusterIP |
| **stats-api** | REST API `GET /api/stats`. No database access: fetches quotes from quotes-api over REST and caches the result until the count changes. ClusterIP. |
| **mongo** | StatefulSet with a 1Gi PVC, so data survives restarts. One instance (allowed). |

```
Browser
   |
   v
gateway (:30080)
   |--------------------> quotes-api -----> MongoDB (PVC)
   |
   +--------------------> stats-api ------> quotes-api
```

## 3. Repository Structure
```
├── quotes-api/              # Quotes service: server.js, public/ (web UI), Dockerfile
├── stats-api/               # Stats service: server.js, Dockerfile
├── k8s/
│   ├── namespace.yaml       # Namespace definition
│   ├── mongo.yaml           # Database persistence (StatefulSet + PVC)
│   ├── quotes-api.yaml      # Quotes service manifests
│   ├── stats-api.yaml       # Stats service manifests
│   └── gateway.yaml         # Nginx entry point
├── kind-cluster.yaml        # Local cluster configuration
└── README.md                # Project documentation
```

**Why a separate gateway:** letting quotes-api forward stats would create a cycle (stats-api → quotes-api → stats-api). Cyclic dependency is a microservice anti-pattern, with an API-Gateway as the fix (Taibi, Lenarduzzi & Pahl, *Microservices Anti-Patterns: A Taxonomy*).

## 4. APIs
**quotes-api**
- `GET /api/quotes`: Retrieves a list of quotes.
- `POST /api/quotes`: Submits a new quote, validates input and returns 400 or 413 on bad requests.
- `GET /healthz`: Readiness probe: returns 200 only if the database is connected.
- `GET /livez`: Liveness probe that verifies the API is running, independently of the database.

**stats-api**
- `GET /api/stats`: Reads quote data from quotes-api, calculates statistics (count, average length, top words excluding stopwords, quotes per author), and caches the result until the quote count changes.
- `GET /healthz`: Readiness and liveness probe.

## 5. Deployment
**Prerequisites:** Docker, kubectl, kind (Kubernetes in Docker).

**Steps:** create the cluster, create the namespace, create the Mongo credentials (the Secret is never stored in git), apply the manifests, then open http://localhost:30080 in a browser.
```bash
kind create cluster --name quoteboard --config kind-cluster.yaml
kubectl apply -f k8s/namespace.yaml
kubectl -n quoteboard create secret generic mongo-credentials \
  --from-literal=username=root --from-literal=password="$(head -c 18 /dev/urandom | base64)"
kubectl apply -f k8s/
```

Verification: To verify a successful deployment, check that all expected pods are running with 0 restarts:
```bash
kubectl get pods -n quoteboard
kubectl get svc -n quoteboard
```

## 6. Evaluating Assignment Requirements

Requirement and How to Verify:
- Two different microservices: run `kubectl get deployments -n quoteboard` to see quotes-api and stats-api.
- REST communication: post a quote, then view stats-api logs (`kubectl -n quoteboard logs -l app=stats-api`) to observe it calling quotes-api via REST.
- Separate database: MongoDB is deployed as a separate StatefulSet workload.
- External access: The gateway is exposed via NodePort 30080.
- Independent scaling: run `kubectl scale deploy/stats-api -n quoteboard --replicas=4` and observe the scaling independent of quotes-api.
- Persistent storage: MongoDB utilizes a 1Gi PersistentVolumeClaim (PVC).
- Container images: Deployments reference public Docker Hub images (`kussainassif/quotes-api:1.0.2` and `kussainassif/stats-api:1.0.2`).
