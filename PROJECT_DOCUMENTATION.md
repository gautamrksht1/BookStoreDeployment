# Online Bookstore Microservices

## 1. Project Summary

This project is a small online bookstore implemented as a server-rendered web application backed by two independently deployable Python APIs and a shared MySQL database. The application demonstrates service decomposition, API-based communication, container builds, Kubernetes service discovery, replica scaling, and an administrator workflow for maintaining the catalog.

The system supports these business cases:

- Customers browse available books and submit a purchase request.
- The Order Service retrieves book details, asks the Catalog Service to reserve stock, and records a successful order.
- Administrators sign in and add books with price and initial stock quantity.
- Kubernetes maintains multiple Catalog and Order replicas and distributes requests across healthy pods.

The project is intended as a local learning/demo deployment. It does not currently implement payment processing, customer accounts, shipping, order cancellation, or production-grade identity management.

## 2. Repository and Folder Layout

The deployment folder is the orchestration root. Each application folder can be maintained as its own Git repository.

```text
H:/DockerKubernetes-MicroServices/
  BookStoreDeployment/
    docker-compose.yml
    k8s.yaml
    .env                 # local values; ignored by Git
    .env.example         # template; safe to commit
    mysql-init/
  BookStoreUI/            # ASP.NET Core MVC web application
  CatalogService/         # FastAPI book and inventory API
  OrderService/           # FastAPI order API
```

The Compose build contexts in `docker-compose.yml` point to the sibling application folders. The Python services each contain their own `database.py`; they do not depend on the older `OnlineBookStoreSolution` folder.

## 3. Architecture

```mermaid
flowchart LR
    Browser[Customer or Admin browser]
    UI[BookStoreUI\nASP.NET Core MVC]
    Order[Order Service\nFastAPI]
    Catalog[Catalog Service\nFastAPI]
    DB[(MySQL\nbookcatalog)]

    Browser -->|HTTP| UI
    UI -->|GET /books| Catalog
    UI -->|POST /orders| Order
    Order -->|GET /books/{id}| Catalog
    Order -->|POST /inventory/reserve| Catalog
    Catalog -->|books and inventory| DB
    Order -->|orders| DB
    UI -. Admin POST /books .-> Catalog
```

In Kubernetes, the three APIs are addressed by their in-cluster Service DNS names. The Catalog and Order Services are `ClusterIP` Services; the UI is exposed using a `NodePort`. In Docker Compose, all containers join the Compose network and use service names such as `catalog-service` and `order-service` for internal traffic.

## 4. Services and Ownership

### BookStoreUI

- ASP.NET Core MVC application targeting .NET 8.
- Renders the customer book listing and purchase forms.
- Calls Catalog and Order APIs from the server side using `HttpClient`.
- Hosts the Admin sign-in and catalog management pages.
- Reads `CATALOG_SERVICE_URL`, `ORDER_SERVICE_URL`, and `BOOKSTORE_ADMIN_PASSWORD` from environment variables.

### Catalog Service

- FastAPI application owning the `books` table and stock quantity.
- Creates its SQLAlchemy schema at startup and retries database startup while MySQL is unavailable.
- Validates new book input with Pydantic.
- Reserves stock atomically with a conditional SQL update.

### Order Service

- FastAPI application owning the `orders` table.
- Reads `CATALOG_SERVICE_URL` from the environment.
- Obtains current book pricing and reserves inventory through the Catalog API before recording an order.
- Closes SQLAlchemy sessions after requests so Kubernetes health probes and normal traffic do not leak database connections.

### MySQL

- Stores `books` and `orders` in the `bookcatalog` database.
- Is a single shared instance in local Compose and Minikube deployments.
- Uses persistent storage in Kubernetes through a PersistentVolumeClaim.

## 5. Main Business Workflows

### Customer Browse

1. The browser requests the UI home page.
2. The UI calls `GET /books` on the Catalog Service.
3. Catalog reads the book list and current stock from MySQL.
4. The UI renders book title, author, price, quantity, and a purchase form.

### Place Order

