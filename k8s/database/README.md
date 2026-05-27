# 🚀 EBS Storage Setup on EKS — Complete Guide

> Step-by-step guide to provision AWS EBS (`gp3`) volumes for your Kubernetes workloads on EKS using the EBS CSI Driver.

---

## 📋 Prerequisites

Before starting, make sure you have the following ready:

| Tool | Purpose |
|------|---------|
| `kubectl` | Configured and pointing to your EKS cluster |
| `eksctl` | EKS cluster management CLI |
| `aws` CLI | AWS API access |
| An EKS cluster | Running and accessible |

---

## Architecture Overview

```
OIDC Provider → IAM Role → EBS CSI Driver → StorageClass (ebs-sc) → PVC → EBS Volume (gp3)
```

---

## Step 1 — Verify Your Cluster

```bash
# Check cluster is reachable
kubectl get nodes

# List your clusters (note your cluster name)
aws eks list-clusters --region <YOUR_REGION>

# Get OIDC issuer URL — SAVE THIS OUTPUT
aws eks describe-cluster \
  --name <YOUR_CLUSTER_NAME> \
  --region <YOUR_REGION> \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

**Expected output:**
```
https://oidc.eks.ap-south-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE
```

> **Why?** The EBS CSI driver needs IAM permissions via OIDC to create and attach EBS volumes.

---

## Step 2 — Enable OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster <YOUR_CLUSTER_NAME> \
  --region <YOUR_REGION> \
  --approve
```

**Verify it worked:**
```bash
aws iam list-open-id-connect-providers | grep -i oidc
# You should see at least one entry
```

> **Why?** This allows Kubernetes service accounts to assume IAM roles (IRSA = IAM Roles for Service Accounts).

---

## Step 3 — Create IAM Service Account

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster <YOUR_CLUSTER_NAME> \
  --region <YOUR_REGION> \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --override-existing-serviceaccounts
```

**Verify it was created:**
```bash
kubectl get serviceaccount -n kube-system ebs-csi-controller-sa
# Expected: ebs-csi-controller-sa   1         Xm
```

> **Why?** This creates an IAM role and links it to a Kubernetes service account so the CSI driver can call AWS EBS APIs.

---

## Step 4 — Install EBS CSI Driver Add-on

**Get your AWS Account ID:**
```bash
aws sts get-caller-identity --query Account --output text
# Example: 123456789012
```

**Get the IAM Role ARN created in Step 3:**
```bash
aws iam list-roles \
  --query "Roles[?contains(RoleName, 'ebs-csi')].Arn" \
  --output text
[O```

**Install the addon:**
```bash
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster <YOUR_CLUSTER_NAME> \
  --region <YOUR_REGION> \
  --service-account-role-arn <ROLE_ARN_FROM_ABOVE> \
  --force
```

**Verify CSI driver pods are running:**
```bash
kubectl get pods -n kube-system | grep ebs-csi
```

**Expected output — all pods must show `Running`:**
```
ebs-csi-controller-xxxx   6/6   Running   0   2m
ebs-csi-node-xxxx         3/3   Running   0   2m
```

> ⚠️ **Do not proceed to Step 5 until all pods show `Running`.**

---

## Step 5 — Create StorageClass

Create the file `ebs-storageclass.yaml`:

```bash
cat > ebs-storageclass.yaml << 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
parameters:
  type: gp3
EOF
```

**Apply it:**
```bash
kubectl apply -f ebs-storageclass.yaml
```

**Verify:**
```bash
kubectl get storageclass
```

**Expected output:**
```
NAME               PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      AGE
ebs-sc (default)   ebs.csi.aws.com         Delete          WaitForFirstConsumer   10s
```

---

## Step 6 — Deploy MongoDB StatefulSet

Create the file `mongo-statefulset.yaml`:

```bash
cat > mongo-statefulset.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec:
  selector:
    app: mongo
  ports:
    - port: 27017
  clusterIP: None
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: "mongo"
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
        - name: mongo
          image: mongo:6
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongo-data
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: mongo-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "ebs-sc"                    # this should be match with ebs volume name
        resources:
          requests:
            storage: 1Gi
EOF
```

**Apply it:**
```bash
kubectl apply -f mongo-statefulset.yaml
```

---

## Step 7 — Verify Everything is Working

Run these commands one by one:

```bash
# 1. Watch pod come up (wait 60–90 seconds)
[Ikubectl get pods -w
# Wait until: mongo-0   1/1   Running

# 2. PVC must show "Bound" — not "Pending"
kubectl get pvc
# Expected:
# NAME                STATUS   VOLUME         CAPACITY   STORAGECLASS   AGE
# mongo-data-mongo-0  Bound    pvc-xxxx-xxxx  1Gi        ebs-sc         2m

# 3. Confirm PV was auto-created
kubectl get pv
# You will see a pvc-xxxx entry with STORAGECLASS=ebs-sc

# 4. Full PVC details
kubectl describe pvc mongo-data-mongo-0

# 5. Check for any errors
kubectl get events --sort-by='.lastTimestamp' | tail -20
```

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| PVC stuck at `Pending` | Normal with `WaitForFirstConsumer` — wait for pod to schedule |
| CSI pods not starting | Run `kubectl describe pod <ebs-pod> -n kube-system` to see error |
| `failed to provision volume` | IAM Role ARN is incorrect in Step 4 — re-check |
| Pod stuck at `ContainerCreating` | Run `kubectl describe pod mongo-0` to see the volume error |
| `node has no capacity` | Check if node instance type supports EBS attachment |

---

## File Structure

```
.
├── ebs-storageclass.yaml      # StorageClass using ebs.csi.aws.com
└── mongo-statefulset.yaml     # StatefulSet + Headless Service for MongoDB
```

---

## Notes

- Replace `<YOUR_CLUSTER_NAME>` and `<YOUR_REGION>` in every command above.
- EBS volumes are **AZ-specific** — `WaitForFirstConsumer` handles this automatically by binding the volume to the same AZ as the pod.
- The `reclaimPolicy: Delete` means the EBS volume is **deleted** when the PVC is deleted. Change to `Retain` if you want to keep the data.
