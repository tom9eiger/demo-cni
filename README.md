# Kubernetes Load Balancing and Network Policy Demo

This project demonstrates a simple Kubernetes setup with a backend and frontend application. The backend is a Flask API, and the frontend is a Flask application that communicates with the backend. This setup showcases Kubernetes load balancing and network policies to control traffic between services.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Architecture](#architecture)
- [Setup](#setup)
  - [Clone the Repository](#clone-the-repository)
  - [Setup Minikub](#setup-minikube)
  - [Docker Images](#docker-images))
  - [Deploy to Kubernetes](#deploy-to-kubernetes)
- [Demo Kubernetes Services](#demo-kubernetes-services)
  - [Demo Service Plugin (kube-proxy)](#demo-service-plugin-kube-proxy)
  - [Demo Network Policy (CNI)](#demo-network-policy-cni)
- [Cleanup](#cleanup)
- [License](#license)
- [Acknowledgments](#acknowledgments)

## Prerequisites

- Docker installed locally
- GitHub account and repository
- Install Minikube: Follow the [Minikube installation guide](https://minikube.sigs.k8s.io/docs/start/)
- Install kubectl: Follow the [kubectl installation guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

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
### Setup Minikube

1. **Start Minikube with Calico**:

    Start Minikube with 1 Control Plane Node and 3 Worker Nodes  the Calico CNI plugin:

    ```bash
    minikube start --network-plugin=cni --cni=calico --nodes=4
    ```

    Verify that Calico is running:

    ```bash
    kubectl get pods -n kube-system | grep calico
    ```

3. **Install MetalLB**

    Apply the MetalLB manifests:

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml
    ```

4. **Configure MetalLB**

    Determine the IP address range that your Minikube cluster is using:

    ```bash
    minikube ip
    ```

    Let's assume your Minikube IP is `192.168.49.2`. Use an IP range in the same subnet for MetalLB (e.g., `192.168.49.30-192.168.49.40`).

    Create a MetalLB configuration file:

    ```yaml
    # metallb-config.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
        namespace: metallb-system
        name: config
    data:
        config: |
        address-pools:
        - name: default
            protocol: layer2
            addresses:
            - 192.168.49.30-192.168.49.40
    ```

    Apply the configuration:

    ```bash
    kubectl apply -f deployment/metallb-config.yaml
    ```

### Docker Images

1. **Build and Push Docker Image for backend**:

   Build and push the Docker image for the backend application:

   ```bash
   cd backend
   docker build -t ghcr.io/your-username/backend-demo:latest .
   docker push ghcr.io/your-username/backend-demo:latest
   cd ..
   ```
2. **Build and Push Docker Image for frontend**:

   Build and push the Docker image for the frontend application:

   ```bash
   cd frontend
   docker build -t ghcr.io/your-username/frontend-demo:latest .
   docker push ghcr.io/your-username/frontend-demo:latest
   cd ..
   ```

### Deploy to Kubernetes

1. **Apply Namespace**:

   ```bash
   kubectl apply -f deployment/app-namespace.yaml
   ```
  
2. **Apply Backend Deployment**:

   ```bash
   kubectl apply -f deployment/backend-deployment.yaml
   kubectl apply -f deployment/backend-service.yaml
   ```

3. **Apply Frontend Deployment**:

   ```bash
   kubectl apply -f deployment/frontend-deployment.yaml
   kubectl apply -f deployment/frontend-service.yaml
   ```
## Demo Kubernetes Services

### Demo Service Plugin (kube-proxy)

1. **Get the Cluster IP and End Point Adresses**:

   ```bash
    # Login to minikube
    minikube ssh

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

1. **Apply Demo Pod**:

   ```bash
   kubectl apply -f deployment/demo-pod.yaml
   ```

2. **Test from demo pod without policy**:

   Execute a shell inside the demo pod to test connectivity to the backend service:

   ```bash
   kubectl exec -it demo-pod -n demo-namespace -- sh
   ```

   Inside the shell, try to connect to the backend service:

   ```sh
   wget -qO- http://flask-api-service.default.svc.cluster.local/api
   ```

   You should see that the connection is succesfull


3. **Apply Network Policy**:

   ```bash
   kubectl apply -f deployment/network-policy.yaml
   ```

4. **Test from Demo Pod with Policy**:

   Execute a shell inside the demo pod to test connectivity to the backend service:

   ```bash
   kubectl exec -it demo-pod -n demo-namespace -- sh
   ```

   Inside the shell, try to connect to the backend service:

   ```sh
   wget -qO- http://flask-api-service.default.svc.cluster.local/api
   ```

   You should see that the connection is refused or times out, demonstrating that the network policy is effectively blocking traffic from the demo pod to the backend service.

5. **Test from Frontend Pod**:

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

This `README.md` includes instructions for:

- Cloning the repository.
- Building and pushing Docker images for both the backend and frontend applications.
- Deploying the applications and network policy to a Kubernetes cluster.
- Verifying the network policy.
- Cleaning up resources.

This should provide a comprehensive guide for anyone looking to understand and deploy the project.