1. The customer submits a book ID and quantity to the UI.
2. The UI sends `POST /orders` to the Order Service.
3. Order requests book details from Catalog.
4. Order asks Catalog to reserve the requested quantity.
5. Catalog decrements stock only when enough stock remains.
6. If reservation succeeds, Order writes an order row and returns an order ID.
7. If reservation fails, Order returns a failure response for insufficient stock.

### Admin Adds a Book

1. An administrator opens `/Admin/Books` and is redirected to `/Admin/Login` if not authenticated.
2. The administrator signs in with the password configured as `BOOKSTORE_ADMIN_PASSWORD`.
3. The Admin page displays the current catalog and a form for title, author, price, and available quantity.
4. The UI posts the validated book data to Catalog `POST /books`.
5. Catalog persists the book and returns the created record.

## 6. API Reference

All API routes are currently unauthenticated at the API layer. Admin authentication is enforced by the MVC UI route, not by Catalog itself.

### Catalog Service

| Method | Route | Purpose | Successful response |
|---|---|---|---|
| `GET` | `/books` | List all books | Array of book records |
| `GET` | `/books/{book_id}` | Get a book by ID | Book record, or a not-found message |
| `POST` | `/books` | Create a catalog entry (used by Admin UI) | `201` and created book |
| `POST` | `/inventory/reserve` | Reserve stock for a book | `{"success": true}` or insufficient-stock result |

Create-book request:

```json
{
  "title": "Example Book",
  "author": "A. Writer",
  "price": 19.95,
  "available_quantity": 12
}
```

Reservation request:

```json
{
  "book_id": 1,
  "quantity": 2
}
```

### Order Service

| Method | Route | Purpose | Successful response |
|---|---|---|---|
| `POST` | `/orders` | Reserve inventory and create an order | `{"success": true, "order_id": 1}` |
| `GET` | `/orders` | List persisted orders; also used by Kubernetes probes | Array of order records |

Order request:

```json
{
  "book_id": 1,
  "quantity": 2
}
```

FastAPI publishes interactive API documentation at `/docs` for each service when reached on its service port.

## 7. Data Model

### Book

| Field | Type | Meaning |
|---|---|---|
| `id` | Integer, primary key | Catalog identifier |
| `title` | String, up to 150 characters | Book title |
| `author` | String, up to 100 characters | Author name |
| `price` | Numeric/float | Unit price |
| `available_quantity` | Integer | Current available stock |

### Order

| Field | Type | Meaning |
|---|---|---|
| `id` | Integer, primary key | Order identifier |
| `book_id` | Integer | Catalog book identifier |
| `quantity` | Integer | Units ordered |
| `total_amount` | Numeric/float | Price multiplied by quantity |
| `order_status` | String | Current implementation records `SUCCESS` |

The Order table stores `book_id` but does not currently declare a cross-service database foreign key. Services share the same MySQL database but own separate tables and logic.

## 8. Configuration and Secrets

`BookStoreDeployment/.env` supplies local Docker Compose values and is ignored by Git. Use `.env.example` as the variable-name template. Never commit real passwords.

| Variable | Consumer | Purpose |
|---|---|---|
| `DB_NAME` | MySQL and Python services | Database name |
| `DB_USER` | Python services | Database login name |
| `DB_PASSWORD` | MySQL and Python services | Database credential |
| `CATALOG_SERVICE_URL` | UI and Order Service | Catalog API base URL |
| `ORDER_SERVICE_URL` | UI | Order API base URL |
| `BOOKSTORE_ADMIN_PASSWORD` | UI | Local Admin sign-in credential |

In Kubernetes, DB credentials are read from Secret `bookstore-db`; the Admin credential is read from Secret `bookstore-admin`. Non-sensitive service names and database settings are stored in ConfigMap `bookstore-config`.

The Admin implementation uses a single shared password and an ASP.NET Core authentication cookie. For production, replace this with an identity provider or managed user accounts, TLS, password rotation, and an authorization policy at the Catalog API boundary.

## 9. Container and Kubernetes Deployment

### Docker Compose

Run commands from the deployment directory:

```powershell
Set-Location H:\DockerKubernetes-MicroServices\BookStoreDeployment
docker compose config
docker compose up --build -d
docker compose ps
docker compose logs -f
```

