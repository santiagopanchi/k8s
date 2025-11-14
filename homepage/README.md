# Kubernetes Homepage Deployment

This repository contains Kubernetes manifests for deploying the homepage and API services.

## Overview

This setup deploys two main services:
- **hello-one**: Homepage service (Next.js) - accessible at `santiagopanchi.com` and `www.santiagopanchi.com`
- **hello-two**: Career Conversations Backend API - accessible at `api.santiagopanchi.com`

## Prerequisites

- Kubernetes cluster with kubectl configured
- Ingress NGINX controller installed
- cert-manager installed for SSL/TLS certificates
- DNS records configured to point to your ingress controller's LoadBalancer IP

## Required Secrets

Before deploying the applications, you need to create the following Kubernetes secrets:

### 1. DockerHub Secret (for hello-one)

The `hello-one` deployment uses a private Docker image and requires authentication to pull from DockerHub.

**Create the secret:**
```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<your-dockerhub-username> \
  --docker-password=<your-dockerhub-password-or-token> \
  --docker-email=<your-email> \
  --namespace=default
```

**Using DockerHub Access Token (Recommended):**
```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<your-dockerhub-username> \
  --docker-password=<your-access-token> \
  --docker-email=<your-email> \
  --namespace=default
```

**Note:** Replace the placeholders with your actual DockerHub credentials. Using an access token is more secure than using your password.

### 2. OpenAI API Key Secret (for hello-two)

The `hello-two` deployment requires an OpenAI API key to function properly.

**Create the secret:**
```bash
kubectl create secret generic openai-secret \
  --from-literal=api-key=<your-openai-api-key> \
  --namespace=default
```

**Note:** Replace `<your-openai-api-key>` with your actual OpenAI API key.

## Environment Variables

### hello-one (Homepage)
- No environment variables required
- Uses private Docker image: `santiagopanchi/homepage`
- Runs on port `3000` internally

### hello-two (Career Conversations Backend)
- **OPENAI_API_KEY**: Retrieved from the `openai-secret` Kubernetes secret
- Uses public Docker image: `santiagopanchi/career-conversations-backend`
- Runs on port `8000` internally

## Deployment Steps

### 1. Create Required Secrets

First, create all the required secrets as described above:
```bash
# Create DockerHub secret
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<your-username> \
  --docker-password=<your-token> \
  --docker-email=<your-email> \
  --namespace=default

# Create OpenAI secret
kubectl create secret generic openai-secret \
  --from-literal=api-key=<your-openai-api-key> \
  --namespace=default
```

### 2. Apply ClusterIssuer (if not already applied)

```bash
kubectl apply -f acme-issuer-prod.yaml
```

### 3. Deploy Services

```bash
# Deploy homepage service
kubectl apply -f hello-one.yaml

# Deploy API service
kubectl apply -f hello-two.yaml

# Deploy ingress
kubectl apply -f hello-app-ingress.yaml
```

### 4. Verify Deployment

Check that all pods are running:
```bash
kubectl get pods -l app=hello-one
kubectl get pods -l app=hello-two
```

Check ingress status:
```bash
kubectl get ingress hello-app-ingress
```

Check certificate status:
```bash
kubectl get certificate example-tls
```

## Updating Docker Images

When you update your Docker images, the deployments will automatically pull the latest version because `imagePullPolicy: Always` is set.

To force a restart and pull the latest images:
```bash
kubectl rollout restart deployment/hello-one
kubectl rollout restart deployment/hello-two
```

## Troubleshooting

### Pods are not starting

1. **Check pod status:**
   ```bash
   kubectl get pods -l app=hello-one
   kubectl get pods -l app=hello-two
   ```

2. **Check pod logs:**
   ```bash
   kubectl logs -l app=hello-one
   kubectl logs -l app=hello-two
   ```

3. **Check pod events:**
   ```bash
   kubectl describe pod <pod-name>
   ```

### Image pull errors (hello-one)

If you see `ImagePullBackOff` or `ErrImagePull` errors for hello-one:
- Verify the `dockerhub-secret` exists: `kubectl get secret dockerhub-secret`
- Check that your DockerHub credentials are correct
- Ensure the image `santiagopanchi/homepage` exists and is accessible with your credentials

### Application crashes (hello-two)

If hello-two pods are in `CrashLoopBackOff`:
- Verify the `openai-secret` exists: `kubectl get secret openai-secret`
- Check the pod logs for specific error messages
- Ensure the OpenAI API key is valid

### SSL/TLS Certificate Issues

If you see "connection is not private" errors:
- Check certificate status: `kubectl get certificate example-tls`
- Check certificate details: `kubectl describe certificate example-tls`
- Verify DNS records point to the correct LoadBalancer IP
- Ensure cert-manager is running: `kubectl get pods -n cert-manager`

### 502 Bad Gateway

If you're getting 502 errors:
- Verify pods are running and ready
- Check service endpoints: `kubectl get endpoints hello-one hello-two`
- Verify the container ports match the service targetPorts:
  - hello-one: containerPort 3000, service targetPort 3000
  - hello-two: containerPort 8000, service targetPort 8000
- Check ingress controller logs: `kubectl logs -n default -l app.kubernetes.io/name=ingress-nginx`

## Service Details

### hello-one (Homepage)
- **Image**: `santiagopanchi/homepage` (private)
- **Container Port**: 3000
- **Service Port**: 80
- **Replicas**: 3
- **Domains**: `santiagopanchi.com`, `www.santiagopanchi.com`

### hello-two (Career Conversations Backend)
- **Image**: `santiagopanchi/career-conversations-backend` (public)
- **Container Port**: 8000
- **Service Port**: 80
- **Replicas**: 3
- **Domain**: `api.santiagopanchi.com`
- **Environment Variables**: `OPENAI_API_KEY` (from secret)

## File Structure

```
.
├── README.md                 # This file
├── acme-issuer-prod.yaml     # cert-manager ClusterIssuer for Let's Encrypt
├── hello-one.yaml            # Homepage service deployment
├── hello-two.yaml            # API service deployment
└── hello-app-ingress.yaml    # Ingress configuration with TLS
```

## Notes

- All deployments use `imagePullPolicy: Always` to ensure the latest images are pulled
- SSL/TLS certificates are automatically managed by cert-manager using Let's Encrypt
- The ClusterIssuer email is set to `santiago762000@gmail.com` - update this in `acme-issuer-prod.yaml` if needed
- Secrets are created in the `default` namespace - adjust namespace if deploying elsewhere

