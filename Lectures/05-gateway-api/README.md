# Kubernetes Gateway API (2025): Hands-On with NGINX Gateway Fabric

## Video reference for Gateway API is the following:

[![Watch the video](https://img.youtube.com/vi/MPUJcPAxtvM/maxresdefault.jpg)](https://www.youtube.com/watch?v=MPUJcPAxtvM)

---
## ⭐ Support the Project  
If this **repository** helps you, give it a ⭐ to show your support and help others discover it! 

---

## Table of Contents

- [Introduction](#introduction)  
- [Pre-Requisites for Gateway API Lectures](#pre-requisites-for-gateway-api-lectures)  
  - [Ingress Series](#ingress-series)  
  - [Custom Resources](#custom-resources)  
- [What We Already Know About Ingress](#what-we-already-know-about-ingress)  
  - [Ingress is Native but Minimal](#1-ingress-is-native-but-minimal)  
  - [Advanced Features Come from the Controller](#2-advanced-features-come-from-the-controller)  
  - [Controller-Specific Annotations Are Common Practice](#3-controller-specific-annotations-are-common-practice)  
- [What’s Missing in Ingress](#whats-missing-in-ingress)  
  - [Cross-Namespace, Multi-Tenancy & RBAC](#1-cross-namespace-multi-tenancy--rbac)  
  - [Protocol & Feature Scope](#2-protocol--feature-scope)  
  - [Controller-Specific Annotations and Lack of Validation](#3-controller-specific-annotations-and-lack-of-validation)  
- [What is Gateway API](#what-is-gateway-api)  
- [Gateway API Resource Model and Role Alignment](#gateway-api-resource-model-and-role-alignment)  
  - [GatewayClass](#1-gatewayclass)  
  - [Gateway](#2-gateway)  
  - [Route Objects](#3-route-objects)  
- [Protocol Flexibility and Advanced Traffic Management](#protocol-flexibility-and-advanced-traffic-management)  
- [Gateway API Flow: Controller, GatewayClass, Gateway, and HTTPRoute](#gateway-api-flow-controller-gatewayclass-gateway-and-httproute)  
- [Gateway API Setup: Cloud-Managed vs In-Cluster Data Planes](#gateway-api-setup-cloud-managed-vs-in-cluster-data-planes)  
- [Demo: Gateway API with NGINX Gateway Fabric](#demo-gateway-api-with-nginx-gateway-fabric)  
  - [Demo Introduction](#demo-introduction)  
  - [What We’re Going to Do in This Demo](#what-were-going-to-do-in-this-demo)  
  - [Pre-requisites](#pre-requisites)  
  - [Step 1: Configure KIND Cluster](#step-1-configure-kind-cluster)  
    - [1.1 Delete any existing cluster](#11-delete-any-existing-cluster)  
    - [1.2 KIND cluster config (00-kind-clusteryaml)](#12-kind-cluster-config-00-kind-clusteryaml)  
    - [1.3 Create the cluster](#13-create-the-cluster)  
  - [Step 2: Install Gateway API CRDs](#step-2-install-gateway-api-crds)  
  - [Step 3: Install the NGINX Gateway Fabric Controller](#step-3-install-the-nginx-gateway-fabric-controller)  
  - [Step 4: Deploy Application Manifests](#step-4-deploy-application-manifests)  
    - [4.1 iPhone App (02-iphoneyaml)](#41-iphone-app-02-iphoneyaml)  
    - [4.2 Android App (03-androidyaml)](#42-android-app-03-androidyaml)  
    - [4.3 Desktop App (04-desktopyaml)](#43-desktop-app-04-desktopyaml)  
  - [Step 5: Create the Gateway](#step-5-create-the-gateway)  
  - [Step 6 – Create HTTPRoutes](#step-6--create-httproutes)  
  - [Step 7 – Check Connectivity](#step-7--check-connectivity)  
- [Conclusion](#conclusion)  
- [References](#references)  

--- 


## **Introduction**

The **Kubernetes Gateway API** is the community-driven evolution of Ingress, created by SIG-Network to separate infrastructure from application routing, support multiple protocols, and bring traffic management into the spec. In this hands-on session, we put it to work with **NGINX Gateway Fabric (NGF)** on a local KIND cluster. We install NGF using Helm, expose it via NodePort, and define **GatewayClass, Gateway, Listeners, and HTTPRoute** resources to route traffic across multiple backends. Along the way, you’ll see how the Gateway API’s role oriented model avoids annotation heavy Ingress patterns, and how NGF acts as the data plane for these resources. Everything is easy to reproduce, giving you a clean starting point for experimenting locally and in the cloud.


---

## Pre-Requisites for Gateway API Lectures

Before starting the Gateway API lecture it is highly recommended to watch the following sessions to build the necessary foundation.

### **Ingress Series**

1. **Day 49:** MASTER Kubernetes Ingress | PART 1 | What, Why & Real-World Flow of Ingress  
   GitHub: [Day 49 Notes](https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2049)  
   YouTube: [Watch Video](https://www.youtube.com/watch?v=yj-ZlKTYDUI&ab_channel=CloudWithVarJosh)  

2. **Day 50:** MASTER Kubernetes Ingress | PART 2 | Path-Based Routing on Amazon EKS with AWS ALB   
   GitHub: [Day 50 Notes](https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2050)  
   YouTube: [Watch Video](https://www.youtube.com/watch?v=4d0dkj6Vc70&ab_channel=CloudWithVarJosh)  

3. **Day 51:** MASTER Kubernetes Ingress | PART 3 | TLS & Subdomain Routing on EKS with AWS ALB  
   GitHub: [Day 51 Notes](https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2051)  
   YouTube: [Watch Video](https://www.youtube.com/watch?v=ABqaWXSIFXc&ab_channel=CloudWithVarJosh)  

> **Short on time?** At the very least, watch **Day 49 – Part 1** for the theory.

---

### **Custom Resources & Operators**

* **Day 39:** Custom Resources (CR) and Custom Resource Definitions (CRD) Explained with Demo  
  GitHub: [Day 39 Notes](https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2039)  
  YouTube: [Watch Video](https://www.youtube.com/watch?v=y4e7nQzu_8E)  

* **Day 40:** Kubernetes Operators Deep Dive with Hands-On Demo  
  GitHub: [Day 40 Notes](https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2040)  
  YouTube: [Watch Video](https://www.youtube.com/watch?v=hxgmG1qYU2M&ab_channel=CloudWithVarJosh)  

---


## What We Already Know About Ingress


### 1. Ingress is Native but Minimal

Ingress is a **built-in Kubernetes resource** designed to handle **HTTP(S) host-based and path-based routing**. Out of the box, that is all it does. The specification does not natively include advanced routing logic, traffic splitting, or multi-protocol support. Its purpose is to provide a **generic interface** that different ingress controllers can implement.

Annotations are often used to extend the Ingress behavior with controller-specific features, but **Kubernetes itself does not validate these annotations**. They are treated as metadata for informational purposes or for consumption by external tools. It is the responsibility of the **Ingress controller** (for example, AWS Load Balancer Controller) to read these annotations and take action based on them. In our demos, AWS LBC interpreted the annotations we defined to configure ALB-specific features such as SSL redirects, and target group health checks.

---


### 2. Advanced Features Come from the Controller

Everything beyond simple host and path routing — such as **SSL redirects**, **custom health checks**, **rate limiting**, **canary or weighted traffic splitting**, and even **TCP/UDP routing** — comes from the **ingress controller**, not the Ingress API itself. These capabilities are exposed by the controller through **annotations** or CRD extensions, which vary by vendor.

---

### 3. Controller-Specific Annotations Are Common Practice

In our Ingress demos, we used the **AWS Load Balancer Controller (an Ingress Controller)**, which provides a large set of annotations to control ALB behavior. For example:

* SSL policy selection
* Target group health check settings
* Redirect and rewrite rules
* Weighted target group forwarding for canary deployments

We leveraged some of these annotations in our demos. The full AWS LBC annotation list is available here:
[https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/ingress/annotations/#ingress-annotations](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/ingress/annotations/#ingress-annotations)

---

## What’s Missing in Ingress


### 1. Cross-Namespace, Multi-Tenancy & RBAC

![Alt text](/Lectures/images/52a.png)

**a) Namespace Scope Limitation**
Ingress is a **namespace-scoped** resource, meaning it can only reference Services **within the same namespace**.
If you have multiple applications (`app1`, `app2`, … `app10`) that you want to expose through the same Ingress, all of them would need to live in the **same namespace**. This conflicts with the common best practice of **namespace-level segregation**, where each application or team operates in its own namespace for isolation, lifecycle independence, and access control.


**b) RBAC and Resource-Level Isolation**
When multiple applications share the same Ingress resource, **all teams must have write access to that shared resource** to manage their own routes. This creates a situation where any team could unintentionally change or break another team’s configuration.
Ingress has **no built-in API-level mechanism** to delegate control over specific hosts or paths within a single Ingress. The only way to achieve isolation is to use separate Ingress resources per team or namespace — but this is implementation-dependent and not enforced by Kubernetes itself.

---

**Note:** Some Ingress controllers, such as the **AWS Load Balancer Controller** with its **IngressGroup** feature, address *both* of these problems by allowing:

* Multiple Ingress resources from **different namespaces** to share a single ALB.
* Teams to manage their own Ingress manifests while still pointing to the same load balancer.

However, this functionality is **controller-specific** and **not part of the native Kubernetes Ingress specification**, meaning it is not portable to controllers that do not support it.

---


### 2. Protocol & Feature Scope

Ingress natively supports only **HTTP(S) host-based and path-based routing**. It does not provide native capabilities for other protocols such as **TCP**, **UDP**, or **gRPC**. In addition, it lacks first-class support for advanced routing and traffic management features such as:

* SSL redirect
* Custom health checks
* Rate limiting
* Weighted/canary traffic splitting
* Header-, query-, or method-based routing

While these capabilities can be achieved with certain Ingress controllers through annotations, this again creates **controller lock-in** and reduces portability. If you move to another controller, you must verify whether it supports the same features — and if it does, you’ll likely need to change the annotation format to match.

> **What is gRPC?**
**gRPC** is a high-performance, open-source protocol for remote procedure calls (RPC).
It enables services to communicate directly, even across different languages and platforms.
It uses HTTP/2 under the hood for faster, more efficient data exchange.

---

### 3. Controller-Specific Annotations and Lack of Validation

Almost every advanced Ingress feature outside the spec is implemented using **annotations**, and these annotations are **controller-specific**. An annotation that configures a weighted forward action for AWS Load Balancer Controller will not work on NGINX or Traefik, because each vendor has its own annotation set.

Kubernetes does not validate these annotations. The API server only checks for YAML syntax correctness, not whether the annotation is valid or meaningful for the controller. This means a typo or unsupported value in an annotation might silently be ignored by the controller, creating a **hidden misconfiguration** risk.

As a result, annotations create a **tight coupling** between your manifests and the specific controller in use. Changing controllers later often requires significant manifest rewrites.

---

## What is Gateway API

Gateway API is a **Kubernetes-standard API** (developed by the Kubernetes SIG-Network community) for managing service networking in a way that is more expressive, extensible, and role-oriented than the older Ingress API.
It is not a built-in resource in Kubernetes by default — instead, it is provided through **Custom Resource Definitions (CRDs)**, which you must install along with a compatible controller to use it.

Gateway API introduces a **new resource model** that:

* Separates **infrastructure** (Gateways) from **application routing** (Routes)
* Enables **multi-tenancy** with API-level isolation
* Supports **multiple protocols** (HTTP, HTTPS, gRPC, TCP, UDP)
* Moves advanced traffic management features into the **core API** (removing the heavy reliance on controller-specific annotations)

---

> **Note on SIG:**
A **Special Interest Group (SIG)** is a working group within the Kubernetes open-source project that focuses on a specific area of the system.
**SIG-Network** is responsible for Kubernetes networking APIs (like Services, Ingress, and Gateway API), ensuring that these APIs evolve in a standardized, portable, and vendor-neutral way across the ecosystem.

> **Note:** Gateway API resources are not available in a fresh Kubernetes cluster by default. You must install the Gateway API CRDs and a compatible controller before you can use them.

---

## Gateway API Resource Model and Role Alignment

![Alt text](/Lectures/images/52c.png)

Gateway API introduces a resource model that separates **infrastructure** from **application routing**, making it easier to manage multi-tenant environments and delegate responsibilities across teams. These resources are intentionally modeled after the roles that typically exist in organizations running Kubernetes.

### Core Resource Types

Gateway API has three stable API kinds:

#### 1. **GatewayClass**

*Blueprint for Gateway implementation and capabilities*

* **Purpose:** Defines a class of Gateways with common configuration and behavior, managed by a specific controller.
* **Analogy:** Similar to a **StorageClass** in Kubernetes — it specifies *what type* of load balancer or data plane you want (e.g., external LB, internal LB, specific vendor).
* **Behavior:** Associated with a specific controller; creates nothing by itself.
* **Parameters:** Can define vendor-specific settings and default behavior for all Gateways referencing it.
* **Ownership:** Usually created and maintained by the **Infrastructure Provider** or platform engineering team.

#### 2. **Gateway**

*Provisions actual traffic-handling infrastructure*

* **Purpose:** Represents an instance of traffic-handling infrastructure, such as a cloud load balancer, with associated listeners (ports, protocols, hostnames).
* **Lifecycle:** **Creating a Gateway typically provisions the actual load balancer**.
* **Configuration:** Configures listeners and references a specific `GatewayClass`.
* **Function:** Connects attached Routes to backend services.
* **Ownership:** Managed by the **Cluster Operator** or platform team.
* **Requirement:** Needs a controller to implement its behavior.

#### 3. **Route Objects**

*Define how traffic is routed to backends*

* **Purpose:** Define routing rules that attach to a Gateway listener and forward traffic to backend endpoints (commonly Kubernetes Services).
* **Protocol Types:** Support **HTTPRoute**, **TCPRoute**, **UDPRoute**, and **GRPCRoute** for multi-protocol flexibility.
* **Routing Logic:** Can match requests based on protocol-specific criteria (e.g., paths, headers, methods).
* **Scope:** Forward traffic to backend services across namespaces (when allowed via `ReferenceGrant`).
* **Ownership:** Managed by **Application Developers** or service owners.

---

## Protocol Flexibility and Advanced Traffic Management

Gateway API goes beyond the HTTP/S-only limitation of Ingress by supporting **multiple protocols** and **spec-defined advanced traffic features**, all within a single, consistent API. This allows both web and non-web traffic to be managed through the same resource model.

* **Multi-Protocol Support** – Gateway API supports:

  * **HTTP and HTTPS** (`HTTPRoute`)
  * **gRPC** (`GRPCRoute`)
  * **TCP** (`TCPRoute`)
  * **UDP** (`UDPRoute`)
    This enables a unified approach to routing for diverse application protocols in one cluster.

* **First-Class Traffic Management Features** – Capabilities that previously required controller-specific annotations are now part of the API specification, making them **portable** across conformant implementations and **validated** by the Kubernetes API server:

  * Weighted/canary traffic splitting (weighted routing)
  * Path and header rewrites
  * Timeouts and retries
  * Request/response filters
  * Header-, query-, and method-based routing

* **Safe Cross-Namespace Routing** – With `ReferenceGrant` and `allowedRoutes`, Gateway API provides explicit, controlled permissions for cross-namespace routing. Teams can securely reference backends in other namespaces without relying on non-standard workarounds.

* **Reduced Annotation Dependency** – Because these features are now part of the spec, reliance on vendor-specific annotations is significantly reduced. While controllers may still offer additional extensions, the portable core remains consistent across vendors.

---

## Gateway API Flow: Controller, GatewayClass, Gateway, and HTTPRoute

This diagram illustrates how the **Gateway API** components interact in a Kubernetes environment using the **NGINX Gateway Fabric** as an example. It follows the sequence from the controller detecting configuration changes to routing traffic to the correct backend.

![Alt text](/Lectures/images/52b.png)

---

### 1. **Controller Watches Gateways and Routes**

The **nginx-gateway-fabric controller** is a pod (part of a deployment) running in the cluster that continuously watches for changes to `Gateway` and `HTTPRoute` resources.
When a new `Gateway` or `HTTPRoute` object is created or modified, the controller reacts and reconciles the configuration to ensure the routing and load balancing behavior matches the desired state.

---

### 2. **GatewayClass Defines LB Type and Controller**

A `GatewayClass` is a cluster-scoped resource that defines:

* **Load Balancer (LB) implementation type** — e.g., NGINX, HAProxy, or a cloud provider LB.
* **Associated controller** — the controller responsible for managing any `Gateway` objects referencing this `GatewayClass`.

This is purely a logical configuration and doesn’t run code itself.
In our example, the `GatewayClass` named `nginx` points to the NGINX Gateway Fabric controller.

---

### 3. **Gateway Implements Listener and Traffic Forwarding**

When the controller detects a `Gateway` resource that uses its `GatewayClass`, it:

* Configures listeners (e.g., HTTP on port 80, HTTPS on port 443).
* Prepares to forward traffic to backend services.

At this stage, depending on the controller implementation, the **data plane** is provisioned:
– **Cloud-managed**: a provider-managed LB is created.
– **In-cluster**: proxy pods (e.g., NGINX, Envoy) are deployed or configured.

---

### 4. **Data Plane**

This is where the **actual traffic forwarding** happens. Proxy pods (in-cluster) or external cloud load balancers (managed by providers) receive requests and forward them to Kubernetes Services and Pods.

#### Control Plane vs Data Plane

* **Control Plane** → Decides & reconciles the desired state. The controller (`nginx-gateway-controller`) watches resources like `GatewayClass`, `Gateway`, and `HTTPRoute` and configures the data plane.
  *Example:* The NGINX Gateway controller deployment updates proxy pods when you apply a new route.

* **Data Plane** → Forwards & processes traffic as per the rules programmed by the control plane.
  *Example:* AWS ALB in cloud setups or NGINX/Envoy proxy pods in on-prem clusters.

---

### 5. **HTTPRoute Defines Routing Rules**

An `HTTPRoute` specifies the routing rules for incoming traffic.
It references the `Gateway` (via `parentRefs`) and defines:

* **Match conditions** (e.g., `PathPrefix: /iphone`).
* **Backend references** (Kubernetes Services and ports).

When created, the controller binds these routes to the corresponding `Gateway` listeners so that traffic flows to the correct backend.

---

> The exact mechanics **depend on the controller implementation** — some lightweight/demo setups may skip a dedicated proxy tier, while production-grade controllers always program a defined data plane.

---

### **One-Line Flows for Quick Recall**

```
Cloud-Managed: Client → Cloud Load Balancer → Kubernetes Service → Backend Pods
In-Cluster: Client → LoadBalancer/NodePort Service → Proxy Pods → Backend Service → Backend Pods
```

---

## **Gateway API Setup: Cloud-Managed vs In-Cluster Data Planes**

**Cloud-Managed (AWS / GCP / Azure Gateway Controllers)**
The controller runs **inside the cluster** but **only programs** an **external, cloud-managed data plane** based on your `Gateway` and `Route` manifests.
It configures listeners, routing rules, TLS settings, and backend service mappings on the cloud load balancer. The cloud load balancer itself handles all client traffic and sends it directly to your Kubernetes Services — the controller is never in the traffic path.

* **AWS** – AWS Gateway API Controller provisions **VPC Lattice services** (or ALB/NLB in other controller variants) as the data plane.
* **GCP** – GKE Gateway Controller provisions **Google Cloud Load Balancers** (external or internal).
* **Azure** – Azure Gateway Controller (for Containers) provisions **Azure Application Gateway** as the data plane.

---

**In-Cluster (NGINX Gateway Fabric, Istio, Envoy Gateway, HAProxy)**
The controller reconciles `GatewayClass`, `Gateway`, and `Route` objects and writes configuration for an **in-cluster proxy** (e.g., NGINX, Envoy) running as pods or DaemonSets.
These proxy pods form the data plane and handle the actual L4/L7 load balancing, TLS termination, and routing. They’re exposed via a Kubernetes `Service` — typically `LoadBalancer` in cloud environments or `NodePort` for on-prem/KIND clusters.
⚠ In some lightweight/demo implementations (like certain KIND setups), the controller may bundle a minimal proxy or bypass a dedicated proxy tier, relying on kube-proxy/L4 forwarding. This is fine for demos, but not production-grade.

---

### **Key Differences**

| Scenario          | Controller Location | Data Plane Location     | Example on AWS / GCP / Azure                  | Other In-Cluster / Cloud-Managed Examples                                                  |
| ----------------- | ------------------- | ----------------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **Cloud-Managed** | In-cluster          | External cloud LB       | AWS VPC Lattice / ALB, GCP GCLB, Azure App GW | AWS Load Balancer Controller (ALB/NLB), GCP Multi-Cluster Gateway, Azure Front Door        |
| **In-Cluster**    | In-cluster          | In-cluster proxy (pods) | —                                             | NGINX Gateway Fabric, Envoy Gateway, Istio Ingress Gateway, HAProxy Ingress, Traefik Proxy |


---

**Mental Model**

* **Controller** = Control plane (watches Gateway API resources, writes config, signals reloads).
* **Data plane** = Where packets actually flow (cloud LB or in-cluster proxy pods).
* In **cloud-managed setups**, the cloud provider’s LB is the data plane.
* In **in-cluster setups**, the proxy pods are the data plane — the controller never sits on the hot path for traffic.
* **Implementation detail matters**: some demos collapse control and data plane for simplicity.

---



# **Demo: Gateway API with NGINX Gateway Fabric**


## **Demo Introduction**

In this demo, we’ll deploy **NGINX Gateway Fabric (NGF)** with the Kubernetes **Gateway API** to route HTTP traffic to multiple backend applications running in a local **KIND** (Kubernetes in Docker) cluster.

We’ll configure a **NodePort** so requests from our host machine can reach the gateway directly — without needing a cloud LoadBalancer.
The NGINX installation is based on the official documentation:
[NGINX Gateway Fabric - Helm Install Guide](https://docs.nginx.com/nginx-gateway-fabric/install/helm/)
I’ve streamlined the steps to make the setup easier to understand and replicate.

---

## **What We’re Going to Do in This Demo**

![Alt text](/Lectures/images/53a.png)

1. Deploy NGF in a KIND cluster using Helm.
2. Expose the NGF Gateway via NodePort for local access.
3. Create a Gateway, Listeners, and HTTPRoutes for different applications.
4. Route traffic to multiple backends (`/iphone`, `/android`, `/`) using Gateway API.

---

## **Flow**

```
User Browser (http://localhost:31000/iphone) → NodePort (31000) → Gateway Listener (80) → HTTPRoute (iphone-routes) → Backend Service (iphone-svc) → Target Pods → HTML Response
```

---


## **Pre-requisites**

Before we start, ensure you have the following installed:

* **KIND** – [Install guide](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
* **Helm** – [Install guide](https://helm.sh/docs/intro/install/)
* **kubectl** (includes Kustomize support, which we’ll use to install CRDs)

---

## **Step 1: Configure KIND Cluster**

### 1.1 Delete any existing cluster

```bash
kind get clusters
kind delete cluster --name=<cluster-name>
```

### 1.2 KIND cluster config (`00-kind-cluster.yaml`)

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.33.1@sha256:050072256b9a903bd914c0b2866828150cb229cea0efe5892e2b644d5dd3b34f
    extraPortMappings:
      - containerPort: 31000  # Port inside the KIND container
        hostPort: 31000       # Port on your local machine
```

> **Why this matters:**
> This mapping allows your local machine (`localhost:31000`) to directly hit the Kubernetes NodePort service running inside the KIND control-plane container.

### 1.3 Create the cluster

```bash
kind create cluster --name=gateway-api --config=00-kind-cluster.yaml
```

---

## **Step 2: Install Gateway API CRDs**

The Gateway API introduces several new resource types that NGF uses. Install them with:

```bash
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.1.0" | kubectl apply -f -
```

**What gets installed (1-liners):**

* **GatewayClass** – Defines a class of gateways (cluster-wide template for data planes)
* **Gateway** – An instance of a GatewayClass (control plane + network listener config)
* **HTTPRoute** – Rules mapping incoming HTTP requests to backend services
* **GRPCRoute** – Same as HTTPRoute but for gRPC traffic
* **ReferenceGrant** – Allows cross-namespace resource referencing

Verify:

```bash
kubectl api-resources | grep gateway
```

> NGF currently supports `HTTPRoute` and `GRPCRoute` from the Gateway API’s standard channel.
`TLSRoute`, `TCPRoute`, and `UDPRoute` are not supported yet, but may be introduced in future releases as the Gateway API evolves and NGF adds more protocol capabilities.


---

## **Step 3: Install the NGINX Gateway Fabric Controller**

We’ll run NGF in its own namespace and expose it via NodePort **31000**.

```bash
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --create-namespace -n ngf-gatewayapi-ns \
  --set nginx.service.type=NodePort \
  --set-json 'nginx.service.nodePorts=[{"port":31000,"listenerPort":80}]'
```

> **Note:**
> This `helm install` command deploys the **NGINX Gateway Fabric** controller into the `ngf-gatewayapi-ns` namespace and configures it to run as a **Gateway API controller**.
>
> * The `--create-namespace` flag ensures the namespace is created if it doesn’t already exist.
> * `nginx.service.type=NodePort` exposes the NGINX service externally via a fixed node port.
> * The `--set-json 'nginx.service.nodePorts=[{"port":31000,"listenerPort":80}]'` option maps:
>
>   * **ListenerPort 80** (from the Gateway’s HTTP listener)
>   * to **NodePort 31000** on the cluster nodes, allowing you to reach the Gateway from outside the cluster by hitting `<NodeIP>:31000`.




Verify:

```bash
kubectl get deploy -n ngf-gatewayapi-ns
```

You should see:

```
deployment.apps/ngf-nginx-gateway-fabric
```

**GatewayClass auto-created:**

```bash
kubectl get gatewayclasses.gateway.networking.k8s.io -o wide
```

```
NAME    CONTROLLER                                   ACCEPTED   AGE   DESCRIPTION
nginx   gateway.nginx.org/nginx-gateway-controller   True       12m
```

> **Note:** GatewayClasses are cluster-scoped. Any namespace can reference them.

---

## **Step 4: Deploy Application Manifests**

We’ll deploy three simple Python-based HTTP servers to represent different device-specific frontends (iPhone, Android, and Desktop).
Each app will live in the **`app1-ns`** namespace and expose a single web page.

---

### **4.1 iPhone App (`02-iphone.yaml`)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iphone-deploy
  namespace: app1-ns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: iphone-page
  template:
    metadata:
      labels:
        app: iphone-page
    spec:
      containers:
      - name: python-http
        image: python:alpine
        command: ["/bin/sh", "-c"]
        args:
          - |
            mkdir -p /iphone && echo '<html>
              <head><title>iPhone Users</title></head>
              <body>
                <h1>iPhone Users</h1>
                <p>Welcome to Cloud With VarJosh</p>
              </body>
            </html>' > /iphone/index.html && cd / && python3 -m http.server 5678
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: iphone-svc
  namespace: app1-ns
spec:
  selector:
    app: iphone-page
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
```

> **Notes:**
> * The Python container serves an HTML page from `/iphone/index.html` on port `5678`.

---

### **4.2 Android App (`03-android.yaml`)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: android-deploy
  namespace: app1-ns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: android-page
  template:
    metadata:
      labels:
        app: android-page
    spec:
      containers:
      - name: python-http
        image: python:alpine
        command: ["/bin/sh", "-c"]
        args:
          - |
            mkdir -p /android && echo '<html>
              <head><title>Android Users</title></head>
              <body>
                <h1>Android Users</h1>
                <p>Welcome to Cloud With VarJosh</p>
              </body>
            </html>' > /android/index.html && cd / && python3 -m http.server 5678
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: android-svc
  namespace: app1-ns
spec:
  selector:
    app: android-page
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
```

---

### **4.3 Desktop App (`04-desktop.yaml`)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: desktop-deploy
  namespace: app1-ns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: desktop-page
  template:
    metadata:
      labels:
        app: desktop-page
    spec:
      containers:
      - name: python-http
        image: python:alpine
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo '<html>
              <head><title>Desktop Users</title></head>
              <body>
                <h1>Desktop Users</h1>
                <p>Welcome to Cloud With VarJosh</p>
              </body>
            </html>' > /index.html && python3 -m http.server 5678
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: desktop-svc
  namespace: app1-ns
spec:
  selector:
    app: desktop-page
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
```

---

**Apply all manifests:**

```bash
kubectl apply -f 02-iphone.yaml
kubectl apply -f 03-android.yaml
kubectl apply -f 04-desktop.yaml
```

**Verify:**

```bash
kubectl get all -n app1-ns
```

---

## **Step 5: Create the Gateway**

`05-gateway.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1  # API group and version for Gateway API
kind: Gateway                             # Resource type: Gateway
metadata:
  name: gateway                           # Name of the Gateway resource
  namespace: ngf-gatewayapi-ns            # Namespace where this Gateway resides
spec:
  gatewayClassName: nginx                 # References the GatewayClass to determine LB type/implementation
  listeners:                              # Listeners define how the Gateway accepts traffic
    - name: http                          # Listener name (unique within this Gateway)
      port: 80                            # Port on which the listener will accept traffic
      protocol: HTTP                      # Protocol for this listener (HTTP, HTTPS, TCP, etc.)
      allowedRoutes:                      # Rules for which Routes can bind to this listener
        namespaces:
          from: All                       # Allows Routes from any namespace to attach
```

### Apply

```bash
kubectl apply -f 05-gateway.yaml
```

### Verify (controller picked it up, data plane exposed)

1. **Gateway object exists & is Ready**

```bash
kubectl get gateway -n ngf-gatewayapi-ns
kubectl describe gateway gateway -n ngf-gatewayapi-ns
```

You should see:

* **Listeners:** `http` on port **80**
* **Addresses:** (may be empty on KIND until the Service is ready)
* **Conditions:** `Programmed=True`, `Ready=True` (names may vary by controller; any “all green” is fine)

2. **GatewayClass is Accepted (controller is responsible)**

```bash
kubectl get gatewayclasses.gateway.networking.k8s.io -o wide
```

Look for the class you referenced (here **nginx**) with **ACCEPTED True** and the controller name
`gateway.nginx.org/nginx-gateway-controller`.

3. **NGF Service created and listening on NodePort 31000**

> This is driven by your Helm install flags from Step 3.

```bash
kubectl get svc -n ngf-gatewayapi-ns
```

Expect a Service for the gateway (name varies by chart, e.g. `gateway-nginx` or similar) showing:

```
TYPE      CLUSTER-IP      EXTERNAL-IP   PORT(S)
NodePort  10.x.x.x        <none>        80:31000/TCP
```

4. **Controller and (if applicable) data-plane pods are healthy**

```bash
kubectl get pods -n ngf-gatewayapi-ns -o wide
```

You should see the **controller** Deployment running. Depending on implementation/version, NGF may also create data-plane pods when the Gateway is reconciled.

5. **Events (optional)**

```bash
kubectl get events -n ngf-gatewayapi-ns --sort-by=.lastTimestamp | tail -n 20
```

Helpful if something didn’t wire up.

---

### What happens next

* NGF **sees** the `Gateway` resource and **programs the data plane** for the HTTP listener on **port 80**.
* Because you installed NGF with **NodePort 31000**, the Service in `ngf-gatewayapi-ns` exposes the listener externally on **`<NodeIP>:31000`** (KIND mapping sends this to `localhost:31000` on your machine).
* **Note:** At this point, without any `HTTPRoute` objects, requests will typically return a default response (e.g., 404). Routing will work after you create the routes in the next step.

**Example Service output**

```
ngf-gatewayapi-ns   service/gateway-nginx   NodePort   10.96.188.186   <none>   80:31000/TCP   42s
```

---

## **Step 6 – Create HTTPRoutes**

`06-routes.yaml`

```yaml
# iPhone route: matches /iphone prefix and sends traffic to iphone-svc
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: iphone-routes
  namespace: app1-ns
spec:
  parentRefs:                 # Which Gateway and listener to attach to
    - name: gateway           # Gateway name
      namespace: ngf-gatewayapi-ns
      sectionName: http       # Listener name from Gateway spec
  rules:                      # Routing rules
    - matches:
        - path:
            type: PathPrefix  # Match any path starting with /iphone
            value: /iphone
      backendRefs:
        - name: iphone-svc    # Service to send traffic to
          port: 80

---

# Android route: matches /android prefix and sends traffic to android-svc
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: android-routes
  namespace: app1-ns
spec:
  parentRefs:
    - name: gateway
      namespace: ngf-gatewayapi-ns
      sectionName: http
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /android
      backendRefs:
        - name: android-svc
          port: 80

---

# Desktop route: matches "/" (default) and sends traffic to desktop-svc
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: desktop-routes
  namespace: app1-ns
spec:
  parentRefs:
    - name: gateway
      namespace: ngf-gatewayapi-ns
      sectionName: http
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /             # Default route; matches anything not caught earlier
      backendRefs:
        - name: desktop-svc
          port: 80
```

---

### **Apply**

```bash
kubectl apply -f 06-routes.yaml
```

---

### **Verify**

Check that the `HTTPRoute` resources exist:

```bash
kubectl get httproutes -n app1-ns
```

Example output:

```
NAME             HOSTNAMES   AGE
android-routes               4m8s
desktop-routes               4m8s
iphone-routes                4m8s
```

Inspect a route in detail (example: `iphone-routes`):

```bash
kubectl describe -n app1-ns httproutes.gateway.networking.k8s.io iphone-routes
```

Look for these key indicators:

* **Route is accepted** → configuration is valid and bound.
* **All references resolved** → Gateway + listeners + backend Services are all present.

---

### **Notes on Route Types**

We used `kind: HTTPRoute` here since it’s the most common for HTTP(S) traffic.
Other options you can use depending on protocol:

* **`GRPCRoute`** — L7 routing for gRPC calls.
* **`TLSRoute`** — passthrough based on SNI (Gateway doesn’t decrypt).
* **`TCPRoute`** — generic L4 TCP routing.
* **`UDPRoute`** — generic L4 UDP routing (e.g., DNS, syslog).

---

## **Step 7 – Check Connectivity**

Now that your `Gateway` and `HTTPRoute` objects are applied, it’s time to validate that traffic is correctly routed through the Gateway and its proxy pods.

---

### **Test Locally via NodePort**

Since we exposed the Gateway on **NodePort `31000`**, send requests directly from your local machine:

```bash
curl http://localhost:31000/android
curl http://localhost:31000/iphone
curl http://localhost:31000/
```

---

### **Expected Output**

* `http://localhost:31000/iphone` → returns **iPhone Users HTML page**
* `http://localhost:31000/android` → returns **Android Users HTML page**
* `http://localhost:31000/` (or `/desktop`) → returns **Desktop Users HTML page**

---

### **Validate Routing via Gateway Proxy Pod**

For deeper debugging, tail logs from the **Gateway proxy pod** to confirm traffic is hitting it:

```bash
kubectl logs -f -n ngf-gatewayapi-ns gateway-nginx-6bb659b9cf-hclbp
```

You should see incoming requests being processed by the proxy.

---

## **Conclusion**
We took the **Gateway API** from concept to practice: installed **NGINX Gateway Fabric (NGF)** via Helm on KIND, exposed it with **NodePort**, and proved routing to three backends using **GatewayClass → Gateway → Listeners → HTTPRoute**. You saw how the role-oriented model replaces annotation-heavy Ingress flows and cleanly separates platform and app concerns.

**Key takeaways**

* **Separation of duties:** Platform owns *GatewayClass/Gateway*; app teams own *HTTPRoute*.
* **Attachment model:** *Listeners* + *allowedRoutes* safely govern cross-namespace routing.
* **Declarative routing:** Host/path matches, header matches, and weights—no brittle annotations.
* **Portable by design:** Specs first; controller (NGF) implements the data plane cleanly.
* **TLS ready:** Terminate at Gateway with cert refs; SNI maps cleanly to Listeners.
* **Operational clarity:** Status conditions (*Accepted/Programmed*), events, and `kubectl` give fast feedback.

**Next steps**

* Add **TLS** (Gateway `certificateRefs`) and verify SNI/host routing.
* Try **filters**: header add/remove, URL rewrite, request/response mods.
* Do **traffic splits/canaries** with multiple `backendRefs` and weights.
* Enforce **policy** via the attachment pattern (timeouts, retries, timeouts if supported).
* Lift the same manifests to **EKS/GKE/AKS**; swap KIND NodePort for LoadBalancer.
* Explore **gRPC** routes now; track **TCP/UDP** maturity per controller.

**Common gotchas to watch**

* *No routes matching:* Listener `hostname/port` doesn’t align with HTTPRoute `hostnames`.
* *Cross-namespace misses:* `allowedRoutes`/`namespaces.from` not set as expected.
* *Service reachability:* NodePort/LoadBalancer paths not open → health checks fail.

This demo now gives you a repeatable baseline to evolve from lab to production—add TLS, policies, and rollout patterns, then standardize Gateway API as your cluster entry point.

---

## **References**

**Official Documentation**

* Gateway API: [https://gateway-api.sigs.k8s.io](https://gateway-api.sigs.k8s.io)
* Ingress API: [https://kubernetes.io/docs/concepts/services-networking/ingress](https://kubernetes.io/docs/concepts/services-networking/ingress)

**Implementations & Controller Docs**

* NGINX Gateway Fabric Overview: [https://docs.nginx.com/nginx-gateway-fabric](https://docs.nginx.com/nginx-gateway-fabric)
* AWS Load Balancer Controller — Ingress Annotations: [https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/ingress/annotations](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/ingress/annotations)

**Community & Project Pages**

* Kubernetes SIG-Network: [https://github.com/kubernetes/community/tree/master/sig-network](https://github.com/kubernetes/community/tree/master/sig-network)
* CNCF Gateway API Project: [https://www.cncf.io/projects/gateway-api](https://www.cncf.io/projects/gateway-api)

---
