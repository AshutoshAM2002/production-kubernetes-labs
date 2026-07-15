# Kubernetes Networking - Ingress Controller

> **Date:** 15 July 2026
>
> **Lab Goal:** Understand why Ingress exists, how Ingress Controllers work internally, and inspect what actually happens inside NGINX when an Ingress resource is created.

---

# Question I Wanted to Answer

While learning Kubernetes networking, one question kept coming to my mind.

> If a Service of type LoadBalancer already exposes my application to the internet, why do we need an Ingress?

Instead of just reading documentation, I built a small lab to understand how everything works internally.

---

# Step 1 - Why can't Pods communicate using Pod IPs?

Every Pod gets an IP address.

For example,

```

Frontend Pod

10.244.0.5

↓

Backend Pod

10.244.0.8

```

At first glance, this seems perfectly fine.

But Kubernetes Pods are **ephemeral**.

If the backend Pod crashes,

Kubernetes creates another Pod.

Now the IP becomes

```

Old Pod

10.244.0.8

↓

New Pod

10.244.0.15

```

Now the frontend is still trying to communicate with

```
10.244.0.8
```

which no longer exists.

This is exactly the networking problem Kubernetes Services solve.



---



---



# Step 2 - Kubernetes Service

A Service acts like a stable entry point in front of Pods.

Applications never communicate directly with Pods.

Instead,

```

Application

↓

Service

↓

Pods

```

The Service automatically finds Pods using

- Labels
- Selectors

As Pods come and go, Kubernetes updates the Service endpoints automatically.

The Service IP never changes.

That is the beauty of Kubernetes networking.

---

production-kubernetes-labs/networking/ingress-controller/screenshots/Screenshot 2026-07-15 030311.png

---



# Service Types

While reading the documentation, I realized Kubernetes offers different Service types depending on the use case.

---



## ClusterIP

Default Service.

Only accessible from inside the cluster.

```

Frontend Service

↓

Backend Service

```

No external access.

Perfect for microservice communication.

---



## NodePort

Suppose I want to access my application from outside the cluster.

NodePort opens the same port on every worker node.

Example

```
192.168.1.15:30080
```

This works.

But I immediately noticed a drawback.

NodePorts use ports from

```
30000-32767
```

If I have many applications,

I need to manage many ports.

Also,

many production environments don't even expose worker nodes directly to the internet.

So NodePort isn't a great production solution.

---



## LoadBalancer

Cloud providers solve this problem.

When I create

```yaml
type: LoadBalancer
```

Kubernetes asks the cloud provider to provision a Load Balancer.

For example,

AWS creates

- ELB
- NLB
- ALB (depending on controller/configuration)

Azure creates

Azure Load Balancer.

Google Cloud creates

Google Cloud Load Balancer.

Now my application gets

```
35.xx.xx.xx
```

or

```
abc123.elb.amazonaws.com
```

and becomes accessible over the internet.

---



### Interesting Observation

LoadBalancer Services work only when Kubernetes is integrated with a cloud provider.

If I'm running Kubernetes on bare-metal or on-premises,

there is no cloud API available to create a Load Balancer automatically.

Projects like **MetalLB** solve this problem by implementing LoadBalancer functionality for on-prem Kubernetes clusters.

---



# Then Why Do We Need Ingress?

This was the main question of today's lab.

Imagine a company with

- frontend
- backend
- users
- payments
- authentication
- notifications
- analytics

If every application is exposed using

```
type: LoadBalancer
```

then every Service gets its own public IP address.

With hundreds of microservices,

this quickly becomes expensive and difficult to manage.

Instead,

we expose only one external Load Balancer

↓

and place an Ingress behind it.

Now a single public endpoint can route traffic to multiple services.

---

production-kubernetes-labs/networking/ingress-controller/screenshots/Screenshot 2026-07-15 030329.png

---

Ingress supports

### Host-Based Routing

```
api.example.com

shop.example.com

admin.example.com
```

---



### Path-Based Routing

```
example.com/frontend

example.com/backend

example.com/payments
```

One external IP.

Multiple applications.

Much cleaner architecture.

---



# The Biggest Thing I Learned Today

An Ingress resource does **not** route traffic.

This was my biggest misconception.

Ingress is simply a Kubernetes object.

It contains routing rules.

Something still needs to implement those rules.

That component is called the

# Ingress Controller

The Ingress Controller continuously watches the Kubernetes API.

Whenever I create or modify an Ingress,

the controller notices the change.

Then it configures the underlying load balancer.

Different controllers implement this differently.

---



