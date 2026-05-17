---
name: certified-kubernetes-admin
description: >
  Use this skill for ALL Kubernetes tasks — cluster administration, workload management,
  networking, storage, RBAC, troubleshooting, and CKA exam preparation. Trigger when
  the user mentions: kubectl, pod, deployment, service, ingress, configmap, secret,
  namespace, pv/pvc, rbac, node, taint, toleration, helm, kubeadm, etcd, CKA, k8s,
  cluster setup, or any Kubernetes troubleshooting scenario. Also triggers on: "k8s",
  "kube", "cluster admin", "container orchestration", or "kubernetes manifest".
---

# Certified Kubernetes Administrator (CKA) — Skill

This skill governs how Jessy assists with Kubernetes administration, kubectl mastery,
cluster operations, and CKA exam preparation. Always produce production-grade YAML,
explain the "why" behind every resource choice, and teach exam-relevant patterns.

## Core Philosophy

- **Exam-first mindset.** CKA is hands-on — every answer includes `kubectl` commands, not just theory.
- **Imperative + Declarative.** Show `kubectl create --dry-run=client -o yaml` to generate base YAML fast.
- **Always explain `why`.** Why use a Deployment over a Pod? Why does this taint block scheduling?
- **Troubleshoot systematically.** Use `describe → logs → events → exec` as the debugging ladder.
- **Time-aware.** CKA exam is 2 hours. Always show the fastest path first, then the full explanation.

---

## CKA Exam Domain Weights

| Domain | Weight | Key Topics |
|--------|--------|------------|
| Cluster Architecture, Installation & Configuration | 25% | kubeadm, RBAC, kubeconfig, etcd backup |
| Workloads & Scheduling | 15% | Deployments, DaemonSets, taints, affinity, resource limits |
| Services & Networking | 20% | Services, Ingress, NetworkPolicy, CoreDNS |
| Storage | 10% | PV, PVC, StorageClass, volume mounts |
| Troubleshooting | 30% | Node issues, pod failures, logs, events |

> Troubleshooting (30%) is the heaviest domain — always teach root-cause diagnosis.

---

## Imperative Commands (Exam Speed)

Generate YAML skeletons fast — never write from scratch in the exam.

```bash
# Pod
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Deployment
kubectl create deployment myapp --image=nginx --replicas=3 --dry-run=client -o yaml > deploy.yaml

# Service (ClusterIP)
kubectl expose deployment myapp --port=80 --target-port=8080 --dry-run=client -o yaml > svc.yaml

# Service (NodePort)
kubectl expose pod nginx --port=80 --type=NodePort --dry-run=client -o yaml

# ConfigMap
kubectl create configmap app-config --from-literal=ENV=prod --dry-run=client -o yaml

# Secret
kubectl create secret generic db-secret --from-literal=password=s3cr3t --dry-run=client -o yaml

# ServiceAccount
kubectl create serviceaccount my-sa --dry-run=client -o yaml

# Role + RoleBinding
kubectl create role pod-reader --verb=get,list,watch --resource=pods --dry-run=client -o yaml
kubectl create rolebinding read-pods --role=pod-reader --user=jane --dry-run=client -o yaml

# ClusterRole + ClusterRoleBinding
kubectl create clusterrole node-reader --verb=get,list --resource=nodes --dry-run=client -o yaml
kubectl create clusterrolebinding node-read --clusterrole=node-reader --user=jane --dry-run=client -o yaml

# Job
kubectl create job my-job --image=busybox -- echo "hello" --dry-run=client -o yaml

# CronJob
kubectl create cronjob my-cron --image=busybox --schedule="*/5 * * * *" -- date --dry-run=client -o yaml
```

---

## Core Resource Templates

### Pod with Resource Limits & Probes
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: default
  labels:
    app: myapp
spec:
  containers:
  - name: myapp
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
  volumes:
  - name: config-vol
    configMap:
      name: app-config
```

### Deployment with Rolling Update Strategy
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0        # zero downtime rolling update
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx:1.25
        ports:
        - containerPort: 80
```

### Service Types
```yaml
# ClusterIP (internal only)
apiVersion: v1
kind: Service
metadata:
  name: myapp-clusterip
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP

---
# NodePort (external via node IP)
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080        # range: 30000-32767
  type: NodePort
```

### Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-clusterip
            port:
              number: 80
```

### NetworkPolicy (deny all + allow specific)
```yaml
# Deny all ingress to namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: secure-ns
spec:
  podSelector: {}
  policyTypes:
  - Ingress

---
# Allow only from frontend pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: secure-ns
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
```

---

## Storage

### PersistentVolume + PersistentVolumeClaim
```yaml
# PV (cluster-wide, admin creates)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce                # RWO: one node read/write
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/my-pv            # exam: hostPath is simplest for practice

---
# PVC (namespace-scoped, dev creates)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi             # must be <= PV capacity
```

### Mount PVC in Pod
```yaml
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

### Access Mode Reference

| Mode | Short | Meaning |
|------|-------|---------|
| ReadWriteOnce | RWO | One node, read+write |
| ReadOnlyMany | ROX | Many nodes, read only |
| ReadWriteMany | RWX | Many nodes, read+write |
| ReadWriteOncePod | RWOP | One pod, read+write (K8s 1.22+) |

---

## Scheduling

### Taints & Tolerations
```bash
# Add taint to node
kubectl taint node node1 key=value:NoSchedule

# Remove taint
kubectl taint node node1 key=value:NoSchedule-
```

```yaml
# Pod toleration (allows scheduling on tainted node)
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"   # NoSchedule | PreferNoSchedule | NoExecute
```

