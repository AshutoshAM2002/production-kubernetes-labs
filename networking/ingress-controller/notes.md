<div align="center">

# Kubernetes Networking: Ingress Controller

**A hands-on journey from Pod networking to production-ready Ingress routing**

![Kubernetes](https://img.shields.io/badge/Kubernetes-Networking-326CE5?logo=kubernetes&logoColor=white)
![NGINX](https://img.shields.io/badge/Ingress-NGINX-009639?logo=nginx&logoColor=white)
![Lab](https://img.shields.io/badge/Type-Hands--on_Lab-orange)

</div>

> [!NOTE]
> **Lab date:** 15 July 2026  
> **Goal:** Understand why Ingress exists, how Ingress Controllers work internally, and what happens inside NGINX when an Ingress resource is created.

## Contents

- [The question](#the-question)
- [Why Pod IPs are not enough](#1-why-pod-ips-are-not-enough)
- [Kubernetes Services](#2-kubernetes-services)
- [Why we need Ingress](#3-why-we-need-ingress)
- [How Ingress Controllers work](#4-how-ingress-controllers-work)
- [Hands-on lab](#5-hands-on-lab)
- [Inspecting NGINX configuration](#6-inspecting-nginx-configuration)
- [Key takeaways](#key-takeaways)
- [Interview questions](#interview-questions)

---

## The Question

While learning Kubernetes networking, one question kept coming to my mind.

> **If a `LoadBalancer` Service already exposes my application to the internet, why do we need an Ingress?**

Instead of just reading documentation, I built a small lab to understand how everything works internally.

---

## 1. Why Pod IPs Are Not Enough

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

## 2. Kubernetes Services

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

![Kubernetes Service routing](<screenshots/Screenshot 2026-07-15 030311.png>)

---

### Service Types

While reading the documentation, I realized Kubernetes offers different Service types depending on the use case.

---

#### ClusterIP

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

#### NodePort

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

#### LoadBalancer

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

> [!IMPORTANT]
> **Bare-metal consideration**
>
> `LoadBalancer` Services require cloud-provider integration. Bare-metal and on-premises clusters do not have a cloud API that can provision one automatically. Projects such as **MetalLB** provide this functionality outside cloud environments.

---

## 3. Why We Need Ingress

Imagine a company with:

- frontend
- backend
- users
- payments
- authentication
- notifications
- analytics

If every application uses `type: LoadBalancer`, every Service receives its own public IP address. With hundreds of microservices, this quickly becomes expensive and difficult to manage.

Ingress lets us expose one external load balancer and route its traffic to many Services through a single public endpoint.

---

![Ingress routing architecture](<screenshots/Screenshot 2026-07-15 030329.png>)

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

**One external IP. Multiple applications. Cleaner architecture.**

---

### The Key Insight

> [!IMPORTANT]
> An Ingress resource does **not** route traffic by itself.

Ingress is a Kubernetes object containing routing rules. Something still needs to implement those rules: the **Ingress Controller**.

## 4. How Ingress Controllers Work

The Ingress Controller continuously watches the Kubernetes API.

Whenever I create or modify an Ingress,

the controller notices the change.

Then it configures the underlying load balancer.

Different controllers implement this differently.

---

### Case 1 — NGINX Ingress Controller

The NGINX controller watches the Ingress resource.

Then it automatically generates

```
nginx.conf
```

and reloads NGINX.

So my Kubernetes YAML becomes actual NGINX configuration.

---

![NGINX Ingress Controller configuration](<screenshots/Screenshot 2026-07-15 025909.png>)

![Generated NGINX configuration](<screenshots/Screenshot 2026-07-15 025940.png>)

---

### Case 2 — AWS Load Balancer Controller

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

## 5. Hands-on Lab

The lab includes:

- [x] Frontend Deployment
- [x] Backend Deployment
- [x] Frontend Service
- [x] Backend Service
- [x] NGINX Ingress Controller
- [x] Ingress resource

---

### Deploy Resources

```bash
kubectl apply -f frontend.yml

kubectl apply -f backend.yml

kubectl apply -f ingress.yml
```

---

### Verify Pods

```bash
kubectl get pods
```

![Ingress controller pods](<screenshots/Screenshot 2026-07-15 022343.png>)

---

### Verify Ingress

```bash
kubectl get ingress
```
![Ingress resource output](<screenshots/Screenshot 2026-07-15 024507.png>)

---

### Test Routing

#### Frontend

```bash
curl http://<INGRESS-IP>/frontend \
-H "Host: foo.bar.com"
```

![Frontend routing test](<screenshots/Screenshot 2026-07-15 025233.png>)

---

#### Backend

```bash
curl http://<INGRESS-IP>/backend \
-H "Host: foo.bar.com"
```

![Backend routing test](<screenshots/Screenshot 2026-07-15 025233.png>)

---

## 6. Inspecting NGINX Configuration

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

![Generated NGINX location blocks](<screenshots/Screenshot 2026-07-15 025909.png>)

---

## Key Takeaways

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

## Interview Questions

<details>
<summary><strong>Why shouldn't applications communicate directly using Pod IPs?</strong></summary>

Because Pod IPs are dynamic and change whenever Pods are recreated.

</details>

<details>
<summary><strong>How does a Service know which Pods to send traffic to?</strong></summary>

Using labels and selectors.

</details>

<details>
<summary><strong>Why isn't NodePort commonly used in production?</strong></summary>

It requires exposing ports on every node, uses high-numbered ports, and becomes difficult to manage at scale.

</details>

<details>
<summary><strong>If LoadBalancer already exposes applications, why do we still need Ingress?</strong></summary>

Ingress lets multiple Services share a single external endpoint and provides advanced routing (host-based and path-based), reducing cost and simplifying management.

</details>

<details>
<summary><strong>Can an Ingress resource work without an Ingress Controller?</strong></summary>

No. An Ingress resource only defines routing rules. An Ingress Controller is required to implement those rules.

</details>

<details>
<summary><strong>What changes with the AWS Load Balancer Controller?</strong></summary>

The Kubernetes Ingress resource remains the same, but instead of generating `nginx.conf`, the AWS Load Balancer Controller calls AWS APIs to create and manage ALBs, Listener Rules, Target Groups, Security Groups, and related resources.

</details>

## Final Thoughts

Today's biggest realization was that Kubernetes networking is built around layers of abstraction.

Pods focus on running applications.

Services provide stable networking.

Ingress defines how external traffic should be routed.

And the Ingress Controller turns those routing rules into a working load balancer configuration.

Understanding the responsibility of each component made the entire networking flow much easier to reason about.

---

**Next Lab:** TLS with Ingress using cert-manager.
