# kubernetes-workshop

## This file contains instructions for the Uniper Kubernetes workshop
Prerequisites
- You are running the workshop in Google Chrome or Microsoft Edge Explorer
- You have been granted access to this repository and Github Codespaces
- Your user has been assigned to a namespace for this exercise
- You can access the application on URL "https://nonprod-aks-test.uniperapps.com/{your_namespace_name}". Put your namespace in place of "{your_namespace_name}". Example: https://nonprod-aks-test.uniperapps.com/app1

## Set up
- Open the azuredevops project with codespaces. Create a personal codespace and wait until it is configured (2 - 3 minutes)
- Open this Readme file
- Follow the instruction to login in to subscription "" to cluster ""
- Login from the command line:
  - az login
  - az account set --subscription ""
  - az aks get-credentials --resource-group "" --name ""
- run 'kubectl get ns' to see the namespaces of the cluster
- run 'kubectl -n {your_namespace_name} get pods' to see your pods.
  
## Exercises
### 0. Setup
- Check access to the cluster
  - After you run "az aks get-credentials --resource-group "" --name "" ", you will be asked to go to a URL and enter the code, if you do not see any error, it means you have access to the cluster.
- See pods and deployments
  - Run
    - kubectl get deployments -n {your_namespace_name} #Lists all the deployments in your namespace, in this case, you are supposed to see "devopsdemo"
    - kubectl get pods -n {your_namespace_name} #Lists all pods running in your namespace
- Invoke URL
  - https://nonprod-aks-test.uniperapps.com/{your_namespace_name} (place your namespace name)
  - Once you goto the URL, you are supposed to see a web-page with a message.
- Change your name using configMap
  - ConfigMaps are used to store non-confidential data in key-value pairs. These key value pairs are available as environment variables for a Pod.
  - Change the value of the name key in demoappconfigmap.yaml. Ex. name: your_name. After changing the name, run the following command:
  - kubectl apply -f demoappconfigmap.yaml -n "your_namespace"
  - The above command does not trigger rolling update (updates to a deployment). To see the changes, first delete the pods using following command:
  - kubectl delete pods -n "your namespace name" --all #Deletes all pods from your namespace.
  - The above command will trigger the pod re-creation with updated config values.
  - Recheck the URL, you will see your changed name now.
- Check logs
  - To check the logs of your application, run the following command.
  - kubectl get pods -n "namespace name" #First get the pod name, copy the pod name
  - kubectl logs "pod name" -n "namespace name" #Place the copied pod name
  - To get a continuous log, run the following command
  - kubectl logs "pod name" -n "namespace name" -f #Place the copied pod name, to exit press Ctrl + C
  - When your partner tries to get a message from your application (by clicking on the message), you will see text "#### Called method (through API)####" in the logs
### 1. Role-based access control
- Log in to a cluster
- Check roles
  - kubectl get roles -n "your namespace name"
    - You should see "namespace-admin"
- Check rolebindings
  - kubectl get rolebindings -n "your namespace name"
    - You should see "namespace-admin"
- Check whom all have access to the namespace (initially, only your KID should be listed)
  - kubectl describe rolebinding namespace-admin -n "your namespace name"
    - In the description of the rolebinding "namespace-admin", you will see only your KID in the subjects section (only you have access to your namespace)
- Check if your partner can list pods/ deployments of your namespace.
  - Ask your partner to run the following command
    - kubectl get pods -n "your namespace name"
    - Your partner is supposed to get: namespaces "your namespace name" is forbidden error.
- Add a user (to allow access to your namespace)
  - Edit file namespace-admin-rbg.yaml, add the user's KID (all upper case except uniper.energy. Ex. ""@uniper.energy) in the subjects section, one of them should be yours. Run the following command:
  - kubectl apply -f namespace-admin-rbg.yaml -n "your namespace name"
  - After running the above command, the user will have access to your namespace. Check by running the following command:
  - kubectl get pods -n "your namespace name"
- Remove user (to remove user just added in the previous step)
  - Edit file namespace-admin-rbg-single-user.yaml, makes sure only your KID is added.
  - kubectl apply -f namespace-admin-rbg-single-user.yaml -n "your namespace name"
  - The above command will remove the user's access
### 2. Resource Quota
- Check resource quota for a namespace by running the following command. Running the following command will list out the limits and requests for the namespace. Used column represents the amount of memory and CPU used.
  - kubectl describe resourcequota -n "your namespace name"
- Try deployment with resource quota
  - In the file devopsdemo-deployment.yaml, change the values of CPU and memory under containers -> resources -> requests to 200Mi for memory (Ex. 200Mi) and 50m for CPU.
  - Run the following command to apply the changes
    - kubectl apply -f devopsdemo-deployment.yaml -n "your namespace name"
    - The above command will trigger a rollout update for the deployment devopsdemo, which means old pods will get deleted, and new ones will be created. Additionally, the rollout status can be checked using the following command
    - kubectl rollout status deploy/devopsdemo -n "your namespace name"
    - The above command will output "deployment "devopsdemo" successfully rolled out" if the update is successful.
  - Recheck the resource quota of namespace after the deployment is updated.
    - kubectl describe resourcequota -n "your namespace name"
    - You will notice changes in the Used values.
  - Edit the devopsdemo-deployment.yaml file and change the values to the initial values, i.e. memory to 400Mi and CPU to 100m. To apply, run:
    - kubectl apply -f devopsdemo-deployment.yaml -n "your namespace name"
