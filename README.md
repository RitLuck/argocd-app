# A simple app using ArgoCD with GitHub Actions

## Step 1: Create Your GitHub Repository Structure

Create a repository with this structure: 

```
argocd-app/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── my-app/
│   ├── deployment.yaml
│   └── service.yaml
└── README.md
```

## Step 2: Create Kubernetes Manifests

my-app/deployment.yaml: 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21  # We'll update this version via GitHub Actions
        ports:
        - containerPort: 80
```

my-app/service.yaml: 

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
```

## Step 3: Create GitHub Actions Workflow 

```
name: Update Nginx Version

on:
  schedule:
    - cron: '0 2 * * *'
  workflow_dispatch:

jobs:
  update-nginx:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          token: ${{ secrets.PAT }}

      - name: Update Nginx image to latest
        run: |
          # Install required tools
          sudo apt-get update && sudo apt-get install -y jq curl
          
          # Get the latest nginx version with better error handling
          echo "Fetching latest nginx version..."
          
          # Method 1: Try Docker Hub API
          latest_version=$(curl -s https://registry.hub.docker.com/v2/nginx/tags/list | jq -r '.tags[]' | grep -E '^[0-9]+\.[0-9]+$' | sort -V | tail -1)
          
          # If Method 1 fails, try Method 2
          if [ -z "$latest_version" ]; then
            echo "Method 1 failed, trying Method 2..."
            latest_version=$(curl -s https://hub.docker.com/v2/repositories/library/nginx/tags/ | jq -r '.results[].name' | grep -E '^[0-9]+\.[0-9]+$' | sort -V | tail -1)
          fi
          
          # If both methods fail, use a fallback
          if [ -z "$latest_version" ]; then
            echo "Both methods failed, using fallback version"
            latest_version="1.23.1"
          fi
          
          echo "Latest nginx version: $latest_version"
          echo "NGINX_VERSION=$latest_version" >> $GITHUB_ENV
          
          # Check if deployment.yaml exists
          if [ ! -f "my-app/deployment.yaml" ]; then
            echo "Error: my-app/deployment.yaml not found!"
            exit 1
          fi
          
          # Show current content
          echo "Current deployment.yaml:"
          cat my-app/deployment.yaml
          
          # Update the deployment.yaml with the new version
          sed -i "s|image: nginx:.*|image: nginx:$latest_version|" my-app/deployment.yaml
          
          # Show updated content
          echo "Updated deployment.yaml:"
          cat my-app/deployment.yaml
          
          # Verify the change was made
          if grep -q "image: nginx:$latest_version" my-app/deployment.yaml; then
            echo "Successfully updated nginx version to $latest_version"
          else
            echo "Failed to update nginx version"
            exit 1
          fi

      - name: Commit and push changes
        run: |
          git config --global user.name 'GitHub Actions'
          git config --global user.email 'actions@github.com'
          git add my-app/deployment.yaml
          git status
          git diff --cached
          git commit -m "Update nginx to version ${{ env.NGINX_VERSION }}"
          git push origin main
```

## Step 4: Push Everything to GitHub 

```
git add .
git commit -m "Initial setup with ArgoCD manifests and GitHub Actions"
git push origin main
```

## Step 5: Set Up ArgoCD Application 


### Install ArgoCD

install ArgoCD in the new cluster

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Verify ArgoCD Installation

```
kubectl get pods -n argocd -w
```

```
NAME                                                READY   STATUS    RESTARTS        AGE
argocd-application-controller-0                     1/1     Running   5 (5m51s ago)   6h20m
argocd-applicationset-controller-655cc58ff8-8mwvj   1/1     Running   2 (6m50s ago)   6h21m
argocd-dex-server-7d9dfb4fb8-pwr8r                  1/1     Running   2 (6m51s ago)   6h21m
argocd-notifications-controller-6c6848bc4c-nglwg    1/1     Running   6 (6m18s ago)   6h21m
argocd-redis-656c79549c-2xrjb                       1/1     Running   2 (6m51s ago)   6h21m
argocd-repo-server-856b768fd9-p9sfp                 1/1     Running   5 (6m51s ago)   6h21m
argocd-server-99c485944-jz6z9                       1/1     Running   11 (118s ago)   6h20m
```


### Set Up Port Forwarding 
 
Set up port forwarding to access ArgoCD:

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Get Admin Password

```
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d
```

### Login to ArgoCD

```
argocd login localhost:8080 --insecure
```


### Create ArgoCD Application

Create your ArgoCD application:

```
argocd app create my-nginx \
  --repo https://github.com/RitLuck/argocd-app.git \
  --path my-app \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```


## Step 6: Enable Auto-Sync in ArgoCD 

```
argocd app set my-nginx --sync-policy automated
```

## Step 7: Initial Sync 

```
argocd app sync my-nginx
```

## Step 8: Verify Deployment 

```
kubectl get pods -l app=nginx
kubectl get service nginx
minikube service nginx --url
```

