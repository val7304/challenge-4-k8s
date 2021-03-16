
# Challenge 4 - Managing Workloads on Kubernetes

Date limite de remise du challenge: le 06 février 2021 

### Task 1:
L'objectif ici est de déployer l'application WordPress sur le cluster mono-noeud Docker Desktop Kubernetes. 

---
  
### Task 2:
L'objectif ici est d'installer Kubernetes **Metrics Server** et de l'exploiter pour visualiser les métriques sur le dashboard UI de Kubernetes, dashboard vu déjà dans le tout premier lab du module Kubernetes. 

Travail demandé:
- Lire l'article court intitulé from [Heapster to Metrics Server](https://www.educative.io/courses/advanced-kubernetes-techniques/qVYQ5g9pwAk) sur _educative_ afin d'avoir un aperçu de Metrics Server.
   
- Procéder à l'installation de Metrics Server avec le gestionnaire de packages Helm 3. Ci dessous, je vous ai préparé un petit guide pour vous faciliter cette tâche.

- Redeployer l'application Dashboard de Kubernetes et observez qu'elle visualise, grâce à Metrics Server, les métriques associées aux workloads.  Ci dessous, je vous ai préparé un petit guide vous aider a déployer l'application Dashboard.

### Guide pour installation de Metrics-Server

 - Metrics Server is not installed by default. To check this, the following `kubectl top` does not work. We will recheck it once Metrics-server is installed, and then it will work.
  
  ```shell
  kubectl top nodes
  ```
 - Add Kubernetes charts repo to the local Helm repos
   
  ```shell
  helm repo add stable https://charts.helm.sh/stable
  helm repo update
  ```

- List the repos

  ```shell
  helm repo list
  ```
- Search for the chart
  
  ```
  helm search repo metrics-server
  ```
- Save locally the values

  ```shell
  helm show values stable/metrics-server  > my-metrics.values
  ```
- Open `my-metrics.values` and perform  the following two changes:

  1-Change hostNetwork enabled to **true**

    ```shell
    > hostNetwork 
        change enabled : true 
    ```
  2- Activate `kubelet-insecure-tls`
  args Remove empty brackets [] and uncomment `kubelet-insecure-tls` line

    ```shell
    - -- kubelet-insecure-tls
    ```	

- Install the metrics-servicer using the updated values

  ```shell
  helm install my-metrics-server stable/metrics-server --values my-metrics.values
  ```

- Check that metrics server is running by issuing. The metrics-server may take serveralminutes to be ready. 

  ```shell
  kubectl top nodes
  ```

- For uninstalling, use the follwing command

  ```shell
  helm uninstall my-metrics-server
  ```

### Guide pour le déploiement de Kubernetes Dashboard UI

The Dashboard UI is not deployed by default. To deploy it, run the following command:

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml  
```

### Accessing the Dashboard UI 

You can access Dashboard using the kubectl command-line tool by running the following command. Hence, The UI can only be accessed from the machine where the command is executed. 

```shell
kubectl proxy
```
Kubectl will make Dashboard available at <http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/>.
You will then be prompted with this page, to enter the credentials:

<img src="images/dashboard_authentication.png" width="600px" height="280px"/>

Dashboard only supports logging in with a Bearer Token. To get the token type in the following command:

```shell
kubectl -n kubernetes-dashboard describe secret default
```
The dashboard UI is then displayed 

<img src="images/dashboard_metrics.png" width="840px" height="520px"/>


# Task 3:
L'objectif ici est d'expérimenter la fonction de **Horizontal Pod Autoscaler (HPA)** et de comprendre son fonctionnement.


Travail demandé : Implémenter les étapes du parcours suivant et faire une synthèse sur les techniques de configuration d'un HPA.

### Horizontal Pod Autoscaler Walkthrough

Horizontal Pod Autoscaler automatically scales the number of pods in a replication controller, deployment, replica set or stateful set based on observed CPU utilization (or, with beta support, on some other, application-provided metrics).

This step walks you through an example of enabling Horizontal Pod Autoscaler for an application php-apache server. _metrics-server monitoring needs to be deployed in the cluster_ to provide metrics via the resource metrics API, as Horizontal Pod Autoscaler uses this API to collect metrics. 

- Installing [Metrics Server](https://github.com/kubernetes-sigs/metrics-server) 
  
   See the guide given u-in the task 2.

- Run & expose `php-apache server`
  First, we will start a deployment running the image and expose it as a service using the following configuration: (`challenge4-task03-horizontal-pod-autoscaler.yaml`)
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: php-apache
  spec:
    selector:
      matchLabels:
        run: php-apache
    replicas: 1
    template:
      metadata:
        labels:
          run: php-apache
      spec:
        containers:
        - name: php-apache
          image: k8s.gcr.io/hpa-example
          ports:
          - containerPort: 80
          resources:
            limits:
              cpu: 500m
            requests:
              cpu: 200m
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: php-apache
    labels:
      run: php-apache
  spec:
    ports:
    - port: 80
    selector:
      run: php-apache
  ```
  - Apply the application and service.
    ```shell
    kubectl apply -f challenge4-task03-horizontal-pod-autoscaler.yaml
    ```
