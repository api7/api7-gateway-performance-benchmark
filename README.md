# API7 Gateway Performance Benchmark

## Prerequisite(s)

- K8s cluster
- Install API7 Gateway (CP and DP) 
- A test upstream ([NGINX Upstream](./k8s-resources/upstream-nginx.yaml))
- Install [ADC](https://docs.api7.ai/enterprise/best-practices/devops-adc#adc-introduction) (A CLI tool to connect gateway instances and publish configurations)

## Steps

### Config K8s Cluster Node Label

1. Verify cluster is running:

```
$ kubectl get nodes

NAME                            STATUS   ROLES    AGE   VERSION
ip-172-31-22-219.ec2.internal   Ready    <none>   54m   v1.29.3-eks-ae9a62a
ip-172-31-24-73.ec2.internal    Ready    <none>   54m   v1.29.3-eks-ae9a62a
ip-172-31-32-45.ec2.internal    Ready    <none>   61m   v1.29.3-eks-ae9a62a
```

2. Labeling of nodes:

```
$ kubectl label nodes ip-172-31-22-219.ec2.internal nodeName=upstream
$ kubectl label nodes ip-172-31-24-73.ec2.internal nodeName=wrk2
$ kubectl label nodes ip-172-31-32-45.ec2.internal nodeName=api7ee

node/ip-172-31-22-219.ec2.internal labeled
node/ip-172-31-24-73.ec2.internal labeled
node/ip-172-31-32-45.ec2.internal labeled
```

3. Create a new namespace:

We install all resources in the same namespace "api7".

```
$ kubectl create namespace api7

namespace/api7 created
```

### Install API7 EE Control Plane

1. Install API7 Control Plane:

```
$ helm repo add api7 https://charts.api7.ai            
$ helm repo add apisix https://charts.apiseven.com
$ helm repo update
# Specify the Node for the api7ee installation (the label we set for the Node earlier).
$ helm install api7ee3 api7/api7ee3 --set nodeSelector."nodeName"=api7ee --set postgresql.primary.nodeSelector."nodeName"=api7ee --set prometheus.server.nodeSelector."nodeName"=api7ee -n api7
```

By default postgresql and prometheus enable persistent storage and you may receive some errors if you do not configure [StorageClass](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#dynamic) for your cluster. 

You can also temporarily disable persistent storage with the command below, but be careful **not to disable it during a production environment** or the data will be lost after the Pod restarts.

```
$ helm install api7ee3 api7/api7ee3 --set nodeSelector."nodeName"=api7ee --set postgresql.primary.nodeSelector."nodeName"=api7ee --set prometheus.server.nodeSelector."nodeName"=api7ee --set postgresql.primary.persistence.enabled=false --set prometheus.server.persistence.enabled=false -n api7
```

2. Check deployment status:

```
$ kubectl get svc -owide -l app.kubernetes.io/name=api7ee3 -n api7

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE    SELECTOR
api7ee3-dashboard    ClusterIP   10.100.89.120   <none>        7080/TCP,7443/TCP   2m3s   app.kubernetes.io/component=dashboard,app.kubernetes.io/instance=api7ee3,app.kubernetes.io/name=api7ee3
api7ee3-dp-manager   ClusterIP   10.100.3.182    <none>        7900/TCP,7943/TCP   2m3s   app.kubernetes.io/component=dp-manager,app.kubernetes.io/instance=api7ee3,app.kubernetes.io/name=api7ee3
```

3. Forward the dashboard port to the local machine, login to the console and upload the licence:

[License Free Trial](https://api7.ai/try?product=enterprise)

```
$ kubectl -n api7 port-forward svc/api7ee3-dashboard 7443:7443
```

### Install API7 Gateway (Data Plane)

1. Login to the dashboard and configure the "Control Plane Address" in the **Gateway Settings**: `http://api7ee3-dp-manager:7900`

![Gateway Settings](https://static.apiseven.com/uploads/2024/05/13/S5O8D2XJ_gateway-settings.jpeg)

2. Disable global plugin prometheus:

![Disable Prometheus](https://static.apiseven.com/uploads/2024/05/13/2XUKYxfv_disable-prometheus.jpeg)

3. Add Gateway Instance:

![Add Gateway Instance](https://static.apiseven.com/uploads/2024/05/13/mb3sy7wc_add-gateway-instance.jpeg)

Select the **Kubernetes** method and configure the "namespace" to generate an install script and run it. This step will get a token to connect the CP to the DP. For example:

```
$ helm repo add api7 https://charts.api7.ai
$ helm repo update
$ helm install -n api7 --create-namespace api7-ee-3-gateway api7/gateway \
  --set "apisix.extraEnvVars[0].name=API7_CONTROL_PLANE_TOKEN" \
  --set "apisix.extraEnvVars[0].value=a7ee-nw9ZWpFo4U67cYwNkN7456ACkJH6P0waB6PNT1IqCUT6eh8c3Fe48fceb18c474ff496bc8913140c6cec" \
  --set "apisix.image.repository=api7/api7-ee-3-gateway" \
  --set "apisix.image.tag=3.2.11.0" \
  --set "etcd.host[0]=http://api7ee3-dp-manager:7900" \
  --set "apisix.replicaCount=1"
```

**Note:** We should install the Gateway on a separate Node and set up the `worker_process` as needed. You can set it with a flag:
- config gateway worker_process: `--set "nginx.workerProcesses"=1`
- config node selector: `--set apisix.nodeSelector.nodeName=api7ee`
- run as root user:
  - `--set apisix.securityContext.runAsNonRoot=false`
  - `--set apisix.securityContext.runAsUser=0`

### Install Upstream NGINX

```
kubectl apply -f k8s-resources/upstream-nginx.yaml
```

### Install wrk2

```
kubectl apply -f k8s-resources/wrk2.yaml
```

### Install ADC

- https://run.api7.ai/adc/release/adc_0.8.0_linux_amd64.tar.gz
- https://run.api7.ai/adc/release/adc_0.8.0_linux_arm64.tar.gz
- https://run.api7.ai/adc/release/adc_0.8.0_darwin_arm64.tar.gz

1. Generate API Token

ADC needs an API Token to access Admin APIs. You can generate an API Token from the Dashboard:

- Login -> Organization -> Tokens -> Generate New Token

2. Configure ADC Credentials

Create a `.env` file in the same directory as the ADC binary.

```
ADC_BACKEND=api7ee
ADC_SERVER=https://127.0.0.1:7443
ADC_TOKEN=a7ee-6baF8488i8wJ5aj7mEo3BT705573eC8GH905qGrdn889zUWcR37df66a34e9954b61918c5dfd13abce3e
ADC_GATEWAY_GROUP=default
```

3. Verify ADC Status

```
$ ./adc ping

Connected to the backend successfully!
```

## Starting Performance Testing

We have provided adc configurations for each of the 9 scenarios, which you can use directly:

1. [One route without plugins](./api7-resources-configuration/1-one-route-without-plugin.yaml)
2. [One route with limit-count plugin](./api7-resources-configuration/2-one-route-with-limit-count.yaml)
3. [One route with key-auth and limit-count plugin](./api7-resources-configuration/3-one-route-with-key-auth-and-limit-count.yaml)
4. [One route and one consumer with key-auth plugin](./api7-resources-configuration/4-one-route-with-key-auth.yaml)
5. [100 routes without plugins](./api7-resources-configuration/5-100-route-without-plugin.yaml)
6. [100 routes with limit-count plugin](./api7-resources-configuration/6-100-route-with-limit-count.yaml)
7. [100 routes and 100 consumers with key-auth and limit-count plugin](./api7-resources-configuration/7-100-route-and-consumer-with-key-auth-limit-count.yaml)
8. [100 routes and 100 consumers with key-auth plugin](./api7-resources-configuration/8-100-route-and-consumer-with-key-auth.yaml)
9. [One route with mocking plugin](./api7-resources-configuration/9-one-route-with-mocking.yaml)

## Example

```
$ ./adc ping

Connected to backend successfully!

$ ./adc sync -f api7-resources-configuration/<filename>.yaml
```
