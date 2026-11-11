# kubectl-first-steps

Testing k8s-git-kubectl-helm-argocd-prometheus-grafana

# Preparation
1. Create a folder where the project will reside
2. Clone the repo from github  
```git clone git@github.com:Grellissimo/k8s.git```
3. Navigate into the folder, and initialize your preferred git manager. I use lazygit as i find it easy to use compared the CLI
4. Create a new k8s cluster if not already available. This can be done in many different ways, for simplicity, i'll use a KinD (kubernetes in docker) cluster directly created by docker desktop. A much professional solution would definitely be something like AWS EKS.  
4.1 After waiting some minutes, run ```kubectl get nodes``` to ensure cluster is ready

# Kubectl
kubectl is the official command-line interface for communicating directly with the Kubernetes Control Plane API. While it is an indispensable tool for cluster administration, live querying, port-forwarding, and deep incident diagnostics (describe, logs), using manual kubectl apply commands for deploying applications is not considered a production best practice. Relying solely on raw manifests makes multi-environment management brittle and error-prone. In modern cloud-native engineering, kubectl is best reserved for operational triage and cluster inspection, while deployment workflows are delegated to specialized tooling—transitioning first to Helm for templating and packaging, and ultimately to ArgoCD for automated, declarative GitOps continuous delivery.  
Useful info to write good YAML files and reference primitives  
```kubectl api-resources``` will give you all info about APIversion and its kind.  
```kubectl explain <resource>``` will give you even more information for a specific resource.  
Can even drill down the resource if required ```kubectl explain <resource>.spec```

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

#### Deploy a test app

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

3. Scale up and down methods  
    3.1 in line  
        ```kubectl scale deployment web-app --replicas=5 -n demo-lab```  

    3.2 edit deployment  
        ```kubectl edit deployment web-app -n demo-lab```
        make direct changes in the YAML file  
    3.3 update the YAML file, set replicas: 5 and deploy
        ```kubectl apply -f manifests/002-deployment.yaml```  
    3.4 expected output  
        ```kubectl get pods -n demo-lab```
    ```
    NAME                       READY   STATUS    RESTARTS   AGE  
    web-app-69f8b8cf86-cc7hd   1/1     Running   0          22s  
    web-app-69f8b8cf86-f2p7p   1/1     Running   0          22s      
    web-app-69f8b8cf86-l87bz   1/1     Running   0          22s
    web-app-69f8b8cf86-tt5q2   1/1     Running   0          22s
    web-app-69f8b8cf86-zq8wl   1/1     Running   0          170m
    ```
        
#### Deploy a service
A service is used to expose an application to the external side. As of now, the web-app is running but not reachable at all.
Type of services:  
- ClusterIP: exposes the service on an internal-only ip.
- NodePort: Opens a dedicated port (range: 30000-32767) on every node in the cluster accessible by Node-IP:<port> used for direct external access in on-prem or bare metal clusters without a cloud load balancer (LB)
- LoadBalancer: Requests hit the LB with a public IP, and are redirected to the web-app
- ExternalName: Directs traffic to an external DNS name. Used to connect cluster apps to legacy or external cloud databases.  
In short, for this exercise as i will only need to confirm i can reach the web-app, the ClusterIP will suffice. In a prod environment the solution would be
- LB > Ingress controller (i.e.: nginx) > ClusterIP > web-app
- LB > webapp (a target for each deployment/pod)  
The latest is the best at the moment

Deployment:
1. Refer to the file 003-service.yaml  
   Explanation:  
   ```
   1. apiVersion: v1 & kind: Service
   2. namespace: demo-lab - A Service must live in the same namespace as the pods it is trying to connect to.
   3. The Critical Link: selector: app: web-app 
      • This is how the Service discovers pods.
      • It searches for any pod inside demo-lab that has the label app: web-app.
      • Because the deployment has multiple replicas, the Service automatically registers all of their IP addresses and round-robin load-balances traffic across them.
   4. port: 80 vs targetPort: 80
      • port (80): The port the Service itself listens on inside the cluster. Other services talk to web-app-service:80.
      • targetPort (80): The port on the actual Pod container where NGINX is listening.
    ```
2. Deploy with the classic ```kubectl apply -f manifests/003-service.yaml```
3. Inspect it with ```kubectl get svc -n demo-lab``` and ```kubectl get endpointslice -n demo-lab``` 
   Expected output:
   ```
   NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
   web-app-service   ClusterIP   10.96.111.176   <none>        80/TCP    0s
   
   NAME                    ADDRESSTYPE   PORTS   ENDPOINTS                                       AGE
   web-app-service-482dk   IPv4          80      10.244.0.5,10.244.0.13,10.244.0.7 + 2 more...   5m15s
   ```
