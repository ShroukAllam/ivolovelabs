# Lab 16 Kubernetes Pod Management

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
kubectl get pods -n ivolve -w   
```

#### 2. Create ConfigMap and Secret

```bash
kubectl create configmap db-config \
  --from-literal=DB_HOST=mysql-service \
  --from-literal=DB_PORT=3306 \
  --from-literal=DB_NAME=ivolve \
  -n ivolve
<img width="724" height="64" alt="image" src="https://github.com/user-attachments/assets/8c80abce-449f-4bfb-8937-73ff1ecc227e" />


kubectl create secret generic db-secret \
  --from-literal=MYSQL_ROOT_PASSWORD=rootpassword \
  --from-literal=MYSQL_APP_USER=appuser \
  --from-literal=MYSQL_APP_PASSWORD=apppassword \
  -n ivolve
```
<img width="696" height="60" alt="image" src="https://github.com/user-attachments/assets/934f91da-b378-47d3-babb-bf03c0cd8bb8" />


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
<img width="686" height="82" alt="image" src="https://github.com/user-attachments/assets/bfe2a198-852d-4162-8994-bc065cfb90c3" />


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
<img width="271" height="231" alt="image" src="https://github.com/user-attachments/assets/aeed3971-0e9c-4dad-8fbe-8d141ebf692f" />


-- Check user exists
SELECT User, Host FROM mysql.user WHERE User = 'appuser';
<img width="255" height="125" alt="image" src="https://github.com/user-attachments/assets/090f8764-7395-44a1-8579-24e45307c895" />


-- Check privileges
SHOW GRANTS FOR 'appuser'@'%';
```
<img width="570" height="174" alt="image" src="https://github.com/user-attachments/assets/695b1e03-6ca7-4581-857d-a54ac6750d59" />


### Key Concepts

| Concept | Description |
|---|---|
| Init Container | Runs to completion before app containers start |
| ConfigMap | Stores non-sensitive configuration (host, port, DB name) |
| Secret | Stores sensitive data (passwords, usernames) |
| `CREATE USER IF NOT EXISTS` | Idempotent — safe to run on re-deploy |
| `FLUSH PRIVILEGES` | Applies grant changes immediately |

---

pods -n kube-system | grep metrics-server
```
