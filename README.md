# Babbly Infrastructure - Kubernetes Deployment

This directory contains the Kubernetes manifests and GitHub workflow for deploying the Babbly application infrastructure to Kubernetes clusters.

## Architecture Overview

The infrastructure includes the following components:

| Component | Purpose | Exposure |
|-----------|---------|----------|
| **Application Services** |  |  |
| Frontend | User interface | External (NodePort) |
| API Gateway | Request routing | External (LoadBalancer) |
| Auth Service | Authentication | Internal |
| User Service | User management | Internal |
| Post Service | Post management | Internal |
| Comment Service | Comment management | Internal |
| Like Service | Like management | Internal |
| **Data Layer** |  |  |
| Cassandra | NoSQL database | Internal |
| Kafka | Message broker | Internal |
| Zookeeper | Kafka coordination | Internal |
| **Monitoring** |  |  |
| Prometheus | Metrics collection | External (NodePort) |
| Grafana | Monitoring dashboards | External (NodePort) |

## Prerequisites

1. **Kubernetes Cluster:**
   - Any Kubernetes cluster (AKS, EKS, GKE, or local)
   - Sufficient resources for all components

2. **GitHub Secrets (for automated deployment):**
   Set up the following secret in your GitHub repository:
   ```
   KUBE_CONFIG_DATA - Base64 encoded kubeconfig file content
   ```
   
   To get your kubeconfig:
   ```bash
   # Get cluster credentials (example for AKS)
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
- Auth Service - deployed via `babbly-auth-service/release.yml`
- User Service - deployed via `babbly-user-service/release.yml`
- Post Service - deployed via `babbly-post-service/release.yml`
- Comment Service - deployed via `babbly-comment-service/release.yml`
- Like Service - deployed via `babbly-like-service/release.yml`
- API Gateway - deployed via `babbly-api-gateway/release.yml`
- Frontend - deployed via `babbly-frontend/release.yml`

## Manual Infrastructure Deployment

If you need to deploy infrastructure manually (run from repository root):

```bash
# Deploy infrastructure components
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

1. **Get service information:**
   ```bash
   kubectl get services
   ```

2. **Frontend:** Access via NodePort or configured ingress
3. **API Gateway:** Access via LoadBalancer external IP or ingress
4. **Prometheus:** Access via NodePort or port-forward
5. **Grafana:** Access via NodePort or port-forward
   - **Username:** admin
   - **Password:** ${GRAFANA_ADMIN_PASSWORD}

## Configuration Notes

### Database Configuration
- **PostgreSQL:** Using external cloud service (configurable)
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
- `auth-service`
- `user-service` 
- `post-service`
- `comment-service`
- `like-service`
- `kafka`
- `cassandra`

### Monitoring Stack
- **Prometheus:** Collects metrics from all Babbly services and Kubernetes cluster
  - Configured to scrape `/metrics` endpoints from all microservices
  - Monitors Kubernetes nodes, pods, and services
  - Persistent storage with configurable retention
  
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
kubectl port-forward service/prometheus ${PROMETHEUS_PORT}:${PROMETHEUS_PORT} --namespace=default
# Then visit http://localhost:${PROMETHEUS_PORT}/targets

# Check Grafana dashboards
kubectl port-forward service/grafana ${GRAFANA_PORT}:${GRAFANA_PORT} --namespace=default
# Then visit http://localhost:${GRAFANA_PORT} (admin/${GRAFANA_ADMIN_PASSWORD})

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
- Default database credentials (change for production)

## Metrics Requirements

For Prometheus monitoring to work properly, each microservice should expose metrics at `/metrics` endpoint. Consider adding:
- ASP.NET Core health checks (`/health`)
- Prometheus metrics middleware
- Custom business metrics (posts created, users registered, etc.)
- HTTP request/response metrics
- Database connection metrics

## Security Notes

⚠️ **For production deployment:**
- Change all default credentials before deployment
- Use strong passwords and rotate them regularly
- Use proper TLS certificates
- Configure network policies
- Use secure secret management (Azure Key Vault, AWS Secrets Manager, etc.)
- Enable RBAC and pod security policies
- Restrict Prometheus scraping permissions
- Configure Grafana OAuth integration
- Implement proper authentication and authorization
- Regular security audits and updates 