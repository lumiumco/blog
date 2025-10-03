---
date: '2025-09-22T00:00:00+00:00'
draft: false
title: 'Rate Limiting in Practice: A Simple Scenario'
---
## Why Rate Limiting Matters

Modern APIs often serve thousands or even millions of requests every minute.
Without limits, this can quickly lead to problems:
- **Abuse**: a single client may flood the API with many requests.
- **Noisy neighbor**: one misbehaving user can negatively affect everyone else.
- **Unpredictable load**: sudden traffic spikes may overwhelm backend services.

Rate limiting is the standard solution to these issues. By restricting how many requests a client can make in a given time window, we protect the reliability and fairness of our system.

In this series, we will explore how to implement rate limiting, moving step by step from simple scenarios to more advanced use cases.

---
## Theory

At a high level, rate limiting works by introducing a `rate limit service` that evaluates each request against defined policies. 

1. Every request from client first passes through the `ingress gateway`.
2. The `ingress gateway` extracts all necessary parameters, such as the request path, and forwards them to the `rate limit service`.
3. The `rate limit service` uses the `service` configuration to determine the maximum number of requests per minute for the given path. It then queries `redis` to check how many requests have already been made for that path.
4. If the request count is still within the configured limit, the `rate limit service` signals the `ingress gateway` to forward the request to the target `service`.

![name](images/rate-limit-allowed.svg#center)

5. If the request count exceeds the configured limit, the `rate limit service` signals the `ingress gateway` to reject the request with the error code `429 Too Many Requests`.

![name](images/rate-limit-throttled.svg#center)

---
## Environment Setup: Kubernetes and Istio

To demonstrate rate limiting in practice, we will prepare an environment with Kubernetes, Istio, rate limit servic and test service.

#### 1. Install Docker and Colima
On macOS, the simplest way is via [Homebrew](https://brew.sh/):
```bash
brew install docker colima
```

#### 2. Start a Kubernetes Cluster with Colima
Install Kubernetes cli aka kubectl
```bash
brew install kubectl
```

Run Colima with Kubernetes enabled:
```bash
colima start --kubernetes --cpu 8 --memory 16 
```

Verify that the cluster is running:
```bash
kubectl get nodes
```

Expected result:
```bash
NAME     STATUS   ROLES                  AGE   VERSION
colima   Ready    control-plane,master   17m   v1.33.3+k3s1
```
Exact versions may differ, but you should see at least one node in the Ready state.

#### 3. Install Istio
Download and extract the latest release:
```bash
curl -L https://istio.io/downloadIstio | sh -
```

Move to the Istio package directory. For example, if the package is istio-1.27.1:
```bash
cd istio-1.27.1
```

Add the istioctl to your path:
```bash
export PATH=$PWD/bin:$PATH
```

Install Istio using the `default` profile:
```bash
istioctl install --set profile=default --skip-confirmation
```

Expected result:
```bash
        |\          
        | \         
        |  \        
        |   \       
      /||    \      
     / ||     \     
    /  ||      \    
   /   ||       \   
  /    ||        \  
 /     ||         \ 
/______||__________\
____________________
  \__       _____/  
     \_____/        

✔ Istio core installed ⛵️                                                                                                                                                                                               
✔ Istiod installed 🧠                                                                                                                                                                                                   
✔ Ingress gateways installed 🛬                                                                                                                                                                                         
✔ Installation complete 
```

Enable automatic sidecar injection for the default namespace:
```bash
kubectl label namespace default istio-injection=enabled
```

Apply the ingress gateway configuration:
```
kubectl apply -f https://raw.githubusercontent.com/lumiumco/istio-config/main/gateway.yaml
```

#### 4. Install the Rate Limit Service


#### 5. Install Service
