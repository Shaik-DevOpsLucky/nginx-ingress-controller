# nginx-ingress-controller
The flow will be:

```text
Internet
   |
   v
AWS Network / classic / Application Load Balancer
   |
   v
NGINX Ingress Controller
   |
   +--------------------+
   |                    |
   v                    v
nginx-app             httpd-app
   |                    |
nginx Service         httpd Service
   |                    |
nginx Pods            httpd Pods
```

For EKS, `ingress-nginx` can be exposed through a Kubernetes `LoadBalancer` Service, with AWS NLB annotations. The current ingress-nginx documentation specifically provides AWS/NLB configuration examples, including IP target mode and health checks. ([Kubernetes][1])

---

# 1. What we'll build

For the video, I recommend this structure:

### Part 1 — Prerequisites

* EKS cluster
* kubectl
* Helm
* AWS CLI
* eksctl
* Verify cluster access

### Part 2 — Install AWS Load Balancer Controller

This is important for a proper EKS setup. AWS recommends the AWS Load Balancer Controller for managing AWS load balancers rather than relying on the legacy AWS cloud provider. ([AWS Documentation][2])

### Part 3 — Install NGINX Ingress Controller

Using Helm.

### Part 4 — Deploy sample applications

We'll deploy:

```text
nginx-app
httpd-app
```

### Part 5 — Create Kubernetes Services

```text
nginx-service
httpd-service
```

Both will remain `ClusterIP`.

### Part 6 — Create Ingress

We'll demonstrate:

```text
http://nginx.demo.local
        |
        v
NGINX Ingress
        |
        v
nginx-service
```

and:

```text
http://httpd.demo.local
        |
        v
NGINX Ingress
        |
        v
httpd-service
```

### Part 7 — Test routing

Using:

```bash
curl
```

and optionally browser/DNS.

---

# 2. Prerequisites

Make sure these are installed on your Mac/Linux machine.

```bash
aws --version
kubectl version --client
helm version
eksctl version
```

Configure AWS:

```bash
aws configure
```

Verify identity:

```bash
aws sts get-caller-identity
```

---

# 3. Connect to your EKS cluster

Replace the values:

```bash
export AWS_REGION=ap-south-1
export CLUSTER_NAME=my-eks-cluster
```

Update kubeconfig:

```bash
aws eks update-kubeconfig \
  --region $AWS_REGION \
  --name $CLUSTER_NAME
```

Verify:

```bash
kubectl get nodes
```

You should see something similar to:

```text
NAME                                           STATUS   ROLES    AGE
ip-10-0-1-100.ap-south-1.compute.internal    Ready    <none>   10m
ip-10-0-2-200.ap-south-1.compute.internal    Ready    <none>   10m
```

Also verify:

```bash
kubectl get pods -A
```

---

# 4. Check whether AWS Load Balancer Controller already exists

Before installing anything, run:

```bash
kubectl get deployment \
  -n kube-system \
  aws-load-balancer-controller
```

If it exists:

```text
NAME                           READY
aws-load-balancer-controller   2/2
```

then **don't reinstall it**.

You can also check:

```bash
helm list -A | grep aws-load-balancer
```

---

# 5. Install AWS Load Balancer Controller

If you don't already have it, AWS currently documents installation through Helm and `eksctl`. ([AWS Documentation][3])

First determine your cluster OIDC configuration:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster $CLUSTER_NAME \
  --region $AWS_REGION \
  --approve
```

Download the IAM policy:

```bash
curl -O \
https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json
```

Create the IAM policy:

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

Get your AWS account ID:

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account \
  --output text)
```

Create the service account:

```bash
eksctl create iamserviceaccount \
  --cluster=$CLUSTER_NAME \
  --region=$AWS_REGION \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::$AWS_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

Add the Helm repository:

```bash
helm repo add eks https://aws.github.io/eks-charts
```

Update:

```bash
helm repo update
```

Install:

```bash
helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --version 1.14.0
```

AWS's current documentation uses controller chart `1.14.0` in its example and notes that upgrades need to be performed manually. ([AWS Documentation][3])

Verify:

```bash
kubectl get deployment \
  -n kube-system \
  aws-load-balancer-controller
