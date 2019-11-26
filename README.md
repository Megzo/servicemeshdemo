# Service Mesh tutorial

This tutorial runs you thorough some basic Kubernetes concepts, 
and after showcases the usage of different service mesh implementations over it.
We going to show you the basics of Linkerd and Istio.

Disclaimer: this tutorial was tested on a Ubuntu 18.04 environment using K3s version
1.0.0 which build upon Kubernetes version 1.16.3.
Other Linux platforms / versions should work very similarly with slight modification of the commands.

### 1. Preparation

For this tutorial we will use [K3s](https://k3s.io/) which is a lightweight [certified Kubernetes distribution](https://www.cncf.io/certification/software-conformance/)
created by [Rancher](https://rancher.com/)  mainly for testing and IoT use-cases. For the sake of simplicity, we will use K3s with Docker rather than it's 
default mode, which uses Containerd.

So first, install Docker:
```
apt update
apt install -y docker.io
systemctl enable docker.service   # so after a restart Docker will start also
```

Then, install K3s with Docker enabled:
```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--docker" sh -
```

That's it. You have now a running Kubernetes cluster, which has only one node.
You can check whether it is working:
```
kubectl version
kubectl get pods -A
```

Pro tip: enable `kubectl` bash completion
```
source <(kubectl completion bash)
```

### 2. Kubernetes (very) basics

Usually, we interact with Kubernetes using `YAML` files. Actually, it's just a way to interact with the API server of Kubernetes.
You create an object in the API server, and Kubernetes will make sure that the object will run (or in other word, the desired state is fulfilled). 

In Kubernetes you can run containers in so-called Pods. A Pod is a collection of one or more containers.
Usually, we run only one container in a Pod, but you can run more. This will be a very important pattern for the service mesh.

Run your first Pod with an Nginx container:
```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
EOF
```

Check the status of the Pod:
```
kubectl get pods
```

You can also see more details (e.g. the internal IP address of Pod, the node that the Pod is located, etc.):
```
kubectl get pods -o wide
```

Now check the Nginx server with `curl`:
```
curl `kubectl get pod nginx -o jsonpath='{.status.podIP}'`
```

You can delete the Pod:
```
kubectl delete pod nginx
kubectl get pods
```

The second most important object is the Deployment. A Deployment represents a set of multiple, 
identical Pods with no unique identities. A Deployment runs multiple replicas of your application 
and automatically replaces any instances that fail or become unresponsive. In this way, Deployments 
help ensure that one or more instances of your application are available to serve user requests. 
Deployments are managed by the Kubernetes Deployment controller.
Run you first deployment with 3 Nginx images using version 1.16.1:
```
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.1
EOF
```

Check the created Deployment:
```
kubectl get deployments -o wide
```

And check the Pods which were created by the Deployment:
```
kubectl get pods -o wide
``` 

Actually, to be very specific, the Deployment does not create the Pods.
The Deployment creates a so-called ReplicaSet, which then will create the Pods.
Although this might sound complicated, this is the very basic of Kubernetes: you create
a primary resource in the API, and a controller can react to it by creating secondary resources.
This will be also a very basic pattern of the service mesh.
```
kubectl get replicasets -o wide
```

Now a very cool thing you can do with a Deployment is a `RollingUpdate`, when you update a version
of a container, and Kubernetes will replace the effected Pods.
You can update the Nginx image version to 1.17.1 by the following command and see the result 
(notice that the new Pods will get new IP addresses):
```
kubectl patch deployment nginx  -p '{"spec":{"template": {"spec": {"containers":[{"name":"nginx","image":"nginx:1.17.1"}]}}}}'
watch kubectl get pods -o wide
```

Since this update was all managed by the Deployment controller of Kubernetes, you can actually see the history, and if you want
roll back to the previous version (again, the new Pods will get new IPs even though you "rolled back"):
```
kubectl rollout history deployment nginx
kubectl rollout undo deployment nginx
watch kubectl get pods -o wide
```

The next very important objects are Services. In the previous example you could see that the IP address of Pods are constantly changing.
Obviously, you don't want to keep asking Kubernetes for the IP of the Pods, and also you may not want to choose from the available endpoints.
This is where Services come in: you will get one constant(ish) IP that will always point to certain type of Pods (which has the same label sets, so for e.g. every Pod controlled by a Deployment) and also, load balance between them. Create a Service for your `nginx` Deployment:
```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  type: ClusterIP
  ports:
  - protocol: TCP
    port: 80
EOF
kubectl get services -o wide
```

Now use `curl` to access the Service. Also, believe me that if you run this multiple times, your queries will be randomly
load balanced to one of the three Nginx Pods:
```
curl `kubectl get service nginx -o jsonpath='{.spec.clusterIP}'`
```

The other interesting thing with Services, is that Kubernetes will generate an internal DNS entry, that your Pods can use
to access each other. That's the Kubernetes' way of service discovery. Start an `alpine` Pod, and see this for yourself:
```
kubectl run -i --tty alpine --generator=run-pod/v1 --image=alpine -- sh
nslookup nginx
wget -O - nginx
exit
kubectl delete pod alpine
```

This is really cool, but now you only got a DNS entry and an IP address that can be access from within your cluster.
Some of the Pods (typically front-end servers/proxies, API gateways) should be accessible from outside the cluster.
To achieve this you mainly have three ways in Kubernetes: i) use a Service with type NodePort, ii) use a Service with type LoadBalancer or iii) use the Ingress object of Kubernetes.

