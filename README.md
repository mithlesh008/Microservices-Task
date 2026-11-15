# Microservices Deployment Using Docker and Kubernetes

## Project Overview

This project demonstrates the containerization and deployment of a Node.js-based microservices application using Docker and Kubernetes.

The application consists of four independent microservices that communicate through Kubernetes Services. Kubernetes Deployments manage application replicas, while the Traefik Ingress Controller provides external HTTP routing to the microservices.

The project is deployed on a Kubernetes cluster running through k3d (Kubernetes in Docker) on an Amazon Linux EC2 instance.

## Application Components

| Microservice | Description | Port |
| --- | --- | ---: |
| User Service | Handles user-related operations | 3000 |
| Product Service | Handles product-related operations | 3001 |
| Order Service | Handles order-related operations | 3002 |
| Gateway Service | Acts as the API Gateway and handles incoming API requests | 3003 |

## Technologies Used

* Node.js — Microservices application runtime
* Express.js — REST API framework
* Docker — Application containerization
* Dockerfile — Container image creation
* `.dockerignore` — Excludes unnecessary files from the Docker build context
* Kubernetes — Container orchestration
* k3d — Lightweight Kubernetes cluster running in Docker
* Traefik — Kubernetes Ingress Controller
* Amazon Linux EC2 — Cloud infrastructure
* `kubectl` — Kubernetes cluster management
* Helm — Kubernetes package management

## Application Architecture

```text
                    Client / Browser
                           |
                    Traefik Ingress
                           |
                -----------------------
                |                     |
           Direct Routes          API Gateway
                |                     |
          ---------------        Gateway Service
          |      |      |            :3003
          |      |      |              |
         User  Product  Order           |
         :3000 :3001   :3002            |
          |      |      |              |
          --------|---------------------
                 Kubernetes
                  ClusterIP
                   Services
```

## API Routing

| Request path | Destination |
| --- | --- |
| `/users` | User Service |
| `/products` | Product Service |
| `/orders` | Order Service |
| `/api/users` | Gateway Service |
| `/api/products` | Gateway Service |
| `/api/orders` | Gateway Service |

The Gateway Service handles API requests and communicates with the respective microservices using Kubernetes internal service discovery.

## Project Directory Structure

```text
Microservices-Task/
│
├── Microservices/
│   ├── UserService/
│   │   ├── Dockerfile
│   │   └── .dockerignore
│   ├── ProductService/
│   │   ├── Dockerfile
│   │   └── .dockerignore
│   ├── OrderService/
│   │   ├── Dockerfile
│   │   └── .dockerignore
│   └── GatewayService/
│       ├── Dockerfile
│       └── .dockerignore
│
├── deployments/
│   ├── user-service.yaml
│   ├── product-service.yaml
│   ├── order-service.yaml
│   └── gateway-service.yaml
│
├── services/
│   ├── user-service.yaml
│   ├── product-service.yaml
│   ├── order-service.yaml
│   └── gateway-service.yaml
│
├── ingress/
│   └── microservices-ingress.yaml
│
├── screenshots/
│   ├── pods.png
│   ├── logs.png
│   └── service-test.png
│
└── README.md
```

## 1. Docker Containerization

Each microservice has its own Dockerfile to build an independent Docker image.

### Docker Images

| Service | Docker image | Port |
| --- | --- | ---: |
| User Service | `user-service:latest` | 3000 |
| Product Service | `product-service:latest` | 3001 |
| Order Service | `order-service:latest` | 3002 |
| Gateway Service | `gateway-service:latest` | 3003 |

### Build Docker Images

Run these commands from the project root directory:

```bash
docker build -t user-service:latest ./Microservices/UserService
docker build -t product-service:latest ./Microservices/ProductService
docker build -t order-service:latest ./Microservices/OrderService
docker build -t gateway-service:latest ./Microservices/GatewayService
```

### Verify Docker Images

```bash
docker images
```

### Docker Image Optimization

Each microservice includes a `.dockerignore` file to exclude unnecessary files, such as:

* `node_modules`
* `.git`
* `.env`
* `npm-debug.log`
* `coverage`

This reduces the Docker build context and avoids including unnecessary files in container images.

## 2. Kubernetes Cluster Setup

The Kubernetes cluster is configured using k3d on an Amazon Linux EC2 instance.

### Prerequisites

Install the following tools:

* Docker
* `kubectl`
* k3d
* Helm

Verify the installations:

```bash
docker --version
kubectl version --client
k3d version
helm version
```

### Create the k3d Cluster