```

And:

```bash
kubectl get pods \
  -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller
```

You want:

```text
READY   STATUS
1/1     Running
1/1     Running
```

---

# 6. Install NGINX Ingress Controller

Now install ingress-nginx.

Add repository:

```bash
helm repo add ingress-nginx \
  https://kubernetes.github.io/ingress-nginx
```

Update:

```bash
helm repo update
```

Create namespace:

```bash
kubectl create namespace ingress-nginx
```

---

# 7. Recommended Helm values for EKS

For the YouTube video, I recommend creating a `values.yaml`.

Create:

```bash
mkdir nginx-ingress-nginx
cd nginx-ingress-nginx
```

Create:

```bash
vim ingress-nginx-values.yaml
```

Put:

```yaml
controller:
  replicaCount: 2

  ingressClassResource:
    name: nginx
    enabled: true
    default: true
    controllerValue: k8s.io/ingress-nginx

  ingressClass: nginx

  service:
    type: LoadBalancer

    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: "external"
      service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
      service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"

      service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: "http"
      service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "10254"
      service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/healthz"

  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
```
For Classic LB use below:
```yaml
controller:
  replicaCount: 2

  ingressClassResource:
    name: nginx
    enabled: true
    default: true
    controllerValue: k8s.io/ingress-nginx

  ingressClass: nginx

  service:
    type: LoadBalancer

  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

The NLB target type, health check path/port, and related annotations are documented by ingress-nginx for AWS deployments. ([Kubernetes][1])

---

# 8. Install ingress-nginx

Run:

```bash
helm upgrade --install ingress-nginx \
  ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  -f ingress-nginx-values.yaml
```

Check:

```bash
helm list -n ingress-nginx
```

Then:

```bash
kubectl get pods -n ingress-nginx
```

Expected:

```text
NAME                                        READY   STATUS
ingress-nginx-controller-xxxxx              1/1     Running
ingress-nginx-controller-yyyyy              1/1     Running
```

---

# 9. Check the NGINX Service

This is one of the most important commands for the video:

```bash
kubectl get svc -n ingress-nginx
```

You should eventually see:

```text
NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP
ingress-nginx-controller   LoadBalancer   10.100.x.x     xxxxx.elb.amazonaws.com
```

Get only the hostname:

