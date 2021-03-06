# gRPC-GKE Service LoadBalancer "helloworld"


Sample application demonstrating RoundRobin gRPC loadbalancing  on Kubernetes for interenal services.


gRPC can stream N messages over one connection.  When k8s services are involved, a single connection to the destination will terminate
at one pod.  If N messages are sent from the client, all N messages will get handled by that pod resulting in imbalanced load.

One way to address this issue is insteaad define the remote service as [Headless](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) and then use gRPC's client side loadbalancing constructs.

In this mode, k8s service does not return the single destination for Service but instead all destination addresses back to a lookup request.

Given the set of ip addresses, the grpc client will send each rpc to different pods and evenly distribute load.

## Setup

### Setup Applicaiton

Create the image in  ```cd ~app/http_frontend```

or use the one i've setup `docker.io/salrashid123/http_frontend`

What the image under

[app/http_frontend](app/http_frontend) is an app that shows

- `/backend`:  make 10 gRPC requests over one connection via 'ClusterIP` k8s Service.  The expected response is all from one node

- `/backendlb`:  make 10 gRPC requests over one connection via k8s Headless Service.  The expected response is from different nodes


### Create a cluster
```
gcloud container  clusters create cluster-grpc --zone us-central1-a  --num-nodes 4 --enable-ip-alias
```

### Deploy

```
kubectl apply -f be-deployment.yaml  -f be-srv-lb.yaml  -f be-srv.yaml  -f fe-deployment.yaml  -f fe-srv.yaml
```


Wait ~5mins till the Network Loadblancer IP is assigned

```
$ kubectl get no,po,deployment,svc
NAME                                               STATUS    ROLES     AGE       VERSION
node/gke-cluster-grpc-default-pool-aeb308a0-89dt   Ready     <none>    1h        v1.11.7-gke.12
node/gke-cluster-grpc-default-pool-aeb308a0-hv5f   Ready     <none>    1h        v1.11.7-gke.12
node/gke-cluster-grpc-default-pool-aeb308a0-vsf4   Ready     <none>    1h        v1.11.7-gke.12

NAME                                 READY     STATUS    RESTARTS   AGE
pod/be-deployment-757dd4f4bd-cpgl9   1/1       Running   0          10s
pod/be-deployment-757dd4f4bd-dsxdg   1/1       Running   0          10s
pod/be-deployment-757dd4f4bd-f8v2v   1/1       Running   0          10s
pod/be-deployment-757dd4f4bd-msp5f   1/1       Running   0          10s
pod/be-deployment-757dd4f4bd-qr4sd   1/1       Running   0          10s
pod/fe-deployment-59d7bb7df8-w4fwc   1/1       Running   0          11s

NAME                                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/be-deployment   5         5         5            5           10s
deployment.extensions/fe-deployment   1         1         1            1           11s

NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)          AGE
service/be-srv       ClusterIP      10.23.242.113   <none>           50051/TCP        1h
service/be-srv-lb    ClusterIP      None            <none>           50051/TCP        1h
service/fe-srv       LoadBalancer   10.23.246.138   35.226.254.240   8081:30014/TCP   1h
service/kubernetes   ClusterIP      10.23.240.1     <none>           443/TCP          1h
```

### Connect via k8s Service

```golang
	address := fmt.Sprintf("be-srv.default.svc.cluster.local:50051")
	conn, err := grpc.Dial(address, grpc.WithTransportCredentials(ce))
```

```
$ curl -sk https://35.226.254.240:8081/backend | jq '.'
{
  "frontend": "fe-deployment-59d7bb7df8-w4fwc",
  "backends": [
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-cpgl9",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-cpgl9",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-cpgl9",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-cpgl9",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-cpgl9",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-cpgl9",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-cpgl9",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-cpgl9",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-cpgl9",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-cpgl9"
  ]
}
```

Note: all the responses are from one node

### Connect via k8s Headless Service

```golang
import (
   "google.golang.org/grpc/balancer/roundrobin"
   "google.golang.org/grpc/credentials"
)

  address := fmt.Sprintf("dns:///be-srv-lb.default.svc.cluster.local:50051")
	conn, err := grpc.Dial(address, grpc.WithTransportCredentials(ce), grpc.WithBalancerName(roundrobin.Name))
	c := echo.NewEchoServerClient(conn)
```


```
$ curl -sk https://35.226.254.240:8081/backendlb | jq '.'
{
  "frontend": "fe-deployment-59d7bb7df8-w4fwc",
  "backends": [
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-dsxdg",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-msp5f",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-qr4sd",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-f8v2v",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-cpgl9",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-dsxdg",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-msp5f",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-qr4sd",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-f8v2v",
    "Hello unary RPC msg   from hostname be-deployment-757dd4f4bd-cpgl9"
  ]
}
```

Note: responses are distributed evenly.

## References
 - [https://github.com/jtattermusch/grpc-loadbalancing-kubernetes-examples#example-1-round-robin-loadbalancing-with-grpcs-built-in-loadbalancing-policy](https://github.com/jtattermusch/grpc-loadbalancing-kubernetes-examples#example-1-round-robin-loadbalancing-with-grpcs-built-in-loadbalancing-policy)
 - [https://kca.id.au/post/k8s_service/](https://kca.id.au/post/k8s_service/)
