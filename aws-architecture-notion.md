# 🧱 Odyn Network — AWS Infrastructure

> **Branch:** `feat/ray-integration-testing-updates2` · **Phase 1** · **Generated:** 2026-04-18
> 
> Distributed GPU orchestration platform. Control plane on AWS, inference workers on multi-cloud GPU nodes, connected via Tailscale VPN overlay.

---

## 📌 Quick Reference

| Layer | Technology | Key Ports |
|---|---|---|
| API Server | Go + Gin | `:3001` |
| gRPC | NodeRegistry | `:50051` |
| Database | PostgreSQL 17 | `:5432` |
| Reverse Proxy | FRP (frps/frpc) | `:7001` / `:8080` |
| Ray GCS | Ray 2.54.0 | `:6379` |
| Ray Serve | vLLM (OpenAI-compat) | `:8000` |
| Ray Dashboard | Web UI | `:8265` |
| Health API | FastAPI | `:8002` |
| Prometheus | Metrics (federated) | `:9090` |
| Grafana | Dashboards | `:3000` |
| Jupyter | Workspace (tunnelled) | `:8888` |
| Terminal | ttyd (tunnelled) | `:7681` |

---

## ☁️ AWS Region — `us-west-2`

> Services: EC2 · EBS · EIP · ECR · Security Groups · Terraform-managed

---

### 🖥️ Control Plane EC2

**Public IP:** `35.93.57.67` · Ubuntu 22.04 LTS · Elastic IP allocated

#### Go + Gin API Server

- **Port:** `:3001`
- JWT authentication — 24h token lifetime
- Issuer / Audience: `https://odyn.network`
- Embedded FRP server on `:7001` (TCP bind) and `:8080` (HTTP vhost)

**Key routes:**

| Method | Route | Purpose |
|---|---|---|
| `POST` | `/api/clusters` | Create Ray cluster |
| `DELETE` | `/api/clusters/:id` | Destroy cluster |
| `POST` | `/v1/clusters/:id/chat/completions` | Inference (OpenAI-compat) |
| `GET` | `/api/supplier/dashboard` | Supplier earnings + machines |
| `POST` | `/api/nodes/:id/containers` | Start container on GPU node |
| `GET` | `/api/proxy/:containerId/jupyter/*` | Jupyter reverse proxy |
| `GET` | `/api/proxy/:containerId/terminal/*` | Terminal reverse proxy |

---

#### gRPC Server — NodeRegistry

- **Port:** `:50051` · Protobuf · Bidirectional streaming

| RPC | Direction | Purpose |
|---|---|---|
| `Register()` | Node → CP | Node initialisation |
| `StreamHeartbeat()` | Bidirectional | Continuous telemetry |
| `ConnectControlPlane()` | CP → Node | Command dispatch |
| `ReportCommandStatus()` | Node → CP | Command results |
| `ReportContainerAccessState()` | Node → CP | SSH/access readiness |
| `ConnectDataTunnel()` | Bidirectional | Raw TCP reverse tunnel |

---

#### PostgreSQL 17

- **Port:** `:5432` · DB: `marketplace`
- Volume: **EBS 30GB gp3**
- 20+ migrations in `/control-plane/internal/assets/migrations/`

| Table | Purpose |
|---|---|
| `clusters` | Ray cluster definitions (head_address, serve_port, model_name, status) |
| `nodes` | GPU host inventory |
| `containers` | Container lifecycle |
| `container_telemetry` | Performance metrics |
| `earnings` | Supplier revenue tracking |
| `sessions` | User sessions + access tokens |

---

#### FRP Server (frps) — Embedded

- **TCP bind:** `:7001` · **HTTP vhost:** `:8080`
- Token auth: `FRP_TOKEN` env var
- Routes Jupyter / Terminal WebSocket sessions from GPU nodes back to buyers

---

#### Ray Cluster Manager

- Manages Ray cluster lifecycle via PostgreSQL state
- Syncs with Ray Head via HTTP (`ODYN_RAY_HEAD_SYNC_TOKEN`)
- Triggers worker add/remove via control-plane HTTP API

---

#### ☁️ Multi-Cloud GPU Provider Clients

Acquires GPU capacity across 9 providers. Each has a configurable timeout env var.

| Provider | Notes |
|---|---|
| Lambda.ai | `LAMBDA_TIMEOUT=30s` |
| Runpod | `RUNPOD_TIMEOUT=30s` |
| Scaleway | European cloud |
| Nebius | GPU-focused |
| TensorDock | GPU marketplace |
| Crusoe | GPU compute |
| Verda | GPU hosting |
| Hetzner | Bare metal |
| Vultr | Cloud provider |

---

#### 🔐 Security Group — `odyn-controlplane`