```bash
kubectl get svc ingress-nginx-controller \
  -n ingress-nginx \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Save it:

```bash
export NGINX_LB=$(kubectl get svc ingress-nginx-controller \
  -n ingress-nginx \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```

Print:

```bash
echo $NGINX_LB
```

Now explain this in your video:

> "The AWS Network Load Balancer is the public entry point. It forwards traffic to the NGINX Ingress Controller pods. NGINX then decides which Kubernetes Service should receive the request."

---

# 9. Create Application Namespace

Create:

```bash
vi namespace.yaml
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
```

Apply:

```bash
kubectl apply -f namespace.yaml
```

Verify:

```bash
kubectl get namespace ingress-nginx
```

---

# 10. Deploy NGINX Application

Create:

```bash
vi nginx-app.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: ingress-nginx
  labels:
    app: nginx-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: ingress-nginx
spec:
  selector:
    app: nginx-app
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

Apply:

```bash
kubectl apply -f nginx-app.yaml
```

Check:

```bash
kubectl get pods -n ingress-nginx
```

```bash
kubectl get svc -n ingress-nginx
```

Expected:

```text
NAME            TYPE        CLUSTER-IP
nginx-service   ClusterIP   10.100.x.x
```

---

# 11. Deploy HTTPD Application

We'll use Apache HTTP Server (`httpd`) as the second application.

Create:

```bash
vi http-app.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: http-app
  namespace: ingress-nginx
  labels:
    app: http-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: http-app
  template:
    metadata:
      labels:
        app: http-app
    spec:
      containers:
        - name: httpd
          image: httpd:2.4-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi

---
apiVersion: v1
kind: Service
metadata:
  name: http-service
  namespace: ingress-nginx
spec:
  selector:
    app: http-app
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
  ```

Apply:

```bash
kubectl apply -f http-app.yaml
```

Verify:

```bash
kubectl get pods -n ingress-nginx
```

You should now have:

```text
nginx-app-xxxxx    Running
nginx-app-xxxxx    Running
http-app-xxxxx     Running
http-app-xxxxx     Running
```

---

Services:

```bash
kubectl get svc -n ingress-nginx
```

Expected:

```text
NAME             TYPE        CLUSTER-IP
nginx-service    ClusterIP   10.100.x.x
httpd-service    ClusterIP   10.100.x.x
```

---

# 11. Test Services internally

Before introducing Ingress, demonstrate that the applications work independently.

Check endpoints:

```bash
kubectl get endpoints -n ingress-nginx
```

You should see pod IPs behind each Service.

For example:

```text
nginx-service    10.0.1.10:80,10.0.2.20:80
httpd-service    10.0.1.30:80,10.0.2.40:80
```

This is a good point in the video to explain:

```text
Pod
 |
 v
Service
 |
 v
Ingress
```

Actually, traffic from outside will be:

```text
Client
  |
  v
AWS NLB
  |
  v
NGINX Ingress Controller
  |
  +------> nginx-service ------> nginx pods
  |
  +------> httpd-service ------> httpd pods
```

---

# 12. Create Ingress resource

Now create:

```bash
vim ingress.yaml
```

Use:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  namespace: ingress-nginx
spec:
  ingressClassName: nginx

  rules:

    - host: nginx.demo.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80

    - host: httpd.demo.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: httpd-service
                port:
                  number: 80
```
or
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  namespace: ingress-nginx
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx

  rules:
    - host: a5e4fe81f898a4720a773c88e2a067e6-969133511.us-east-1.elb.amazonaws.com
      http:
        paths:
          - path: /nginx(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: nginx-service
                port:
                  number: 80

          - path: /httpd(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: http-service
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f ingress.yaml
```

Check:

```bash
kubectl get ingress -n ingress-nginx
```

You should see:

Example:

```text
NAME            CLASS   HOSTS   ADDRESS
demo-ingress    nginx   *       k8s-ingressn-xxxxx.elb.amazonaws.com
```
Detailed:

```bash
kubectl describe ingress demo-ingress \
  -n ingress-nginx
```

---

# 13. Test without DNS

For the YouTube demo, this is actually a great technique.

Get NLB hostname:

```bash
echo $NGINX_LB
```

Then use `curl --resolve`.

For nginx:

```bash
curl --resolve nginx.demo.local:80:$NGINX_LB \
  http://nginx.demo.local/
```

For httpd:

```bash
curl --resolve httpd.demo.local:80:$NGINX_LB \
  http://httpd.demo.local/
```

The important concept:

```text
nginx.demo.local
        |
        | Host header
        v
      NGINX
        |
        v
nginx-service
```

and:

```text
httpd.demo.local
        |
        | Host header
        v
      NGINX
        |
        v
httpd-service
```

---

# 14. If you want browser testing

You can temporarily add entries to your local `/etc/hosts`.

First resolve the NLB:

```bash
dig +short $NGINX_LB
```

However, **don't use the resolved NLB IP as a permanent DNS configuration** because AWS NLB IPs can change.

For a simple lab/video you can use:

```text
<temporary-ip> nginx.demo.local
<temporary-ip> httpd.demo.local
```

Then:

```text
http://nginx.demo.local
```

and:

```text
http://httpd.demo.local
```

For a real environment, create Route 53 records pointing the hostnames to the NLB.

---

# 15. Better YouTube demonstration — path-based routing

After demonstrating host-based routing, I recommend adding a second Ingress example.

For example:

```text
http://demo.example.com/nginx
http://demo.example.com/httpd
```

Create:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  namespace: ingress-nginx
spec:
  ingressClassName: nginx

  rules:
    - host: demo.example.com
      http:
        paths:

          - path: /nginx
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80

          - path: /httpd
            pathType: Prefix
            backend:
              service:
                name: httpd-service
                port:
                  number: 80
```

This lets you explain the two major routing methods:

### Host-based

```text
nginx.example.com  ---> nginx
httpd.example.com  ---> httpd
```

### Path-based

```text
example.com/nginx   ---> nginx
example.com/httpd   ---> httpd
```

---

# 16. Commands for troubleshooting

For your video, keep this section.

### NGINX pods

```bash
kubectl get pods -n ingress-nginx
```

### NGINX logs

```bash
kubectl logs \
  -n ingress-nginx \
  deployment/ingress-nginx-controller
```

Or:

```bash
kubectl logs \
  -n ingress-nginx \
  -l app.kubernetes.io/component=controller
```

### NGINX Service

```bash
kubectl get svc -n ingress-nginx
```

### Ingress

```bash
kubectl get ingress -A
```

### Detailed Ingress

```bash
kubectl describe ingress \
  demo-ingress \
  -n ingress-nginx
```

### Services

```bash
kubectl get svc -n ingress-nginx
```

### Endpoints

```bash
kubectl get endpoints -n ingress-nginx
```

### EndpointSlices

```bash
kubectl get endpointslices -n ingress-nginx
```

### Events

```bash
kubectl get events \
  -n ingress-nginx \
  --sort-by=.lastTimestamp
```

---

# 17. Very important troubleshooting flow

If:

```bash
kubectl get ingress
```

works but application doesn't respond:

Follow this:

```text
Ingress
   |
   v
Service
   |
   v
Endpoints
   |
   v
Pods
```

Run:

```bash
kubectl describe ingress demo-ingress -n ingress-nginx
```

Then:

```bash
kubectl get svc -n ingress-nginx
```

Then:

```bash
kubectl get endpoints -n ingress-nginx
```

Then:

```bash
kubectl get pods -n ingress-nginx -o wide
```

Then test the application directly:

```bash
kubectl port-forward \
  -n ingress-nginx \
  svc/nginx-service \
  8080:80
```

And:

```bash
curl http://localhost:8080
```

If this works, the application and Service are healthy and your investigation moves toward Ingress/NLB.

---

**Ingress is a Kubernetes API resource that defines HTTP/HTTPS routing rules.**

**Ingress Controller is the component that implements those rules.**

In our setup:

```text
                 INTERNET
                    |
                    |
                    v
        +-----------------------+
        | AWS Network           |
        | Load Balancer         |
        +-----------------------+
                    |
                    |
                    v
        +-----------------------+
        | NGINX Ingress         |
        | Controller            |
        +-----------------------+
             /           \
            /             \
           v               v
  +---------------+  +---------------+
  | nginx-service |  | httpd-service |
  | ClusterIP     |  | ClusterIP     |
  +---------------+  +---------------+
          |                  |
          v                  v
    +-----------+      +-----------+
    | nginx pod |      | httpd pod |
    +-----------+      +-----------+
```

And the routing decision happens based on the HTTP request:

```text
Host: nginx.demo.local
             |
             v
       NGINX Ingress
             |
             v
      nginx-service
```

while:

```text
Host: httpd.demo.local
             |
             v
       NGINX Ingress
             |
             v
      httpd-service
```

That makes the video much more valuable than simply showing installation commands.

One important EKS distinction: **AWS Load Balancer Controller and NGINX Ingress Controller are different components**. The AWS controller manages AWS load-balancer resources; NGINX performs the HTTP routing inside the cluster. AWS documents the Load Balancer Controller as the recommended way to manage AWS load balancers on EKS. ([AWS Documentation][2])

If your existing EKS cluster **already has AWS Load Balancer Controller installed**, you can skip that entire installation section and go straight to the NGINX Helm installation.

[1]: https://kubernetes.github.io/ingress-nginx/deploy/?utm_source=chatgpt.com "Installation Guide - Ingress-Nginx Controller"
[2]: https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html?utm_source=chatgpt.com "Route internet traffic with AWS Load Balancer Controller - Amazon EKS"
[3]: https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html?utm_source=chatgpt.com "Install AWS Load Balancer Controller with Helm - Amazon EKS"

# Prepared by:

**Shaik Moulali**

*Lead DevOps Consultant*
