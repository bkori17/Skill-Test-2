# Microservices Kubernetes Deployment Using Minikube

**Source repository:** https://github.com/mohanDevOps-arch/Microservices-Task  
**Platform:** AWS EC2 (Ubuntu 24.04 LTS), Docker, Minikube, Kubernetes  
**Ingress:** Not used (optional bonus excluded)

## 1. Objective

Deploy four Node.js microservices using **four Kubernetes Deployments** and **four ClusterIP Services**, and verify communication using Kubernetes DNS and a Gateway. Each Deployment includes image reference, matching labels/selectors, environment variables, resource requests/limits, and HTTP readiness/liveness probes.

| Application | Container port | Kubernetes Service | API endpoints |
| --- | ---: | --- | --- |
| User Service | 3000 | `user-service` | `/health`, `/users` |
| Product Service | 3001 | `product-service` | `/health`, `/products` |
| Order Service | 3002 | `order-service` | `/health`, `GET /orders`, `POST /orders` |
| Gateway Service | 3003 | `gateway-service` | `/health`, `/api/users`, `/api/products`, `/api/orders` |

The Gateway calls `user-service:3000`, `product-service:3001`, and `order-service:3002` through Kubernetes-internal service discovery.

**Database:** None. User/product information is defined in the supplied JavaScript code, and orders are stored **in memory**. Orders will be lost if the Order Service Pod restarts. No MongoDB, MySQL, PersistentVolume or PVC is required.

## 2. Architecture

```text
Windows browser (http://EC2_PUBLIC_IP:3003)
                 |
       AWS EC2 Security Group
           TCP 3003 (your IP only)
                 |
       kubectl port-forward
       (Kubernetes, NOT SSH tunneling)
                 |
        Gateway Service:3003
                 |
           Gateway Pod
                 |
         Kubernetes DNS
          /      |      \
         /       |       \
 User Service  Product Service  Order Service
  ClusterIP        ClusterIP       ClusterIP
    :3000            :3001           :3002
      |                |               |
   User Pod       Product Pod       Order Pod
```

Ingress is deliberately omitted because it is an optional bonus and the assessment's main tasks do not require it. External browser testing uses a temporary Kubernetes port-forward bound to the EC2 network interface; this is suitable for an assessment demonstration, not production exposure.

## 3. Prerequisites

- AWS EC2: **t3.large** (2 vCPU, 8 GiB RAM), **30 GiB gp3**, **Ubuntu Server 24.04 LTS x86_64**.
- EC2 inbound rules: **TCP 22** from your own IP for administration, **TCP 3003** from your own IP for browser demonstration. Do **not** open 3000–3002 publicly.
- Outbound internet access to install tools and build Node images.
- PuTTY/SSH access to EC2.

## 4. Install Docker, Minikube and kubectl

Run on the Ubuntu EC2 terminal:

```bash
sudo apt update
sudo apt install -y curl wget git ca-certificates docker.io unzip zip
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
```

**Disconnect and reconnect your SSH/PuTTY session** to activate Docker group permissions.

```bash
docker ps
docker --version
```

Install Minikube:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
```

Install kubectl:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/kubectl
kubectl version --client
```

Start the single-node Kubernetes cluster:

```bash
minikube start --driver=docker --cpus=2 --memory=5120
minikube status
kubectl get nodes
```

**📸 Capture :** `screenshots/minikube.png` showing Minikube running and the node `Ready`.

## 5. Clone original source and create Dockerfiles

```bash
cd ~
git clone https://github.com/mohanDevOps-arch/Microservices-Task.git
cd Microservices-Task
ls Microservices
```

There should be `user-service`, `product-service`, `order-service`, and `gateway-service`. Each uses `app.js` as its entry point and has a `package.json`; the provided packages do not define an npm start command.

Create one Dockerfile in each source directory:

```bash
for name in user product order gateway; do
  cat > "Microservices/${name}-service/Dockerfile" <<'DOCKERFILE'
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY app.js ./
CMD ["node", "app.js"]
DOCKERFILE
done

find Microservices -name Dockerfile
```

Build Docker images **inside Minikube**, so no Docker Hub login is needed:

```bash
minikube image build -t user-service:1.0 Microservices/user-service
minikube image build -t product-service:1.0 Microservices/product-service
minikube image build -t order-service:1.0 Microservices/order-service
minikube image build -t gateway-service:1.0 Microservices/gateway-service
minikube image ls | grep service
```

**📸 Capture :** `screenshots/docker-images.png` showing all four images.

## 6. Kubernetes manifest structure

