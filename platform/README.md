## GKE Cluster Installation with RunAI components

Google Kubernetes Engine (GKE) Cluster with GCS FUSE CSI Driver and GPU Node Pool (H200)

This document outlines the steps to create a Google Kubernetes Engine (GKE) cluster with the GCS FUSE CSI driver enabled and an additional node pool configured for GPU workloads using the h200 (or equivalent A3 series) machine type with the latest NVIDIA drivers.

### Prerequisites:

- You have a Google Cloud project.
- You have the gcloud command-line tool installed and configured to connect to your Google Cloud project.

Step 1: Enable the GKE API (if you haven't already)

```shell
gcloud services enable container.googleapis.com
```

Step 2: Set your default project, zone, and region (optional, but recommended)

Replace YOUR_PROJECT_ID, YOUR_ZONE, and YOUR_REGION with your actual values. Ensure the zone you choose supports the a3-highgpu-8g machine type (or the specific machine type offering H200 GPUs when available).

```shell

gcloud config set project YOUR_PROJECT_ID
gcloud config set compute/zone YOUR_ZONE
gcloud config set compute/region YOUR_REGION
```

Step 3: Create the GKE cluster with the gcsfuse-csi-driver addon and the initial CPU node pool

Replace YOUR_CLUSTER_NAME with your desired name for the cluster and adjust other parameters as needed.

```shell
gcloud container clusters create YOUR_CLUSTER_NAME \
    --addons GCSFuseCsiDriver \
    --region <YOUR_REGION>\
    --num-nodes 3 \
    --machine-type e2-medium \
    --workload-pool=<YOUR_PROJECT_ID>.svc.id.goog
```

Step 4: Add the GPU node pool with the h200 (A3) machine type and NVIDIA drivers

Replace gpu-pool with your desired name for the GPU node pool. Adjust the --node-count and other parameters as needed. Ensure the zone supports the chosen machine type and GPU accelerators.

Locations for H100

NAME ZONE DESCRIPTION
nvidia-h100-80gb us-east4-a NVIDIA H100 80GB
nvidia-h100-80gb us-east4-b NVIDIA H100 80GB
nvidia-h100-80gb us-east4-c NVIDIA H100 80GB
nvidia-h100-80gb us-east7-b NVIDIA H100 80GB
nvidia-h100-80gb us-east5-a NVIDIA H100 80GB

```shell
gcloud container node-pools create gpu-pool \
    --cluster=YOUR_CLUSTER_NAME \
    --project=PROJECT_ID \
    --accelerator type=nvidia-h100-80gb,count=8,gpu-driver-version=latest  \
    --location=REGION \
    --node-locations=<REGION-a> \
    --machine-type=a3-highgpu-8g \
    --node-count=1 \
```

Locations for h200

NAME ZONE DESCRIPTION
nvidia-h200-141gb us-central1-b NVIDIA H200 141GB
nvidia-h200-141gb europe-west1-b NVIDIA H200 141GB
nvidia-h200-141gb us-west1-c NVIDIA H200 141GB
nvidia-h200-141gb us-east4-b NVIDIA H200 141GB
nvidia-h200-141gb us-east7-c NVIDIA H200 141GB
nvidia-h200-141gb us-east5-a NVIDIA H200 141GB

```shell
gcloud container node-pools create gpu-pool \
    --cluster=YOUR_CLUSTER_NAME \
    --project=PROJECT_ID \
    --accelerator type=nvidia-h200-141gb,count=8,gpu-driver-version=latest  \
    --location=REGION \
    --node-locations=REGION-a \
    --machine-type=a3-highgpu-8g \
    --node-count=1 \
```

Note: The h200 machine type is part of the A3 series. Update the --machine-type and --accelerator flags based on the actual H100/H200 configuration and availability in your chosen zone.

Important Considerations:

Verify the availability of the h200 (or its equivalent A3 series) machine type and GPU accelerators in your chosen zone.
GKE manages NVIDIA driver compatibility. Rely on GKE's mechanisms for driver management.
Test any manual upgrades in a non-production environment first.

Step 5: Get the credentials for your new cluster

```shell

gcloud container clusters get-credentials YOUR_CLUSTER_NAME --region YOUR_REGION
```

Step 6: Ensure the Latest NVIDIA Drivers are Enabled on the GPU Node Pool

GKE automatically manages NVIDIA drivers on GPU node pools. The recommended approach is to ensure node auto-upgrade is enabled.

Check if Node Auto-Upgrade is Enabled:

```shell
gcloud container node-pools describe gpu-pool \
    --cluster=YOUR_CLUSTER_NAME \
    --location=YOUR_ZONE \
    --format="value(management.autoUpgrade)"
```

Enable Node Auto-Upgrade (if disabled):

```shell
gcloud container node-pools update gpu-pool \
    --cluster=YOUR_CLUSTER_NAME \
    --location=YOUR_ZONE \
    --enable-autoupgrade
```

(Optional) Enable NVIDIA GPU Driver Installer Addon (for more control):

```shell
gcloud container node-pools update gpu-pool \
    --cluster=YOUR_CLUSTER_NAME \
    --location=YOUR_ZONE \
    --enable-gpu-driver-installation
(Optional) Trigger a Node Pool Upgrade (to expedite driver updates):
```

```shell
gcloud container node-pools upgrade gpu-pool \
    --cluster=YOUR_CLUSTER_NAME \
    --location=YOUR_ZONE \
    --node-version=latest
```

### Persistent Volumes for GKE Cluster

The manifest `sample-persistent-volume.yaml` describes a request for a disk with 30 gibibytes (GiB) of storage whose access mode allows it to be mounted as read-write by a single node. It also creates a Pod that consumes the PersistentVolumeClaim as a volume.

When you create this PersistentVolumeClaim object, Kubernetes dynamically creates a corresponding PersistentVolume object.

Because the storage class standard-rwo uses volume binding mode WaitForFirstConsumer, the PersistentVolume will not be created until a Pod is scheduled to consume the volume.

```shell
kubectl apply -f sample-persistent-volume.yaml
```

```shell
kubectl get pv
```

### Mounting a GCS Bucket

1. Create a GCS bucket (if you haven't already).

```shell
gsutil mb -p <YOUR_PROJECT_ID> gs://data-bucket34 # Change to your bucket-name
```

2. Mount the bucket using the gcsfuse-csi-driver addon.

Create a namespace

```shell
kubectl create namespace app-gcs-fuse
```

Create a service account

```shell
kubectl create serviceaccount gcs-sa \
    --namespace app-gcs-fuse
```

Grant roles to the service account to access GCS fuse bucket

roles/storage.objectUser or roles/storage.objectViewer

```shell
gcloud storage buckets add-iam-policy-binding gs://BUCKET_NAME \
    --member "principal://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/PROJECT_ID.svc.id.goog/subject/ns/app-gcs-fuse/sa/gcs-sa" \
    --role "ROLE_NAME"
```

```shell
kubectl apply -f manifests/gcs-fuse.yaml -n app-gcs-fuse
```

### Anywhere cache

Create anywhere cache for the storage bucket
In this example we are using the storage bucket gs://data-bucket34

```shell
gcloud storage buckets anywhere-caches create gs://data-bucket34 us-east1-b us-east1-c
```

```shell
gcloud storage buckets anywhere-caches list gs://data-bucket34
```

### Install run-ai components

### Pre-requistes

1. Runai control plane requires a default storage class for the runai-backend namespace.
   Check you default storage class for the GKE cluster.

2. Apply token registry-creds to the cluster .

Note: Apply your registry-creds as per the instruction provided by Nvidia
Set the `TOKEN` provided by the NVIDIA team

```shell
TOKEN= ''
```

```shell
kubectl create namespace runai-backend
kubectl create secret docker-registry runai-reg-creds --docker-server=https://runai.jfrog.io --docker-username=self-hosted-image-puller-prod --docker-password=$TOKEN --docker-email=support@run.ai --namespace=runai-backend
```

(Optional) Use below commands if you have the registry creds stored in the yaml file

```shell
kubectl create namespace runai-backend
kubectl apply -f runai-reg-creds.yaml
```

### Install the Control Plane

```shell
helm repo add runai-backend https://runai.jfrog.io/artifactory/cp-charts-prod
helm repo update
helm upgrade -i runai-backend -n runai-backend runai-backend/control-plane --version "~2.20.0" \
    --set global.domain=<Domain>  #
```

You would see run-ai components being spinned up in the runai-backend namespace.

```shell
 kubectl get pods -n runai-backend
NAME                                                   READY   STATUS    RESTARTS        AGE
keycloak-0                                             1/1     Running   0               4m1s
runai-backend-assets-service-69757fb9b-9kpwk           1/1     Running   0               4m8s
runai-backend-audit-service-64656cb78d-bd6vd           1/1     Running   0               4m8s
runai-backend-authorization-68c7dd5c9c-fvzf7           1/1     Running   0               4m7s
runai-backend-backend-7bd5c44dbc-ljfdl                 1/1     Running   1 (2m18s ago)   4m7s
runai-backend-cli-exposer-54c987bfb-xz5mt              1/1     Running   0               4m7s
runai-backend-cluster-service-5c6654c44d-9fqt6         1/1     Running   0               4m6s
runai-backend-datavolumes-6565f68bb7-p6sf2             1/1     Running   0               4m6s
runai-backend-frontend-7847d46d75-rxvzc                1/1     Running   0               4m6s
runai-backend-grafana-5f79564466-zxb6v                 2/2     Running   0               4m9s
runai-backend-identity-manager-75c4cb4c85-m4wmc        1/1     Running   0               4m5s
runai-backend-k8s-objects-tracker-77b7d6ddcc-wpkxr     1/1     Running   0               4m5s
runai-backend-metrics-service-7695f654fb-9tphz         1/1     Running   0               4m5s
runai-backend-notifications-proxy-64fcdb76b5-wvjml     1/1     Running   0               4m5s
runai-backend-notifications-service-59c98bc5d4-rvrzq   1/1     Running   0               4m9s
runai-backend-org-unit-helper-bc5769db9-h6jvm          1/1     Running   0               4m4s
runai-backend-org-unit-service-59896fd86d-6gml8        1/1     Running   0               4m4s
runai-backend-policy-service-7795d7ff47-q7wxv          1/1     Running   0               4m4s
runai-backend-postgresql-0                             1/1     Running   0               4m1s
runai-backend-redis-queue-master-0                     1/1     Running   0               4m1s
runai-backend-redoc-5cb54888dd-2trjd                   1/1     Running   0               4m4s
runai-backend-tenants-manager-6c68774f78-6v6lq         2/2     Running   0               4m3s
runai-backend-thanos-query-6d5bb46576-qpx24            1/1     Running   0               4m9s
runai-backend-thanos-receive-0                         1/1     Running   0               4m1s
runai-backend-traefik-5c8b5c845f-w8sfd                 1/1     Running   0               4m3s
runai-backend-workloads-5c8ffb647c-4chpj               1/1     Running   0               4m2s
runai-backend-workloads-helper-7dc686b4c4-5p5cv        1/1     Running   0               4m2s

```

```shell
kubectl get svc -n runai-backend
NAME                                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                               AGE
keycloak-headless                     ClusterIP   None             <none>        80/TCP                                17h
keycloak-http                         ClusterIP   34.118.239.156   <none>        80/TCP,8443/TCP                       17h
runai-backend-assets-service          ClusterIP   34.118.236.177   <none>        8080/TCP,9090/TCP                     17h
runai-backend-audit-service           ClusterIP   34.118.228.117   <none>        8080/TCP,9090/TCP                     17h
runai-backend-authorization           ClusterIP   34.118.226.169   <none>        8080/TCP,9090/TCP                     17h
runai-backend-backend                 ClusterIP   34.118.226.65    <none>        7000/TCP                              17h
runai-backend-cli-exposer             ClusterIP   34.118.235.180   <none>        8080/TCP,9090/TCP                     17h
runai-backend-cluster-service         ClusterIP   34.118.232.175   <none>        8080/TCP,9090/TCP                     17h
runai-backend-datavolumes             ClusterIP   34.118.232.145   <none>        8080/TCP,9090/TCP                     17h
runai-backend-frontend                ClusterIP   34.118.236.42    <none>        8080/TCP                              17h
runai-backend-grafana                 ClusterIP   34.118.229.30    <none>        80/TCP                                17h
runai-backend-identity-manager        ClusterIP   34.118.236.26    <none>        8080/TCP,9090/TCP                     17h
runai-backend-k8s-objects-tracker     ClusterIP   34.118.233.147   <none>        8080/TCP,9090/TCP                     17h
runai-backend-metrics-service         ClusterIP   34.118.239.34    <none>        8080/TCP,9090/TCP                     17h
runai-backend-notifications-proxy     ClusterIP   34.118.229.83    <none>        8080/TCP,9090/TCP                     17h
runai-backend-notifications-service   ClusterIP   34.118.225.198   <none>        5000/TCP,9093/TCP,9090/TCP,8080/TCP   17h
runai-backend-org-unit-service        ClusterIP   34.118.236.103   <none>        8080/TCP,9090/TCP                     17h
runai-backend-policy-service          ClusterIP   34.118.235.103   <none>        8080/TCP,9090/TCP                     17h
runai-backend-postgresql              ClusterIP   34.118.231.222   <none>        5432/TCP                              17h
runai-backend-postgresql-hl           ClusterIP   None             <none>        5432/TCP                              17h
runai-backend-redis-queue-master      ClusterIP   34.118.228.208   <none>        6379/TCP                              17h
runai-backend-redis-queue-master-hl   ClusterIP   None             <none>        6379/TCP                              17h
runai-backend-redoc                   ClusterIP   34.118.237.29    <none>        8080/TCP                              17h
runai-backend-tenants-manager         ClusterIP   34.118.228.245   <none>        8080/TCP,9090/TCP                     17h
runai-backend-thanos-query            ClusterIP   34.118.239.67    <none>        9090/TCP                              17h
runai-backend-thanos-query-grpc       ClusterIP   34.118.229.119   <none>        10901/TCP                             17h
runai-backend-thanos-receive          ClusterIP   34.118.232.91    <none>        10902/TCP,10901/TCP,19291/TCP         17h
runai-backend-traefik                 ClusterIP   34.118.236.138   <none>        8080/TCP,9100/TCP                     17h
runai-backend-workloads               ClusterIP   34.118.235.90    <none>        8080/TCP,9090/TCP                     17h
```

### Install the run-ai cluster
