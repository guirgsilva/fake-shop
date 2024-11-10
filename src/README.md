```markdown
# Fake Shop - Flask Application with PostgreSQL on EKS

This project deploys a Flask application with a PostgreSQL database on an AWS EKS (Elastic Kubernetes Service) cluster using Helm for easy configuration and management.

## Prerequisites

- **Docker** installed for building and pushing images.
- **Kubernetes (kubectl)** and **Helm** installed for deploying and managing resources on EKS.
- **AWS CLI** configured to access your EKS cluster.

## Getting Started

### 1. Build and Push Docker Image

1. Navigate to the directory containing your Dockerfile.
2. Build the Docker image:
   ```bash
   docker build -t guilhermergsilva/fake-shop:latest .
   ```
3. Log in to Docker Hub:
   ```bash
   docker login
   ```
4. Push the Docker image:
   ```bash
   docker push guilhermergsilva/fake-shop:latest
   ```

### 2. Kubernetes YAML Files for Flask App and PostgreSQL

1. **Create Secret for PostgreSQL Credentials**:
   - File: `postgres-secret.yaml`
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: postgres-secret
   type: Opaque
   data:
     POSTGRES_USER: ZWNvbW1lcmNl    # Base64 encoded 'ecommerce'
     POSTGRES_PASSWORD: UGcxMjM0    # Base64 encoded 'Pg1234'
     POSTGRES_DB: ZWNvbW1lcmNl      # Base64 encoded 'ecommerce'
   ```

2. **ConfigMap for Flask App Environment Variables**:
   - File: `flask-configmap.yaml`
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: flask-config
   data:
     DB_HOST: "postgres-service"
     DB_PORT: "5432"
     FLASK_ENV: "production"
     DATABASE_URL: "postgresql://$(DB_USER):$(DB_PASSWORD)@$(DB_HOST):$(DB_PORT)/$(DB_NAME)"
   ```

3. **Deployment and Service for PostgreSQL**:
   - File: `postgres-deployment.yaml`
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: postgres
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: postgres
     template:
       metadata:
         labels:
           app: postgres
       spec:
         containers:
         - name: postgres
           image: postgres:latest
           env:
           - name: POSTGRES_USER
             valueFrom:
               secretKeyRef:
                 name: postgres-secret
                 key: POSTGRES_USER
           - name: POSTGRES_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: postgres-secret
                 key: POSTGRES_PASSWORD
           - name: POSTGRES_DB
             valueFrom:
               secretKeyRef:
                 name: postgres-secret
                 key: POSTGRES_DB
           ports:
           - containerPort: 5432
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: postgres-service
   spec:
     selector:
       app: postgres
     ports:
     - protocol: TCP
       port: 5432
       targetPort: 5432
   ```

4. **Deployment and Service for Flask Application**:
   - File: `flask-deployment.yaml`
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: flask-app
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: flask-app
     template:
       metadata:
         labels:
           app: flask-app
       spec:
         containers:
         - name: flask-app
           image: guilhermergsilva/fake-shop:latest
           envFrom:
           - configMapRef:
               name: flask-config
           - secretRef:
               name: postgres-secret
           ports:
           - containerPort: 5000
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: flask-app-service
   spec:
     type: LoadBalancer
     selector:
       app: flask-app
     ports:
     - protocol: TCP
       port: 80
       targetPort: 5000
   ```

5. **Apply the Kubernetes Configurations**:
   ```bash
   kubectl apply -f postgres-secret.yaml
   kubectl apply -f flask-configmap.yaml
   kubectl apply -f postgres-deployment.yaml
   kubectl apply -f flask-deployment.yaml
   ```

### 3. Automate Database Migrations

1. Update `entrypoint.sh` to apply migrations automatically:
   ```bash
   #!/bin/bash
   export FLASK_APP=index.py
   flask db upgrade
   exec gunicorn -w 4 -b 0.0.0.0:5000 index:app
   ```

2. Update Dockerfile:
   ```Dockerfile
   FROM python:3.11.0
   EXPOSE 5000
   COPY requirements.txt .
   RUN python -m pip install -r requirements.txt
   WORKDIR /app
   COPY . /app
   RUN chmod +x /app/entrypoint.sh
   ENV PROMETHEUS_MULTIPROC_DIR=/tmp/metrics
   ENV FLASK_APP=index.py
   RUN mkdir -p /tmp/metrics 
   CMD ["/app/entrypoint.sh"]
   ```

### 4. Create a Helm Chart

1. **Initialize a Helm Chart**:
   ```bash
   helm create fake-shop
   ```

2. **Move Kubernetes YAML Files to Helm Chart Templates**:
   - Place `flask-deployment.yaml`, `postgres-deployment.yaml`, `flask-configmap.yaml`, and `postgres-secret.yaml` in `fake-shop/templates`, adjusting them to use Helm’s templating syntax (e.g., `{{ .Release.Name }}`).

3. **Update `values.yaml`**:
   ```yaml
   flaskApp:
     replicaCount: 2
     image: guilhermergsilva/fake-shop:latest

   postgres:
     user: "ecommerce"
     password: "Pg1234"
     db: "ecommerce"
   ```

4. **Install the Helm Chart**:
   ```bash
   helm install fake-shop .
   ```

### 5. Verify and Access the Application

1. Check the status of all resources:
   ```bash
   kubectl get all
   ```
2. Access the application using the external IP of `flask-app-service`.

---
