# Lab 15: Node.js Application Deployment with ClusterIP Service

## Objectives
- Create Deployment `nodejs-app` with 2 replicas (only 1 pod runs due to node taint)
- Use custom Docker image from Docker Hub
- Inject environment variables from ConfigMap and Secret
- Add toleration `node=worker:NoSchedule` to pod spec
- Mount existing PVC `app-logs-pvc` to `/var/log/app`
- Expose deployment via ClusterIP service `nodejs-service`

## Prerequisites
- Lab 10: Node taint `node=worker:NoSchedule` on worker node
- Lab 11: Namespace `ivolve` with ResourceQuota (2 pods limit)
- Lab 12: ConfigMap `mysql-config` and Secret `mysql-secret`
- Lab 13: PVC `app-logs-pvc` bound and ready
- Custom Node.js image uploaded to Docker Hub

- ## Step 1: Verify Prerequisites

```bash
kubectl get namespace ivolve
kubectl get configmap mysql-config -n ivolve
kubectl get secret mysql-secret -n ivolve
kubectl get pvc app-logs-pvc -n ivolve
kubectl get nodes
kubectl describe node <worker-node> | grep Taints
```
## Step 2: Build and Push Docker Image

```bash
cd nodejs-app
docker build -t shroukallam/nodejs-k8s-app:latest .
docker push shroukallam/nodejs-k8s-app:latest
```

## Step 3: Create Deployment YAML

```bash
cat > nodejs-deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
  namespace: ivolve
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nodejs-app
  template:
    metadata:
      labels:
        app: nodejs-app
    spec:
      tolerations:
        - key: node
          operator: Equal
          value: worker
          effect: NoSchedule
      containers:
        - name: nodejs-app
          image: YOUR_USERNAME/nodejs-k8s-app:latest
          ports:
            - containerPort: 3000
          env:
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: mysql-config
                  key: DB_HOST
            - name: DB_USER
              valueFrom:
                configMapKeyRef:
                  name: mysql-config
                  key: DB_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: DB_PASSWORD
          volumeMounts:
            - name: app-logs
              mountPath: /var/log/app
      volumes:
        - name: app-logs
          persistentVolumeClaim:
            claimName: app-logs-pvc
EOF
```


## Step 4: Verify Toleration and Environment Variables

```bash
kubectl describe pod -l app=nodejs-app -n ivolve | grep -A 3 Tolerations
```
<img width="682" height="296" alt="image" src="https://github.com/user-attachments/assets/a2162fe3-7323-488d-acf4-5d52664ac600" />

```bash
kubectl describe pod nodejs-app-58b477f49c-csbl8 -n ivolve | grep -A 5 "Mounts"
```
<img width="681" height="181" alt="image" src="https://github.com/user-attachments/assets/d8e04e43-fd69-40e4-a04e-58034a17440a" />


## Step 5: Create and Apply ClusterIP Service

```bash
cat > nodejs-service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nodejs-service
  namespace: ivolve
spec:
  type: ClusterIP
  selector:
    app: nodejs-app
  ports:
    - port: 80
      targetPort: 3000
EOF
```

```bash
kubectl get service -n ivolve
```
<img width="684" height="127" alt="image" src="https://github.com/user-attachments/assets/038dfece-a7dd-4127-81f8-a0574595bd7b" />


```bash
kubectl describe service nodejs-service -n ivolve
```
<img width="682" height="371" alt="image" src="https://github.com/user-attachments/assets/10c373f0-85a3-490d-a210-98560e3ece56" />


## Step 6: Test the Application

```bash
kubectl run test-client --image=busybox -n ivolve --rm -it --restart=Never -- sh
```
<img width="980" height="231" alt="image" src="https://github.com/user-attachments/assets/1cd9e985-259a-4103-ae61-590b0983ff38" />


<img width="833" height="297" alt="image" src="https://github.com/user-attachments/assets/8c5d6fb1-d2de-4fd2-9d57-00c78dae3cac" />



