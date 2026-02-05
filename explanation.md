## Kubernetes Deployment Plan for IP2 Application

The public URL of the frontend will be the external IP provisioned by the GKE `LoadBalancer` service.

Public URL (frontend): http://136.119.149.215

## Overview

This document describes the comprehensive process for preparing the **IP2 Application** (React frontend, Node.js backend, MongoDB) for deployment on Google Kubernetes Engine (GKE). It records architecture, design choices, and debugging steps taken to ensure a reliable production-ready deployment.

---

## 1. Application Structure

The application is composed of three main components that communicate within the Kubernetes cluster:

1. **Frontend**

	- A React.js single-page application packaged into a Docker image and pushed to Docker Hub as `snngure/yoloip2-frontend:v1.0`.
	- For this deployment the frontend bundle was temporarily patched in the running pod to point at the backend external IP `http://136.119.149.215` so the UI can POST products.

2. **Backend**

	- A Node.js REST API packaged into a Docker image and pushed to Docker Hub as `snngure/yoloip2-backend:v1.0`.
	- The MongoDB connection string is stored in a Kubernetes Secret (`mongodb-secret`) and provided to the backend as `MONGO_URL`.

3. **Database**

	- MongoDB is deployed inside the cluster using a `StatefulSet` with `volumeClaimTemplates`. On GKE this provisions Persistent Disks for stable storage so data persists when pods are rescheduled.

---

## 2. Why Kubernetes on GKE?

- **Scalability**: Deployments allow easy scaling of frontend and backend replicas. Rolling updates are supported out of the box.
- **Networking & Service Discovery**: Kubernetes DNS lets services communicate by name (e.g., `backend:5000`).
- **Managed Control Plane**: GKE manages the control plane, reducing operational overhead.
- **LoadBalancer**: GKE provisions a cloud load balancer for `Service(type=LoadBalancer)`, which provides a stable public IP for the frontend.

---

## 3. Kubernetes Manifests (high-level)

3.1 Frontend (Deployment + `LoadBalancer` Service)

- Deployment uses image `snngure/yoloip2-frontend:v1.0` and exposes port 80.
- Environment variable `REACT_APP_BACKEND_URL` is provided via a Secret named `frontend-config`.
- Service of type `LoadBalancer` exposes the frontend externally.

3.2 Backend (Deployment + `ClusterIP` Service)

- Deployment uses image `snngure/yoloip2-backend:v1.0` and exposes port 5000.
- The backend reads `MONGO_URL` from the `mongodb-secret`.
- Service type is `ClusterIP` to restrict external access; frontend reaches backend through internal DNS.

3.3 Database (StatefulSet + PVCs + Headless Service)

- A `StatefulSet` with `volumeClaimTemplates` is used for MongoDB to guarantee stable storage and pod identity.
- A headless Service (`clusterIP: None`) provides stable network identities for the StatefulSet replicas.

---

## 4. Debugging and Key Learnings

### Image tagging and Kubernetes image pulls

- Use immutable version tags (for example, `snngure/yoloip2-frontend:v1.0`) rather than `:latest`. Kubernetes will only pull a new image when the image reference in the manifest changes. Using unique tags guarantees that `kubectl apply` triggers a rollout.

### Service types and security

- Frontend uses `LoadBalancer` for public access; backend uses `ClusterIP` to keep it internal to the cluster. This reduces attack surface and enforces proper separation of concerns.

### Persistence

- Using `StatefulSet` + `volumeClaimTemplates` on GKE ensures Persistent Disks back database volumes so data survives pod deletion and rescheduling. Verify persistence by deleting a MongoDB pod and confirming application data remains.

---

## 5. Ansible + Vagrant (IP3) — local automation

This project also contains an Ansible + Vagrant setup used to provision a local VM and deploy the app for development/testing. The `Vagrantfile` provisions an Ubuntu VM and runs the Ansible `playbook.yaml`, which contains roles for installing Docker, cloning the repo, and starting the application containers. This local automation is intended for development and testing; the production deployment is targeted at GKE.

---

## 6. How to deploy (short)

1. Build and push images:
```bash
docker build -t snngure/yoloip2-backend:v1.0 ./backend
docker push snngure/yoloip2-backend:v1.0

docker build -t snngure/yoloip2-frontend:v1.0 ./client
docker push snngure/yoloip2-frontend:v1.0
```

2. Configure GCP and export variables:
```bash
export PROJECT=project-5a16ba60-1bd6-4e78-a9f
export CLUSTER=yoloip2-cluster
export ZONE=us-central1-a
```

3. Run `kubectl apply -f k8s/deployment.yaml` then get the frontend IP:
```bash
kubectl get svc 
```

---

## 7. What I've Learnt


- **K8s objects**: StatefulSet for DB, Deployments for backend/frontend, Services (ClusterIP and LoadBalancer) are used.