```bash
k3d cluster create Microservices \
  --servers 1 \
  --agents 2 \
  --port "80:80@loadbalancer" \
  --port "443:443@loadbalancer"
```

Verify the cluster:

```bash
kubectl cluster-info
kubectl get nodes
```

### Import Local Docker Images into k3d

Since the Docker images are built locally, import them into the k3d cluster:

```bash
k3d image import \
  user-service:latest \
  product-service:latest \
  order-service:latest \
  gateway-service:latest \
  -c Microservices
```

This makes the locally built images available to Kubernetes without requiring an external container registry.

> **Note:** Kubernetes Deployments use `imagePullPolicy: Never` for locally imported images.

## 3. Kubernetes Deployments

Separate Deployment manifests are created for all four microservices. Each Deployment includes:

* Container image reference
* Replica configuration
* CPU and memory requests
* CPU and memory limits
* Environment variables
* Liveness and readiness probes
* Labels and selectors
* Container ports

### Deployment Files

| File | Deployment |
| --- | --- |
| `deployments/user-service.yaml` | User Service |
| `deployments/product-service.yaml` | Product Service |
| `deployments/order-service.yaml` | Order Service |
| `deployments/gateway-service.yaml` | Gateway Service |

### Deploy Applications

Apply the manifests individually:

```bash
kubectl apply -f deployments/user-service.yaml
kubectl apply -f deployments/product-service.yaml
kubectl apply -f deployments/order-service.yaml
kubectl apply -f deployments/gateway-service.yaml
```

Or apply all Deployment manifests at once:

```bash
kubectl apply -f deployments/
```

### Verify Deployments

```bash
kubectl get deployments
kubectl get pods -o wide
kubectl describe deployments
```

## 4. Kubernetes Services

Kubernetes Services provide stable network endpoints for communication between microservices. All four microservices use the `ClusterIP` service type to enable internal cluster-level communication.

### Service Configuration

| Service | Kubernetes Service name | Port | Type |
| --- | --- | ---: | --- |
| User Service | `user-service` | 3000 | `ClusterIP` |
| Product Service | `product-service` | 3001 | `ClusterIP` |
| Order Service | `order-service` | 3002 | `ClusterIP` |
| Gateway Service | `gateway-service` | 3003 | `ClusterIP` |

### Service Files

```text
services/
├── user-service.yaml
├── product-service.yaml
├── order-service.yaml
└── gateway-service.yaml
```

Apply and verify Services:

```bash
kubectl apply -f services/
kubectl get services
kubectl get endpoints
```

### Inter-Service Communication

Services can communicate with each other using Kubernetes DNS names:

```text
http://user-service:3000
http://product-service:3001
http://order-service:3002
http://gateway-service:3003
```

This allows services to communicate without requiring individual pod IP addresses.

## 5. Traefik Ingress Controller (Bonus Task)

Traefik Ingress Controller exposes application endpoints through a single entry point. Traefik is available as the default Ingress Controller in the k3s-based k3d cluster.

### Verify Traefik

```bash
kubectl get pods -n kube-system
kubectl get deployment traefik -n kube-system
kubectl get ingressclass
```

### Ingress Configuration

The Ingress resource uses the `traefik` IngressClass.

Manifest: `ingress/microservices-ingress.yaml`

| URL path | Backend service | Port |
| --- | --- | ---: |
| `/users` | User Service | 3000 |
| `/products` | Product Service | 3001 |
| `/orders` | Order Service | 3002 |
| `/api/users` | Gateway Service | 3003 |
| `/api/products` | Gateway Service | 3003 |
| `/api/orders` | Gateway Service | 3003 |

The `/api` routes are forwarded to the Gateway Service, which handles API requests and communicates with the respective microservices.

Apply and verify the Ingress:

```bash
kubectl apply -f ingress/microservices-ingress.yaml
kubectl get ingress
kubectl describe ingress microservices-ingress
```

### Configure the Hostname

For hostname-based testing, add the following entry to the client's hosts file, replacing `<EC2_PUBLIC_IP>` with the EC2 instance's public IP address:

```text
<EC2_PUBLIC_IP> microservices.local
```

On Amazon Linux, the hosts file is `/etc/hosts`.

For external access, configure the EC2 Security Group to allow inbound HTTP traffic on port 80 and HTTPS traffic on port 443.

> **Note:** The k3d load balancer must publish these ports to the EC2 host. DNS or hosts-file configuration alone does not expose the application.

## 6. Application Testing and Validation

### Check Running Pods and Services

```bash
kubectl get pods
kubectl get svc
```

### Check Application Logs

