# Lab 16 & Lab 17 — Kubernetes Pod Management

---

## Lab 16: Init Container for Pre-Deployment Database Setup

### Objective
Modify an existing Node.js Deployment to include an init container that sets up a MySQL database before the application starts.

### Architecture Overview

```
ConfigMap (DB_HOST, DB_PORT, DB_NAME)  ──┐
                                          ├──► Pod
Secret (ROOT_PASSWORD, APP_USER, PASS) ──┘     │
                                               ├── Init Container (mysql:5.7)
                                               │     └── Creates 'ivolve' DB
                                               │     └── Grants appuser privileges
                                               │     └── ✅ Exits successfully
                                               │
                                               └── App Container (Node.js)
                                                     └── Starts only after init completes
```

### Steps

#### 1. Deploy MySQL

```yaml
# mysql-deployment.yaml
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
          value: "rootpassword"
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
kubectl apply -f mysql-deployment.yaml
kubectl get pods -n ivolve -w   # wait for Running
```

#### 2. Create ConfigMap and Secret

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

#### 3. Update Node.js Deployment with Init Container

```yaml
# deployment.yaml
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
      initContainers:
      - name: db-init
        image: mysql:5.7
        env:
        - name: MYSQL_HOST
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: DB_HOST
        - name: MYSQL_PORT
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: DB_PORT
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: MYSQL_ROOT_PASSWORD
        - name: APP_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: MYSQL_APP_USER
        - name: APP_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: MYSQL_APP_PASSWORD
        command:
        - sh
        - -c
        - |
          echo "Waiting for MySQL..."
          until mysql -h "$MYSQL_HOST" -P "$MYSQL_PORT" -u root -p"$MYSQL_ROOT_PASSWORD" -e "SELECT 1;" > /dev/null 2>&1; do
            echo "MySQL not ready, retrying in 5s..."
            sleep 5
          done
          echo "MySQL ready. Running setup..."
          mysql -h "$MYSQL_HOST" -P "$MYSQL_PORT" -u root -p"$MYSQL_ROOT_PASSWORD" <<EOF
          CREATE DATABASE IF NOT EXISTS ivolve;
          CREATE USER IF NOT EXISTS '$APP_USER'@'%' IDENTIFIED BY '$APP_PASSWORD';
          GRANT ALL PRIVILEGES ON ivolve.* TO '$APP_USER'@'%';
          FLUSH PRIVILEGES;
          EOF
          echo "Init complete."

      containers:
      - name: nodejs-app
        image: node:18-alpine
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: DB_PORT
        - name: DB_NAME
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: DB_NAME
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: MYSQL_APP_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: MYSQL_APP_PASSWORD
        ports:
        - containerPort: 3000
```

```bash
kubectl apply -f deployment.yaml
```

#### 4. Verify Init Container Ran Successfully

```bash
# Check pod status (should show Init:0/1 → Running)
kubectl get pods -n ivolve -w

# View init container logs
kubectl logs <pod-name> -c db-init -n ivolve
```

#### 5. Verify Database and User in MySQL

```bash
kubectl exec -it <mysql-pod-name> -n ivolve -- mysql -u root -p
```

```sql
-- Check database exists
SHOW DATABASES;

-- Check user exists
SELECT User, Host FROM mysql.user WHERE User = 'appuser';

-- Check privileges
SHOW GRANTS FOR 'appuser'@'%';
```

**Expected output:**
```
GRANT ALL PRIVILEGES ON `ivolve`.* TO 'appuser'@'%'
```

### Key Concepts

| Concept | Description |
|---|---|
| Init Container | Runs to completion before app containers start |
| ConfigMap | Stores non-sensitive configuration (host, port, DB name) |
| Secret | Stores sensitive data (passwords, usernames) |
| `CREATE USER IF NOT EXISTS` | Idempotent — safe to run on re-deploy |
| `FLUSH PRIVILEGES` | Applies grant changes immediately |

---

## Lab 17: Pod Resource Management with CPU and Memory Requests and Limits

### Objective
Update the existing Node.js Deployment to define resource requests and limits, then verify and monitor them.

### Resource Configuration

| Type | CPU | Memory |
|---|---|---|
| **Request** | 1 vCPU | 1Gi |
| **Limit** | 2 vCPUs | 2Gi |

### Steps

#### 1. Patch the Deployment

```bash
kubectl patch deployment nodejs-app -n ivolve --patch '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "nodejs-app",
          "resources": {
            "requests": {
              "cpu": "1",
              "memory": "1Gi"
            },
            "limits": {
              "cpu": "2",
              "memory": "2Gi"
            }
          }
        }]
      }
    }
  }
}'
```

#### 2. Verify Rollout

```bash
kubectl rollout status deployment/nodejs-app -n ivolve
kubectl get pods -n ivolve
```

#### 3. Verify with `kubectl describe pod`

```bash
kubectl describe pod <pod-name> -n ivolve
```

Look for this section in the output:

```
Containers:
  nodejs-app:
    Requests:
      cpu:     1
      memory:  1Gi
    Limits:
      cpu:     2
      memory:  2Gi
```

#### 4. Monitor Real-Time Usage with `kubectl top`

```bash
# Verify metrics-server is running
kubectl get deployment metrics-server -n kube-system

# Monitor pod resource usage
kubectl top pod -n ivolve
```

**Expected output:**
```
NAME                          CPU(cores)   MEMORY(bytes)
nodejs-app-xxxxxxx-xxxxx      5m           128Mi
```

> **Note:** If `kubectl top` returns `error: Metrics API not available`, install metrics-server:
> ```bash
> kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
> ```

### Key Concepts

| Field | Behaviour |
|---|---|
| `requests.cpu` | Amount of CPU Kubernetes **reserves** on the node for scheduling |
| `requests.memory` | Amount of memory Kubernetes **reserves** on the node for scheduling |
| `limits.cpu` | Maximum CPU the pod can use — **throttled** if exceeded |
| `limits.memory` | Maximum memory the pod can use — **OOMKilled** if exceeded |

### Troubleshooting

**Pod stuck pending after adding resources:**
```bash
# Check available node resources
kubectl describe nodes | grep -A 6 "Allocated resources"
```
If the node has less than 1 CPU free, reduce the request value (e.g. `"500m"`) to fit your cluster.

**`kubectl top` not working:**
Metrics Server must be deployed. Verify with:
```bash
kubectl get pods -n kube-system | grep metrics-server
```
