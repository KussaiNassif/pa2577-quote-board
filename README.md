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


## 7. Benefits, Challenges and Security
This section follows the components in the same order as section 2, and for each component it lists the benefit of the design, the challenges, what is already done in this repository and what can be done next. 

### gateway (`k8s/gateway.yaml`)
**Benefit:** one external entry point (NodePort 30080). The browser only needs one address, and quotes-api never has to call stats-api, so there is no cyclic dependency.
- **Challenge:** all traffic passes through it, so it is a single point of failure and a possible bottleneck.
  - *Done:* stateless nginx with 2 replicas and readiness/liveness probes on `/healthz`. `proxy_connect_timeout 2s` on `/api/stats` means that if stats-api is down, the gateway returns an error quickly instead of hanging, and the quotes keep working (can be checked with `kubectl scale deploy/stats-api -n quoteboard --replicas=0`).
  - *Next:* a HorizontalPodAutoscaler, or a managed load balancer/Ingress at a cloud provider.
- **Security:** there is no TLS and no authentication, so anyone who can reach port 30080 can read and post quotes over plain HTTP. The gateway uses the stock `nginx` image, which starts as root, and has no resource limits.
  - *Done:* the gateway is the only Service of `type: NodePort`; everything else is ClusterIP and only reachable inside the cluster.
  - *Next:* terminate TLS and add authentication and rate limiting here, since this is the one place all traffic passes. Use an unprivileged nginx image and add resource limits.
### quotes-api (`quotes-api/`, `k8s/quotes-api.yaml`)
- **Benefit:** the only service with database access (Database-per-Service). The data model can change without affecting stats-api as long as the REST API stays the same.
- **Challenge:** MongoDB can start slowly, and quotes-api cannot work without it.
  - *Done:* `server.js` retries the connection every 3 s instead of crashing, and `/healthz` only returns 200 when the database is connected, so Kubernetes does not send traffic to it before it is ready. `/livez` is separate so that a database problem does not make Kubernetes restart the API.
- **Security:** user input is stored and later shown to other users.
  - *Done:* `POST /api/quotes` only accepts `text` and `author` as strings (this also blocks NoSQL operator injection such as `{"$gt": ""}`), with a max length of 500/120 characters, a 10 kB body limit (`413`) and `400` on invalid data. `limit`/`skip` are clamped. The web UI (`public/app.js`) escapes all user text (`escapeHtml`) against XSS. The container runs as a non-root user (`USER app` in the Dockerfile) with CPU/memory limits.
  - *Next:* rate limiting against spam, and a Content-Security-Policy header.

### stats-api (`stats-api/`, `k8s/stats-api.yaml`)
- **Benefit:** no database access, so it can be scaled independently of quotes-api (e.g. `--replicas=4`) when many users read statistics.
- **Challenge:** it reads all quotes from quotes-api over REST every time the number of quotes changes. With a lot of data this becomes slow and puts load on quotes-api.
  - *Done:* the result is cached until the quote count changes; quotes are fetched in pages of 500 with a 3 s timeout per request (at most 200 pages = 100,000 quotes; above that the statistics are incomplete).
  - *Next:* let quotes-api publish an event for each new quote (a message queue) so that stats-api can update its statistics incrementally.
- **Security:** non-root user (`USER app`) and resource limits, same as quotes-api. It is ClusterIP, so from outside the cluster it is only reachable through the gateway.

### mongo (`k8s/mongo.yaml`)
- **Benefit:** a StatefulSet with a 1Gi PersistentVolumeClaim, so the data survives restarts of pods and nodes.
- **Challenge:** only one instance (based on the assignment), so the database is a single point of failure. A PVC is not a backup.
  - *Next:* a MongoDB replica set and scheduled backups.
- **Security:** the password is generated at deploy time and stored in a Kubernetes Secret, so it is never in git (section 5). However, quotes-api connects with the root account, and since there is no NetworkPolicy, any pod in the cluster can reach MongoDB on port 27017 (it still needs the password).
  - *Next:* a least-privilege MongoDB user for quotes-api, a NetworkPolicy that only lets quotes-api reach MongoDB, and encryption of Secrets at rest.

### Overall
- **Business view:** each service can be scaled separately, so in a cloud you would only pay for more capacity where it is actually needed, instead of scaling the whole application.
- **No autoscaling yet:** CPU requests are already set for quotes-api and stats-api, which a HorizontalPodAutoscaler uses; it would also need metrics-server, which is not installed in kind by default.
- **The application is too small to need scaling:** to make sense of it, pretend there are many users posting quotes (scale quotes-api) and many more reading statistics (scale stats-api).

Thank you for reading, Kussai Nassif