| Direction | Port | Source | Purpose |
|---|---|---|---|
| Inbound | `22` | `0.0.0.0/0` | SSH |
| Inbound | `3001` | `0.0.0.0/0` | HTTP API |
| Inbound | `50051` | `0.0.0.0/0` | gRPC |
| Inbound | `7001` | `0.0.0.0/0` | FRP TCP |
| Inbound | `8080` | `0.0.0.0/0` | FRP HTTP vhost |
| Outbound | All | `0.0.0.0/0` | Unrestricted |

---

### 🌿 Ray Head Node EC2

**Tailscale IP:** `100.102.64.114` · Python 3.10.12-slim · Ray 2.54.0

#### Ray Core — Global Control Store (GCS)

- **Port:** `:6379`
- All workers join via: `ray start --address=100.102.64.114:6379 --node-ip-address=<WORKER_TAILSCALE_IP>`
- Manages cluster membership, object store, scheduling

---

#### 🚀 Ray Serve + vLLM Deployment

- **Port:** `:8000` · OpenAI-compatible `/v1/chat/completions`
- `vLLMDeployment` class spawns vLLM subprocess on `:9000`
- Replicas: `NUM_REPLICAS` env var (configurable)

| Config | Value |
|---|---|
| `GPU_MEMORY_UTILIZATION` | `0.85–0.90` |
| `TENSOR_PARALLEL_SIZE` | Configurable |
| `KV_CACHE_FRACTION` | `0.90` |

**Supported models:**

- `meta-llama/Llama-3.1-8B-Instruct`
- `Qwen/Qwen2.5-32B-Instruct`
- `sentence-transformers/all-MiniLM-L6-v2` (embeddings)

---

#### 📈 Autoscaler

- **Env:** `ODYN_AUTOSCALE_ENABLED=true`
- Poll interval: `30s`
- Scale **up** trigger: queue depth `> 5.0`
- Scale **down** trigger: queue depth `< 0.5`
- Min workers: `ODYN_MIN_WORKERS` (default `1`)
- Max workers: `ODYN_MAX_WORKERS` (default `10`)

---

#### 🏥 FastAPI Health Server

- **Port:** `:8002`
- Proxies: Ray Dashboard (`:8265`), Grafana (`:3000`), Prometheus (`:9090`)
- Ray Metrics Export: `:8081`
- File SD: `/tmp/ray/prom_metrics_service_discovery.json`

---

#### Ray Dashboard

- **Port:** `:8265`
- Job submission · Cluster monitoring
- Auto-generates Prometheus session dashboards to `/tmp/ray`

---

#### 📦 ECR — Elastic Container Registry

```
123456789012.dkr.ecr.us-east-1.amazonaws.com/runtime:test
```

- Env: `ODYN_RUNTIME_IMAGE_REF`
- Distributes node-agent runtime and vLLM worker images
- Used for auto-updates on GPU nodes

---

### 📊 Monitoring Stack EC2 — Terraform-managed

**Instance type:** `t3.small` · EIP allocated · `30GB gp3 EBS` · Security group: `odyn-monitoring`

> **Terraform location:** `/monitoring/aws/terraform/`
> Files: `main.tf`, `variables.tf`, `outputs.tf`

#### Prometheus

- **Port:** `:9090` · Retention: `30 days`
- Mode: **federated scrape** across all nodes

| Scrape Target | Address | Metrics |
|---|---|---|
| Ray H100 federation | `189.1.169.93:9090` | Ray + GPU |
| vLLM A100 chat | `202.78.161.249:9001` | Inference (socat relay) |
| vLLM A100 embeddings | `202.78.161.249:8001` | Embedding model |
| Self | `localhost:9090` | Monitoring node |

> GPU nodes must allow monitoring EC2 EIP on port `:9090` via UFW

---

#### Grafana

- **Port:** `:3000`
- Anonymous access enabled (no login required)
- Datasource: Prometheus (auto-provisioned)
- Dashboards provisioned from `/monitoring/aws/grafana/provisioning/dashboards/`
- Persistent volume: `/var/lib/grafana` on EBS

---

#### Terraform Outputs

| Output | Value |
|---|---|
| `ec2_public_ip` | Elastic IP address |
| `ec2_instance_id` | Instance ID |
| `grafana_url` | `http://<EIP>:3000` |
| `prometheus_url` | `http://<EIP>:9090` |

---

## 🔐 Tailscale VPN Overlay — `100.64.0.0/10`

> Replaces AWS private subnets for Ray inter-node communication. All Ray GCS traffic flows over Tailscale — AWS internal IPs (`172.31.x.x`) are unreachable from external GPU workers.