- Scale deployment (Increasing number of application replicas to handle more load)
  - Scaling a deployment creates a new pod in the same namespace. The application can be scaled up using the following command:
  - kubectl scale deployment.v1.apps/devopsdemo --replicas=2 -n "your namespace name"
  - Currently, the namespace's resource quota is configured to accommodate only two replicas in a namespace. Increasing the replicas to 3 or more will result in "Error creating: pods "devopsdemo-67f7957cf8-pzk5l" is forbidden: exceeded quota". This error message can be seen in the events of your namespace. Run following command to check the Events
  - kubectl get events -n "your namespace name"
- Validate cluster scaling 
  - Applications are deployed on a Node, a computing unit with its own CPU capacity and memory capacity. Once a Node exhausts its CPU and Memory, a new Node is created to accommodate new application deployments, this is called cluster scaling.
  - We have 20 namespaces, each of these namespaces can accommodate at most one replica without triggering scaling. But when each namespace runs two replicas each, scaling is triggered.
  - kubectl get nodes #Lists the nodes in a cluster
  - Scale up your application using the following command
    - kubectl scale deployment.v1.apps/devopsdemo --replicas=2 -n "namespace name"
  - Scaling up takes time (5-10 mins), you can check the number pods in yournamespace has now increased to 2, run the following command to check
    - kubectl get pods -n "your namespace name"
### 3. Ingress Controller
- Check ingress-controller namespace (You will have just readers access to this namespace). Run following command to see the pods in ingress-controller namespace
  - kubectl get pods -n ingress-controller
- Check logs. To check the logs of the ingress-controller, run the following command with pod name of the first pod.
  - kubectl logs "pod_name" -n ingress-controller -f
- Access your application again using the URL, you will notice your each access is getting logged.
### 5. Network Policies
- Network Policies are configured to regulate the traffic between two namespaces. Here, the network policies are configured to allow bi-directional communication between your namespace and your partner's namespace. Run following command to check the network policy of your namespace:
  - kubectl get networkpolicy -n "your namespace name"
  - Currently only one Network Policy is in place: kubernetes-namespace-network-policy, you can describe it using the following command:
    - kubectl describe networkpolicy -n "your namespace name"
    - Under spec -> ingress in `from` you specify from which namespace you will allow the incoming traffic from. Currently one of the `from` is your partner's namespace. To stop incoming traffic from your partner's namespace, remove just the `from` with your partner's namespace, re-apply the file using following command
      - kubectl apply -f kubernetesnetworkpolicy.yaml -n "your namespace name"
      - Verify that when your partner tries to access the message from your namespace, it keeps on waiting. Revert the changes and apply the changed file again using above command.
- Validate result (By getting inside the Pod)
  - To get inside the pod, first get the pod name.
  - kubectl get pods -n "your namespace name" #Copy the pod name
  - Next, run following command to get inside the pod
    - kubectl exec -ti "pod name" -n "your namespace name" -- /bin/sh
    - When you just see "/#" you are inside the pod
  - Considering you have already reverted the changes done in the above step, run the following command
    - curl devopsdemo."partner namespace_name".svc.cluster.local -I
    - Replace "namespace_name" from the above command with your partner's namespace. `devopsdemo."namespace_name".svc.cluster.local` in the above command means you are trying to connect to devopsdemo (application name/ service name) deployed in "your partner namespace". Here, nonprod-aks-test.uniperapps.com/app2 will not work because its an external URL which everyone can access, but in the background pods communicate using service names, and currently you are inside the pod, so you should use service name. The above command will give you following message:
      - HTTP/1.1 200 <br>
        Content-Type: text/html;charset=UTF-8 <br>
        Content-Language: en-US <br>
        Content-Length: 865 <br>
        Date: Wed, 11 Nov 2020 12:58:27 GMT <br>
      - The above output means the communication is allowed (HTTP/1.1 200)
    - You can exit the pod using the command `exit`. Once you are outside, again change the networkpolicy file by removing the incoming traffic, apply the changed file, get into the pod again and run the above mentioned CURL command. You will notice, it keeps you waiting for the output which means communication is not allowed.
    - After checking, exit from the pod and revert the changes and apply the network policy again.
### 6. Pod Security Policies
- Containers can optionally run in a privileged mode, which means that the process inside the container has unrestricted access to the host system's resources. In most cases, it’s a security risk to let the container run in privileged mode. Setting privileged to false will prevent the container from running in privileged mode.
- Try to deploy a privileged pod
  - Edit devopsdemo-deployment.yaml file, change value of `privileged` to true under `securityContext`. Apply the changes by running the following command:
  - kubectl apply -f devopsdemo-deployment.yaml -n "your namespace name"
  - You will see error "cannot set `allowPrivilegeEscalation` to false and `privileged` to true"
