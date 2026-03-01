# Lab 1: Continuous Deployment Using GitHub Actions

## Overview

In this hands-on lab, you will learn how to deploy a Node.js application to a local Kubernetes cluster (Minikube) using GitHub Actions for Continuous Deployment.

### What is Minikube?

[Minikube](https://minikube.sigs.k8s.io/docs/start/) is a tool that makes it easy to run a Kubernetes cluster on your local machine. It is useful for testing and development purposes, allowing you to experiment with Kubernetes without access to a remote cluster.

Once installed, you can start a cluster by running:

```bash
minikube start
```

You can then interact with the cluster using `kubectl`, just as you would with a remote Kubernetes cluster — deploying and managing applications locally before promoting them to production.

### GitHub Actions Integration

This lab uses the [`medyagh/setup-minikube`](https://github.com/marketplace/actions/setup-minikube) action to install and start Minikube inside a GitHub Actions workflow, enabling automated deployment testing on every push.

---

## Prerequisites

- Docker (or a compatible container runtime) installed on your machine.
- A GitHub account with a new repository created for this lab.
- Push your local code to that repository before running the workflow.

---

## Project Structure

```
.
├── Dockerfile
├── package.json
├── server.js
├── k8s-node-app.yaml
└── .github/
    └── workflows/
        └── deploy-to-minikube-github-actions.yaml
```

---

## Setup Steps

### Step 1 — Create the `Dockerfile`

```dockerfile
FROM node:14
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
RUN npm install express
COPY . .
EXPOSE 3000
CMD [ "node", "server.js" ]
```

---

### Step 2 — Create `package.json`

```json
{
  "name": "docker_web_app",
  "version": "1.0.0",
  "description": "Node.js on Docker",
  "author": "First Last <first.last@example.com>",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.16.1"
  }
}
```

---

### Step 3 — Create `server.js`

```javascript
'use strict';

const express = require('express');

// Constants
const PORT = 3000;
const HOST = '0.0.0.0';

// App
const app = express();
app.get('/', (req, res) => {
  res.send('Hello World');
});

app.listen(PORT, HOST);
console.log(`Running on http://${HOST}:${PORT}`);
```

---

### Step 4 — Create `k8s-node-app.yaml`

```yaml
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nodejs-app
  namespace: default
  labels:
    app: nodejs-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nodejs-app
  template:
    metadata:
      labels:
        app: nodejs-app
    spec:
      containers:
      - name: nodejs-app
        image: "devopshint/node-app:latest"
        ports:
          - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: nodejs-app
  namespace: default
spec:
  selector:
    app: nodejs-app
  type: NodePort
  ports:
  - name: http
    targetPort: 3000
    port: 80
```

---

### Step 5 — Create the GitHub Actions Workflow

Create the file at `.github/workflows/deploy-to-minikube-github-actions.yaml`:

```yaml
name: Deploy to Minikube using GitHub Actions

on: [push]

jobs:
  job1:
    runs-on: ubuntu-latest
    name: Build Node.js Docker Image and deploy to Minikube
    steps:
      - uses: actions/checkout@v2

      - name: Start Minikube
        uses: medyagh/setup-minikube@master

      - name: Check cluster status
        run: kubectl get pods -A

      - name: Build Docker image
        run: |
          export SHELL=/bin/bash
          eval $(minikube -p minikube docker-env)
          docker build -f ./Dockerfile -t devopshint/node-app:latest .
          echo -n "Verifying images:"
          docker images

      - name: Deploy to Minikube
        run: kubectl apply -f k8s-node-app.yaml

      - name: Test service URLs
        run: |
          minikube service list
          minikube service nodejs-app --url
```

---

## Verification

After pushing your code to GitHub, navigate to the **Actions** tab in your repository to monitor the workflow run. A successful execution means your Node.js application was built, deployed to Minikube, and is accessible via the service URL.
