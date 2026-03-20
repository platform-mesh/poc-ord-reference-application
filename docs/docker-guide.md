# Docker & Kubernetes Guide

This guide covers running the ORD Reference Application as a container locally and deploying it to a Kubernetes cluster.

## Container Image

The pre-built image is available on GitHub Container Registry:

```
ghcr.io/platform-mesh/poc-ord-reference-application:latest
```

Pull it directly:

```bash
docker pull ghcr.io/platform-mesh/poc-ord-reference-application:latest
```

## Local Testing with Docker

### Run the container

```bash
docker run --rm -p 8080:8080 ghcr.io/platform-mesh/poc-ord-reference-application:latest
```

### Verify endpoints

```bash
# Health check
curl http://localhost:8080/health/v1/

# ORD configuration
curl http://localhost:8080/.well-known/open-resource-discovery

# CRM API (BasicAuth: foo/bar)
curl -u foo:bar http://localhost:8080/crm/v1/customers

# Astronomy API (open)
curl http://localhost:8080/astronomy/v1/constellations

# Static HTML page
curl http://localhost:8080/
```

### Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `8080` | HTTP listen port |
| `PUBLIC_URL` | `https://ord-reference-application.cfapps.sap.hana.ondemand.com` | Base URL reported in ORD metadata |

Example with custom `PUBLIC_URL`:

```bash
docker run --rm -p 8080:8080 \
  -e PUBLIC_URL=http://localhost:8080 \
  ghcr.io/platform-mesh/poc-ord-reference-application:latest
```

## Deploy to a Kind Cluster

### Prerequisites

- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)

### Create cluster and deploy

```bash
# Create a Kind cluster
kind create cluster --name ord-ref-app

# Apply the Kubernetes manifests (pulls image from GHCR)
kubectl apply -f k8s/

# Wait for the pod to be ready
kubectl wait --for=condition=ready pod -l app=ord-reference-app --timeout=90s

# Port-forward to access the app
kubectl port-forward svc/ord-reference-app 8080:8080
```

Then verify endpoints as shown above.

### Clean up

```bash
kind delete cluster --name ord-ref-app
```

## Building the Image Locally

```bash
# Build
docker build -t ghcr.io/platform-mesh/poc-ord-reference-application:latest .

# Run
docker run --rm -p 8080:8080 ghcr.io/platform-mesh/poc-ord-reference-application:latest
```

### Load a local build into Kind (instead of pulling from GHCR)

```bash
kind load docker-image ghcr.io/platform-mesh/poc-ord-reference-application:latest --name ord-ref-app
```

## Pushing to GHCR

### Prerequisites

- [GitHub CLI](https://cli.github.com/) authenticated with `write:packages` scope
- Push access to `platform-mesh/poc-ord-reference-application`

### Manual push

```bash
# Add write:packages scope (one-time)
gh auth refresh -h github.com -s write:packages

# Login to GHCR
echo "$(gh auth token)" | docker login ghcr.io -u <your-github-username> --password-stdin

# Build and tag
docker build -t ghcr.io/platform-mesh/poc-ord-reference-application:latest .

# Push
docker push ghcr.io/platform-mesh/poc-ord-reference-application:latest
```

### Automated builds

Every push to `main` and every version tag (`v*`) triggers the [Docker Publish](./../.github/workflows/docker-publish.yml) GitHub Actions workflow, which builds and pushes the image to GHCR automatically.

Tags produced:
- `latest` — from every push to `main`
- `<version>` — from version tags (e.g. `v1.1.0` produces `1.1.0` and `1.1`)
- `sha-<commit>` — from every push
