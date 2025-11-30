---
date: '2025-09-22T00:00:00+00:00'
draft: false
title: 'Rate Limiting in Practice: Simple Scenario'
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

1. Every request from a client first passes through the `ingress gateway`.
2. The `ingress gateway` extracts all necessary parameters, such as the request path, and forwards them to the `rate limit service`.
3. The `rate limit service` uses the `service` configuration to determine the maximum number of requests per minute for the given path. It then queries `redis` to check how many requests have already been made for that path.
4. If the request count is still within the configured limit, the `rate limit service` signals the `ingress gateway` to forward the request to the target `service`.

![ratelimit-allowed](images/ratelimit-allowed.svg#center)

5. If the request count exceeds the configured limit, the `rate limit service` signals the `ingress gateway` to reject the request with the error code `429 Too Many Requests`.

![ratelimit-throttled](images/ratelimit-throttled.svg#center)

---
## Environment Setup: Kubernetes and Istio

To demonstrate rate limiting in practice, we will prepare an environment with Kubernetes, Istio, rate limit service and test service.

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
kubectl apply -f https://raw.githubusercontent.com/lumiumco/ratelimit-configs/refs/heads/main/istio/gateway.yaml
```

#### 4. Install the Rate Limit Service
We will use [Helm](https://helm.sh/), the package manager for Kubernetes, to install our services.
If you don’t have Helm yet, install it using Homebrew:
```bash
brew install helm
```

Next, add the Lumiumco Helm repository, which contains the charts for services:
```bash
helm repo add lumiumco https://lumiumco.github.io/helm-charts/
helm repo update
```

Now install the rate limit service chart into the `istio-system` namespace:
```bash
helm install ratelimit lumiumco/ratelimit -n istio-system
```

This will deploy the service alongside Istio components.
After a short while, you can verify that it’s running:
```bash
kubectl get pods -n istio-system -l app=ratelimit
```

Expected output:
```bash
NAME                         READY   STATUS    RESTARTS   AGE
ratelimit-7689c576f6-rpvnr   1/1     Running   0          1m
```

Create EnvoyFilter to apply this configuration:
```bash
kubectl apply -f https://raw.githubusercontent.com/lumiumco/ratelimit-configs/refs/heads/main/simple/envoy-filter.yaml
```

#### 5. Install the Light Service
Next, let’s install the `light` service. 
It is simple [Java application](https://github.com/lumiumco/light) that exposes a single REST endpoint:
```
GET /light
```

We deploy this service using a common [Helm chart](https://github.com/lumiumco/helm-charts/tree/main/charts/service).
Each service has its own values.yaml file to define specific parameters such as the image, and virtual service.
```bash
helm install light lumiumco/service -f https://raw.githubusercontent.com/lumiumco/light/refs/heads/main/values.yaml
```

After a short while, you can verify that `light` service running:
```bash
kubectl get pods -l app.kubernetes.io/name=light
```

Expected output:
```bash
NAME                     READY   STATUS    RESTARTS   AGE
light-66cc85fd65-bn6nc   2/2     Running   0          1m
```

Now we can test the endpoint.
Run the following command:
```bash
curl http://localhost/light/public
```

You should see the result:
```bash
{"company":"n/a","result":"Light result (public)","client":"n/a","type":"public"}
```

#### 6. Apply and Verify Rate Limiting Configuration
With the `light` service running, we can now apply rate limiting configuration to control how many requests clients can make within a specific time window.

The rate limiting configuration is defined as a `ConfigMap`. Each configuration specifies a **domain**, **descriptors** (which identify what is being limited), and the **limit** itself (requests per unit of time).

We prepared the simplest rate limit configuration:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: light-ratelimit
  namespace: istio-system
data:
  light-ratelimit.yaml: |
    domain: api
    descriptors:
      - key: path
        value: /light*
        rate_limit:
          unit: minute
          requests_per_unit: 3
```

This configuration limits access to all `/light*` endpoints to **3 requests per minute** for all clients.

Apply this configuration:
```bash
kubectl apply -f https://raw.githubusercontent.com/lumiumco/ratelimit-configs/refs/heads/main/simple/config-map.yaml
```

Restart `ratelimit` deployment:
```bash
kubectl rollout restart deployment ratelimit -n istio-system
```

To verify that the configuration is active, send multiple requests in quick succession:
```bash
for i in {1..6}; do curl -s -o /dev/null -w "%{http_code}\n" http://localhost/light/public; done
```

The first few requests should return `200`, and once the limit is reached, you should start seeing `429`:
```bash
200
200
200
429
429
429
```
indicating `Too Many Requests`.
This confirms that rate limiting is working as intended.

#### 7. How It All Fits Together
Now that everything is running, let’s review what we actually configured and how the pieces interact.

![ratelimit-configs](images/ratelimit-configs.svg#center)

1. Istio Gateway

We installed Istio and applied a **Gateway** configuration.
The gateway acts as the single entry point to our Kubernetes cluster
It receives all incoming traffic and routes it to `light` services.
In our setup, every request to the cluster first passes through this gateway.

2. Rate Limit Service

Next, we deployed the **Rate Limit Service**.
It is responsible for tracking how many requests have been made for each defined path and for deciding whether a new request should be allowed or rejected.

3. EnvoyFilter

We then applied an **EnvoyFilter** configuration.
This filter modifies the behavior of the Istio ingress gateway so that, before forwarding a request to its target service, the gateway:

 1. Extracts the request path, for example, `/light/public`.
 2. Sends this information to the rate limit service.
 3. Waits for a response indicating whether the request is still within the allowed limit.

If the rate limit service responds with an allow, the request proceeds to the target service.
If the limit is exceeded, the gateway immediately returns `429 Too Many Requests`.

4. ConfigMap

Finally, we applied a **ConfigMap** that defines the actual rate limiting policy.
This configuration tells the `rate limit service` how many requests are allowed for specific paths and over what time period.
In our example, all endpoints matching `/light*` are limited to `3 requests per minute` across all clients.

When a client sends a request:

 - the **Istio Gateway** receives it.
 - the **EnvoyFilter** ensures that the gateway contacts the **Rate Limit Service**.
 - the **Rate Limit Service** checks Redis and the **ConfigMap** definitions to determine if the request is within the limit.
 - based on the result, the gateway either forwards the request to the **service** or responds with a 429.

#### 8. Conclusion

With this setup, we have built a rate limit on top of Kubernetes and Istio.
It demonstrates how a few declarative configurations can work together to enforce traffic control at the edge of your cluster.

In this article, we built a working rate limiting setup using Kubernetes and Istio.
With just a few declarative configurations — the Gateway, EnvoyFilter, and ConfigMap — we established a foundation for controlling traffic and protecting services from overload.


In the next articles of this series, we will gradually introduce more complex scenarios:

 - multiple services with different limits
 - clients with different limits
 - metrics collection

#### Thanks for reading. See you in the next part!