The assessment submission contains **eight YAML files**:

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
├── screenshots/
│   ├── pods.png
│   ├── logs.png
│   └── service-test.png
└── README.md
```

**Manifests:**
- `deployments/*.yaml`: Deployment, one replica, service-name labels/selectors, locally built `:1.0` image with `imagePullPolicy: Never`, container port, `NODE_ENV` and `SERVICE_NAME`, resource requests `100m` CPU / `64Mi` memory, limits `500m` CPU / `256Mi` memory, `/health` readiness and liveness probes.
- `services/*.yaml`: Kubernetes `ClusterIP` Service, matching Pod selector and correct `port` / `targetPort` (3000/3001/3002/3003).

The actual eight YAML files must exist under `submission/deployments/` and `submission/services/` before the following commands are run. Use the accompanying starter-kit files or the generated manifests from the project instructions.

## 7. Validate and deploy

From the directory **containing** `submission/`:

```bash
kubectl apply --dry-run=client -f submission/deployments/
kubectl apply --dry-run=client -f submission/services/
```

**📸 Capture** `screenshots/yaml-validation.png` showing successful dry-run output.

Deploy:

```bash
kubectl apply -f submission/deployments/
kubectl apply -f submission/services/
kubectl rollout status deployment/user-service --timeout=120s
kubectl rollout status deployment/product-service --timeout=120s
kubectl rollout status deployment/order-service --timeout=120s
kubectl rollout status deployment/gateway-service --timeout=120s
```

Verify:

```bash
kubectl get pods
```


**📸 Capture(MANDATORY):** `screenshots/pods.png` showing `kubectl get pods` and all four running Pods.

Next:

```bash
kubectl get svc
```




## 8. Verify service communication inside Kubernetes

Use Node.js inside the Gateway Pod to query each backend using its **Kubernetes DNS name**:

```bash
kubectl exec deployment/gateway-service -- node -e "fetch('http://user-service:3000/users').then(r=>r.json()).then(console.log).catch(e=>{console.error(e);process.exit(1)})"

kubectl exec deployment/gateway-service -- node -e "fetch('http://product-service:3001/products').then(r=>r.json()).then(console.log).catch(e=>{console.error(e);process.exit(1)})"

kubectl exec deployment/gateway-service -- node -e "fetch('http://order-service:3002/orders').then(r=>r.json()).then(console.log).catch(e=>{console.error(e);process.exit(1)})"
```

**Expected:** User JSON (John Doe/Jane Smith), Product JSON (Laptop/Phone), and initially empty Order JSON (`[]`).

**📸 Capture :** `screenshots/communication.png` showing successful DNS-based responses.

Check startup logs:

```bash
kubectl logs deployment/user-service
kubectl logs deployment/product-service
kubectl logs deployment/order-service
kubectl logs deployment/gateway-service
```

**📸 Capture** `screenshots/logs.png` showing service startup logs and ideally successful communication test output. The sample apps log startup but may not log each HTTP call; successful API responses are important evidence.

## 9. Test Gateway with curl (no Ingress, no SSH tunnel)

In the **first EC2 terminal**, run and leave running:

```bash
kubectl port-forward --address 0.0.0.0 service/gateway-service 3003:3003
```

**Expected:** `Forwarding from 0.0.0.0:3003 -> 3003`.

In a **second EC2 terminal**, run:

```bash
curl http://127.0.0.1:3003/health
curl http://127.0.0.1:3003/api/users
curl http://127.0.0.1:3003/api/products
curl http://127.0.0.1:3003/api/orders
```

**Expected:** healthy Gateway status, two users, two products and initially `[]` orders.

Create an order through the Gateway:

```bash
curl -X POST http://127.0.0.1:3003/api/orders \
  -H 'Content-Type: application/json' \
  -d '{"userId":1,"productId":1}'

curl http://127.0.0.1:3003/api/orders
```

**Expected:** the new order appears in the GET response; its timestamp will reflect the actual test time.

**📸 Capture now (recommended):** `screenshots/order-test.png` showing successful POST and subsequent GET.

## 10. Test through a Windows browser

In AWS Console, copy the EC2 **Public IPv4 address**. Ensure the EC2 Security Group allows inbound TCP 3003 **from your public IP only** and that the port-forward command is still running.

Open these **HTTP** links (replace `EC2_PUBLIC_IP`):

```text
http://EC2_PUBLIC_IP:3003/health
http://EC2_PUBLIC_IP:3003/api/users
http://EC2_PUBLIC_IP:3003/api/products
http://EC2_PUBLIC_IP:3003/api/orders
```

The browser will display **JSON**, not a graphical frontend. That is expected for these API services.

**📸 Capture ** `screenshots/service-test.png` showing the public-IP browser URL and successful JSON output.

The port-forward process is temporary. Stop it with `Ctrl+C` after the demo and remove the temporary 3003 inbound Security Group rule.


**Project outcome:** A working four-service Node.js application on Minikube, using Kubernetes Service DNS, Deployment probes and resource controls, and a simple EC2 public-IP Gateway demonstration without Ingress or a database.