```bash
kubectl logs deployment/user-service
kubectl logs deployment/product-service
kubectl logs deployment/order-service
kubectl logs deployment/gateway-service
```

### Test Services Using Port Forwarding

Run each command in a separate terminal session:

```bash
kubectl port-forward svc/user-service 3000:3000
kubectl port-forward svc/product-service 3001:3001
kubectl port-forward svc/order-service 3002:3002
kubectl port-forward svc/gateway-service 3003:3003
```

### Test Gateway API Endpoints

```bash
curl http://localhost:3003/api/users
curl http://localhost:3003/api/products
curl http://localhost:3003/api/orders
```

### Test Direct Services

Use the service endpoints supported by the respective microservices:

```bash
curl http://localhost:3000
curl http://localhost:3001
curl http://localhost:3002
```

The exact response depends on the routes implemented by each service.

### Test Through Ingress

Using the configured hostname:

```bash
curl http://microservices.local/users
curl http://microservices.local/products
curl http://microservices.local/orders
curl http://microservices.local/api/users
curl http://microservices.local/api/products
curl http://microservices.local/api/orders
```

These commands assume that the hostname resolves to the EC2 host and the Ingress ports are correctly exposed.

## 7. Troubleshooting

### Pods Not Starting

Check pod status, describe the problematic pod, and inspect its logs:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### `ImagePullBackOff` or `ErrImageNeverPull`

Verify the local Docker images and import them into k3d again if needed:

```bash
docker images
k3d image import \
  user-service:latest \
  product-service:latest \
  order-service:latest \
  gateway-service:latest \
  -c Microservices
```

Also verify the Deployment image reference and image pull policy.

### Service Communication Issues

```bash
kubectl get svc
kubectl get endpoints
```

Verify that Service selectors match the Deployment pod labels.

### Ingress Not Working

Check the Ingress and Traefik resources:

```bash
kubectl get ingress
kubectl describe ingress microservices-ingress
kubectl get pods -n kube-system
kubectl get svc -n kube-system
```

Verify that:

* The IngressClass is `traefik`.
* Backend Service names and ports are correct.
* Kubernetes Services have healthy endpoints.
* The k3d load balancer exposes the required host ports.
* The EC2 Security Group allows inbound traffic.

### Application Startup Issues

Check the application logs:

```bash
kubectl logs deployment/gateway-service
```

Ensure the Dockerfile uses the correct Node.js application entry point. For example:

```dockerfile
CMD ["node", "app.js"]
```

The entry-point filename must match the actual application file.

## 8. Screenshots and Evidence

The `screenshots/` directory contains evidence of deployment and testing.

| Screenshot | Description |
| --- | --- |
| `pods.png` | Shows all four microservice pods running successfully |
| `logs.png` | Shows application logs and service communication |
| `service-test.png` | Shows successful API requests using `curl` or port-forwarding |

These screenshots demonstrate the successful deployment and validation of the application.

## 9. Deployment Workflow

1. Build Docker images for all four microservices.
2. Create the k3d Kubernetes cluster.
3. Import local Docker images into the cluster.
4. Apply Kubernetes Deployment manifests.
5. Apply Kubernetes `ClusterIP` Service manifests.
6. Install and verify the Traefik Ingress Controller.
7. Apply the Ingress configuration.
8. Validate pods, Services, logs, and API endpoints.
9. Capture screenshots and document the results.

## 10. Assignment Deliverables

The submission contains the following components:

```text
submission/
├── deployments/
│   ├── user-service.yaml
│   ├── product-service.yaml
│   ├── order-service.yaml
│   └── gateway-service.yaml
├── services/
│   ├── user-service.yaml
│   ├── product-service.yaml
│   ├── order-service.yaml
│   └── gateway-service.yaml
├── ingress/
│   └── microservices-ingress.yaml
├── screenshots/
│   ├── pods.png
│   ├── logs.png
│   └── service-test.png
└── README.md
```

The Dockerfiles and `.dockerignore` files are maintained alongside their respective microservice source directories.

## 11. Conclusion

This project demonstrates the deployment of a containerized Node.js microservices application using Docker and Kubernetes. The four independent services are deployed using Kubernetes Deployments and exposed internally through `ClusterIP` Services. Traefik Ingress Controller provides centralized HTTP routing, while Kubernetes DNS enables service-to-service communication.

The application is deployed on a k3d Kubernetes cluster running on Amazon Linux EC2, demonstrating container orchestration, service discovery, Ingress routing, and cloud-based deployment.

## Project Repository

GitHub Repository — Microservices Task
