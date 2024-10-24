# Multi-Container-Application-Deployment-with-Docker-Compose-and-Kubernetes
This project demonstrates the deployment of a multi-container application using Docker Compose and Kubernetes. The application consists of a frontend, backend, and database.
Containerize the Application
Build the Docker Images:

Ensure you have Dockerfiles for both the frontend and backend apps in your Git repositories.
For the backend:

Dockerfile
Copy code
FROM node:16
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "start"]
For the frontend:

Dockerfile
Copy code
FROM node:16
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "start"]
Push Docker Images to ECR:

Create a repository in Amazon ECR.
Authenticate Docker with ECR and push your images:
bash
Copy code
aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-west-2.amazonaws.com
docker build -t frontend-app ./frontend
docker tag frontend-app:latest <account-id>.dkr.ecr.us-west-2.amazonaws.com/frontend-app:latest
docker push <account-id>.dkr.ecr.us-west-2.amazonaws.com/frontend-app:latest
Repeat these steps for the backend image.

Step 3: Create Kubernetes Manifests for AWS EKS
You’ll now create Kubernetes manifests to deploy the frontend, backend, and database on AWS EKS.

Backend Kubernetes Manifest (backend-deployment.yaml):
yaml
Copy code
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: <account-id>.dkr.ecr.us-west-2.amazonaws.com/backend-app:latest
          ports:
            - containerPort: 5000
          env:
            - name: DATABASE_URL
              value: "postgres://db-service:5432/mydb"
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
  type: LoadBalancer
Frontend Kubernetes Manifest (frontend-deployment.yaml):
yaml
Copy code
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: <account-id>.dkr.ecr.us-west-2.amazonaws.com/frontend-app:latest
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
  type: LoadBalancer
Database:
You have two options:

Use Amazon RDS for a managed PostgreSQL database.
Deploy PostgreSQL as a container in Kubernetes (less recommended in production).
If using RDS, you don’t need to containerize the database, but you can connect your backend to RDS using the DATABASE_URL.

Step 4: Apply Kubernetes Manifests to EKS
Once your cluster is up and running, use kubectl to deploy the application:

bash
Copy code
kubectl apply -f backend-deployment.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f db-deployment.yaml # If using containerized DB, else configure RDS.
Check the status of your deployments:

bash
Copy code
kubectl get deployments
kubectl get services
Your frontend and backend should be accessible via their respective LoadBalancer URLs.

Step 5: Monitoring and Autoscaling (Bonus)
To enable autoscaling for your Kubernetes cluster on AWS EKS, you can set up Horizontal Pod Autoscaling (HPA). Here’s an example for scaling based on CPU utilization:

yaml
Copy code
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-deployment
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 50
Apply the autoscaler:

bash
Copy code
kubectl apply -f backend-hpa.yaml
Step 6: Rollback Strategy (Bonus)
In Kubernetes, you can rollback to a previous deployment version if something goes wrong. Use the following command to rollback:

bash
Copy code
kubectl rollout undo deployment/backend-deployment
