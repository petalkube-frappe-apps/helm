# Cleanup Commands

```bash
# Delete everything in erpnext namespace
kubectl delete all --all -n erpnext

# Delete PVCs
kubectl delete pvc --all -n erpnext

# Delete configmaps and secrets
kubectl delete configmap --all -n erpnext
kubectl delete secret --all -n erpnext

# Or delete the entire namespace (removes everything)
kubectl delete namespace erpnext
```

# Pre-Step 0: Setup Docker Buildx

```bash
# Install QEMU emulators for multi-arch builds
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

# Check if Docker buildx is properly configured
docker buildx ls
```

# Step 0: Build Docker Image

```bash
# Remove existing builders
docker buildx rm multiarch 2>/dev/null || true
docker buildx rm arm64-builder 2>/dev/null || true

# Create a new builder with proper driver
docker buildx create --name arm64-builder --driver docker-container --bootstrap --use

# Verify it's active
docker buildx ls

# Build for ARM64 and push (use --no-cache to ensure fresh build)
docker buildx build --platform linux/arm64 --build-arg FRAPPE_PATH=https://github.com/petalkube-frappe-apps/frappe --build-arg FRAPPE_BRANCH=develop --build-arg PYTHON_VERSION=3.11.6 --build-arg NODE_VERSION=20.19.2 --build-arg APPS_JSON_BASE64=$APPS_JSON_BASE64 --tag harrismajeed/erpnext-custom:develop --file images/production/Containerfile --push --no-cache .

# Verify the image is ARM64
docker buildx imagetools inspect harrismajeed/erpnext-custom:develop | grep -i platform

# OLD METHOD (DOESN'T WORK - for comparison only):
# docker build --platform linux/arm64 --build-arg=FRAPPE_PATH=https://github.com/petalkube-frappe-apps/frappe --build-arg=FRAPPE_BRANCH=develop --build-arg=PYTHON_VERSION=3.11.6 --build-arg=NODE_VERSION=20.19.2 --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 --tag=harrismajeed/erpnext-custom:develop --file=images/production/Containerfile .
# docker push harrismajeed/erpnext-custom:develop
```

# NFS Setup in Kubernetes

```bash
kubectl create namespace nfs

helm repo add nfs-ganesha-server-and-external-provisioner https://kubernetes-sigs.github.io/nfs-ganesha-server-and-external-provisioner

helm upgrade --install -n nfs in-cluster nfs-ganesha-server-and-external-provisioner/nfs-server-provisioner --set 'storageClass.mountOptions={vers=4.1}' --set persistence.enabled=true --set persistence.size=8Gi

# List pods and wait till all pods come up
kubectl get pods -n nfs -w
```

# Galera Cluster Setup

```bash
kubectl create namespace db

touch galera-values.yaml
nano galera-values.yaml

helm install mariadb-galera oci://registry-1.docker.io/bitnamicharts/mariadb-galera \
  -n db \
  -f galera-values.yaml \
  --set image.registry=docker.io \
  --set image.repository=bitnamilegacy/mariadb-galera \
  --set image.tag=10.6.16

# List pods and wait till all pods come up
kubectl get pods -n db -w
```

# Install ERPNext

```bash
kubectl create namespace erpnext

helm install frappe-bench ./erpnext -n erpnext -f custom_values.yaml

# List pods and wait till all pods come up
kubectl get pods -n erpnext -w
```

# Create New Site

```bash
helm template frappe-bench -n erpnext ./erpnext -f erpnext/custom_values_create_site.yaml -s templates/job-create-site.yaml > create-new-site-job.yaml

kubectl apply -f create-new-site-job.yaml
```

# remove default ingress if any 
kubectl delete ingress nginx-ingress -n default
kubectl delete deployment nginx-deployment -n default

# Apply IP-Based Ingress

```bash
kubectl apply -f erpnext/ipbased-erpaccess-ingress.yaml
```

# Build ERPNext Assets

```bash
# Get the gunicorn pod name
kubectl get pods -n erpnext | grep gunicorn

# Build ERPNext assets
kubectl exec -n erpnext frappe-bench-erpnext-gunicorn-75f7cfcd9b-btsr7 -- bench build --app erpnext

# Or exec into pod and build interactively
kubectl exec -it -n erpnext frappe-bench-erpnext-gunicorn-75f7cfcd9b-wtxl4 -- bash
# Then run: bench build --app erpnext
```

# Build Assets in Nginx Pod

```bash
# Get the nginx pod name
kubectl get pods -n erpnext | grep nginx

# Build ERPNext assets in nginx pod
kubectl exec -n erpnext frappe-bench-erpnext-nginx-b8ff7d696-m94mr -- bench build --app erpnext

# Or exec into nginx pod and build interactively
kubectl exec -it -n erpnext frappe-bench-erpnext-nginx-b8ff7d696-m94mr -- bash
# Then run: bench build --app erpnext
```

# Clear Cache and Build All Apps

```bash
# Clear cache in gunicorn pod
kubectl exec -n erpnext frappe-bench-erpnext-gunicorn-75f7cfcd9b-btsr7 -- bench --site all clear-cache

# Clear cache in nginx pod
kubectl exec -n erpnext frappe-bench-erpnext-nginx-b8ff7d696-m94mr -- bench --site all clear-cache

# Build all apps in gunicorn pod
kubectl exec -n erpnext frappe-bench-erpnext-gunicorn-75f7cfcd9b-btsr7 -- bench build --force

# Build all apps in nginx pod
kubectl exec -n erpnext frappe-bench-erpnext-nginx-b8ff7d696-m94mr -- bench build --force

clear cach and clear website cache for both the pods 
```