- Create Horizontal Pod Autoscaler    
   Now that the server is running, we will create the autoscaler using kubectl autoscale. The following command will create a Horizontal Pod Autoscaler that maintains between 1 and 10 replicas of the Pods controlled by the php-apache deployment we created in the first step of these instructions. Roughly speaking, HPA will increase and decrease the number of replicas (via the deployment) to maintain an average CPU utilization across all Pods of 50% (since each pod requests 200 milli-cores by kubectl run), this means average CPU usage of 100 milli-cores).
  ```shell
  kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
  ```
  We may check the current status of autoscaler by running:
  ```shell
  kubectl get hpa
  NAME         REFERENCE               TARGETS      MINPODS   MAXPODS   REPLICAS   AGE  
  php-apache   Deployment/php-apache    0%/50%      1         10        0          9s 
  ```
  Please note that the current CPU consumption is 0% as we are not sending any requests to the server (the TARGET column shows the average across all the pods controlled by the corresponding deployment).

- Increase load
   
   Now, we will see how the autoscaler reacts to increased load. We will start a container, and send an infinite loop of queries to the php-apache service (please run it in a different terminal):
  ```shell
  kubectl run -it --rm load-generator --image=k8s.gcr.io/busybox
  ```
  Hit enter for command prompt:
  ```shell
  while true; do wget -q -O- http://php-apache; done
  ```
  Within a minute or so, we should see the higher CPU load by executing:

  ```shell
  kubectl get hpa  
  NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
  php-apache   Deployment/php-apache   248%/50%   1         10        1          6m24s
  ```
  Here, CPU consumption has increased to `248%` of the request. As a result, the deployment was resized to 5 replicas:
  ```shell
  kubectl get deployment php-apache
  NAME         READY   UP-TO-DATE   AVAILABLE   AGE
  php-apache   5/5     5            5           8m7s
  ```
    > Note: It may take a few minutes to stabilize the number of replicas. Since the amount of load is not controlled in any way it may happen that the final number of replicas will differ from this example.

- Stop load
     
  We will finish our example by stopping the user load.
  In the terminal where we created the container with busybox image, terminate the load generation by typing `<Ctrl> + C`.
  Then we will verify the result state (after a minute or so):
  ```shell
  kubectl get hpa
  NAME         REFERENCE                     TARGET       MINPODS   MAXPODS   REPLICAS   AGE
  php-apache   Deployment/php-apache/scale   0% / 50%     1         10        1          11m
  ```
  ```shell
  kubectl get deployment php-apache
  NAME         READY   UP-TO-DATE   AVAILABLE   AGE
  php-apache   1/1     1            1           27m
  ```
  Here CPU utilization dropped to 0, and so HPA autoscaled the number of replicas back down to 1.

  > Note: Autoscaling the replicas may take a few minutes

- Clean Up
     
  Remove the deployment and the service
  ```shell 
  kubectl delete -f challenge4-task03-horizontal-pod-autoscaler.yaml
  ```  
  Remove the HPA autoscaler
  ```shell 
  kubectl delete hpa php-apache
  ```
