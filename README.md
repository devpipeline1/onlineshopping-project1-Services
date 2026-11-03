# Kubernetes Microservices Workshop — Live Product Catalogue

This version upgrades the original shopping demo with a PostgreSQL database and a live product-management screen. Docker Compose uses a named database volume locally; the Kubernetes demo intentionally uses no PV or PVC, so database data is ephemeral in EKS.

## Application components

- **Frontend:** Nginx serves the HTML page and reverse-proxies `/api/products` requests to Product API and `/api/orders` to Order API.
- **Product API:** Python API supporting Create, Read, Update and Delete (CRUD) for products.
- **PostgreSQL:** Shared persistent database. Both Product API replicas use the same database, so changes made through one replica are visible through the other.
- **Order API:** Existing sample order API, unchanged.

The product list automatically refreshes every 5 seconds. This is periodic polling (not WebSockets/server push), which is simple and reliable for a Kubernetes teaching lab.

## Product API endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/products` | List products |
| POST | `/products` | Create a product |
| PUT | `/products/{id}` | Update a product |
| DELETE | `/products/{id}` | Delete a product |
| GET | `/health` | API and database health |

Example create request:

```bash
curl -X POST http://localhost:8080/api/products \
  -H 'Content-Type: application/json' \
  -d '{"name":"Monitor","description":"27-inch display","price":12500,"stock":8}'
```

## 1. Run locally with Docker Compose

From the project root:

```bash
docker compose up --build -d
docker compose ps
```

Open http://localhost:8080. Add a product, edit it, and delete it. Open the page in a second browser window: the catalogue refreshes every five seconds and shows changes made in the other window.

Test the API:

```bash
curl http://localhost:8080/api/products
curl http://localhost:8080/api/products/1
```

Create a product:

```bash
curl -X POST http://localhost:8080/api/products \
  -H 'Content-Type: application/json' \
  -d '{"name":"Monitor","description":"27-inch display","price":12500,"stock":8}'
```

Update a product (replace `1` with the returned product ID):

```bash
curl -X PUT http://localhost:8080/api/products/1 \
  -H 'Content-Type: application/json' \
  -d '{"name":"Laptop Pro","description":"Updated model","price":72000,"stock":6}'
```

Delete a product:

```bash
curl -X DELETE http://localhost:8080/api/products/1
```

Stop the application but keep database data:

```bash
docker compose down
```

To delete the local database and all its data too:

```bash
docker compose down -v
```

## 2. Build images

```bash
docker build -t workshop-frontend:2.0 ./frontend
docker build -t workshop-product-api:2.0 ./product-api
docker build -t workshop-order-api:1.0 ./order-api
```

## 3. Push images to Amazon ECR

```bash
export AWS_REGION=ap-south-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws ecr create-repository --repository-name workshop-frontend --region "$AWS_REGION" 2>/dev/null || true
aws ecr create-repository --repository-name workshop-product-api --region "$AWS_REGION" 2>/dev/null || true
aws ecr create-repository --repository-name workshop-order-api --region "$AWS_REGION" 2>/dev/null || true

docker login --username AWS --password "$(aws ecr get-login-password --region "$AWS_REGION")" "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"

docker tag workshop-frontend:2.0 "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/workshop-frontend:2.0"
docker tag workshop-product-api:2.0 "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/workshop-product-api:2.0"
docker tag workshop-order-api:1.0 "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/workshop-order-api:1.0"

docker push "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/workshop-frontend:2.0"
docker push "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/workshop-product-api:2.0"
docker push "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/workshop-order-api:1.0"
```

## 4. Deploy to EKS

The Kubernetes manifest intentionally creates **no PersistentVolume (PV) or PersistentVolumeClaim (PVC)**. PostgreSQL writes to the container filesystem. No EBS CSI driver or `gp3` StorageClass is required for this demo.

Set the cluster context and ECR values:

```bash
kubectl config use-context application-eks
kubectl get nodes
export AWS_REGION=ap-south-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
sed "s/ACCOUNT_ID/$ACCOUNT_ID/g; s/AWS_REGION/$AWS_REGION/g" k8s/deployments.yaml > k8s/deployments.rendered.yaml
kubectl apply -f k8s/deployments.rendered.yaml
```

