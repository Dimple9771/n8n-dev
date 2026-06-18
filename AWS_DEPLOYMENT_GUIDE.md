# N8N AWS EKS Deployment Guide

## Prerequisites Checklist

Before deploying to AWS, ensure you have:

- [ ] AWS CLI configured and authenticated
- [ ] `kubectl` installed and configured to connect to your EKS cluster
- [ ] Helm 3.x installed
- [ ] AWS EKS cluster created and running
- [ ] Custom Docker image `dimple9771/n8n-custom:latest` pushed to a registry (Docker Hub, ECR, etc.)
- [ ] Domain `n8n.dimple.com` pointing to your AWS Load Balancer (or ready to configure DNS)
- [ ] AWS Load Balancer Controller installed on your EKS cluster

## Pre-Deployment Verification

### 1. Verify AWS Credentials
```powershell
aws sts get-caller-identity
```

### 2. Verify kubectl Connection
```powershell
kubectl cluster-info
kubectl get nodes
```

### 3. Verify Helm Chart
```powershell
cd "C:\Users\Dimple Ramisetti\OneDrive\Desktop\aws deploy\n8n-dev"
helm template ./helm/n8n-enterprise --namespace n8n-dev
helm lint ./helm/n8n-enterprise
```

### 4. Verify Docker Image Availability
```powershell
# Check if image exists in your registry
docker pull dimple9771/n8n-custom:latest
```

## Deployment Steps

### Step 1: Create Namespace (Optional - Helm will create if not exists)
```powershell
kubectl create namespace n8n-dev
```

### Step 2: Deploy Using Helm

#### Option A: Using ArgoCD (GitOps approach)
```powershell
# Apply the ArgoCD Application resource
kubectl apply -f argocd/n8n-app.yaml
```

#### Option B: Direct Helm Deployment (Recommended for testing)
```powershell
cd "C:\Users\Dimple Ramisetti\OneDrive\Desktop\aws deploy\n8n-dev"

# Deploy the Helm chart
helm install n8n-enterprise ./helm/n8n-enterprise `
  --namespace n8n-dev `
  --values ./helm/n8n-enterprise/values.yaml

# OR upgrade if already deployed
helm upgrade n8n-enterprise ./helm/n8n-enterprise `
  --namespace n8n-dev `
  --values ./helm/n8n-enterprise/values.yaml
```

### Step 3: Verify Deployment
```powershell
# Check all resources
kubectl get all -n n8n-dev

# Check pods status
kubectl get pods -n n8n-dev

# Check services
kubectl get svc -n n8n-dev

# Check ingress
kubectl get ingress -n n8n-dev

# View ingress details (get ALB URL)
kubectl describe ingress n8n-ingress -n n8n-dev
```

### Step 4: Monitor Deployment Progress
```powershell
# Watch pods coming up
kubectl get pods -n n8n-dev -w

# Check pod logs
kubectl logs -f deployment/n8n-main -n n8n-dev
kubectl logs -f deployment/postgres -n n8n-dev
```

### Step 5: Configure DNS
Once the ALB is created (visible in `kubectl describe ingress`):

1. Get the ALB hostname from the Ingress status
2. Create a CNAME record in Route53 or your DNS provider:
   - **From**: `n8n.dimple.com`
   - **To**: `<ALB-hostname>`
3. Wait for DNS propagation (5-30 minutes)

### Step 6: Access N8N
- Navigate to: `http://n8n.dimple.com` (or `https://` if you configure TLS)
- Create admin user and login

## Troubleshooting

### Pods Not Starting
```powershell
# Check pod events
kubectl describe pod <pod-name> -n n8n-dev

# Check pod logs
kubectl logs <pod-name> -n n8n-dev

# Check events in namespace
kubectl get events -n n8n-dev --sort-by='.lastTimestamp'
```

### Database Connection Issues
```powershell
# Check postgres pod
kubectl get pod -n n8n-dev | grep postgres

# Check postgres logs
kubectl logs -f statefulset/postgres -n n8n-dev

# Exec into postgres pod
kubectl exec -it postgres-0 -n n8n-dev -- psql -U n8n -d n8n
```

### Image Pull Errors
```powershell
# Check if image exists in registry
docker pull dimple9771/n8n-custom:latest

# If using ECR, create a secret for authentication
kubectl create secret docker-registry ecr-secret \
  --docker-server=<account_id>.dkr.ecr.<region>.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password) \
  -n n8n-dev

# Update values.yaml to reference this secret
```

### Ingress Not Creating ALB
```powershell
# Verify AWS Load Balancer Controller is installed
kubectl get deployment -A | grep aws-load-balancer-controller

# Check ingress controller logs
kubectl logs -f deployment/aws-load-balancer-controller -n kube-system
```

## Current Configuration

- **Namespace**: n8n-dev
- **Domain**: n8n.dimple.com
- **Image**: dimple9771/n8n-custom:latest
- **Database**: PostgreSQL (on-cluster)
- **Cache**: Redis (on-cluster)
- **Ingress**: AWS ALB
- **Execution Mode**: Queue (with Redis)
- **Runners**: Enabled (External mode)

## Rollback

If deployment fails:
```powershell
helm rollback n8n-enterprise -n n8n-dev
```

## Cleanup (if needed)

```powershell
# Delete the helm release
helm uninstall n8n-enterprise -n n8n-dev

# Delete namespace
kubectl delete namespace n8n-dev
```

## Next Steps

1. Verify all prerequisites are met
2. Run the verification commands in Step 1-3
3. Execute deployment using Option B
4. Monitor deployment progress (Step 4)
5. Configure DNS (Step 5)
6. Access and test the N8N application (Step 6)
