# Babbly Infrastructure - AKS Deployment

This directory contains the Kubernetes manifests and GitHub workflow for deploying the Babbly application to Azure Kubernetes Service (AKS).

## Architecture Overview

The application uses the following ports and services:

| Service | Internal Port | External Port | Type |
|---------|---------------|---------------|------|
| Frontend | 3000 | 30000 (NodePort) | NodePort |
| API Gateway | 8080 | 5010 | LoadBalancer |
| Auth Service | 5001 | - | ClusterIP |
| User Service | 8081 | - | ClusterIP |
| Post Service | 8080 | - | ClusterIP |
| Comment Service | 8082 | - | ClusterIP |
| Like Service | 8083 | - | ClusterIP |
| Cassandra | 9042 | - | Headless |
| Kafka | 9092 | - | ClusterIP |
| Zookeeper | 2181 | - | ClusterIP |
| **Monitoring** |  |  |  |
| Prometheus | 9090 | 30090 (NodePort) | NodePort |
| Grafana | 3000 | 30030 (NodePort) | NodePort |

## Prerequisites

1. **Azure Resources:**
   - Azure Kubernetes Service (AKS) cluster
   - Azure Container Registry (ACR) (optional, using Docker Hub currently)

2. **GitHub Secrets:**
   Set up the following secret in your GitHub repository:
   ```
   KUBE_CONFIG_DATA - Base64 encoded kubeconfig file content
   ```
   
   To get your kubeconfig:
   ```bash
   # Get AKS credentials
   az aks get-credentials --resource-group your-resource-group --name your-aks-cluster
   
   # Base64 encode the config
   cat ~/.kube/config | base64 | tr -d '\n'
   ```
   
   Copy the output and paste it as the `KUBE_CONFIG_DATA` secret value in GitHub.

## Deployment Approach

**Infrastructure Workflow (this repo):**
The GitHub workflow deploys ONLY infrastructure components:
- Secrets (Auth0, Cassandra credentials)
- Zookeeper
- Kafka 
- Cassandra
- Prometheus (monitoring)
- Grafana (dashboards)

**Microservices (separate workflows):**
Each microservice has its own CI/CD pipeline in their respective repositories:
- Auth Service (port 5001) - deployed via `babbly-auth-service/release.yml`
- User Service (port 8081) - deployed via `babbly-user-service/release.yml`
- Post Service (port 8080) - deployed via `babbly-post-service/release.yml`
- Comment Service (port 8082) - deployed via `babbly-comment-service/release.yml`
- Like Service (port 8083) - deployed via `babbly-like-service/release.yml`
- API Gateway (port 8080 → 5010) - deployed via `babbly-api-gateway/release.yml`
- Frontend (port 3000 → 30000) - deployed via `babbly-frontend/release.yml`

## Manual Infrastructure Deployment

If you need to deploy infrastructure manually (run from repository root):

```bash
# Deploy infrastructure only
kubectl apply -f infrastructure/k8s/secrets.yaml --namespace=default
kubectl apply -f infrastructure/k8s/kafka-zookeeper.yaml --namespace=default
kubectl apply -f infrastructure/k8s/cassandra.yaml --namespace=default
kubectl apply -f infrastructure/k8s/prometheus.yaml --namespace=default
kubectl apply -f infrastructure/k8s/grafana.yaml --namespace=default

# Wait for infrastructure to be ready
kubectl wait --for=condition=available --timeout=300s deployment/kafka --namespace=default

# Cassandra takes longer to start, so wait for StatefulSet first
kubectl wait --for=jsonpath='{.status.readyReplicas}'=1 --timeout=300s statefulset/cassandra --namespace=default
kubectl wait --for=condition=ready --timeout=600s pod -l app=cassandra --namespace=default

kubectl wait --for=condition=available --timeout=300s deployment/prometheus --namespace=default
kubectl wait --for=condition=available --timeout=300s deployment/grafana --namespace=default
```

**Note:** Microservices are deployed separately via their individual CI/CD workflows when code changes are pushed to each service repository.

## Accessing the Application

After deployment:

1. **Get external IPs:**
   ```bash
   kubectl get services
   ```

