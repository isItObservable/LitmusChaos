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
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)
kubectl apply -f nginx/deploy.yaml
```
this command will install the nginx controller on the nodes having the label `observability` 

##### 2. get the ip adress of the ingress gateway
Since we are using Ingress controller to route the traffic , we will need to get the public ip adress of our ingress.
With the public ip , we would be able to update the deployment of the ingress for :
* hipstershop
* grafana
* litmus chaos
```
IP=$(kubectl get svc nginx-ingress-nginx-controller -n ingress-nginx -ojson | jq -j '.status.loadBalancer.ingress[].ip')
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
helm install prometheus prometheus-community/kube-prometheus-stack --set server.nodeSelector.node-type=observability --set prometheusOperator.nodeSelector.selector.node-type=observability  --set prometheus.nodeSelector.selector.node-type=observability --set grafana.nodeSelector.selector.node-type=observability  
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
GRAFANAID=$(kubectl edit svc -l app.kubernetes.io/name=grafana)
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
helm  install chaos litmuschaos/litmus --namespace=litmus --set ingress.enabled=true --set ingress.host.name="litmus.$IP.nip.io" --set portal.frontend.nodeSelector.node-type=observability --set portal.server.nodeSelector.node-type=observability --set mongo.nodeSelector.node-type=observability
```

#### 6. update the litmus service

The kubernetes service needs to be `NodePort`, make sure to update the service type  of:
- litmusportal-frontend-service
- litmusportal-server-service

if you run 
```
kubectl get svc -n litmus
```
you should have :
```
NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
chaos-litmus-portal-mongo       ClusterIP   10.104.107.117   <none>        27017/TCP                       2m
litmusportal-frontend-service   NodePort    10.101.81.70     <none>        9091:30385/TCP                  2m
litmusportal-server-service     NodePort    10.108.151.79    <none>        9002:32456/TCP,9003:31160/TCP   2m
```


#### 8. update the litmus ingress 
To make the litmus ingress work you will need to update the current ingress deployed by Helm by adding the following annotations :
* `nginx.ingress.kubernetes.io/rewrite-target: /$1`
* `nginx.ingress.kubernetes.io/use-regex: "true"`

To update the ingress you will need to run :
```
kubectl edit ingress litmus-ingress -n litmus
```
here is the update version of the ingress :
```
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
      annotations:
          meta.helm.sh/release-name: chaos
          meta.helm.sh/release-namespace: litmus
          nginx.ingress.kubernetes.io/rewrite-target: /$1
          nginx.ingress.kubernetes.io/use-regex: "true"
      generation: 3
      labels:
          app.kubernetes.io/component: litmus-frontend
          app.kubernetes.io/instance: chaos
          app.kubernetes.io/managed-by: Helm
          app.kubernetes.io/name: litmus
          app.kubernetes.io/part-of: litmus
          app.kubernetes.io/version: 2.4.4
          helm.sh/chart: litmus-2.4.4
          litmuschaos.io/version: 2.4.0
         name: litmus-ingress
  namespace: litmus
  spec:
    ingressClassName: nginx
    rules:
    - host: litmus.<YOUR IP>.nip.io
      http:
      paths:
        - backend:
          service:
            name: chaos-litmus-frontend-service
            port:
              number: 9091
          path: /(.*)
          pathType: ImplementationSpecific
        - backend:
            service:
              name: chaos-litmus-server-service
              port:
                number: 9002
          path: /backend/(.*)
          pathType: ImplementationSpecific
