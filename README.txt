================================================================
CAPSTONE PROJECT - Python Flask REST App
DevOps Pipeline: GitHub → CodePipeline → ECR → Kubernetes
================================================================

AUTHOR: Arun
DATE: 2026

----------------------------------------------------------------
1. DESCRIPTION
----------------------------------------------------------------
This project Dockerizes a Python Flask REST application and 
deploys it using a complete DevOps pipeline:

- Source Control: GitHub
- CI/CD: AWS CodePipeline + CodeBuild
- Container Registry: Amazon ECR
- Orchestration: Kubernetes (kubeadm, multi-node)
- Database: MySQL 8.0
- Cache: Redis 7

The application has 7 endpoints:
  /               - Hello World
  /status         - Health check (returns 200 OK when ready)
  /palindrom/<t>  - Check palindrome, store in DB if true
  /admin          - Admin area
  /prepare-for-deploy    - Start deployment preparation
  /ready-for-deploy      - Check if ready for deployment
  /redis-hits     - Test Redis connection (increments counter)

----------------------------------------------------------------
2. PREREQUISITES
----------------------------------------------------------------
- AWS Account with ECR, CodePipeline, CodeBuild access
- GitHub repository with the application code
- 2 EC2 instances (t3.medium, Ubuntu 22.04) for Kubernetes
- Docker installed on EC2 instances
- kubectl installed on master node

----------------------------------------------------------------
3. EXECUTION STEPS - LOCAL (Docker Compose)
----------------------------------------------------------------

Step 1: Clone the repository
  git clone https://github.com/YOUR_USERNAME/capstone-devops.git
  cd capstone-devops

Step 2: Start all containers
  docker-compose up -d

Step 3: Verify all containers are running
  docker-compose ps

Step 4: Check application is ready (wait ~15 seconds)
  curl http://localhost:5000/status

Step 5: Access app instances
  App 1: http://localhost:5000
  App 2: http://localhost:5001

Step 6: Stop all containers
  docker-compose down

Step 7: Stop and remove volumes (full cleanup)
  docker-compose down -v

----------------------------------------------------------------
4. EXECUTION STEPS - KUBERNETES
----------------------------------------------------------------

Step 1: Apply all Kubernetes manifests
  kubectl apply -f k8s/namespace.yml
  kubectl apply -f k8s/configmap.yml
  kubectl apply -f k8s/mysql-pvc.yml
  kubectl apply -f k8s/mysql-deployment.yml
  kubectl apply -f k8s/redis-deployment.yml
  kubectl apply -f k8s/db-init-job.yml
  kubectl apply -f k8s/app-deployment.yml

Step 2: Verify all pods are running
  kubectl get pods -n capstone

Step 3: Get the NodePort to access the app
  kubectl get svc -n capstone
  Access: http://<WORKER_NODE_IP>:30000

Step 4: Check application status
  curl http://<WORKER_NODE_IP>:30000/status

----------------------------------------------------------------
5. TESTING STEPS
----------------------------------------------------------------

Test 1: Hello World endpoint
  curl http://localhost:5000/
  Expected: Hello, World!

Test 2: Status endpoint (app ready check)
  curl http://localhost:5000/status
  Expected: OK (status 200)
  Note: Wait 10 seconds after start for ready state

Test 3: Palindrome endpoint (stores in DB if palindrome)
  curl http://localhost:5000/palindrom/racecar
  Expected: Text is palindrom

  curl http://localhost:5000/palindrom/hello
  Expected: Text is not palindrom

Test 4: Redis hits endpoint
  curl http://localhost:5000/redis-hits
  Expected: Redis hits 1 (increments each call)
  Note: Should NOT return "No response from redis"

Test 5: Admin endpoint
  curl http://localhost:5000/admin
  Expected: admin area

Test 6: Prepare for deploy endpoint
  curl http://localhost:5000/prepare-for-deploy
  Expected: preparing

Test 7: Ready for deploy endpoint
  curl http://localhost:5000/ready-for-deploy
  Expected: Ready (after prepare-for-deploy called)

Test 8: Both app instances (Docker Compose)
  curl http://localhost:5000/redis-hits  (App 1)
  curl http://localhost:5001/redis-hits  (App 2)
  Note: Both should share Redis counter

----------------------------------------------------------------
6. CI/CD PIPELINE FLOW
----------------------------------------------------------------

1. Developer pushes code to GitHub
2. CodePipeline detects the change (webhook)
3. CodeBuild runs buildspec.yml:
   a. Logs into Amazon ECR
   b. Builds Docker image
   c. Tags with commit hash + latest
   d. Pushes to ECR
   e. Updates Kubernetes deployment (rolling update)
4. Kubernetes pulls new image from ECR
5. Rolling update ensures zero downtime
6. New version is live!

----------------------------------------------------------------
7. ENVIRONMENT VARIABLES
----------------------------------------------------------------

Variable          Default         Description
APP_HOST          0.0.0.0         Flask host (use 0.0.0.0 in Docker)
APP_PORT          5000            Flask port
REDIS_HOST        redis           Redis container hostname
REDIS_PORT        6379            Redis port
DATABASE_URI      (required)      MySQL connection string

DATABASE_URI format:
mysql+pymysql://appuser:apppass@mysql-service:3306/capstonedb

----------------------------------------------------------------
8. DOCKER IMAGES
----------------------------------------------------------------

Application image: Built from Dockerfile
  - Base: python:3.11-slim
  - Installs: gcc, mysqlclient, pip packages
  - Exposes: port 5000
  - Health check: GET /status

MySQL image: mysql:8.0 (official)
Redis image: redis:7-alpine (official, lightweight)

----------------------------------------------------------------
9. AUTO-RESTART POLICY
----------------------------------------------------------------

Docker Compose: restart: unless-stopped
  - Containers automatically restart on failure
  - Only stop if manually stopped

Kubernetes: restartPolicy: Always
  - Pods automatically restart on failure
  - Deployment maintains desired replica count (2)

----------------------------------------------------------------
10. ISSUES FACED AND SOLUTIONS
----------------------------------------------------------------

Issue 1: App not ready immediately
  Cause: App has 10-second startup delay (is_ready() function)
  Solution: Added healthcheck with start-period of 15 seconds
  depends_on: condition: service_healthy for DB and Redis

Issue 2: MySQL not ready when app starts
  Cause: MySQL takes 30+ seconds to initialize
  Solution: Used healthcheck + depends_on with condition

Issue 3: Database tables not created automatically
  Cause: SQLAlchemy doesn't auto-create tables
  Solution: Added db-init service/job that runs create_db.py

================================================================
END OF README
================================================================