## Case 1 — NGINX Ingress Controller

The NGINX controller watches the Ingress resource.

Then it automatically generates

```
nginx.conf
```

and reloads NGINX.

So my Kubernetes YAML becomes actual NGINX configuration.

---

📷 **Insert Screenshot**

(nginx.conf showing `/frontend` and `/backend` location blocks.)

---



## Case 2 — AWS Load Balancer Controller

AWS doesn't use nginx.conf.

Instead,

the controller calls AWS APIs.

It automatically creates or updates

- Application Load Balancer (ALB)
- Listener Rules
- Target Groups
- Target Registrations
- Security Groups
- ACM Certificates (if configured)
- Resource Tags

The Kubernetes Ingress resource is exactly the same.

Only the implementation changes.

This was another "aha!" moment for me.

---



# My Lab

Today I created

✅ Frontend Deployment

✅ Backend Deployment

✅ Frontend Service

✅ Backend Service

✅ NGINX Ingress Controller

✅ Ingress Resource

---



## Deploy Resources

```bash
kubectl apply -f frontend.yml

kubectl apply -f backend.yml

kubectl apply -f ingress.yml
```

---



## Verify Pods

```bash
kubectl get pods
```

📷 **Insert Screenshot**

(Output of `kubectl get pods`.)

---



## Verify Ingress

```bash
kubectl get ingress
```

📷 **Insert Screenshot**

(`kubectl get ingress` output.)

---



## Test Routing

Frontend

```bash
curl http://<INGRESS-IP>/frontend \
-H "Host: foo.bar.com"
```

📷 **Insert Screenshot**

(Curl output for frontend.)

---

Backend

```bash
curl http://<INGRESS-IP>/backend \
-H "Host: foo.bar.com"
```

📷 **Insert Screenshot**

(Curl output for backend.)

---



# Inspecting nginx.conf

This was probably the coolest part of today's lab.

I executed

```bash
kubectl exec -it \
-n ingress-nginx \
<controller-pod> -- cat /etc/nginx/nginx.conf
```

and searched for my routes.

Sure enough,

the controller had generated location blocks matching my Ingress resource.

Something similar to

```nginx
location /frontend {
    proxy_pass http://frontend-service;
}

location /backend {
    proxy_pass http://backend-service;
}
```

Seeing my Kubernetes YAML transformed into actual NGINX configuration made the entire flow much easier to understand.

---

📷 **Insert Screenshot**

(Generated nginx.conf.)

---



# Key Takeaways

- Pods are temporary and their IP addresses change.
- Services provide a stable endpoint for Pods.
- ClusterIP is used for internal communication.
- NodePort exposes applications through worker nodes.
- LoadBalancer provisions an external load balancer (mainly on cloud platforms).
- Ingress allows multiple applications to share a single external endpoint.
- An Ingress resource does not process traffic by itself.
- An Ingress Controller watches Ingress resources and configures the actual load balancer.
- NGINX Ingress Controller updates `nginx.conf`.
- AWS Load Balancer Controller creates and manages AWS resources like ALBs, Listener Rules, Target Groups, and Security Groups.

---



# Interview Questions



### Why shouldn't applications communicate directly using Pod IPs?

Because Pod IPs are dynamic and change whenever Pods are recreated.

---



### How does a Service know which Pods to send traffic to?

Using labels and selectors.

---



### Why isn't NodePort commonly used in production?

It requires exposing ports on every node, uses high-numbered ports, and becomes difficult to manage at scale.

---



### If LoadBalancer already exposes applications, why do we still need Ingress?

Ingress lets multiple Services share a single external endpoint and provides advanced routing (host-based and path-based), reducing cost and simplifying management.

---



### Can an Ingress resource work without an Ingress Controller?

No. An Ingress resource only defines routing rules. An Ingress Controller is required to implement those rules.

---



### What changes when using the AWS Load Balancer Controller instead of the NGINX Ingress Controller?

The Kubernetes Ingress resource remains the same, but instead of generating `nginx.conf`, the AWS Load Balancer Controller calls AWS APIs to create and manage ALBs, Listener Rules, Target Groups, Security Groups, and related resources.

---



# Final Thoughts

Today's biggest realization was that Kubernetes networking is built around layers of abstraction.

Pods focus on running applications.

Services provide stable networking.

Ingress defines how external traffic should be routed.

And the Ingress Controller turns those routing rules into a working load balancer configuration.

Understanding the responsibility of each component made the entire networking flow much easier to reason about.

---

**Next Lab:** TLS with Ingress using cert-manager.