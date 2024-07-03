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