# Production Keycloak Deployment Guide on Kubernetes via Helm

This repository contains the production configuration and step-by-step instructions for deploying Keycloak (Quarkus distribution) on Kubernetes using Helm.

## Prerequisites

- **Kubernetes Cluster**: v1.23+ with `kubectl` configured.
- **Helm**: Helm v3 installed (`helm version`).
- **Ingress Controller**: NGINX Ingress (or Traefik / ALB) installed.
- **TLS Issuer**: `cert-manager` installed or custom SSL certificate secret.
## Deployment Choices Overview

When setting up Keycloak in production on Kubernetes, you have 3 main paths:

1. **Option 1: Codecentric Community Helm Chart (Recommended for Helm)**
   - Open-source, community-maintained chart designed for Keycloak on Quarkus.
   - Easy to customize with external PostgreSQL and Ingress.
2. **Option 2: Create a Custom Lightweight Helm Chart**
   - Ideal if you want complete control over manifests without third-party chart dependencies.
3. **Option 3: Official Keycloak Operator (Recommended Upstream)**
   - Native Kubernetes Operator maintained directly by the Keycloak project. Uses `Keycloak` Custom Resources.

---

## Step 1: Create Namespace and Secrets

1. Create the `keycloak` namespace:
   ```bash
   kubectl create namespace keycloak
   ```

2. Edit `secrets-template.yaml` with your actual secure passwords, then apply:
   ```bash
   kubectl apply -f secrets-template.yaml
   ```

---

## Step 2: Customize `values-production.yaml`

- `extraEnvVars.KC_HOSTNAME`: Set to your production FQDN (e.g. `auth.yourdomain.com`).
- `externalDatabase.host`: Point to your PostgreSQL database host/endpoint.
- `ingress.hostname` & `ingress.extraTls`: Update domain names.
- `ingress.annotations`: Ensure your `cert-manager.io/cluster-issuer` or ingress annotation matches your cluster setup.

---

## Step 3: Deploy Keycloak using Custom Helm Chart

Deploy Keycloak to your Kubernetes cluster using the custom Helm chart in [`chart/`](chart/):

```bash
# Dry-run test template rendering (optional):
helm template keycloak ./chart --namespace keycloak

# Install or Upgrade Keycloak release:
helm upgrade --install keycloak ./chart \
  --namespace keycloak \
  --values chart/values.yaml
```

---

## Chart Customization Guide

Edit [`chart/values.yaml`](chart/values.yaml) to configure:

- **Domain / Hostname**: `hostname: "auth.yourdomain.com"`
- **External PostgreSQL**: `database.host`, `database.name`, `database.username`, `database.password`
- **Replicas & HA Anti-Affinity**: `replicaCount: 2`, `podAntiAffinity.type: hard`
- **TLS / Cert-Manager**: `ingress.hostname`, `ingress.annotations`

---

## Step 4: Verify Deployment & HA Cluster

1. Check pod status:
   ```bash
   kubectl get pods -n keycloak -w
   ```

2. Check Infinispan cluster discovery in logs:
   ```bash
   kubectl logs -n keycloak -l app.kubernetes.io/name=keycloak --tail=100
   ```
   Look for lines indicating JGroups node discovery (`ISPN000094: Received new cluster view`).

3. Access the Admin Console at `https://auth.yourdomain.com/admin` using user `admin` and the password configured in `keycloak-admin-secret`.

---

## Production Security & Architecture Notes

- **Reverse Proxy Headers**: `KC_PROXY_HEADERS=xforwarded` is required when terminating SSL at the Ingress controller.
- **Infinispan Clustering**: `podAntiAffinityPreset: hard` ensures replicas are placed on separate Kubernetes nodes for high availability.
- **Database Connection Pooling**: Ensure your PostgreSQL database supports connection pooling (or max_connections is sized appropriately for `replicaCount * connection_pool_size`).