| Node | Tailscale IP | Role |
|---|---|---|
| Ray Head | `100.102.64.114` | GCS `:6379` · Dashboard `:8265` |
| A100 Worker | `100.92.148.18` | Chat `:9001` · Embeddings `:8001` |
| H100 / DGX Spark | `202.78.161.249` | Inference `:8000` · Prometheus `:9090` |

New workers enrol in Tailscale automatically via `install_odyn_node.sh`.

---

## 🖥️ GPU Worker Nodes — Multi-cloud / On-prem

> Node-Agent (Go daemon) · Docker runtime · `HostNetwork=true` · XFS workspace quotas

---

### Node Agent (Go Daemon)

Runs on every GPU host. Installed via `install_odyn_node.sh`.

- **State directory:** `/var/lib/odyn`
- **Install root:** `/opt/odyn`
- **Active version symlink:** `/opt/odyn/current`

| Responsibility | Detail |
|---|---|
| Docker container lifecycle | Create, start, stop, remove GPU containers |
| gRPC client | → Control Plane `:50051` — heartbeat + command stream |
| FRP client (frpc) | → Control Plane `:7001` — Jupyter/Terminal tunnels |
| XFS quota management | Per-project workspace quotas via `xfs_quota` |
| Preflight validation | Bootstrap checks on startup |
| Ray worker spawn | `ray start --address=HEAD:6379 --node-ip-address=<TAILSCALE_IP>` |

---

### 🟢 A100 Worker Container

- **Base image:** `nvidia/cuda:12.4-devel-ubuntu22.04`
- **Host:** `gcore-a100` · 8× NVIDIA A100 GPUs
- **Mode:** `HostNetwork=true`

| Service | Port | Model |
|---|---|---|
| vLLM inference server | `:9000` | Qwen2.5-32B-Instruct |
| Chat relay (socat) | `:9001` | Proxies :9000 → external |
| Embeddings server | `:8001` | all-MiniLM-L6-v2 |
| Ray worker | joins `:6379` | — |

- Model cache: `/opt/odyn/models`
- Prometheus metrics relayed via socat

---

### 🔴 H100 / DGX Spark Container

- **Base image:** `nvidia/cuda:12.9.0-devel-ubuntu22.04`
- **Host:** `dgx-spark` · NVIDIA H100 Blackwell (`sm_121a`)
- **CUDA:** 12.9 · Triton JIT with ptxas
- **Mode:** `HostNetwork=true`

| Service | Port |
|---|---|
| vLLM inference | `:8000` |
| Ray Dashboard | `:8265` |
| Prometheus metrics | `:9090` (federated) |
| Ray worker | joins `:6379` |

- `GPU_MEMORY_UTILIZATION=0.85`
- `KV_CACHE_FRACTION=0.90`
- Triton PTXAS path: `/usr/local/cuda/bin/ptxas`
- Also runs local Prometheus + Grafana (`monitoring/docker-compose.yml`)

---

### 🟡 Runtime Container (Buyer Workspace)

- **Base image:** `nvidia/cuda:12.4.0-runtime-ubuntu22.04`
- Managed by Node Agent — one per buyer session

| Service | Port | Access |
|---|---|---|
| Jupyter Lab | `:8888` | Via FRP tunnel → `/api/proxy/:id/jupyter` |
| Terminal (ttyd) | `:7681` | Via FRP tunnel → `/api/proxy/:id/terminal` |

- Workspace: `/workspace` with XFS quota (configurable GB per project)
- PyTorch + CUDA tools pre-installed

---

## 🗂️ Repository Structure

```
phase1/
├── control-plane/          # Go · Gin · gRPC · FRP server · Ray manager
│   ├── internal/
│   │   ├── cloudproviders/ # Lambda, Runpod, Scaleway, Nebius, TensorDock…
│   │   ├── grpcserver/     # NodeRegistry gRPC service
│   │   ├── frp/            # Embedded frps
│   │   ├── handlers/       # HTTP handlers
│   │   ├── databases/      # PostgreSQL migrations
│   │   └── services/       # Business logic
│   ├── proto/              # Protobuf definitions
│   └── docker-compose.yml  # PostgreSQL 17-alpine
│
├── node-agent/             # Go daemon · Docker · gRPC client · frpc
│   ├── internal/
│   │   ├── docker/         # Container lifecycle
│   │   ├── frp/            # Embedded frpc
│   │   ├── grpc/           # NodeRegistry client
│   │   └── workspace/      # XFS quota management
│   ├── images/
│   │   ├── docker/         # Runtime Dockerfile (CUDA 12.4 + PyTorch + Jupyter)
│   │   └── vllm-a6000-amd64/
│   └── scripts/
│       └── install_odyn_node.sh
│
├── ray-head/               # Python · Ray 2.54.0 · vLLM · FastAPI
│   └── src/
│       ├── main.py         # Ray startup + cluster discovery + Serve deployment
│       ├── vllm_deployment.py
│       ├── autoscaler.py
│       ├── health.py
│       └── config.py
│
├── frontend/               # React + TypeScript SPA
│   └── src/
│       └── pages/          # Supplier dashboard, Buyer marketplace, Jupyter, Terminal
│
├── monitoring/
│   ├── aws/
│   │   ├── terraform/      # EC2 · SG · EIP · EBS
│   │   ├── docker-compose-aws.yml
│   │   └── prometheus-aws.yml
│   └── docker-compose.yml  # Local H100 Prometheus + Grafana
│
└── docker/
    ├── runtime-vllm-dgx-spark.Dockerfile  # H100 Blackwell
    └── runtime-vllm-a100.Dockerfile       # A100
```

