# Lab 18, 19 & 20 — Kubernetes Network Policy, DaemonSet & RBAC


## Lab 18: Control Pod-to-Pod Traffic via Network Policy

### Objective
Define a NetworkPolicy that restricts access to MySQL pods — only the Node.js application pods can reach MySQL on port 3306.

### Architecture

```
nodejs-app pod  ──► port 3306 ──►  mysql pod   
other pods      ──► port 3306 ──►  mysql pod  
```

### Steps

#### 1. Create Secret and ConfigMap

```bash
kubectl create configmap db-config \
  --from-literal=DB_HOST=mysql-service \
  --from-literal=DB_PORT=3306 \
  --from-literal=DB_NAME=ivolve \
  -n ivolve

kubectl create secret generic db-secret \
  --from-literal=MYSQL_ROOT_PASSWORD=rootpassword \
  --from-literal=MYSQL_APP_USER=appuser \
  --from-literal=MYSQL_APP_PASSWORD=apppassword \
  -n ivolve
```

#### 2. Deploy MySQL

```yaml
# mysql.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: ivolve
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: ivolve
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: MYSQL_ROOT_PASSWORD
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: ivolve
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
```

```bash
kubectl apply -f mysql.yaml
kubectl get pods -n ivolve -w   # wait for Running
```

#### 3. Deploy Node.js App

```yaml
# nodejs.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
  namespace: ivolve
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nodejs-app
  template:
    metadata:
      labels:
        app: nodejs-app
    spec:
      containers:
      - name: nodejs-app
        image: node:18-alpine
        command: ["node", "-e", "setInterval(()=>{},1000)"]
        ports:
        - containerPort: 3000
```

```bash
kubectl apply -f nodejs.yaml
kubectl get pods -n ivolve -w   # wait for Running
```

#### 4. Apply the NetworkPolicy

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-mysql
  namespace: ivolve
spec:
  podSelector:         
    matchLabels:
      app: mysql
  policyTypes:
  - Ingress             
  ingress:
  - from:
    - podSelector:      
        matchLabels:
          app: nodejs-app
    ports:
    - protocol: TCP
      port: 3306        
```

```bash
kubectl apply -f network-policy.yaml
```

#### 5. Verify the NetworkPolicy

```bash
kubectl get networkpolicy -n ivolve
kubectl describe networkpolicy allow-app-to-mysql -n ivolve
```

Expected output:
```
Name:         allow-app-to-mysql
Namespace:    ivolve
PodSelector:  app=mysql
Allowing ingress traffic:
  From:
    PodSelector: app=nodejs-app
  To Port: 3306/TCP
Not affecting egress traffic
```

#### 6. Test the Policy

```bash
#  Allowed — from nodejs-app pod
kubectl exec -it <nodejs-pod-name> -n ivolve -- nc -zv mysql-service 3306
# Expected: mysql-service (10.x.x.x:3306) open

#  Blocked — from a random pod
kubectl run test-pod --image=busybox --rm -it -n ivolve -- nc -zv mysql-service 3306
# Expected: connection times out
```

### Key Concepts

| Field | Description |
|---|---|
| `podSelector` | Selects which pods the policy applies TO |
| `policyTypes: Ingress` | Controls incoming traffic only |
| `ingress.from.podSelector` | Whitelists specific source pods |
| `port: 3306` | Restricts to MySQL port only |

---

## Lab 19: Node-Wide Pod Management with DaemonSet

### Objective
Deploy Prometheus node-exporter as a DaemonSet in the monitoring namespace so it runs on every node and exposes metrics on port 9100.

### Steps

#### 1. Create the monitoring namespace

```bash
kubectl create namespace monitoring
kubectl get namespace monitoring
```

#### 2. Check existing node taints

```bash
kubectl describe nodes | grep -A 5 "Taints"
```

In this setup, the following taints were found:

| Node | Taint |
|---|---|
| `minikube` | No taint |
| `minikube-m02` | `node=worker:NoSchedule` |

#### 3. Deploy the DaemonSet

```yaml
# node-exporter-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      tolerations:
      - key: "node"               
        operator: "Equal"
        value: "worker"
        effect: "NoSchedule"
      - operator: "Exists"        
        effect: "NoSchedule"

      hostNetwork: true           
      hostPID: true               

      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        args:
        - --path.rootfs=/host/root
        ports:
        - containerPort: 9100
          hostPort: 9100
        securityContext:
          privileged: true
        volumeMounts:
        - name: root-fs
          mountPath: /host/root
          readOnly: true

      volumes:
      - name: root-fs
        hostPath:
          path: /
