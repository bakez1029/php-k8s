php-k8s

Kubernetes deployment repository for the php-app.

This repository is deploy-only.
It does not build container images. It consumes versioned images pushed to GHCR by the php-poc repository.

Architecture Overview

This deployment uses a production-style multi-container Pod:

Browser
→ Service (ClusterIP)
→ Nginx container (port 8080)
→ PHP-FPM container (port 9000)
→ MySQL service (port 3306)

Components:

nginx – serves HTTP traffic

php-fpm – executes PHP application

MySQL – internal database (for local cluster)

Secret – provides .env configuration

initContainer – copies application files into a shared volume

This mirrors a real-world containerized PHP deployment pattern.

Repository Structure

helm/
php-app/
Chart.yaml
values.yaml
templates/

This repository contains:

Helm chart for deployment

Kubernetes templates

Environment configuration logic

It does NOT contain:

Application source code

Dockerfile

Secrets

Prerequisites

Docker

k3d

kubectl

Helm

Create cluster (if needed):

k3d cluster create php-lab --agents 2 -p "8081:80@loadbalancer"

Verify cluster:

kubectl get nodes

Database Setup (Local Dev Only)

Deploy MySQL inside the cluster:

kubectl create namespace apps

kubectl create deployment mysql --image=mysql:8.0 -n apps

kubectl set env deployment/mysql
MYSQL_ROOT_PASSWORD=rootpass
MYSQL_DATABASE=app
MYSQL_USER=app
MYSQL_PASSWORD=apppass
-n apps

kubectl expose deployment mysql
--port=3306
--name=mysql
-n apps

Verify:

kubectl get pods -n apps
kubectl get svc -n apps

You should see a Service named "mysql".

Environment Configuration

Create a local .env.local file (do NOT commit this):

DB_HOST=mysql
DB_NAME=app
DB_USER=app
DB_PASS=apppass

Create Kubernetes Secret:

kubectl create secret generic php-app-env
--from-file=.env=.env.local
-n apps

Deploy Application

helm upgrade --install php-app ./helm/php-app
-n apps
--create-namespace

Verify:

kubectl get pods -n apps

Access Application

Port forward:

kubectl port-forward svc/php-app 8083:80 -n apps

Open:

http://localhost:8083

If everything is configured correctly, the app should connect to MySQL and load successfully.

Upgrade Deployment

To deploy a new image version:

helm upgrade php-app ./helm/php-app
-n apps
--set php.image.tag=<new-tag>

Teardown

Remove application:

helm uninstall php-app -n apps

Delete namespace:

kubectl delete namespace apps

Delete cluster:

k3d cluster delete php-lab

Deployment Flow

This repository is part of a two-repo architecture:

php-poc → Builds and pushes Docker image to GHCR
php-k8s → Deploys that image to Kubernetes using Helm
