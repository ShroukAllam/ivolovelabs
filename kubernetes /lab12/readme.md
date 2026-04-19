Lab 12: Managing Configuration and Sensitive Data with ConfigMaps and Secrets 

Prerequisites
Kubernetes cluster running 

ivolve namespace from Lab 11 (verify with kubectl get namespace ivolve)

kubectl configured to communicate with your cluster

Steps
Step 1: Verify Namespace - Run kubectl get namespace ivolve to confirm the ivolve namespace exists 
<img width="607" height="143" alt="image" src="https://github.com/user-attachments/assets/db9e23a3-24cf-49a6-be84-509fbf156437" />


Step 2: Create ConfigMap
Step 3: Verify ConfigMap Creation - Run kubectl get configmap mysql-config -n ivolve -o yaml to view the ConfigMap details
<img width="683" height="261" alt="image" src="https://github.com/user-attachments/assets/cc920fdd-7540-49ec-9f89-297098a338ee" />

Step 4: Encode Secret Values Using Base64 

Step 5: Create Secret YAML File - Create a file named secret.yaml 
<img width="441" height="213" alt="image" src="https://github.com/user-attachments/assets/17e4a39f-00fd-41ab-9a78-7d227c93fcf5" />

Step 6: Apply the Secret - Execute kubectl apply -f secret.yaml 

Step 7: Verify Secret Creation - Run kubectl get secret mysql-secret -n ivolve -o yaml 

Step 8: Decode and Verify Secret Values 

Step 9: Create Test Pod to Access ConfigMap and Secret - Create a file named test-pod.yaml 
<img width="488" height="676" alt="image" src="https://github.com/user-attachments/assets/6ab61278-2301-4dcd-bc1a-d0d09a1008de" />

Step 10: Apply and Verify Test Pod 

<img width="674" height="112" alt="image" src="https://github.com/user-attachments/assets/0268f5ad-021e-47bc-a55b-3443ede8ec09" />