2. **Frontend:** Access via NodePort on any node IP:30000
3. **API Gateway:** Access via LoadBalancer external IP:5010
4. **Prometheus:** Access via NodePort on any node IP:30090
5. **Grafana:** Access via NodePort on any node IP:30030
   - **Username:** admin
   - **Password:** admin123

## Configuration Notes

### Database Configuration
- **PostgreSQL:** Using Neon DB (external cloud service)
- **Cassandra:** Deployed in cluster with persistent storage
- **Keyspaces:** babbly_posts, babbly_comments, babbly_likes

### Message Queue
- **Kafka Topics:** 
  - user-events
  - auth-events  
  - post-events
  - comment-events
  - like-events

### Service Discovery
Services communicate using Kubernetes DNS:
- `auth-service:5001`
- `user-service:8081` 
- `post-service:8080`
- `comment-service:8082`
- `like-service:8083`
- `kafka:9092`
- `cassandra:9042`

### Monitoring Stack
- **Prometheus:** Collects metrics from all Babbly services and Kubernetes cluster
  - Configured to scrape `/metrics` endpoints from all microservices
  - Monitors Kubernetes nodes, pods, and services
  - 8GB persistent storage with 200h retention
  
- **Grafana:** Provides visualization dashboards
  - Pre-configured Prometheus datasource
  - Built-in dashboards:
    - **Babbly Application Overview:** Service health, HTTP request rates, response times
    - **Kubernetes Cluster Overview:** Pod status, CPU/memory usage
  - Supports custom dashboard creation

## Troubleshooting

### Common Issues

**Cassandra Startup Issues:**
If Cassandra readiness probe fails after configuration changes:
```bash
# Check Cassandra status
kubectl get statefulset cassandra --namespace=default
kubectl get pods -l app=cassandra --namespace=default

# If probe config didn't update, restart the pod
kubectl delete pod cassandra-0 --namespace=default

# Wait for it to come back up
kubectl wait --for=condition=ready --timeout=600s pod -l app=cassandra --namespace=default

# Check logs if still having issues
kubectl logs cassandra-0 --namespace=default --tail=100
```

### Check pod status:
```bash
kubectl get pods --namespace=default
kubectl describe pod <pod-name> --namespace=default
kubectl logs <pod-name> --namespace=default
```

### Check service connectivity:
```bash
kubectl get services --namespace=default
kubectl get endpoints --namespace=default
```

### Check Kafka topics:
```bash
kubectl exec deployment/kafka --namespace=default -- kafka-topics --bootstrap-server localhost:9092 --list
```

### Check Cassandra keyspaces:
```bash
kubectl exec cassandra-0 --namespace=default -- cqlsh -e "DESCRIBE KEYSPACES;"
```

### Check monitoring stack:
```bash
# Check Prometheus targets
kubectl port-forward service/prometheus 9090:9090 --namespace=default
# Then visit http://localhost:9090/targets

# Check Grafana dashboards
kubectl port-forward service/grafana 3000:3000 --namespace=default
# Then visit http://localhost:3000 (admin/admin123)

# Check Prometheus metrics collection
kubectl logs deployment/prometheus --namespace=default

# Check Grafana logs
kubectl logs deployment/grafana --namespace=default
```

## Development Environment

All services are configured for development environment:
- `ASPNETCORE_ENVIRONMENT=Development`
- Detailed logging enabled
- CORS configured for local development
- Default Cassandra credentials (cassandra/cassandra)

## Metrics Requirements

For Prometheus monitoring to work properly, each microservice should expose metrics at `/metrics` endpoint. Consider adding:
- ASP.NET Core health checks (`/health`)
- Prometheus metrics middleware
- Custom business metrics (posts created, users registered, etc.)
- HTTP request/response metrics
- Database connection metrics

## Security Notes

⚠️ **For production deployment:**
- Change default Cassandra credentials
- Change default Grafana admin password (currently: admin123)
- Use proper TLS certificates
- Configure network policies
- Use Azure Key Vault for secrets
- Enable RBAC and pod security policies
- Restrict Prometheus scraping permissions
- Configure Grafana OAuth integration 