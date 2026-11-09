# k8s

Testing k8s-git-kubectl-helm-argocd-prometheus-grafana

# Preparation
1. Create a folder where the project will reside
2. Clone the repo from github  
```git clone git@github.com:Grellissimo/k8s.git```
3. Navigate into the folder, and initialize your preferred git manager. I use lazygit as i find it easy to use compared the CLI
4. Create a new k8s cluster if not already available. This can be done in many different ways, for simplicity, i'll use a KinD (kubernetes in docker) cluster directly created by docker desktop. A much professional solution would definitely be something like AWS EKS.  
4.1 After waiting some minutes, run ```kubectl get nodes``` to ensure cluster is ready


## Deploy a service using kubectl

As a first, i just want to deploy a simple service using kubectl.
Focus of this exercise is to learn and observe how:
1. Create a dedicated namespace
2. Deploy a simple service via kubectl via YAML file
3. scale-up, scale-down, delete
4. Cleanup where required


Note: I keep all Yaml files in a subfolder named manifests for easier reference
### Steps
#### Deploy a new namespace

1. Create a file 001-namespace.yaml. Refer to the manifests folder  
    This will deploy our demo-lab namespace where our service will be deployed.

2. Deploy: ```kubectl apply -f manifests/001-namespace.yaml```  
    expected output ```namespace/demo-lab created```. Verify with ```kubectl get ns```  
    ```
    NAME                 STATUS   AGE
    default              Active   26m
    demo-lab             Active   57s <-- new namespace created!!!
    kube-node-lease      Active   26m
    kube-public          Active   26m
    kube-system          Active   26m
    local-path-storage   Active   26m
    ```

#### Deploy a test service

1. Create the file for the service. 002-deployment.yaml  
   Notice how in the file I have specified the newly crated namespace.
2. Deploy the service ```kubectl apply -f manifests/002-deployment.yaml```  
   Expected output ```deployment.apps/web-app created```.  
   Verify with ```kubectl get pods -n demo-lab``` and ```kubectl get deployments -n demo-lab```  
   ```
   NAME                       READY   STATUS    RESTARTS   AGE
   web-app-69f8b8cf86-zq8wl   1/1     Running   0          4m49s
   AND
   NAME      READY   UP-TO-DATE   AVAILABLE   AGE
   web-app   1/1     1            1           4m53s
   ```