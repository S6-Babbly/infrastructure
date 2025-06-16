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
   - Service Principal with appropriate permissions

2. **GitHub Secrets:**
   Set up the following secrets in your GitHub repository:
   ```
   AZURE_CREDENTIALS - Service principal credentials in JSON format:
   {
     "clientId": "your-client-id",
     "clientSecret": "your-client-secret", 
     "subscriptionId": "your-subscription-id",
     "tenantId": "your-tenant-id"
   }
   ```

3. **Update Workflow Variables:**
   Edit `.github/workflows/aks-deploy.yml` and update:
   - `AZURE_CONTAINER_REGISTRY`: Your ACR name (if using)
   - `RESOURCE_GROUP`: Your Azure resource group name
   - `CLUSTER_NAME`: Your AKS cluster name

## Deployment Order

The GitHub workflow deploys components in this order:

1. **Infrastructure Components:**
   - Secrets (Auth0, Cassandra credentials)
   - Zookeeper
   - Kafka 
   - Cassandra
   - Prometheus (monitoring)
   - Grafana (dashboards)

2. **Backend Services:**
   - Auth Service (port 5001)
   - User Service (port 8081)
   - Post Service (port 8080)
   - Comment Service (port 8082)
   - Like Service (port 8083)

3. **Gateway & Frontend:**
   - API Gateway (port 8080 → 5010)
   - Frontend (port 3000 → 30000)

## Manual Deployment

If you need to deploy manually:

```bash
# Deploy infrastructure
kubectl apply -f k8s/secrets.yaml
kubectl apply -f k8s/kafka-zookeeper.yaml
kubectl apply -f k8s/cassandra.yaml
kubectl apply -f k8s/prometheus.yaml
kubectl apply -f k8s/grafana.yaml

# Wait for infrastructure to be ready
kubectl wait --for=condition=available --timeout=300s deployment/kafka
kubectl wait --for=condition=ready --timeout=300s pod -l app=cassandra
kubectl wait --for=condition=available --timeout=300s deployment/prometheus
kubectl wait --for=condition=available --timeout=300s deployment/grafana

# Deploy services
kubectl apply -f ../babbly-auth-service/k8s/
kubectl apply -f ../babbly-user-service/k8s/
kubectl apply -f ../babbly-post-service/k8s/
kubectl apply -f ../babbly-comment-service/k8s/
kubectl apply -f ../babbly-like-service/k8s/
kubectl apply -f ../babbly-api-gateway/k8s/
kubectl apply -f ../babbly-frontend/k8s/
```

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

### Check pod status:
```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### Check service connectivity:
```bash
kubectl get services
kubectl get endpoints
```

### Check Kafka topics:
```bash
kubectl exec -it deployment/kafka -- kafka-topics --bootstrap-server localhost:9092 --list
```

### Check Cassandra keyspaces:
```bash
kubectl exec -it cassandra-0 -- cqlsh -e "DESCRIBE KEYSPACES;"
```

### Check monitoring stack:
```bash
# Check Prometheus targets
kubectl port-forward service/prometheus 9090:9090
# Then visit http://localhost:9090/targets

# Check Grafana dashboards
kubectl port-forward service/grafana 3000:3000
# Then visit http://localhost:3000 (admin/admin123)

# Check Prometheus metrics collection
kubectl logs deployment/prometheus

# Check Grafana logs
kubectl logs deployment/grafana
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