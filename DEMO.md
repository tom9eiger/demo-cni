# Kubernetes Load Balancing and Network Policy Demo

This project demonstrates a simple Kubernetes setup with a backend and frontend application. The backend is a Flask API, and the frontend is a Flask application that communicates with the backend. This setup showcases Kubernetes load balancing and network policies to control traffic between services.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Architecture](#architecture)
- [Setup](#setup)
  - [Clone the Repository](#clone-the-repository)
  - [Docker Images](#docker-images)
    - [Backend](#backend)
    - [Frontend](#frontend)
  - [Deploy to Kubernetes](#deploy-to-kubernetes)
    - [Backend Deployment](#backend-deployment)
    - [Frontend Deployment](#frontend-deployment)
    - [Network Policy](#network-policy)
    - [Demo Pod for Verification](#demo-pod-for-verification)
- [Verify Network Policy](#verify-network-policy)
- [Cleanup](#cleanup)
- [License](#license)
- [Acknowledgments](#acknowledgments)

## Prerequisites

- A running Kubernetes cluster
- `kubectl` configured to interact with your cluster
- Docker installed locally
- GitHub account and repository

## Architecture

- **Backend**: A Flask API that responds with a simple JSON message.
- **Frontend**: A Flask application that makes requests to the backend API and displays the responses.
- **Network Policy**: Restricts access to the backend service, allowing only the frontend to communicate with it.

## Setup

### Clone the Repository

Clone the repository to your local machine:

```bash
git clone https://github.com/your-username/your-repo.git
cd your-repo
```

### Docker Images

#### Backend

1. **Create `requirements.txt`**:

   Create a `requirements.txt` file in the `backend` directory:

   ```plaintext
   flask
   ```

2. **Create `Dockerfile`**:

   Create a `Dockerfile` in the `backend` directory:

   ```dockerfile
   FROM python:3.9-slim

   WORKDIR /app

   COPY requirements.txt requirements.txt
   RUN pip install -r requirements.txt

   COPY app.py /app

   CMD ["python", "app.py"]
   ```

3. **Build and Push Docker Image**:

   Build and push the Docker image for the backend application:

   ```bash
   cd backend
   docker build -t ghcr.io/your-username/backend-demo:latest .
   docker push ghcr.io/your-username/backend-demo:latest
   cd ..
   ```

#### Frontend

1. **Create `requirements.txt`**:

   Create a `requirements.txt` file in the `frontend` directory:

   ```plaintext
   flask
   requests
   ```

2. **Create `Dockerfile`**:

   Create a `Dockerfile` in the `frontend` directory:

   ```dockerfile
   FROM python:3.9-slim

   WORKDIR /app

   COPY requirements.txt requirements.txt
   RUN pip install -r requirements.txt

   COPY frontend.py /app

   CMD ["python", "frontend.py"]
   ```

3. **Build and Push Docker Image**:

   Build and push the Docker image for the frontend application:

   ```bash
   cd frontend
   docker build -t ghcr.io/your-username/frontend-demo:latest .
   docker push ghcr.io/your-username/frontend-demo:latest
   cd ..
   ```

### Deploy to Kubernetes

#### Backend Deployment

1. **Create `backend-deployment.yaml`**:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: flask-api
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: flask-api
     template:
       metadata:
         labels:
           app: flask-api
       spec:
         containers:
         - name: flask-api
           image: ghcr.io/your-username/backend-demo:latest
           ports:
           - containerPort: 80
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: flask-api-service
   spec:
     selector:
       app: flask-api
     ports:
       - protocol: TCP
         port: 80
         targetPort: 80
     type: ClusterIP
   ```

2. **Apply Backend Deployment**:

   ```bash
   kubectl apply -f deployment/backend-deployment.yaml
   kubectl apply -f deployment/backend-service.yaml
   ```

#### Frontend Deployment

1. **Create `frontend-deployment.yaml`**:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: frontend
   spec:
     replicas: 1
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
           image: ghcr.io/your-username/frontend-demo:latest
           ports:
           - containerPort: 80
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
         port: 80
         targetPort: 80
     type: LoadBalancer
   ```

2. **Apply Frontend Deployment**:

   ```bash
   kubectl apply -f deployment/frontend-deployment.yaml
   kubectl apply -f deployment/frontend-service.yaml
   ```
## Demo Kubernetes Services

### Demo Service Plugin (kube-proxy)

1. **Get the Cluster IP and End Point Adresses**:

   ```bash
    # Get Kubernetes Service
    docker@minikube:~$ kubectl get svc

    # Get the endpoints
    docker@minikube:~$ kubectl get ep
   ```
2. **Get the prerouting Rule for KUBE-SERVICE**:

   ```bash
    docker@minikube:~$ sudo iptables -t nat -L KUBE-SERVICES
    Chain KUBE-SERVICES (2 references)
    target     prot opt source               destination         
    KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  anywhere             10.96.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:https
    KUBE-SVC-TCOU7JCQXEZGVUNU  udp  --  anywhere             10.96.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:domain
    KUBE-SVC-ERIFXISQEP7F7OF4  tcp  --  anywhere             10.96.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:domain
    KUBE-SVC-JD5MR3NA4I4DYORP  tcp  --  anywhere             10.96.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
    KUBE-SVC-6YNYFUIKGNIA7RFX  tcp  --  anywhere             10.108.198.28        /* demo-cni-app/flask-api-service cluster IP */ tcp dpt:http
    KUBE-NODEPORTS  all  --  anywhere             anywhere             /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
   ```

3. **Get the NAT Rule for ClusterIP**:

   ```bash
    docker@minikube:~$ sudo iptables -t nat -L KUBE-SVC-6YNYFUIKGNIA7RFX
    Chain KUBE-SVC-6YNYFUIKGNIA7RFX (1 references)
    target     prot opt source               destination         
    KUBE-MARK-MASQ  tcp  -- !10.244.0.0/16        10.108.198.28        /* demo-cni-app/flask-api-service cluster IP */ tcp dpt:http
    KUBE-SEP-J7YQFRES3OILODCJ  all  --  anywhere             anywhere             /* demo-cni-app/flask-api-service -> 10.244.0.3:80 */
   ```
4. **Get the Rule for the Service End Point**:

   ```bash
    docker@minikube:~$ sudo iptables -t nat -L KUBE-SEP-J7YQFRES3OILODCJ
    Chain KUBE-SEP-J7YQFRES3OILODCJ (1 references)
    target     prot opt source               destination         
    KUBE-MARK-MASQ  all  --  10.244.0.3           anywhere             /* demo-cni-app/flask-api-service */
    DNAT       tcp  --  anywhere             anywhere             /* demo-cni-app/flask-api-service */ tcp to:10.244.0.3:80
   ```

### Demo Network Policy (CNI)
In this Demo we will work with Network Policy and how Network Policy effects traffic between Pods

1. **Create `demo-pod.yaml`**:

   ```yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: demo-namespace
   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: demo-pod
     namespace: demo-namespace
   spec:
     containers:
     - name: demo-container
       image: busybox
       command: ["sh", "-c", "sleep 3600"]
   ```

2. **Apply Demo Pod**:

   ```bash
   kubectl apply -f deployment/demo-pod.yaml
   ```

3. **Test from demo pod without policy**:

   Execute a shell inside the demo pod to test connectivity to the backend service:

   ```bash
   kubectl exec -it demo-pod -n demo-namespace -- sh
   ```

   Inside the shell, try to connect to the backend service:

   ```sh
   wget -qO- http://flask-api-service.default.svc.cluster.local/api
   ```

   You should see that the connection is succesfull

3. **Create `network-policy.yaml`**:

   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: allow-frontend-to-backend
   spec:
     podSelector:
       matchLabels:
         app: backend
     policyTypes:
     - Ingress
     ingress:
     - from:
         podSelector:
           matchLabels:
             app: frontend
       ports:
       - protocol: TCP
         port: 80
   ```

2. **Apply Network Policy**:

   ```bash
   kubectl apply -f deployment/network-policy.yaml
   ```



## Verify Network Policy

1. **Test from Demo Pod**:

   Execute a shell inside the demo pod to test connectivity to the backend service:

   ```bash
   kubectl exec -it demo-pod -n demo-namespace -- sh
   ```

   Inside the shell, try to connect to the backend service:

   ```sh
   wget -qO- http://flask-api-service.default.svc.cluster.local/api
   ```

   You should see that the connection is refused or times out, demonstrating that the network policy is effectively blocking traffic from the demo pod to the backend service.

2. **Test from Frontend Pod**:

   Similarly, you can verify that the frontend pod can communicate with the backend service.

   ```bash
   kubectl exec -it <frontend-pod-name> -- sh
   ```

   Inside the shell, try to connect to the backend service:

   ```sh
   wget -qO- http://flask-api-service.default.svc.cluster.local/api
   ```

   You should see a successful response from the backend service.

## Cleanup

To clean up the resources, delete the created Kubernetes resources and namespaces:

```bash
kubectl delete -f deployment/backend-deployment.yaml
kubectl delete -f deployment/backend-service.yaml
kubectl delete -f deployment/frontend-deployment.yaml
kubectl delete -f deployment/frontend-service.yaml
kubectl delete -f deployment/network-policy.yaml
kubectl delete namespace demo-namespace
```

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

## Acknowledgments

- [Flask](https://flask.palletsprojects.com/)
- [Kubernetes](https://kubernetes.io/)

This `DEMO.md` includes instructions for:

- Cloning the repository.
- Building and pushing Docker images for both the backend and frontend applications.
- Deploying the applications and network policy to a Kubernetes cluster.
- Verifying the network policy.
- Cleaning up resources.

This should provide a comprehensive guide for anyone looking to understand and deploy the project.