NodePort will open you a port between 30000-32767 on every(!) host in your cluster to forward that to a given port of your service. Obviously, this is not the recommended way to externally access a Service from the outside, but it is quick so good for testing purposes. You can actually use the same port number to access the Service from you browser (make sure you firewall allows this port range): 
```
kubectl patch service nginx  -p '{"spec":{"type": "NodePort"}}'
kubectl get services -o wide
curl 127.0.0.1:`kubectl get service nginx -o jsonpath='{.spec.ports[].nodePort}'`
```

The second way of external access is the `LoadBalancer` type Service. Although, this name might be a bit confusing, since
the default `ClusterIP` type Service also does load balancing between the Pods, but on the other hand it tells you that usually
to make this work you need an actual (hardware) load balancer. Kubernetes has to communicate with this load balancer to request a
public IP that will then point to your internal Service. This does not come automatically, and in a basic cluster it won't work without an extra component. However, usually in managed clusters (e.g. GKE, AKS, EKS) it is configured for you. Luckily, also in K3s you have a default way that let's you deal with this, and will open the same ports that are defined in your Service on every node similarly to the `NodePort` type, but using the exact same port numbers (not between the 30000-32767). This will actually make your ports collide very frequently since for e.g. every web server tends to same 80/443 port numbers.

Actually, changing the `nignx` service to `LoadBalancer` type won't work, since there is already web server listening on the port 80 and 443, that is the so-called Ingress Controller that is the 3rd way to make your services exposed to the Internet as long as you use HTTP/HTTPS.
K3s by default uses the `Traefik` proxy as an Ingress Controller. You can see the details with the following two commands:

```
kubectl describe deployment -n kube-system traefik 
kubectl get services -n kube-system -o wide
```

Note that the `traefik` Service has type `LoadBalancer` and has the same External IP as the primary IP of you machine.
You can also open this in your browser and see that its working (although you will get a `404 page not found` error since you have no Ingress configured yet).

```
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: nginx
          servicePort: 80
EOF
kubectl get ingresses.networking.k8s.io
```

You can refresh the page in you browser and see that the default Nginx page loaded, which means that your query did reach one
of the Nginx Pods.

