# Kubernetes Hello World with Ingress

## Awesome tools

* <https://github.com/kubernetes-helm/monocular>
* Helm (package and version)
* [minikube](https://kubernetes.io/docs/getting-started-guides/minikube) - quick local setup
* kubectl - cluster commandline
* kops - sets up the AWS infrastructure

## Local Minikube Setup

Install minikube for local dev: ([quickstart](https://kubernetes.io/docs/getting-started-guides/minikube/))

```bash
minikube start
kubectl config use-context minikube # point kubectl at minikube
```

Open the dashboard:

```
minikube dashboard
```

Deploy something using a `.yaml` file or `helm`

```bash
$ kubectl create -f deployment.yaml
```

Ingress - expose the service to the outside

```
kubectl expose deployment my-ingress-service --type=LoadBalancer
```

Open dashboard in a browser

```
minikube service my-ingress-service
```

## Real clusters

For AWS deployment you can setup the cluster either using the [kops](https://github.com/kubernetes/kops) production cluster builder or, of course, homebrew with terraform. Given the learning curve for kubernetes anyway I'd suggest to use kops for your first cluster. You also get some nice out of the box features with kops, like rolling cluster uprades.

### Cost savings

If you want a cheap dev cluster you can get the cost of a basic kubernetes cluster in AWS down to about 120 bucks. This article covers a **70% AWS bill reduction**: <https://carlosbecker.com/posts/k8s-sandbox-costs> by tweaking things. [This](https://medium.com/cognitoiq/how-cognitoiq-are-using-application-load-balancers-to-cut-elastic-load-balancing-cost-by-90-78d4e980624b) article covers a **90% cost reduction** for the load balancers.

### Warning about absolute paths

*WARNING: `kubectl` doesn't do ~ expansion. Avoid using ~ in any paths you give for kubectl or credentials files. It will fuck up with stupid error messages otherwise. i.e.:*

```bash
$ kubectl --kubeconfig=~/k8/$CLUSTER.kubeconfig get po # BAD
```

> The connection to the server localhost:8080 was refused - did you specify the right host or port?

Ditto if you tried to use absolute paths to credentials and certificate files:

```bash
$ kubectl --kubeconfig=$CLUSTER.kubeconfig get po
```

> Error in configuration:
> unable to read client-cert /Users/my-username/\~/k8/my-cluster.com/my-username.crt for my-username due to open /Users/my-username/\~/k8/my-cluster.com/my-username.crt: no such file or directory

### Connect to a kubernetes cluster

View the web ui at `https://cluster.foo.com/ui`

Let's point `kubectl` at an existing cluster. Note the (somewhat ugly) use of KUBECONFIG and $CLUSTER.kubeconfig - this is for multi-cluster environments where you need to switch between clusters. There are other ways to approach this: you can instead just have all your various clusters in the default kubeconfig file and use `kubectl config use-context` to switch between them. That will be system wide rather than per terminal window though. You can also look at `KUBECONFIG=cluster-merge` docs. 

Unpack credentials

```bash
$ unzip my-username.zip
Archive:  my-username.zip
  inflating: my-username.crt
  inflating: my-username.csr
  inflating: my-username.key
```
  
Build a definition in $CLUSTER.kubeconfig. The explicit use of KUBECONFIG is for multi-cluster environments.

```bash
export CLUSTER=my-cluster.com # domain name for the cluster
export USERNAM=my-username # your k8 username
export KOPS_STATE_STORE=s3://my-bucket-name # whatever you used when setting up with kops

# Get kops to create a .kubeconfig file for us
KUBECONFIG=$CLUSTER.kubeconfig kops export kubecfg $CLUSTER
```

Setup the credentials to use with that cluster and switch to having that cluster selected:

```bash
kubectl --kubeconfig=$CLUSTER.kubeconfig config set-credentials $USERNAM --client-key=$USERNAM.key --client-certificate=$USERNAM.crt
# Make it so we don't need to specify --user all the time. Can also do this for --namespace
# Using the full cluster name for the name of the context is slightly overkill, we could use any string
kubectl --kubeconfig=$CLUSTER.kubeconfig config set-context $CLUSTER --user $USERNAM
kubectl --kubeconfig=$CLUSTER.kubeconfig config use-context $CLUSTER
```

**Note whenever you use kubectl you MUST pass --kubeconfig**

```
kubectl --kubeconfig=$CLUSTER.kubeconfig ...

```

## Inspecting a Cluster

Inspect a cluster: `bash kubectl get all`

See where kubectl is currently pointing: `kubectl config view`

View deployment history:

```bash
$ kubectl --kubeconfig=$CLUSTER.kubeconfig rollout history deployment gateway
deployments "gateway"
REVISION	CHANGE-CAUSE
7	kubectl apply --kubeconfig=/root/.kube/office.kubeconfig --filename=/root/.kube/app-configs/gateway-v53.yaml --record=true
8	kubectl apply --kubeconfig=/root/.kube/office.kubeconfig --filename=/root/.kube/app-configs/gateway-v57.yaml --record=true
9	kubectl apply --kubeconfig=/root/.kube/office.kubeconfig --filename=/root/.kube/app-configs/gateway-v58.yaml --record=true
```

## Ingress, Deployments and Services

Ingress (ingestion) with kubernetes can be confusing at first. There are multiple ways to do things so often each example you look at has a different approach. This is a good article: <https://medium.com/@cashisclay/kubernetes-ingress-82aa960f658e> that covers ingress and how to find your endpoints.

If you use "Kind: Ingress" you can then go in and point Route53 at the created E/ALB after.

> Note: kubernetes may not work so well with docker registries that change their IP address. ECRs seem to be fine.

The Ingress:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: myapp-ingress
spec:
  rules:
  - host: myapp.k8s.dev.my-domain.com
    http:
      paths:
      - path: /
        backend:
         serviceName: myapp-http-ingestion
         servicePort: 80
```

The service. This acts like a little load balancer and routes to whatever Deployments exist that match it's `spec`. In this case that's any deployments with the label `app myapp-ingestion`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-ingestion
spec:
  selector:
    app: myapp-ingestion
  sessionAffinity: None
  ports:
  - port: 80
    targetPort: 3000
```

The Deployment (your code)

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: myapp-ingestion
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: myapp-ingestion
    spec:
      containers:
      - name: myapp-http-ingestion
        image: docker.my-domain.com/myapp-http-ingestion:20170814-114105-a7df29e.12
        imagePullPolicy: Always
        ports:
        - containerPort: 3000
```

Next you need to go into the AWS console and use Route53 to point the domain at the new load balancer.

## Finding your ingress (ingestion) points

List all the ingress points in short form:

```
kubectl --kubeconfig=$CLUSTER.kubeconfig get ingress
```

Examine a particular ingress in detail:

```
kubectl --kubeconfig=$CLUSTER.kubeconfig describe ingress myapp-ingress
```

`get` and `describe` are interchangeable, one just shows more detail.