```
Now that Litmus has been installed you can connect to the front service using :
* Url : http://litmus.<YOU INgress IP>.nip.io/
* Username : admin
* Password : litmus

### 9. Update the new deployment generated by the ChaosCenter
When connecting the first time to front end of Litmus, it will deploy in the same names space a `self agent`
Check if the deployment of the self agent is successful :
```
kubectl get pods -n litmus
```
Expected result :
```
NAME                                     READY   STATUS    RESTARTS   AGE
chaos-exporter-fd9667569-ztt7s           1/1     Running   0          9m9s
chaos-litmus-frontend-68d8d97586-84ws2   1/1     Running   0          47h
chaos-litmus-mongo-0                     1/1     Running   0          2d15h
chaos-litmus-server-df8fff796-xjqlc      2/2     Running   0          2d15h
chaos-operator-ce-7785cf959b-z7pgz       1/1     Running   0          8m7s
event-tracker-76fc96fb87-qwbxc           1/1     Running   0          7m13s
subscriber-77b87b5597-kdsb8              1/1     Running   0          10m
workflow-controller-5dd664cf4b-lxvng     1/1     Running   0          10m
```
The new deployment done byt the Chaos Center doesn't have the nodeSelector defined. 
Therefore we need to update the new deployment.
let's get the deployment name:
```
kubectl get deployment -n litmus
```
Expected result:
```
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
chaos-exporter          1/1     1            1           2d14h
chaos-litmus-frontend   1/1     1            1           2d15h
chaos-litmus-server     1/1     1            1           2d15h
chaos-operator-ce       1/1     1            1           2d14h
event-tracker           1/1     1            1           2d14h
subscriber              1/1     1            1           2d14h
workflow-controller     1/1     1            1           2d14h
```
We need to add a `nodeSelector` to  the following deployemnts :
* `chaos-exporter`
* `chaos-operator-ce`
* `event-tracker`
* `subscriber`
* `workflow-controller`

Edit each of the deployment and add :
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
   .....
  
spec:
  ....
  template:
    ...
    spec:
      dnsPolicy: ClusterFirst
      containers:
        ...
      nodeSelector:
        node-type: observability
     
```

### 10. Deploy the Litmus exporter
Litmus is automatically exposing metrics in a prometheus format.
In order to take advantage to it , we simply need to deploy a ServiceMonitor ( CRD created by the Prometheus Operator)
```
kubectl apply -f prometheus/podmonitor.yaml
```

### 11. Configure Prometheus by enabling the feature remo-writer

To measure the impact of our experiments on use traffic , we will use the load testing tool named K6.
K6 has a Prometheus integration that writes metrics to the Prometheus Server.
This integration requires to enable a feature in Prometheus named: remote-writer

To enable this feature we will need to edit the CRD containing all the settings of promethes: prometehus

To get the Prometheus object named use by prometheus we need to run the following command: 
```
kubectl get Prometheus
```
here is the expected output:
```
NAME                                    VERSION   REPLICAS   AGE
prometheus-kube-prometheus-prometheus   v2.32.1   1          22h
```
We will need to add an extra property in the configuration object :
```
enableFeatures:
- remote-write-receiver
```
so to update the object :
```
kubectl edit Prometheus prometheus-kube-prometheus-prometheus
```
After the update your Prometheus object should look  like :
```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus
    meta.helm.sh/release-namespace: default
  generation: 2
  labels:
    app: kube-prometheus-stack-prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: kube-prometheus-stack
    app.kubernetes.io/version: 30.0.1
    chart: kube-prometheus-stack-30.0.1
    heritage: Helm
    release: prometheus
  name: prometheus-kube-prometheus-prometheus
  namespace: default
spec:
  alerting:
  alertmanagers:
  - apiVersion: v2
    name: prometheus-kube-prometheus-alertmanager
    namespace: default
    pathPrefix: /
    port: http-web
  enableAdminAPI: false
  enableFeatures:
  - remote-write-receiver
  externalUrl: http://prometheus-kube-prometheus-prometheus.default:9090
  image: quay.io/prometheus/prometheus:v2.32.1
  listenLocal: false
  logFormat: logfmt
  logLevel: info
  paused: false
  podMonitorNamespaceSelector: {}
  podMonitorSelector:
  matchLabels:
  release: prometheus
  portName: http-web
  probeNamespaceSelector: {}
  probeSelector:
  matchLabels:
  release: prometheus
  replicas: 1
  retention: 10d
  routePrefix: /
  ruleNamespaceSelector: {}
  ruleSelector:
  matchLabels:
  release: prometheus
  securityContext:
  fsGroup: 2000
  runAsGroup: 2000
  runAsNonRoot: true
  runAsUser: 1000
  serviceAccountName: prometheus-kube-prometheus-prometheus
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector:
  matchLabels:
  release: prometheus
  shards: 1
  version: v2.32.1
```
### 12. Deploy K6 test
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