---

## ⚙️ Key Environment Variables

### Control Plane

```env
DATABASE_URL=postgres://marketplace:marketplace@localhost:5432/marketplace
PORT=3001
GRPC_PORT=50051
FRP_BIND_PORT=7001
FRP_VHOST_HTTP_PORT=8080
FRP_TOKEN=secret007
ODYN_RAY_HEAD_SYNC_TOKEN=secret007
RAY_HEAD_GCS_PORT=6379
RAY_HEAD_DASHBOARD_PORT=8265
RAY_HEAD_SERVE_PORT=8000
RAY_HEAD_HEALTH_PORT=8002
RAY_WORKER_STORAGE_GB=200
```

### Ray Head

```env
RAY_HEAD_NODE_IP=100.102.64.114
RAY_HEAD_PORT=6379
MODEL_NAME=meta-llama/Llama-3.1-8B-Instruct
HF_TOKEN=<huggingface-token>
GPU_MEMORY_UTILIZATION=0.90
TENSOR_PARALLEL_SIZE=1
NUM_REPLICAS=1
ODYN_CONTROL_PLANE_URL=http://35.93.57.67:3001
ODYN_AUTOSCALE_ENABLED=false
ODYN_MIN_WORKERS=1
ODYN_MAX_WORKERS=10
```

### Node Agent

```env
ODYN_BACKEND_URL=http://<CONTROL_PLANE_HOST>:50051
ODYN_NODE_TOKEN=<NODE_RUNTIME_TOKEN>
ODYN_NODE_TAILSCALE_IP=100.92.148.18
ODYN_FRP_SERVER_ADDR=<CONTROL_PLANE_HOST>
ODYN_FRP_SERVER_PORT=7001
ODYN_FRP_TOKEN=secret007
```

---

## 🔄 Data & Request Flow

### Inference Request (Buyer → GPU)

```
Buyer HTTP request
  → Control Plane :3001  POST /v1/clusters/{id}/chat/completions
    → Ray Head :8000  (Ray Serve OpenAI endpoint)
      → vLLMDeployment (Ray actor)
        → vLLM subprocess :9000  (on GPU worker via Tailscale)
          → Model inference (A100 / H100)
        ← tokens streamed back
      ← SSE / JSON response
    ← proxied response
  ← buyer receives output
```

### Node Registration (Supplier → Platform)

```
Supplier runs install_odyn_node.sh on GPU host
  → Tailscale enrolled
  → Node Agent installed to /opt/odyn
  → Node Agent gRPC Register() → Control Plane :50051
  → StreamHeartbeat() loop starts (CPU, GPU, mem, disk telemetry)
  → FRP client connects → Control Plane :7001
  → Node visible in supplier dashboard
```

### Autoscaling Loop

```
Ray Head Autoscaler (30s poll)
  → Check queue depth
  → If > 5.0: POST /api/clusters/{id}/workers → Control Plane
    → Control Plane picks available GPU node from DB
    → gRPC ConnectControlPlane() → Node Agent
      → Node Agent starts Ray worker container (HostNetwork)
        → ray start --address=HEAD:6379
  → If < 0.5: remove worker via same path
```

---

## 🏷️ Tech Stack Summary

| Component | Language / Runtime | Version |
|---|---|---|
| Control Plane | Go | 1.21+ |
| Node Agent | Go | 1.21+ |
| Ray Head | Python | 3.10.12 |
| Ray | Ray[serve] | 2.54.0 |
| Inference | vLLM | Latest |
| Frontend | React + TypeScript | Vite |
| Database | PostgreSQL | 17-alpine |
| Containers | Docker | 24+ |
| IaC | Terraform | — |
| Networking | Tailscale VPN | — |
| Observability | Prometheus + Grafana | — |
| GPU (A100) | CUDA | 12.4 |
| GPU (H100) | CUDA | 12.9 (Blackwell) |