4. Verification. Port-forward tunnel from laptop port to the service inside the k8s cluster.  
   Run ```kubectl port-forward svc/web-app-service 8080:80 -n demo-lab```  
   Expected output after running it
   ```
   Forwarding from 127.0.0.1:8080 -> 80
   Forwarding from [::1]:8080 -> 80
   ```
   Output after testing connection via curl (can also open a browser if needed)
   ```
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   -truncated output-
   ```

## Troubleshooting scenario

Common issue: ImagePullBackOff

1. Simulate issue where the image pointer is wrong  
   ```kubectl set image deployment/web-app nginx=nginx:this-tag-does-not-exist -n demo-lab```  
   OR  
   Update the 002-deployment.yaml to point to the wrong image tag and apply (as above)
2. Observe the failure  
   ```kubectl get pods -n demo-lab```
   ```
   NAME                       READY   STATUS             RESTARTS   AGE
   web-app-5df77999f-7zfhg    0/1     ErrImagePull       0          3m29s
   web-app-5df77999f-8p8zw    0/1     ImagePullBackOff   0          3m29s
   web-app-5df77999f-gr65r    0/1     ImagePullBackOff   0          3m29s
   web-app-69f8b8cf86-cc7hd   1/1     Running            0          17m
   web-app-69f8b8cf86-f2p7p   1/1     Running            0          17m
   web-app-69f8b8cf86-l87bz   1/1     Running            0          17m
   web-app-69f8b8cf86-zq8wl   1/1     Running            0          3h7m
   ```
3. Investigation  
   run ```kubectl describe pod <failing-pod-name> -n demo-lab``` to find out details about the issue - cluster/infrastructure  
   expected output  
   ```
   Events:
   Type     Reason   Age                   From     Message
   ----     ------   ----                  ----     -------
   Normal   BackOff  3m6s (x750 over 21h)  kubelet  spec.containers{nginx}: Back-off pulling image "nginx:nginx"
   Warning  Failed   3m6s (x750 over 21h)  kubelet  spec.containers{nginx}: Error: ImagePullBackOff
   ```
   run ```kubectl logs <failing-pod-name> -n demo-lab``` to find out logs about the issue - app code crashes  
   expected output  
   ```
   Error from server (BadRequest): container "nginx" in pod "web-app-5df77999f-7zfhg" is waiting to start: trying and failing to pull image
   ```
4. Revert the wrong update. In an event of catastrophic failure, where the service needs to be working asap, we could do the rollout. However, it is best practice to just revert the changes just pushed. In a professional environment there more mechanism in place and GitOps and IaC principles are followed. So in theory just switching to the main/master branch and redeploying will be reverting back to the latest working configuration.  
Note: There are much better and advanced solutions to do this but for the sake of the exercise, we are not (yet) implementing it.  
   For This time we will fix it using the rollout command:
   ```kubectl rollout undo deployment/web-app -n demo-lab```  
   Expected output: (we are advised to use another approach)
   ```
   Warning: resource deployments/web-app was previously managed with 'kubectl apply'. Rolling back will not update the kubectl.kubernetes.io/last-applied-configuration annotation, which may cause unexpected behavior on future 'kubectl apply' operations. Consider using 'kubectl apply' with your previous configuration file instead.
   deployment.apps/web-app rolled back
   ```  
   Status:
   ```
   NAME                       READY   STATUS    RESTARTS   AGE
   web-app-69f8b8cf86-4wnxr   1/1     Running   0          9m18s  <--- notice only one pod was affected as the default RollingUpdate deployment strategy is to ensure zero downtime. So that there
   web-app-69f8b8cf86-cc7hd   1/1     Running   0          22h
   web-app-69f8b8cf86-f2p7p   1/1     Running   0          22h
   web-app-69f8b8cf86-l87bz   1/1     Running   0          22h
   web-app-69f8b8cf86-zq8wl   1/1     Running   0          25h
   ```

# Branching
This is a good time to insert the branching concept. So far i have been using the main branch (also known as master sometimes). However, In production environments, the main branch represents the stable, deployable source of truth and is strictly protected. Committing directly to main introduces the risk of deploying unvetted configurations that could cause outages. Instead, all new features, bug fixes, and infrastructure changes are developed in dedicated branches. This workflow isolates changes, enables automated CI/CD validation and peer code reviews via Pull Requests (PRs), and ensures thatfaulty code can be revised or abandoned without jeopardizing the stability of the live cluster.
There is some standard to follow (i.e.: feature/ or fix/ or doc/ branches) but this can be customized depending on the company internal standards.



# This continues in the HELM repo