### Node Affinity
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
```

### Pod Affinity / Anti-Affinity
```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: myapp
        topologyKey: kubernetes.io/hostname   # spread across nodes
```

### Static Pods
- Managed by kubelet directly — manifest in `/etc/kubernetes/manifests/`
- Control plane components (etcd, kube-apiserver) are static pods
- To create: drop YAML in `/etc/kubernetes/manifests/` on the node

---

## RBAC

```yaml
# Role — namespace-scoped
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-manager
  namespace: dev
rules:
- apiGroups: [""]                  # "" = core group (pods, services, configmaps)
  resources: ["pods"]
  verbs: ["get", "list", "create", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jane-pod-manager
  namespace: dev
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-manager
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Check permissions
kubectl auth can-i create pods --as=jane -n dev
kubectl auth can-i create pods --as=system:serviceaccount:dev:my-sa -n dev
```

---

## etcd Backup & Restore

```bash
# Backup
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table

# Restore
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore

# Update etcd static pod manifest to point to restored data dir
# Edit: /etc/kubernetes/manifests/etcd.yaml → --data-dir=/var/lib/etcd-restore
```

---

## Cluster Administration

### kubeadm Init & Join
```bash
# Init control plane
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<IP>

# Join worker node (token from kubeadm init output)
kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

# Generate new join token (if expired)
kubeadm token create --print-join-command

# Upgrade cluster
kubeadm upgrade plan
kubeadm upgrade apply v1.29.0

# Drain node before maintenance
kubectl drain node1 --ignore-daemonsets --delete-emissary-data

# Uncordon after maintenance
kubectl uncordon node1
```

### kubeconfig Management
```bash
# Set context
kubectl config use-context <context-name>

# View contexts
kubectl config get-contexts

# Create kubeconfig entry
kubectl config set-credentials jane --client-certificate=jane.crt --client-key=jane.key
kubectl config set-context jane-dev --cluster=kubernetes --user=jane --namespace=dev
```

---

## Troubleshooting Ladder

Use this order every time a pod/node issue appears:

```
1. kubectl get pod <name> -o wide          → check node, status, restarts
2. kubectl describe pod <name>             → events section (ImagePullBackOff, OOMKilled, etc.)
3. kubectl logs <name>                     → app-level errors
   kubectl logs <name> --previous          → crashed container logs
4. kubectl exec -it <name> -- sh           → live debug inside container
5. kubectl get events --sort-by=.metadata.creationTimestamp
```

### Common Pod Failure States

| Status | Root Cause | Fix |
|--------|-----------|-----|
| `ImagePullBackOff` | Wrong image name / no registry access | Fix image name, check imagePullSecret |
| `CrashLoopBackOff` | App crashes at start | Check `logs --previous`, fix app config |
| `OOMKilled` | Exceeded memory limit | Increase `resources.limits.memory` |
| `Pending` | No node fits (resources/taints) | `describe pod` → events show reason |
| `ContainerCreating` | Volume mount issue | Check PVC bound, secret/configmap exists |
| `Error` | Container exited non-zero | Check logs |
| `Terminating` (stuck) | Finalizer blocking | `kubectl patch pod <name> -p '{"metadata":{"finalizers":[]}}' --type=merge` |

### Node Troubleshooting
```bash
# Check node status
kubectl get nodes
kubectl describe node <node-name>

# SSH to node and check kubelet
systemctl status kubelet
journalctl -u kubelet -f              # live kubelet logs

# Common node issues
# NotReady → kubelet stopped, network plugin issue, disk pressure
systemctl restart kubelet

# Check node conditions
kubectl get node <name> -o jsonpath='{.status.conditions[*].type}'
```

### Service Connectivity Debug
```bash
# Check if service endpoint exists
kubectl get endpoints <svc-name>

# Test DNS resolution from a pod
kubectl run tmp --image=busybox --rm -it --restart=Never -- nslookup myapp-svc.default.svc.cluster.local

# Test HTTP from inside cluster
kubectl run tmp --image=curlimages/curl --rm -it --restart=Never -- curl http://myapp-svc.default.svc.cluster.local
```

---

## CKA Exam Tips

### Alias Setup (do this first in the exam)
```bash
alias k=kubectl
export do="--dry-run=client -o yaml"
export now="--force --grace-period 0"

# Usage
k run nginx --image=nginx $do > pod.yaml
k delete pod nginx $now
```

### Namespace Shortcut
```bash
# Set default namespace for session
kubectl config set-context --current --namespace=dev
```

### Vim for YAML (exam editor)
```bash
# Add to ~/.vimrc for YAML formatting
set expandtab
set tabstop=2
set shiftwidth=2
```

### Quick Copy & Edit Pattern
```bash
# Copy existing resource, edit, apply
kubectl get deployment myapp -o yaml > myapp.yaml
vim myapp.yaml
kubectl apply -f myapp.yaml
```

### Key File Paths (memorize)
| Path | Purpose |
|------|---------|
| `/etc/kubernetes/manifests/` | Static pod manifests (control plane) |
| `/etc/kubernetes/pki/` | Cluster certificates |
| `/var/lib/kubelet/` | Kubelet data |
| `/var/lib/etcd/` | etcd data directory |
| `~/.kube/config` | kubeconfig |

---

## Level History
- **Lv.1** — Full CKA skill: all 5 exam domains, imperative commands, YAML templates,
  troubleshooting ladder, etcd backup/restore, RBAC, scheduling, storage, networking.
  Exam tips and alias setup included.
