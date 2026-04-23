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
<img width="695" height="83" alt="image" src="https://github.com/user-attachments/assets/7bc75b95-aa34-43c8-b50b-fc9bd9dc04aa" />


#### 3. Verify with `kubectl describe pod`

```bash
kubectl describe pod nodejs-app-845f4777-k9cds -n ivolve
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
<img width="532" height="238" alt="image" src="https://github.com/user-attachments/assets/1ab96f37-e613-40af-b943-edf839fcf788" />


#### 4. Monitor Real-Time Usage with `kubectl top`

```bash
# Verify metrics-server is running
kubectl get deployment metrics-server -n kube-system

# Monitor pod resource usage
kubectl top pod -n ivolve
```
<img width="605" height="79" alt="image" src="https://github.com/user-attachments/assets/5d6e7d7b-2704-4720-a44a-9617229270de" />


kubectl get pods -n kube-system | grep metrics-server
```
