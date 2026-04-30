# Deployment Guide

## Docker Deployment

```bash
docker-compose -f docker/docker-compose.prod.yml up -d
```

## Kubernetes Deployment

```bash
kubectl apply -f k8s/
```

## Cloud Deployments

### AWS
- ECS + Bedrock
- RDS + ElastiCache + OpenSearch

### GCP
- GKE + Vertex AI
- Cloud SQL + Memorystore + Vertex Vector Search

### Azure
- AKS + Azure OpenAI
- Azure SQL + Redis + AI Search

See [deployment docs](docs/deployment/) for details.
