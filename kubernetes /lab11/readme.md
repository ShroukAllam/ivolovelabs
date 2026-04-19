Lab 11: Namespace Management and Resource Quota Enforcement 

Prerequisites
Kubernetes cluster running

kubectl configured to communicate with your cluster

Steps
Step 1: Verify Initial Node Status - Run kubectl get nodes to see your available nodes 
<img width="607" height="82" alt="image" src="https://github.com/user-attachments/assets/d61a21f4-6a6b-48d7-800a-b0f999d871db" />

 
Step 2: Create Namespace - Execute kubectl create namespace ivolve to create a new namespace called "ivolve" that will isolate your resources 

Step 3: Verify Namespace Creation - Run kubectl get namespaces to confirm the ivolve namespace appears with Active status 
<img width="589" height="141" alt="image" src="https://github.com/user-attachments/assets/94a720ec-a40d-4af5-8a8f-8cb49f3b9b8e" />


Step 4: Create Resource Quota YAML - Create a file named resource-quota.yaml 
<img width="234" height="164" alt="image" src="https://github.com/user-attachments/assets/55fa17b8-8910-46b0-bd80-bc339cc661b1" />


Step 5: Apply Resource Quota - Execute kubectl apply -f resource-quota.yaml to apply the quota to the ivolve namespace 

Step 6: Verify Resource Quota - Run kubectl describe resourcequota ivolve-quota -n ivolve to see the quota shows Used: 0 pods and Hard: 2 pods 
<img width="667" height="140" alt="image" src="https://github.com/user-attachments/assets/a654ac75-ffc2-43ec-9255-0c566d8a7861" />


Step 7: Create First Pod - Execute kubectl run pod1 --image=nginx --namespace=ivolve --restart=Never to create the first test pod 

Step 8: Verify First Pod and Quota - Run kubectl get pods -n ivolve to see pod1 running, then kubectl get resourcequota -n ivolve to see usage shows 1/2 pods
<img width="676" height="201" alt="image" src="https://github.com/user-attachments/assets/b78a31a4-a76b-41cd-8724-42f5d8fed42e" />


Step 9: Create Second Pod - Execute kubectl run pod2 --image=nginx --namespace=ivolve --restart=Never to create the second test pod 

Step 10: Verify Both Pods - Run kubectl get pods -n ivolve to see both pod1 and pod2 running, and kubectl get resourcequota -n ivolve to see usage shows 2/2 pods 
<img width="680" height="167" alt="image" src="https://github.com/user-attachments/assets/f6bc275c-21cd-4460-9e3c-e5922c6f87d7" />


Step 11: Test Quota Enforcement - Attempt to create a third pod with kubectl run pod3 --image=nginx --namespace=ivolve --restart=Never which should fail with error 
<img width="681" height="111" alt="image" src="https://github.com/user-attachments/assets/43cc3dcf-cc42-4d81-b891-a9d9ab3e791a" />