Finally, we can put all these together. The great thing in Kubernetes is that you can define multiple API objects in one file.
This means that you can run a more complex application with just one command. A very good demo is [this](https://microservices-demo.github.io/) 
by Weaveworks. You can use their complete [yaml file](https://raw.githubusercontent.com/microservices-demo/microservices-demo/master/deploy/kubernetes/complete-demo.yaml) to run the whole web shop in a few commands. You just have to create a [Namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) first, apply the yaml manifest and create an ingress to access it (and delete the old ingress we just created):

```
kubectl create namespace sock-shop
kubectl apply -f https://raw.githubusercontent.com/rexx4314/microservices-demo/patch-1/deploy/kubernetes/complete-demo.yaml
kubectl get pods -n sock-shop -o wide
kubectl get services -n sock-shop -o wide
kubectl delete ingresses.networking.k8s.io test-ingress # delete the old ingress
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: sock-shop
  namespace: sock-shop
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: front-end
          servicePort: 80
EOF
kubectl get ingresses.networking.k8s.io -n sock-shop
```

Wait until all the Pods in the `sock-shop` namespace is running. And hopefully, after refreshing you browser you should see the sock shop application.

### 3. Using Linkerd

Take a look again at the application you just deployed:
```
kubectl get pods -n sock-shop -o wide
```

So now you have 13 Pods running and have no clue who talks to who, and where is the problem in case you have any.
That's what service mesh for, to see through this chaos. Originally, [Linkerd](https://linkerd.io/) was developed by a
group who was working on a way more complex microservice environment at Twitter. Read this [blog post](https://thenewstack.io/history-service-mesh/) for more info.
The current 2.0 version of Linkerd was specifically made for Kubernetes as a lightweight but powerful Service Mesh implementation to help you see through this microservice chaos. So let's see who this works. Mostly, I will follow [this](https://linkerd.io/2/getting-started/) guide, so take a look on it, since they update this very frequently as new versions of Linkerd are coming out.

First, intall the `linkerd` CLI tool:
```
curl -sL https://run.linkerd.io/install | sh
export PATH=$PATH:`pwd`/.linkerd2/bin
cp /etc/rancher/k3s/k3s.yaml .kube/config # sudo might be needed linkerd version
```

Check that everything is ready in your Kubernetes cluster to install Linkerd:
```
linkerd check --pre
```

If everything went OK, you can add Linkerd to your culster, and check the status:
```
linkerd install | kubectl apply -f -
linkerd check
kubectl get pods -n linkerd
```

<!--
Linkerd has a built-in dashbord that we going to expose via an Ingress:
This does not work due to some URL rewrite issues... :(
```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: web-ingress-auth
  namespace: linkerd
data:
  auth: YWRtaW46JGFwcjEkbjdDdTZnSGwkRTQ3b2dmN0NPOE5SWWpFakJPa1dNLgoK
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: linkerd-dashbord
  namespace: linkerd
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/rewrite-target: /
    ingress.kubernetes.io/custom-request-headers: l5d-dst-override:linkerd-web.linkerd.svc.cluster.local:8084
    traefik.ingress.kubernetes.io/auth-type: basic
    traefik.ingress.kubernetes.io/auth-secret: web-ingress-auth
    traefik.ingress.kubernetes.io/redirect-regex: ^/(.*)
    traefik.ingress.kubernetes.io/redirect-replacement: /linkerd/$1
spec:
  rules:
  - http:
      paths:
      - path: /linkerd
        backend:
          serviceName: linkerd-web
          servicePort: 8084
EOF
kubectl get ingresses.networking.k8s.io -n linkerd
```
-->

Linkerd has a built-in dashbord that we going to expose by changing its service from
`ClusterIP` to `LoadBalancer`:
```
kubectl patch service -n linkerd linkerd-web -p '{"spec":{"type": "LoadBalancer"}}'
kubectl get services -n linkerd -o wide
```

You should now able to access the Linkerd web dashboard via your browser on port 8084 using HTTP (not HTTPS).

Now it's time to inject the sidecar proxies next to your application Pod to implement a Servce Mesh.
Linkerd can do such thing by the `linkerd inject` command. (You can actually also tag a namespace to
set up auto-inject, but that's for another time.)

```
kubectl get deployments -n sock-shop -o yaml | linkerd inject - | kubectl apply -f -
watch kubectl get pods -n sock-shop
```

If everything is ready you should see the same webshop in the browser. From the users' viewpoint nothing has changed. But from the operators' perspective, we now have a much more deeper visibility into our environment. Try generate some load and see the
numbers on the dashboard:
```
while true; do sleep 0.5; curl -s 127.0.0.1 > /dev/null; date;done
```

So that's Linkerd for you. Pretty easy, very lightweight but quite powerful.
There are many more advanced features, which we won't cover here. Please take a look [here](https://linkerd.io/2/features/) to see all the features. And keep in mind, Linkerd is in very active development so features are appearing in a very fast pace.

So for now, let's clean up before we move on to Istio:
```
kubectl get deployments -n sock-shop -o yaml | linkerd uninject - | kubectl apply -f -
linkerd install --ignore-cluster | kubectl delete -f -
kubectl get pods -A
```

### 3. Using Istio

[Istio](https://istio.io/) is another popular Service Mesh implementation. Compared to
Linkerd, it is much more feature rich (you can do crazy things with it), but as a result 
it is much more heavy, meaning that it's control plane requires much more resource to run, and
operating it might need a dedicated team (how has much to learn). But some people say, it's definitely
worth the effort. 

The other major difference is that while Linkerd uses it's own proxy implementation (written in Rust),
Istio uses a proxy called [Envoy](https://www.envoyproxy.io/). It's actually Envoy who can implement 
these rich features. Istio is basically a control plane over Envoy. 

So let's install Istio. First, download the latest release of `istioctl` (as of now 1.4.0):
```
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.4.0
export PATH=$PWD/bin:$PATH
```

Before we actually install Istio to our cluster, we going to remove the built in Ingress 
controller of K3s, since Istio has it's own (also based on Envoy):
```
kubectl patch service -n kube-system traefik -p '{"spec":{"type": "NodePort"}}'
kubectl delete ingresses.networking.k8s.io -n sock-shop sock-shop
kubectl delete -n kube-system `kubectl get pods -n kube-system -o name | grep svclb-traefik`
```

Now we are ready to install Istio. Type the following and wait until every Pod is in running state:
```
istioctl manifest apply --set profile=demo
kubectl get pods -n istio-system 
```

Now we can inject side car proxies next to our existing application, similarly to what we
did in case in of Linkerd (but remember, you can also enable auto injection in a namespace):
```
kubectl get deployments -n sock-shop -o yaml | istioctl kube-inject -f - | kubectl apply -f -
watch kubectl get pods -n sock-shop
```

Now here comes the tricky part. Most of the shenanigans you can do with Istio is done via
so-called Custom Resource Definitions (CRDs). These are similar object to what we discussed 
in the beginning (so Pods, Deployments, Services), but they are not part of a basic Kubernetes
cluster. We can only create them because we just installed Istio and they are specific to Istio.
When we talk about the control plane of Istio, we mostly mean custom controllers which can watch
these custom API endpoints and do funky things when you add something (mainly they program the 
Envoy sidecars).

Let's check these Custom Resource Definitions by the following commands (ignore the *.cattle.io
one, those are very similar resources, but added by the K3s itself).

```
kubectl get crds
```

Now, let's expose the `sock-shop` front-end, but not by a regular Ingress object, but
an Istio specific one:
```
cat << EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: sock-shop
  namespace: sock-shop
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
EOF
kubectl get gateways.networking.istio.io -n sock-shop
cat << EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: sock-shop
  namespace: sock-shop
spec:
  hosts:
  - "*"
  gateways:
  - sock-shop
  http:
  - route:
    - destination:
        port:
          number: 80
        host: front-end
EOF
kubectl get virtualservices.networking.istio.io -n sock-shop
```

If everything went well, you should now be able to open the website from you browser.
But again, from this perspective the page looks the same as before.
So, now lets see what Istio provides for visibility.
First, let's expose Kiali which is a dashboard for Istio (make sure port 20001 isn't blocked
by a firewall).
```
kubectl patch service -n istio-system kiali -p '{"spec":{"type": "LoadBalancer"}}'
kubectl get services -n istio-system -o wide
```

Now open Kiali at port 20001. Use admin/admin as credentials.

There are other interesting obeservability tools included in Istion. For e.g. Prometheus
for metrics collection with a Grafana dashboard, and Jaeger for tracing. You can take a
look at these dashboards as well by exposing them:
``` 
kubectl patch service -n istio-system grafana -p '{"spec":{"type": "LoadBalancer"}}'
kubectl patch service -n istio-system jaeger-query -p '{"spec":{"type": "LoadBalancer"}}'
kubectl get services -n istio-system -o wide
```


You can clean up Istio by the following command:
```
istioctl manifest generate --set profile=demo | kubectl delete -f -
```