Current host mappings are chosen to coexist with the older solution: UI `http://localhost:5001`, Catalog `localhost:8101`, Order `localhost:8102`, and MySQL `localhost:3308`. Container-to-container calls continue using internal ports 8080, 8001, 8002, and 3306.

### Minikube

The Kubernetes resources are in `k8s.yaml`, namespace `bookstore`:

- MySQL Deployment, ClusterIP Service, and persistent claim.
- Catalog Deployment and ClusterIP Service.
- Order Deployment and ClusterIP Service.
- BookStoreUI Deployment and NodePort Service.
- ConfigMap and Secret references for runtime configuration.
- Startup, readiness, and liveness probes.
- CPU requests and limits for the scalable API pods.
- CPU-based HorizontalPodAutoscalers for Catalog and Order.

Build, load, and roll out only the image for a changed application:

```powershell
# Catalog
Set-Location H:\DockerKubernetes-MicroServices\CatalogService
docker build -t bookstore/catalog-service:local -f dockerfile .
minikube image load bookstore/catalog-service:local
kubectl -n bookstore rollout restart deployment/catalog-service
kubectl -n bookstore rollout status deployment/catalog-service --timeout=180s
```

Use the corresponding image tags and Deployment names for Order (`bookstore/order-service:local`, `order-service`) and UI (`bookstore/ui:local`, `bookstore-ui`). If Kubernetes configuration changes, apply the manifest:

```powershell
kubectl apply -f H:\DockerKubernetes-MicroServices\BookStoreDeployment\k8s.yaml
```

Inspect rollout and autoscaling:

```powershell
kubectl -n bookstore get deployments,pods,services,hpa
kubectl -n bookstore get hpa -w
kubectl top pods -n bookstore
```

The current Catalog and Order Deployments each run at least two replicas and can scale to five replicas when average CPU exceeds the 70% utilization target. Kubernetes Services route traffic to ready endpoints; current scaling was verified with two ready endpoints per API. Actual scaling above two depends on sustained load and available cluster capacity.

## 10. Health Checks and Operations

- Catalog readiness and liveness use `GET /books`.
- Order readiness and liveness use `GET /orders`.
- UI readiness and liveness use `GET /`.
- MySQL probes use `mysqladmin ping`.
- `kubectl -n bookstore logs deployment/<name>` displays service logs.
- `kubectl -n bookstore describe pod <pod-name>` and `kubectl -n bookstore get events --sort-by=.lastTimestamp` help diagnose scheduling, probes, and container failures.
- The Kubernetes Dashboard can be opened through the Minikube dashboard addon and set to the `bookstore` namespace.

The Python API handlers close their SQLAlchemy sessions in `finally` blocks. This is important because health probes call database-backed endpoints repeatedly.

## 11. Known Limitations and Follow-up Work

- MySQL is a single instance. API replicas improve request handling but do not make persistence highly available; database capacity and reliability can become bottlenecks.
- Inventory is reserved before the Order row is committed. A failure during order persistence can leave stock reduced without a matching order. A production workflow should use a transaction/outbox, compensation, or another consistency strategy.
- Requests to Catalog and inventory reservations do not currently use explicit HTTP timeouts or comprehensive response/error handling in the Order Service.
- The UI catches connectivity exceptions but does not consistently display API error bodies or non-success statuses from order placement.
- Book prices and totals currently use floating-point columns in the Python models; currency-safe decimal storage and rounding rules should be introduced before real payments.
- Admin is a single shared password, not a user/role management system. API endpoints are not independently authenticated.
- The deployment is configured for local Compose/Minikube. A shared remote Kubernetes cluster would require registry-hosted immutable image tags, managed secrets, ingress/TLS, resource planning, backups, and monitoring.
- The `mysql-init` directory is currently empty; schema tables are created by the services at startup, and the catalog starts without sample books unless an Admin adds them.

## 12. Current Local Scaling Snapshot

At the last verified Minikube check, Catalog and Order each had two Ready replicas, two EndpointSlice addresses, and valid CPU metrics. Their HPAs were configured for a minimum of two, maximum of five, and 70% average CPU utilization. Low observed CPU kept them at two replicas; no artificial load test is included in this project document.