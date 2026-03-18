# Agent OS — Kubernetes Template

Deploy [Agent OS](https://github.com/agno-agi/agno) on Kubernetes.

This is a self-contained template: it includes the application code, Dockerfile, and Kubernetes manifests. Clone it, add your API key, and deploy. Defaults to **Docker Desktop Kubernetes** for local development.

## Prerequisites

- Docker Desktop with Kubernetes enabled (Settings > Kubernetes > Enable Kubernetes)
- `kubectl` configured (Docker Desktop does this automatically)
- API key for your model provider (e.g., OpenAI)

Verify your cluster is running:

```bash
kubectl get nodes
# Should show: docker-desktop   Ready
```

## Quick Start

### 1. Clone this repo

```bash
git clone https://github.com/agno-agi/agentos-k8s-template.git
cd agentos-k8s-template
```

### 2. Build the Docker image locally

Docker Desktop shares images between Docker and its Kubernetes cluster, so there is no need to push to a registry.

```bash
docker build -t agentos:latest .
```

### 3. Add your API key

```bash
# Generate the base64-encoded value
echo -n "sk-your-actual-openai-key" | base64
```

Edit `k8s/secrets.yaml` and replace the `OPENAI_API_KEY` value with your encoded key.

### 4. Deploy

```bash
kubectl apply -k k8s/
```

### 5. Wait for pods

```bash
kubectl get pods -n agentos -w

# Wait until both show Running:
#   agentos-db-0       1/1   Running
#   agentos-xxxxx      1/1   Running
```

### 6. Open Agent OS

```bash
open http://localhost:30080/docs
```

No port-forward needed — the service is exposed on NodePort 30080.

## Repository Structure

```
agentos-k8s-template/
├── Dockerfile              # Container image definition
├── requirements.txt        # Python dependencies
├── pyproject.toml          # Project metadata
├── example.env             # Example environment variables
├── app/                    # FastAPI application (AgentOS entrypoint)
│   ├── main.py
│   └── config.yaml
├── agents/                 # Agent definitions
│   ├── knowledge_agent.py
│   └── mcp_agent.py
├── db/                     # Database configuration
│   ├── session.py
│   └── url.py
├── scripts/                # Build and entrypoint scripts
│   ├── entrypoint.sh
│   ├── build_image.sh
│   └── ...
└── k8s/                    # Kubernetes manifests
    ├── namespace.yaml      # Isolated 'agentos' namespace
    ├── configmap.yaml      # App configuration (DB host, runtime env)
    ├── secrets.yaml        # API keys and DB credentials (base64-encoded)
    ├── postgres.yaml       # PostgreSQL + PgVector StatefulSet
    ├── agentos.yaml        # Agent OS Deployment + NodePort Service
    ├── ingress.yaml        # Ingress for production (optional)
    ├── hpa.yaml            # Horizontal Pod Autoscaler (optional)
    └── kustomization.yaml  # Apply everything with one command
```

## Common Commands

```bash
# View logs
kubectl logs -n agentos -l app=agentos -f

# View database logs
kubectl logs -n agentos -l app=agentos-db -f

# Get pod status
kubectl get pods -n agentos

# Describe a pod (useful for debugging)
kubectl describe pod -n agentos <pod-name>

# Open a shell in the Agent OS pod
kubectl exec -it -n agentos deploy/agentos -- bash

# Connect to the database
kubectl exec -it -n agentos agentos-db-0 -- psql -U ai -d ai
```

## Tear Down

```bash
# Remove all resources
kubectl delete -k k8s/

# Also delete persistent volume data
kubectl delete pvc -n agentos --all
```

## Production Deployment

The default configuration targets Docker Desktop for local development. For production:

### 1. Use a container registry

Update `k8s/agentos.yaml`:

```yaml
image: your-registry/agentos:v1
imagePullPolicy: Always  # change from Never
```

### 2. Use a managed database

Replace the PostgreSQL StatefulSet with a managed service (RDS, Cloud SQL, AlloyDB, etc.) and update `k8s/configmap.yaml` with the connection details.

### 3. Enable Ingress

Uncomment `ingress.yaml` in `k8s/kustomization.yaml`. Install an ingress controller and update the host in `k8s/ingress.yaml`.

### 4. Enable autoscaling

Uncomment `hpa.yaml` in `k8s/kustomization.yaml`. Increase `replicas` in `k8s/agentos.yaml`.

### 5. Secure your secrets

Replace plain Kubernetes secrets with a secrets manager (HashiCorp Vault, AWS Secrets Manager, Sealed Secrets, etc.).

### 6. Increase resources

Bump the CPU and memory limits in `k8s/agentos.yaml` and `k8s/postgres.yaml` to match your workload.

## Customizing Your Agents

Edit the files in `agents/` to define your own agents. The template includes two examples:

- **Knowledge Agent** (`agents/knowledge_agent.py`) — RAG agent with PgVector knowledge base
- **MCP Agent** (`agents/mcp_agent.py`) — Agent with MCP tool access

Register your agents in `app/main.py` and rebuild the image:

```bash
docker build -t agentos:latest .
kubectl rollout restart deployment/agentos -n agentos
```

## Related

- [Agno](https://github.com/agno-agi/agno) — The agent framework
- [Agent OS Docker Template](https://github.com/agno-agi/agentos-docker-template) — Deploy Agent OS with Docker Compose
