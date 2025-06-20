```markdown
# Spring Petclinic on GKE with Helm

This project demonstrates how to deploy a Spring Petclinic application backed by MySQL on Google Kubernetes Engine (GKE) using Helm charts.

---

## Prerequisites

* Google Cloud Platform account with billing enabled
* [Google Cloud SDK (gcloud)](https://cloud.google.com/sdk/docs/install) installed and configured
* [kubectl](https://kubernetes.io/docs/tasks/tools/) installed
* [Helm](https://helm.sh/docs/intro/install/) installed

---

## Step 1: Create GKE Cluster

```bash
# Set your preferred GCP project
gcloud config set project YOUR_PROJECT_ID

# Create a GKE cluster
gcloud container clusters create petclinic-cluster \
  --zone us-central1-a \
  --num-nodes=2

# Get credentials for kubectl
gcloud container clusters get-credentials petclinic-cluster --zone us-central1-a
```

---

## Step 2: Deploy Helm Chart

```bash
# Navigate to your Helm chart directory
cd petclinic/

# Install or upgrade the Helm release
helm upgrade --install petclinic .

# Check deployments and services
kubectl get pods
kubectl get svc
```

---

## Step 3: Access the Application

Find the external IP of the `petclinic-service`:

```bash
kubectl get svc petclinic-service
```

Open the external IP in your browser on port 80 (e.g., `http://EXTERNAL_IP`).

---

## Step 4: Troubleshooting Common Issues

### MySQL Pod Crash: Missing root password

If the MySQL pod crashes with an error like:

```
Database is uninitialized and password option is not specified
```

Make sure you have specified the root password in your Helm `values.yaml`:

```yaml
mysql:
  rootPassword: your-root-password # <-- Make sure this is set
  username: appuser
  password: appuser_password
```

And your MySQL deployment template includes:

```yaml
env:
  - name: MYSQL_ROOT_PASSWORD
    value: "{{ .Values.mysql.rootPassword }}"
  - name: MYSQL_DATABASE
    value: "{{ .Values.mysql.database }}"
  - name: MYSQL_USER
    value: "{{ .Values.mysql.username }}"
  - name: MYSQL_PASSWORD
    value: "{{ .Values.mysql.password }}"
```

### Verify MySQL Pod Status

```bash
kubectl get pods -l app=mysql
kubectl logs <mysql-pod-name>
```

### Test Connectivity

You can test MySQL connectivity inside the cluster with:

```bash
kubectl run -it --rm debug --image=mysql:8 -- bash
```

Inside the debug pod:

```bash
mysql -h mysql -P 3306 -u appuser -p
```

---

## Cleanup

To ensure all resources are properly removed, follow these steps.

### 1. Delete the Helm Release

First, uninstall the Helm release from your cluster. This will remove all Kubernetes resources deployed by the Helm chart (Deployments, Services, PVCs, etc.).

```bash
helm uninstall petclinic --namespace default
```

You can verify it's gone:

```bash
helm list
```

### 2. Delete the GKE Cluster

This will tear down the entire GKE cluster, including all nodes, disk volumes, and associated network resources created by GKE itself.

```bash
gcloud container clusters delete petclinic-cluster --zone us-central1-a --quiet
```

The `--quiet` flag bypasses the confirmation prompt. Remove it if you prefer to confirm manually.

### 3. Verify Remaining Resources (Optional but Recommended)

Sometimes, certain resources like Persistent Disks or Load Balancers might remain if not properly managed by Kubernetes or deleted. It's good practice to check if anything is left behind.

#### Check Persistent Disks:

```bash
gcloud compute disks list --filter="zone=(us-central1-a) AND -users:*"
```

If you see any disks that belong to your `petclinic-cluster` and are not in use (indicated by `-users:*`), you might need to delete them manually:

```bash
gcloud compute disks delete DISK_NAME --zone us-central1-a
```

Be very careful when deleting disks; ensure they are not used by other critical resources.

#### Check Load Balancers (if applicable, though Helm usually cleans this):

```bash
gcloud compute forwarding-rules list
gcloud compute target-pools list
```

If you find any resources associated with `petclinic-service` that are still active and not cleaned up by the cluster deletion, you might need to delete them manually using `gcloud compute forwarding-rules delete` or `gcloud compute target-pools delete`.

---

## Notes

* Consider using Kubernetes Secrets for storing sensitive data like passwords instead of plaintext in `values.yaml`.
* Persistent storage is configured via `PersistentVolumeClaim`; adjust size in `values.yaml` as needed.
* Feel free to raise issues or contribute improvements!
```
