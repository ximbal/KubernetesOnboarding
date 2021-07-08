TL;DR;

Use an ingress controller to route requests to the right backend based on a label.

PreRequisites

- docker (https://www.docker.com/get-started)
- Kind (https://kind.sigs.k8s.io/docs/user/quick-start/)
- Kubectl (https://kubernetes.io/docs/tasks/tools/)
- Internet Connection
- A code editor 
- A Clone of the Repo.


Steps.


- Clone the repository
- Deploy a Kubernetes cluster
- Deploy an Ingress Controller (Nginx Ingress Controller, but there are others)
- Deploy a fresh copy of rancher
- Import freshly deployed cluster into rancher.
- Deploy Applications
- Expose Applications to other applications (Create Services per application)
- Expose the Applications to the Outside world (Create a LoadBalancer based on the Nginx Controller)

## Deploy a Kubernetes cluster


```
cat <<EOF | kind create --name sandbox-k8s cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF
```

This command will deploy a Kubernetes Cluster in Docker (using ***Kind***, K in D  get it?)

```kubectl cluster-info --context kind-sandbox-k8s```

This command will give you the info about the Kubernetes API

```VERSION=$(curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/stable.txt)```

***Kind*** should switch or create your context, so that you can access the Kubernetes cluster.

```kubectl config get-contexts```

the context should look like this:

```
$ kubectl config get-contexts
CURRENT   NAME                                                                       CLUSTER                                                                    AUTHINFO                                                                   NAMESPACE
          platform-prod_us-central1-c_production-platform-us-central1-c              platform-prod_us-central1-c_production-platform-us-central1-c   gke_pearll-platform-prod_us-central1-c_production-platform-us-central1-c
*         kind-sandbox-k8s                                                           kind-sandbox-k8s                                                           kind-sandbox-k8s
          kubernetes-admin@kubernetes                                                kubernetes                                                                 kubernetes-admin
```
In my case I have Two contexts, or access to Two different Clusters, there are no limit on how many clusters you can admin
from a single computer.

If you need to change context:

```
$ kubectl config use-context kind-sandbox-k8s
Switched to context "kind-sandbox-k8s".
```
## Deploy an Ingress Controller

Now we're going to deploy the Ingress Controller (if you're not sure what an ingress is, read this:https://medium.com/google-cloud/kubernetes-nodeport-vs-loadbalancer-vs-ingress-when-should-i-use-what-922f010849e0)

```kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/${VERSION}/deploy/static/provider/kind/deploy.yaml```

That will create the Ingress Controller, but in the background, to wait for it to come online:

```kubectl wait --namespace ingress-nginx   --for=condition=ready pod   --selector=app.kubernetes.io/component=controller   --timeout=90s```

And that's it, We have a working Cluster with an Ingress Controller.

## Deploy a fresh copy of rancher

At this point we should have something similar than this:

```
$ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED       STATUS       PORTS                                                                 NAMES
5e920daa0560   kindest/node:v1.21.1   "/usr/local/bin/entr…"   4 hours ago   Up 4 hours   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 127.0.0.1:44213->6443/tcp   sandbox-k8s-control-plane
```

A single node kubernetes cluster.

We need to create a place to store and persist the rancher configurations accross restarts, reboots and upgrades.

```
mkdir ~/rancher
```

Then we launch a new Rancher instance,

```
docker run -d --restart=unless-stopped -p 80:80 -p 443:443 --user root \
-v ~/rancher:/var/lib/rancher \
--name rancher rancher/rancher:latest
```

To confirm we should have this:

```
$ docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED         STATUS         PORTS                                                                 NAMES
735237d1c28f   rancher/rancher:latest   "entrypoint.sh"          7 minutes ago   Up 7 minutes   0.0.0.0:8080->80/tcp, 0.0.0.0:8443->443/tcp                           rancher
5e920daa0560   kindest/node:v1.21.1     "/usr/local/bin/entr…"   4 hours ago     Up 4 hours     0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 127.0.0.1:44213->6443/tcp   sandbox-k8s-control-plane
```

Now you have an instance of rancher and a kubernetes cluster available.

## Import freshly deployed cluster into rancher.

Head to the url: http://localhost:8080

Set an ```admin``` password.

Click on ```Add cluster```

Select ```Other Cluster```

Give a name to the cluster in rancher, i.e.: ```sandbox-k8s```

Click ```Create```

Now you need to deploy the Rancher app on the Kubernetes side of things.

Create a Service Account for Rancher to use on Kubernetes.

```kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin --user {your user account}```

Deploy the Rancher solution aka. the Cattle System into Kubernetes.

The following command is an example, the URL will change and you should use the one shown to you at the time 
you're importing a cluster.

```curl --insecure -sfL https://192.168.10.101:8443/v3/import/54974kcsjqhv4xvsgrm6m4dnxg7n25nkl7tdz2qf7kcklj4msh6jnh_c-db7p5.yaml | kubectl apply -f -```

## Deploy Applications

3 Different Applications will be deployed. 3 instances of the same app.

The 3 instances definitions are in the manifest file separated by ```---```

See file `deploy-workloads.yaml`

To deploy the 3 workloads:

```
kubectl apply -f deploy-workloads.yaml
```

## Expose Applications to other applications (Create Services per application)

Now, let's expose the applications to other apps inside of the cluster

See the file `expose-workloads-using-services.yaml`

To expose the 3 workloads to each other.

```
kubectl apply -f expose-workloads-using-services.yaml
```

## Expose the Applications to the Outside world (Create a LoadBalancer based on the Nginx Controller)

The final step is to create an ingress load balancer that is aware on what to do with each request.

See the file `deploy-ingress-loadbalancer.yaml`

```
kubectl apply -f deploy-ingress-loadbalancer.yaml
```

## See it in action

From a command line try the following:

```$ curl localhost/hola```

You probably got:
```
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>```

Nothing happens, it fails, it means the loadbalancer does not know what to do with the request.

Now try:

```
$ curl localhost/bar
bar
$ curl localhost/foo
foo
$ curl localhost/baz
baz
```


