To set up a local Kubernetes cluster using Minikube with Calico for network policy and MetalLB for load balancing, follow these steps. This setup will help you deploy and test the demo application locally.

### Step-by-Step Guide

### Prerequisites

- Install Minikube: Follow the [Minikube installation guide](https://minikube.sigs.k8s.io/docs/start/)
- Install kubectl: Follow the [kubectl installation guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

### Setup Minikube with Calico and MetalLB

#### 1. Start Minikube with Calico

Start Minikube with the Calico CNI plugin:

```bash
minikube start --network-plugin=cni --cni=calico
```

Verify that Calico is running:

```bash
kubectl get pods -n kube-system | grep calico
```

#### 2. Set Up MetalLB

1. **Install MetalLB**

   Apply the MetalLB manifests:

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml
   kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml
   ```

2. **Configure MetalLB**

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
   kubectl apply -f metallb-config.yaml
   ```