Wait for PostgreSQL before checking the APIs:

```bash
kubectl rollout status deployment/postgres -n microservices-demo
kubectl rollout status deployment/product-api -n microservices-demo
kubectl rollout status deployment/frontend -n microservices-demo
kubectl rollout status deployment/order-api -n microservices-demo
```

Access the frontend:

```bash
kubectl port-forward -n microservices-demo service/frontend-service 8080:80
```

Open http://localhost:8080.

## 5. Workshop demonstrations

### Demonstrate CRUD

Use the frontend form to add, edit, and delete products. Also use the curl examples above to call the API directly through the frontend proxy.

### Demonstrate automatic updates

Open the frontend in two browser windows. Add or edit a product in one window. The other window refreshes its product list within five seconds.

### Demonstrate shared database across API replicas

```bash
kubectl scale deployment product-api -n microservices-demo --replicas=3
kubectl get pods -n microservices-demo -l app=product-api -o wide
kubectl get svc postgres -n microservices-demo
kubectl logs deployment/product-api -n microservices-demo
```

All Product API replicas connect to the same PostgreSQL Service (`postgres:5432`) and shared database. The PostgreSQL workload is intentionally one replica for this teaching setup; it is not a highly available production database architecture.

### Inspect persistence

```bash
kubectl get pvc -n microservices-demo
kubectl get pv
```

The Kubernetes PostgreSQL database has no PV/PVC. Its data is lost if the PostgreSQL Pod is replaced or recreated. This is intentional for a short-lived workshop demo.

## 6. Important workshop notes

- The frontend polls every five seconds. This is near-real-time refresh, not WebSocket-based push.
- Product API replicas are stateless; product data lives in PostgreSQL, not in container memory.
- PostgreSQL uses a demo password in this manifest. For production, use Kubernetes Secrets or an external secret manager, TLS, backups, monitoring, and a managed/high-availability database such as Amazon RDS.
- The sample PostgreSQL Deployment has one replica and no PV/PVC. Do not treat this database as durable or highly available.
- PostgreSQL data can be lost when its Pod is replaced. This configuration is for workshops only.

## 7. Cleanup

```bash
kubectl delete namespace microservices-demo
```

This removes the Kubernetes demo resources. There is no PV/PVC to clean up.


## Place orders and view the order list

The Order API now persists orders in the same PostgreSQL database as the Product API. The frontend shows an **Order** button on each product.

1. Start with `docker compose up --build -d`.
2. Open `http://localhost:8080`.
3. Enter a customer name and click **Order** next to a product.
4. Enter a quantity. The Order API checks stock, saves the order and item, and decreases product stock in one database transaction.
5. The new order appears in the Order list. Products and orders refresh every five seconds.

Order API endpoints: `GET /orders` lists orders; `POST /orders` places an order using `customer_name`, `product_id`, and `quantity`; `GET /health` checks the service and database.

Example:
```bash
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"customer_name":"Asha","product_id":1,"quantity":2}'

curl http://localhost:8080/api/orders
```

Order items store the product name and price at purchase time, preserving the order history if a product is later renamed or deleted.


### Rebuild and deploy the order-enabled version to EKS

Build and push the updated Order API and Frontend images (Product API remains version 2.0):
```bash
docker build -t workshop-order-api:2.0 ./order-api
docker build -t workshop-frontend:3.0 ./frontend

docker tag workshop-order-api:2.0 "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/workshop-order-api:2.0"
docker tag workshop-frontend:3.0 "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/workshop-frontend:3.0"
docker push "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/workshop-order-api:2.0"
docker push "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/workshop-frontend:3.0"

kubectl apply -f k8s/deployments.rendered.yaml
kubectl rollout status deployment/order-api -n microservices-demo
kubectl rollout status deployment/frontend -n microservices-demo
```
The Kubernetes Order API Deployment receives `DATABASE_URL` pointing to the in-cluster `postgres` Service. The PostgreSQL Pod must be running and the database reachable. This Kubernetes configuration does not use a PV/PVC.
