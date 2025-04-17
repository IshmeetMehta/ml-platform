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

```shell
gcloud container node-pools create gpu-pool \
    --cluster=YOUR_CLUSTER_NAME \
    --project=PROJECT_ID \
    --accelerator type=nvidia-h100-80gb,count=8,gpu-driver-version=latest  \
    --location=REGION \
    --node-locations=REGION-a \
    --machine-type=a3-highgpu-8g \
    --node-count=1 \
```

Note: The h200 machine type is likely part of the A3 series. The example uses a3-highgpu-8g which features NVIDIA H100 GPUs. Update the --machine-type and --accelerator flags based on the actual H200 configuration and availability in your chosen zone.

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

Next Steps (Example - Mounting a GCS Bucket):

After creating the cluster, you can deploy workloads that utilize the GCS FUSE CSI driver to mount GCS buckets. Here are example manifests:

PersistentVolumeClaim (PVC) - gcs-pvc.yaml:

YAML

```shell
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gcs-volume-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: gcsfuse
  volumeMode: Filesystem
Pod Definition - gcs-pod.yaml:
```

YAML

```shell
apiVersion: v1
kind: Pod
metadata:
  name: gcs-pod
spec:
  containers:
    - name: my-container
      image: ubuntu:latest
      command: ["sleep", "infinity"]
      volumeMounts:
        - name: gcs-volume
          mountPath: /mnt/gcs
  volumes:
    - name: gcs-volume
      persistentVolumeClaim:
        claimName: gcs-volume-claim
```

Apply these manifests using kubectl apply -f <filename>.yaml.

Important Considerations:

Verify the availability of the h200 (or its equivalent A3 series) machine type and GPU accelerators in your chosen zone.
GKE manages NVIDIA driver compatibility. Rely on GKE's mechanisms for driver management.
Test any manual upgrades in a non-production environment first.

### Install run-ai components

### Pre-requistes

1.

```shell
kubectl apply -f manifests/storageclass-local.yaml
```

```shell
kubectl patch storageclass local-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

2. Apply token registry-creds to the cluster .

Note: Apply your registry-creds as per the instruction provided by Nvidia

```shell
TOKEN= ''
```

```shell
kubectl create namespace runai-backend
kubectl create secret docker-registry runai-reg-creds --docker-server=https://runai.jfrog.io --docker-username=self-hosted-image-puller-prod --docker-password=$TOKEN --docker-email=support@run.ai --namespace=runai-backend
```

(Optional) Use below commands if you are story the registry creds in the yaml file

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
NAME                                                   READY   STATUS              RESTARTS   AGE
keycloak-0                                             0/1     Pending             0          25s
runai-backend-assets-service-8fc8fc676-tmc6j           0/1     Init:0/1            0          25s
runai-backend-audit-service-5769656dcb-68rbw           0/1     Init:0/1            0          21s
runai-backend-authorization-6d6788b4d5-rvlft           0/1     Init:0/1            0          24s
runai-backend-backend-74888f9-hw85q                    0/1     Init:0/2            0          24s
runai-backend-cli-exposer-57ff744bb8-gxz7z             1/1     Running             0          21s
runai-backend-cluster-service-98567c5b4-zgq7s          0/1     Pending             0          21s
runai-backend-datavolumes-69b7bfb5c5-vc4mb             0/1     Init:0/1            0          20s
runai-backend-frontend-766d7f89b7-q4m8k                0/1     ContainerCreating   0          20s
runai-backend-grafana-67fb7f88b-spqjh                  0/2     Init:0/1            0          25s
runai-backend-identity-manager-9f5dbf669-pn9pk         0/1     Init:0/1            0          25s
runai-backend-k8s-objects-tracker-7b8cc8cbfc-jrvgj     0/1     Init:0/1            0          20s
runai-backend-metrics-service-756b4b9d4d-hqg5m         0/1     Init:0/2            0          22s
runai-backend-notifications-proxy-95f7964b9-9d2tc      1/1     Running             0          24s
runai-backend-notifications-service-5d7c5574d8-k6tzh   0/1     Init:0/1            0          25s
runai-backend-org-unit-helper-5bb4cb657-5cxqv          0/1     Pending             0          20s
runai-backend-org-unit-service-86b78df47b-cgnf2        0/1     Init:0/1            0          24s
runai-backend-policy-service-57db4f7455-vk558          0/1     Pending             0          20s
runai-backend-postgresql-0                             0/1     Pending             0          25s
runai-backend-redis-queue-master-0                     0/1     Pending             0          25s
runai-backend-redoc-777c6f6c96-8q2q2                   1/1     Running             0          20s
runai-backend-tenants-manager-6dc567c474-2jzhm         0/2     Pending             0          19s
runai-backend-thanos-query-6bd5b84576-krm8j            0/1     Running             0          21s
runai-backend-thanos-receive-0                         0/1     Pending             0          25s
runai-backend-traefik-5974bccfbc-65cvd                 0/1     Pending             0          20s
runai-backend-workloads-7cfdff78b9-mcgfh               0/1     Init:0/1            0          25s
runai-backend-workloads-helper-6486784bcd-kzbdx        0/1     Init:0/1            0          20s
```

### Install the run-ai cluster