```

```bash
kubectl apply -f node-exporter-daemonset.yaml
kubectl get pods -n monitoring -o wide -w
```

#### 4. Validate a pod is scheduled on each node

```bash
kubectl get pods -n monitoring -o wide
```


#### 5. Confirm metrics on port 9100

```bash
# Get node IPs
kubectl get nodes -o wide

# Access metrics
minikube ssh -- curl http://localhost:9100/metrics | head -20
```


### Key Concepts

| Concept | Description |
|---|---|
| DaemonSet | Ensures one pod runs on every node automatically |
| `tolerations` | Allows pods to schedule on tainted nodes |
| `hostNetwork: true` | Pod uses the node's network namespace |
| `hostPID: true` | Pod can see all node processes |
| `hostPort: 9100` | Exposes port directly on the node's IP |

---

## Lab 20: Securing Kubernetes with RBAC and Service Accounts

### Objective
Create a Service Account with read-only access to pods in the ivolve namespace using RBAC Role and RoleBinding.

### Architecture

```
jenkins-sa (ServiceAccount)
    │
    └──► jenkins-sa-pod-reader (RoleBinding)
                │
                └──► pod-reader (Role)
                          │
                          └──► pods: [get, list] only
```

### Steps

#### 1. Create the Service Account

```bash
kubectl create namespace ivolve
kubectl create serviceaccount jenkins-sa -n ivolve
kubectl get serviceaccount jenkins-sa -n ivolve
```

#### 2. Create the Secret and retrieve the token

```yaml
# jenkins-sa-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-sa-token
  namespace: ivolve
  annotations:
    kubernetes.io/service-account.name: jenkins-sa
type: kubernetes.io/service-account-token
```

```bash
kubectl apply -f jenkins-sa-secret.yaml

# Retrieve the token
kubectl get secret jenkins-sa-token -n ivolve \
  -o jsonpath='{.data.token}' | base64 -d
echo ""
```

#### 3. Create the pod-reader Role

```yaml
# pod-reader-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: ivolve
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]   # read-only — no create/delete/update
```

```bash
kubectl apply -f pod-reader-role.yaml
kubectl describe role pod-reader -n ivolve
```

Expected output:
```
Name:         pod-reader
Namespace:    ivolve
PolicyRule:
  Resources  Verbs
  ---------  -----
  pods       [get list]
```

#### 4. Create the RoleBinding

```yaml
# pod-reader-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-sa-pod-reader
  namespace: ivolve
subjects:
- kind: ServiceAccount
  name: jenkins-sa
  namespace: ivolve
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f pod-reader-rolebinding.yaml
kubectl describe rolebinding jenkins-sa-pod-reader -n ivolve
```

#### 5. Validate permissions

```bash
#  Should be allowed
kubectl auth can-i list pods \
  --as=system:serviceaccount:ivolve:jenkins-sa -n ivolve

kubectl auth can-i get pods \
  --as=system:serviceaccount:ivolve:jenkins-sa -n ivolve

#  Should be FORBIDDEN
kubectl auth can-i delete pods \
  --as=system:serviceaccount:ivolve:jenkins-sa -n ivolve

kubectl auth can-i create pods \
  --as=system:serviceaccount:ivolve:jenkins-sa -n ivolve

kubectl auth can-i list services \
  --as=system:serviceaccount:ivolve:jenkins-sa -n ivolve
```



### Key Concepts

| Resource | Description |
|---|---|
| `ServiceAccount` | Identity for a pod or process inside the cluster |
| `Secret (SA token)` | JWT token used to authenticate the service account |
| `Role` | Defines allowed actions on resources within a namespace |
| `RoleBinding` | Binds a Role to a subject (user, group, or service account) |
| `kubectl auth can-i` | Checks permissions without performing the actual action |

### RBAC Flow

| Step | Resource | Purpose |
|---|---|---|
| 1 | ServiceAccount | Creates the identity `jenkins-sa` |
| 2 | Secret | Generates a token for authentication |
| 3 | Role | Defines `get` and `list` on pods only |
| 4 | RoleBinding | Grants the Role to `jenkins-sa` |
| 5 | Validation | Confirms allowed and forbidden actions |
