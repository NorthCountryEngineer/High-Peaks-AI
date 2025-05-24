# High Peaks AI Platform

High Peaks AI comprises modular services including:

* **highpeaks-identity-service** (Node.js microservice & Keycloak)
* **highpeaks-ml-platform** (MLflow, MinIO, PostgreSQL, Flask inference)
* **highpeaks-dataswarm-agenticai-platform** (Flowise service)
* **highpeaks-devops-agent** (DevOps agent)
* **highpeaks-infrastructure** (Kubernetes & DNS deployment)

---

## Prerequisites

Ensure you have the following installed on your Ubuntu (or macOS) host:

* **Docker** (>=20.x)
* **Kind** (Kubernetes-in-Docker)
* **kubectl**
* **Helm**
* **Node.js** (>=14.x) & npm (for identity service local dev)
* **Python 3.10+** & pip (for ML Platform local dev)
* **Bind9** (optional, for LAN DNS)

---

## 1. Clone All Repositories

```bash
export GH_ORG="NorthCountryEngineer"
mkdir -p ~/Highpeaks-AI && cd ~/Highpeaks-AI

git clone https://github.com/$GH_ORG/highpeaks-identity-service.git
git clone https://github.com/$GH_ORG/highpeaks-ml-platform.git
git clone https://github.com/$GH_ORG/highpeaks-dataswarm-agenticai-platform.git
git clone https://github.com/$GH_ORG/highpeaks-devops-agent.git
git clone https://github.com/$GH_ORG/highpeaks-infrastructure.git
```

---

## 2. (Optional) LAN DNS with Bind9

If you want `*.highpeaks.local` hostnames accessible on your LAN:

1. Install and enable Bind9:

   ```bash
   sudo apt update
   sudo apt install bind9 bind9utils bind9-doc -y
   ```
2. Edit `/etc/bind/named.conf.options`, add your LAN IP (e.g. `10.0.0.44`) in `listen-on` & `allow-query`.
3. Create zone file `/etc/bind/db.highpeaks.local`:

   ```dns
   $TTL 1h
   @ IN SOA ns1.highpeaks.local. admin.highpeaks.local. (
     2025042201 ; Serial
     1h         ; Refresh
     15m        ; Retry
     1w         ; Expire
     1h )      ; Negative cache TTL
   IN NS ns1.highpeaks.local.
   ns1       IN A 10.0.0.44
   keycloak  IN A 10.0.0.44
   flowise   IN A 10.0.0.44
   mlflow    IN A 10.0.0.44
   devops    IN A 10.0.0.44
   ```
4. Enable and restart:

   ```bash
   sudo named-checkconf
   sudo named-checkzone highpeaks.local /etc/bind/db.highpeaks.local
   sudo systemctl restart bind9
   ```
5. Point your other machines (including your Mac) to use `10.0.0.44` as their DNS server.

---

## 3. Deploy Kubernetes Cluster & Namespaces

```bash
cd ~/Highpeaks-AI/highpeaks-infrastructure
chmod +x run_highpeaks_deployment.sh
./run_highpeaks_deployment.sh
```

This script will:

1. Create a Kind cluster named `highpeaks` (`k8s/kind-cluster.yaml`).
2. Apply namespaces (`k8s/namespaces.yaml`):

   * `highpeaks-identity`
   * `highpeaks-ml`
   * `highpeaks-flowise`
   * `highpeaks-devops`
3. Build and load Docker images for identity, ML Platform, DevOps agent.
4. Pull & load the Flowise image.
5. Deploy all services.

If you ever need to reset, delete the Kind cluster:

```bash
kind delete cluster --name highpeaks
```

---

## 4. Services Overview & Access

### Identity Service (Node.js & Keycloak)

* **Helm chart**: `highpeaks-identity-service/charts/highpeaks-identity`
* **Ingress**: `keycloak.highpeaks.local` via NGINX
* **Port-forward (fallback)**:

  ```bash
  kubectl port-forward -n highpeaks-identity svc/highpeaks-identity-service 8081:80
  open http://localhost:8081/health
  ```

### ML Platform (Flask & MLflow)

* **Components**:

  * **MinIO** (object storage)
  * **PostgreSQL** (metadata store)
  * **MLflow server**
  * **Flask inference**
* **YAML manifests**: `highpeaks-ml-platform/infrastructure/k8s`
* **Port-forward**:

  ```bash
  kubectl port-forward -n highpeaks-ml svc/highpeaks-ml-platform 5000:80
  open http://localhost:5000/predict
  ```
* **MLflow UI**:

  ```bash
  kubectl port-forward -n highpeaks-ml svc/mlflow-service 5001:5000
  open http://localhost:5001
  ```

### Flowise Service

* **Manifests**: `highpeaks-dataswarm-agenticai-platform/k8s`
* **Service**: NodePort 32345 → internal 8080
* **Access**: `http://<node-ip>:32345`
* **Port-forward** (optional):

  ```bash
  kubectl port-forward -n highpeaks-flowise svc/highpeaks-flowise-service 8080:80
  open http://localhost:8080
  ```

### DevOps Agent

* **Kustomize overlay**: `highpeaks-devops-agent/k8s/overlays/dev`
* **Port-forward**:

  ```bash
  kubectl port-forward -n highpeaks-devops svc/highpeaks-devops-agent 8000:80
  open http://localhost:8000/health
  ```

---

## 5. Troubleshooting

### Common Errors

* **ImagePullBackOff**: Ensure images are loaded into Kind. The deployment script does this for you; if you rebuild locally, re-run `kind load docker-image`.
* **Service has no endpoints**: Check pod readiness (`kubectl get pods -n <ns>`), describe the service, and ensure selectors match labels.
* **Ingress 404 / Too many redirects**: Verify `proxy: edge` in values, and that NGINX ConfigMap for forwarded headers is present.
* **Port conflicts**: Default port-forwards (8081, 5000, 8000) may collide. Change the local port mapping: e.g., `8082:80`.

### Viewing Logs

To inspect logs for all services:

```bash
# Identity
kubectl logs -n highpeaks-identity -l app=highpeaks-identity

# ML Platform
kubectl logs -n highpeaks-ml deployment/mlflow-server
kubectl logs -n highpeaks-ml deployment/minio
kubectl logs -n highpeaks-ml deployment/postgres
kubectl logs -n default deployment/highpeaks-ml-platform

# Flowise
kubectl logs -n highpeaks-flowise deployment/highpeaks-flowise

# DevOps Agent
kubectl logs -n highpeaks-devops deployment/highpeaks-devops-agent
```

Use `-f` to stream logs:

```bash
kubectl logs -n <namespace> <pod-name> -f
```

---

## Next Steps

* Integrate actual ML models by training via `highpeaks-ml-platform/ml/scripts/train-mnist-model.py` against the MLflow server.
* Add ingress rules for Flowise and MLflow under `highpeaks-infrastructure/k8s/ingress.yaml`.
* Hook up TLS certificates using cert-manager or external Load Balancer.

---

*This README is the canonical blueprint for the High Peaks AI platform. Feed this to your Codex agents to fully scaffold and maintain the project.*
