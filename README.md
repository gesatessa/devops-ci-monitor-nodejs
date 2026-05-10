# DevOps CI Monitor Node

A full-stack Node.js application using:

- React frontend
- Express backend
- MySQL database
- Docker & Docker Compose

---

# Project Structure

```text
.
├── client/              # React frontend
├── server/              # Express backend
├── Dockerfile
├── docker-compose.yml
├── k8-manifests/
└── README.md
```

---

# Architecture

```text
Browser
   ↓
Express Server (Node.js)
   ├── REST API (/api/users)
   └── Serves React frontend

Express
   ↓
MySQL Database
```

---

# Technologies Used

## Frontend
- React
- Webpack
- Babel
- Axios

## Backend
- Node.js
- Express
- mysql2
- body-parser
- cors

## Infrastructure
- Docker
- Docker Compose
- MySQL 8

---

# Running the Application

## Prerequisites

Install:

- Docker
- Docker Compose

---

# Start the Application

```bash
docker compose up --build
```

---

# Access the Application

Frontend + Backend:

```text
http://localhost:5000
```

MySQL:

```text
localhost:3306
```

---

# Stop the Application

```bash
docker compose down
```

---

# Remove Database Volume

WARNING: This deletes all database data.

```bash
docker compose down -v
```

---

# Docker Compose Overview

The compose file starts:

## app
- Builds the Node.js application
- Exposes port 5000
- Connects to MySQL

## mysql
- Runs MySQL 8
- Creates:
  - database: test_db
  - user: appuser
- Stores persistent data in Docker volume

---

# Environment Variables

| Variable | Description |
|---|---|
| PORT | App port |
| DB_HOST | MySQL hostname |
| DB_USER | MySQL username |
| DB_PASSWORD | MySQL password |
| DB_NAME | Database name |

---

# API Endpoints

## Get Users

```http
GET /api/users
```

---

## Create User

```http
POST /api/users
```

Body:

```json
{
  "name": "John",
  "email": "john@example.com",
  "role": "Admin"
}
```

---

## Update User

```http
PUT /api/users/:id
```

---

## Delete User

```http
DELETE /api/users/:id
```

---

# Lessons Learned

## 1. Docker `depends_on` does NOT mean "ready"

`depends_on` only controls startup order.

It does NOT guarantee:
- database readiness
- application availability

A healthcheck or retry logic is still required.

---

## 2. MySQL containers initialize in phases

MySQL 8 container:
1. starts temporary initialization server
2. creates database/users
3. shuts down
4. starts real server

Applications may fail if they connect during this transition.

---

## 3. Retry logic matters in distributed systems

Applications should:
- retry database connections
- tolerate startup delays
- avoid crashing immediately

---

## 4. Reuse of failed DB connections is dangerous

A failed MySQL connection object should NOT be reused.

Correct approach:
- create a new connection for every retry attempt

---

## 5. `mysql` package is outdated

The old Node.js `mysql` package is incompatible with modern MySQL 8 authentication.

Use:

```bash
npm install mysql2
```

instead.

---

## 6. Container networking uses service names

Inside Docker Compose:

```text
localhost != mysql container
```

Applications must connect using:

```text
DB_HOST=mysql
```

where `mysql` is the compose service name.

---

## 7. Non-root containers are best practice

The Dockerfile creates:

- appuser
- appgroup

This improves container security.

---

## 8. Layer caching improves Docker builds

Copying `package.json` before source files allows Docker to cache dependency installation layers.

This dramatically speeds up rebuilds.

---

## 9. Frontend build output matters

Webpack outputs directly into:

```text
client/public/
```

This explains why Express serves static assets from that directory.

Understanding build output paths is critical when debugging Dockerized frontend apps.

---

## 10. Modern Docker Compose no longer needs `version`

This is obsolete:

```yaml
version: '3.9'
```

Modern Docker Compose auto-detects schema versions.

---

# Future Improvements

## Backend
- Use MySQL connection pooling
- Add centralized error handling
- Add request validation
- Add logging

## Frontend
- Use webpack-dev-server or Vite
- Separate build output into `/dist`
- Add environment-based configuration

## Docker
- Use multi-stage builds
- Reduce image size
- Add `.dockerignore`

## Infrastructure
- Add NGINX reverse proxy
- Add Kubernetes deployment improvements
- Add CI/CD pipeline

---

# Useful Commands

## View logs

```bash
docker compose logs -f
```

## App logs only

```bash
docker compose logs -f app
```

## MySQL logs only

```bash
docker compose logs -f mysql
```

## Rebuild containers

```bash
docker compose up --build
```

---

# Final Notes

This project demonstrates:
- containerized full-stack development
- Docker networking
- MySQL integration
- frontend/backend integration
- operational debugging techniques

The debugging process itself was valuable because it exposed several real-world container orchestration behaviors that developers frequently encounter in production systems.


## MySQL

```sh
docker compose exec -it db mysql -uappuser -ppassword123 test_db

```

```sql
SHOW DATABASES;

USE test_db;

SHOW TABLES;

DESCRIBE users;
SELECT * FROM users;

INSERT INTO users (name, email, role)
VALUES ('kimi', 'kimi@example.com', 'Admin');

exit;
```


## AWS 

```sh
aws configure --profile dev

export AWS_PROFILE=dev
#or
aws s3 ls --profile dev

# verify active profile
aws sts get-caller-identity
```


### EKS
```sh
eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --nodes 2 \
  --node-type t3.small \
  --managed \
  --spot


```

Now when you run `kubectl get nodes` this is the flow:
```md
kubectl get nodes
        ↓
Read ~/.kube/config
        ↓
Run:
aws eks get-token
        ↓
AWS CLI uses IAM credentials
        ↓
STS creates temporary token
        ↓
kubectl calls EKS API endpoint
        ↓
EKS validates token
        ↓
API server returns node objects
```


### ECR
```sh
aws ecr create-repository \
  --repository-name $ECR_REPO \
  --region $AWS_REGION


aws ecr get-login-password --region $AWS_REGION | \
docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

docker build -t $ECR_REPO .

IMG_TAG=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:v1
docker tag $ECR_REPO:latest $IMG_TAG
docker push $IMG_TAG
```


### PVC

```sh
aws iam list-open-id-connect-providers

# associate IAM OIDC provider first:
eksctl utils associate-iam-oidc-provider \
  --region $AWS_REGION \
  --cluster $CLUSTER_NAME \
  --approve


# Error: unable to create iamserviceaccount(s) without IAM OIDC provider enabled
ROLE_NAME=AmazonEKS_EBS_CSI_DriverRole

eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster $CLUSTER_NAME \
  --role-name $ROLE_NAME \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve


eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster $CLUSTER_NAME \
  --region $AWS_REGION \
  --service-account-role-arn arn:aws:iam::$ACCOUNT_ID:role/$ROLE_NAME \
  --force

# verify
kubectl get csidrivers
# NAME                 ATTACHREQUIRED   PODINFOONMOUNT
# ebs.csi.aws.com      true             false

eksctl get addon --cluster $CLUSTER_NAME --region $AWS_REGION
```

```sh
k apply -f sc.yml
k apply -f pvc.yml
k apply -f mysql-deploy-svc.yml
```

### clean up
```sh
eksctl delete cluster --name $CLUSTER_NAME --region $AWS_REGION
```
