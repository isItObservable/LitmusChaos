# Is it Observable
<p align="center"><img src="/image/logo.png" width="40%" alt="Is It observable Logo" /></p>

## What Is Chaos Engineering ? What is Litmus Chaos 
<p align="center"><img src="/image/litmus.png" width="40%" alt="Litmus Logo" /></p>

This tutorial will deploy Litmus and run an Experimnets validating the request/limits settings.


## Prerequisite 
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm
### 1.Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
cloudtrace.googleapis.com \
clouddebugger.googleapis.com \
cloudprofiler.googleapis.com \
--project ${PROJECT_ID}
```
### 2.Create a GKE cluster
```
ZONE=us-central1-b
gcloud container clusters create onlineboutique \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```
### 3.Clone Github repo
```
git clone https://github.com/isItObservable/LitmusChaos.git
cd LitmusChaos
```
### 4. Deploy the sample Application

#### 0. Label your nodes to deploy
To run  our simulation, we need to label our ndoes
2 nodes will be dedicated to host :
* Litmus Chaos
* Prometheus


The rest of our nodes will be dedicated to host our Application HipsterShop.
We will create label name type that could have the following values :
* worker
* observability
```
kubectl get nodes -o wide
kubectl label <nodename1> node-type=observability
kubectl label <nodename2> node-type=worker
kubectl label <nodename3> node-type=worker
```
#### 1.Deploy Nginx Ingress Controller
```
helm repo add nginx-stable https://helm.nginx.com/stable
helm install ngninx nginx-stable/nginx-ingress --set controller.enableLatencyMetrics=true --set prometheus.create=true --set controller.nodeSelector.elector.node-type=observability
```
this command will install the nginx controller on the nodes having the label `observability` 

##### 2. get the ip adress of the ingress gateway
Since we are using Ingress controller to route the traffic , we will need to get the public ip adress of our ingress.
With the public ip , we would be able to update the deployment of the ingress for :
* hipstershop
* grafana
* litmus chaos
```
IP=$(kubectl get svc ngninx-nginx-ingress -ojson | jq -j '.status.loadBalancer.ingress[].ip')
```
##### 3. update the deployment file
```
sed -i "s,IP_TO_REPLACE,$IP," hipstershop/k8s-manifest.yaml
sed -i "s,IP_TO_REPLACE,$IP," hipstershop/k8s-manifest_nolimit.yaml 
sed -i "s,IP_TO_REPLACE,$IP," grafana/ingress.yaml
sed -i "s,IP_TO_REPLACE,$IP," k6/k8s-manifiest.yaml
```
#### 4.Prometheus
Our Chaos experiments will utilize the Prometheus as an Observabilty backend
We will neeed to deploy Prometheus only on the nodes having the label `observability`.
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack --set alertmanager.nodeSelector.selector.node-type=observability --set server.nodeSelector.selector.node-type=observability --set prometheusOperator.nodeSelector.selector.node-type=observability  --set prometheus.nodeSelector.selector.node-type=observability --set grafana.nodeSelector.selector.node-type=observability  
```

#### 5.HipsterShop
The hipsterShop deployment files has been customized to be deployed only on nodes having the label `workerThe hipsterShop deployment files has been customized to be deployed only on nodes having the label `worker`
```
kubectl create ns hipster-shop
kubectl -n hipster-shop create rolebinding default-view --clusterrole=view --serviceaccount=hipster-shop:default
kubectl apply -f hipstershop/k8s-manifest.yaml -n hipster-shop
```
#### 6. Expose Grafana
```
kubectl edit svc -l app.kubernetes.io/name=grafana
```
change to type NodePort
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus
    meta.helm.sh/release-namespace: default
  labels:
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: grafana
    app.kubernetes.io/version: 7.0.3
    helm.sh/chart: grafana-5.3.0
  name: prometheus-grafana
  namespace: default
  resourceVersion: "89873265"
  selfLink: /api/v1/namespaces/default/services/prometheus-grafana
spec:
  clusterIP: IPADRESSS
  externalTrafficPolicy: Cluster
  ports:
  - name: service
    nodePort: 30806
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/name: grafana
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```
Deploy the ingress by making sure to replace the service name of your grafana
```
kubectl apply -f grafana/ingress.yaml
```
Get the login user and password of Grafana
* For the password :
```
kubectl get secret --namespace default prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```
* For the login user:
```
kubectl get secret --namespace default prometheus-grafana -o jsonpath="{.data.admin-user}" | base64 --decode
```


### 5. Install Litmus Chaoas
```
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
kubectl create ns litmus
helm install install chaos litmuschaos/litmus --namespace=litmus --set ingress.enabled=true --ingress.host.name=litmus.$IP.nip.io --set portal.frontend.nodeSelector.selector.node-type=observability --set portal.server.nodeSelector.selector.node-type=observability --set mongo.nodeSelector.selector.node-type=observability
```

### 6. Verify you installation
Check the pods in the namespace litmus
```
kubectl get pods -n litmus
```
Expected output
```
NAME                                    READY   STATUS  RESTARTS  AGE
litmusportal-frontend-97c8bf86b-mx89w   1/1     Running 2         6m24s
litmusportal-server-5cfbfc88cc-m6c5j    2/2     Running 2         6m19s
mongo-0                                 1/1     Running 0         6m16s
```

Check the services :
```
kubectl get svc -n litmus
```
Expected output :
```
NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP PORT(S)                       AGE
litmusportal-frontend-service   NodePort    10.100.105.154  <none>      9091:30229/TCP                7m14s
litmusportal-server-service     NodePort    10.100.150.175  <none>      9002:30479/TCP,9003:31949/TCP 7m8s
mongo-service                   ClusterIP   10.100.226.179  <none>      27017/TCP                     7m6s
```

### 5. Deploy K6 test
The repository contains a DockerFile that :
- install the latest version of K6
- deploy their prometheus integration ( this integration will push the load testing data into Prometheus)
- add the loadtesting script `k6/loadgenerator.js` in the container

in the tutorial we won't build this image but utilize the prebuilt image : `hrexed/k6-prometheus:0.1`

Let's deploy the loadgenerator:
```
kubectl create ns k6
kubectl apply -f k6/k8s-manifiest.yaml -n k6
```



