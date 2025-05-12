# vllm-llama3-hdml
Deploy a Llama3.3 70B model on GKE using vllm and Hyperdisk ML for faster loading

## Create the cluster
Set the needed environment variables
```
REGION=us-central1
ZONE=us-central1-c
PROJECT_ID=$(gcloud config get project)
PROJECT_NUMBER=$(gcloud projects describe $(gcloud config get-value project) --format="value(projectNumber)")
NAMESPACE=default
CLUSTER_NAME=ducdo-llama3-hdml
LOG_BUCKET_NAME=ducdo-vllm-log
VLLM_DISK_IMAGE_NAME=ducdo-vllm085
HF_TOKEN=<FILL_ME>
```
Prepare the secondary boot disk image
Create a Cloud Storage bucket to store the execution logs
```
gsutil mb -b on -l $REGION gs://$LOG_BUCKET_NAME
```
Build the **gke-disk-image-builder** tool
```
git clone https://github.com/GoogleCloudPlatform/ai-on-gke.git
cd ai-on-gke/tools/gke-disk-image-builder
go build -o cli ./cli
```
Prepare the secondary boot disk image
```
go run ./cli \
    --project-name=$PROJECT_ID \
    --image-name=$VLLM_DISK_IMAGE_NAME \
    --zone=$ZONE \
    --gcs-path=gs://$LOG_BUCKET_NAME \
    --disk-size-gb=100 \
    --container-image=docker.io/vllm/vllm-openai:v0.8.5
```
Create the cluster
```
gcloud container clusters create $CLUSTER_NAME \
    --location=$ZONE \
    --workload-pool=$PROJECT_ID.svc.id.goog \
    --num-nodes=1 \
    --machine-type=c3-standard-44 \
    --release-channel=rapid \
    --image-type="COS_CONTAINERD" \
    --enable-image-streaming
```
## Create the node pool
Create additional CPU nodepools if needed for the model downloader
```
gcloud container node-pools create c344 \
    --node-locations=$ZONE \
    --num-nodes=1 \
    --machine-type=c3-standard-44 \
    --cluster=$CLUSTER_NAME \
    --image-type="COS_CONTAINERD" \
    --location=$ZONE
```
Create the GPU node pool
```
gcloud container node-pools create a3-high4m \
    --location=$ZONE \
    --node-locations=$ZONE \
    --enable-autoscaling \
    --total-min-nodes=0 \
    --total-max-nodes=1 \
    --disk-type=pd-ssd \
    --machine-type=a3-highgpu-4g \
    --accelerator type=nvidia-h100-80gb,count=4,gpu-driver-version=LATEST  \
    --cluster=$CLUSTER_NAME \
    --spot \
    --image-type="COS_CONTAINERD" \
    --enable-image-streaming \
    --secondary-boot-disk=disk-image=projects/$PROJECT_ID/global/images/$VLLM_DISK_IMAGE_NAME,mode=CONTAINER_IMAGE_CACHE 
```
## Create the Hyperdisk ML

1. Create a StorageClass through hyperdisk-ml.yaml
1. Create a PersistentVolumeClaim through producer-pvc.yaml
1. Create a Kubernetes Job to populate the mounted Google Cloud Hyperdisk volume through producer-job.yaml
```
kubectl create secret generic hf-secret \
    --from-literal=hf_api_token=$HF_TOKEN\
    --dry-run=client -o yaml | kubectl apply -f -
```
4. Create PersistentVolume and PersistentVolumeClaim based from the recently created Hyperdisk ML disk, through hdml-static-pv.yaml.
## Create the deployment
### Deployment in vllm-deployment.yaml
### Run a proxy and test the endpoint

```
kubectl port-forward svc/llm-service 8080:8080
```
Open another tab

```
curl http://localhost:8080/v1/completions -H "Content-Type: application/json" -d '{
    "model": "meta-llama/Llama-3.3-70B-Instruct",
    "prompt": "Where can I find the best biryani in India?",    
    "max_tokens": 1024,
    "temperature": 0
}'
```

## Clean up

```
kubectl delete deployment vllm-deployment
kubectl delete service llm-service
gcloud container clusters delete $CLUSTER_NAME --region=$REGION
```

## Notes
1. This tutorial uses the Spot instance type which may cause delay in node pool scaling
2. Use the node pool name as the free text filtering in Log Explorer returned this log indicating resource quota limitation issue.
```
localizedMessage: [
  0: {
  locale: "en-US"
  message: "A a3-highgpu-4g VM instance with 4 nvidia-h100-80gb accelerator(s) and 8 local SSD(s) is currently unavailable in the us-central1-c zone. Consider trying your request in the us-central1-b, us-central1-a zone(s), which currently has capacity to accommodate your request. Alternatively, you can try your request again with a different VM hardware configuration or at a later time. For more information, see the troubleshooting documentation."
  }
]
```
3. C4 does not support Hyperdisk ML as of May'25, fallback to use c3 machine. Check the latest at https://cloud.google.com/compute/docs/machine-resource#machine_type_comparison
4. Llama 3.3 70B downloaded size is 263GB
```
ducdo@control-tower-25:~/workspaces/gke-vllm-hdml
$ gsutil du -sh gs://ducdo-llm-models/Llama-3.3-70B-Instruct
262.87 GiB   gs://ducdo-llm-models/Llama-3.3-70B-Instruct
```
1. Faced an issue while downloading the model
```
The node was low on resource: ephemeral-storage. Threshold quantity: 10120387530, available: 8773636Ki. Container copy was using 82930028Ki, request is 0, has larger consumption of ephemeral-storage.
```
This was potentially caused by the limited ephemeral settings in the job configurations.
This fixed it:
```
        env:
        - name: HF_HOME
          value: "/data/.hf_cache"
```
Apparently HF downloader uses a local ephemeral storage as cache, and would face storage limit issue while downloading large files (e.g. large models). The above config specifies large cache storage using the persistent disk itself.
