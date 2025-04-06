# vllm-llama3-hdml
Deploy a Llama3.3 70B model on GKE using vllm and Hyperdisk ML for faster loading

## Create the cluster
Set the needed environment variables
```
CLUSTER_NAME=ducdo-llama3-hdml
PROJECT_ID=gpu-launchpad-playground
REGION=us-central1
LOCATION=us-central1-c
LOG_BUCKET_NAME=ducdo-llama3-hdml
DISK_IMAGE_NAME=ducdo-llama3-hdml
```
Prepare the secondary boot disk image
Create a Cloud Storage bucket to store the execution logs
```
gsutil mb -l $REGION gs://$LOG_BUCKET_NAME
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
    --image-name=$DISK_IMAGE_NAME \
    --zone=$LOCATION \
    --gcs-path=gs://$LOG_BUCKET_NAME \
    --disk-size-gb=100 \
    --container-image=docker.io/vllm/vllm-openai:v0.7.3
```
Create the cluster
```
gcloud container clusters create $CLUSTER_NAME \
    --project=$PROJECT_ID \
    --region=$REGION \
    --workload-pool=$PROJECT_ID.svc.id.goog \
    --release-channel=rapid \
    --num-nodes=1 \
    --image-type="COS_CONTAINERD" \
    --enable-image-streaming
```
## Create the node pool
Set the needed environment variables
```
NODE_POOL_NAME=ducdo-a3-high4m
ZONE1=us-central1-a
ZONE2=us-central1-b
ZONE3=us-central1-c
```
Create the node pool
```
gcloud container node-pools create $NODE_POOL_NAME \
    --node-locations=$ZONE1,$ZONE2,$ZONE3 \
    --enable-autoscaling \
    --total-min-nodes=0 \
    --total-max-nodes=3 \
     --disk-type=pd-ssd \
    --machine-type=a3-highgpu-4g \
    --accelerator type=nvidia-h100-80gb,count=4,gpu-driver-version=LATEST  \
    --cluster=$CLUSTER_NAME \
    --spot \
    --image-type="COS_CONTAINERD" \
    --enable-image-streaming \
    --secondary-boot-disk=disk-image=projects/$PROJECT_ID/global/images/$DISK_IMAGE_NAME,mode=CONTAINER_IMAGE_CACHE \
    --location=$REGION
```
## Create the Hyperdisk ML
### Create a StorageClass that supports Hyperdisk ML
Save the following StorageClass manifest in a file named hyperdisk-ml.yaml
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
    name: hyperdisk-ml
parameters:
    type: hyperdisk-ml
    provisioned-throughput-on-create: "2400Mi"
provisioner: pd.csi.storage.gke.io
allowVolumeExpansion: false
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
mountOptions:
  - read_ahead_kb=4096
```
### Create a ReadWriteOnce (RWO) PersistentVolumeClaim
Save the following PersistentVolumeClaim manifest in a file named producer-pvc.yaml
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: producer-pvc
spec:
  storageClassName: hyperdisk-ml
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 300Gi
```
### Create a Kubernetes Job to populate the mounted Google Cloud Hyperdisk volume
```
HF_TOKEN=
```
```
kubectl create secret generic hf-secret \
    --from-literal=hf_api_token=$HF_TOKEN\
    --dry-run=client -o yaml | kubectl apply -f -
```
### Save the following example manifest as producer-job.yaml
```
ZONE=us-central1-c
```
```
apiVersion: batch/v1
kind: Job
metadata:
  name: producer-job
spec:
  template:  # Template for the Pods the Job will create
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: cloud.google.com/compute-class
                operator: In
                values:
                - "Performance"
            - matchExpressions:
              - key: cloud.google.com/machine-family
                operator: In
                values:
                - "c3"
            - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                - $ZONE
      containers:
      - name: copy
        resources:
          requests:
            cpu: "32"
          limits:
            cpu: "32"
        image: huggingface/downloader:0.17.3
        command: [ "huggingface-cli" ]
        args:
        - download
        - meta-llama/Llama-3.3-70B-Instruct
        - --local-dir=/data/Llama-3.3-70B-Instruct
        - --local-dir-use-symlinks=False
        env:
        - name: HUGGING_FACE_HUB_TOKEN
          valueFrom:
            secretKeyRef:
              name: hf-secret
              key: hf_api_token
        volumeMounts:
          - mountPath: "/data"
            name: volume
      restartPolicy: Never
      volumes:
        - name: volume
          persistentVolumeClaim:
            claimName: producer-pvc
  parallelism: 1         # Run 1 Pods concurrently
  completions: 1         # Once 1 Pods complete successfully, the Job is done
  backoffLimit: 4        # Max retries on failure
```
Error
ducdo@cloudshell:~/vllm-100 (gpu-launchpad-playground)$ kubectl apply -f producer-job.yaml
Error from server (GKE Warden constraints violations): error when creating "producer-job.yaml": admission webhook "warden-validating.common-webhooks.networking.gke.io" denied the request: GKE Warden rejected the request because it violates one or more constraints.
Violations details: {"[denied by ccc-node-affinity-selector-limitation]":["Key 'cloud.google.com/machine-family' is not allowed with node affinity; Custom Compute Classes can only be used along labels with keys: cloud.google.com/compute-class,topology.kubernetes.io/region,topology.kubernetes.io/zone,failure-domain.beta.kubernetes.io/region,failure-domain.beta.kubernetes.io/zone,cloud.google.com/gke-accelerator-count,cloud.google.com/gke-tpu-topology,cloud.google.com/gke-tpu-accelerator."]}
Requested by user: 'ducdo@google.com', groups: 'system:authenticated'.

## Create the deployment
### Create deployment manifest vllm-h100-llama370b.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-h100-llama370b-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: model-server
  template:
    metadata:
      labels:
        app: model-server
    spec:
      nodeSelector:
        cloud.google.com/gke-accelerator: nvidia-h100-80gb
      containers:
      - name: inference-server
        image: vllm/vllm-openai:v0.7.3
        env:
        - name: HUGGING_FACE_HUB_TOKEN
          valueFrom:
            secretKeyRef:
              name: ducdo-hf-secret
              key: hf_api_token
        command:
          - sh
          - -c
          - "vllm serve 'meta-llama/Llama-3.3-70B-Instruct' --tensor-parallel-size 4 --max_model_len 128000 --port 8080 --gpu-memory-utilization 0.95 --trust-remote-code"
        resources:
          limits:
            nvidia.com/gpu: "4"
        ports:
          - containerPort: 8080
        readinessProbe:
          tcpSocket:
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
        volumeMounts:
          - mountPath: /dev/shm
            name: dshm
      volumes:
      - name: dshm
        emptyDir:
          medium: Memory
```
### Create the service manifest llm-service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: llm-service
spec:
  ports:
    - name: http
      port: 8080
      protocol: TCP
      targetPort: 8080
  selector:
    app: model-server
  type: ClusterIP
```
### Use kubectl to deploy
```
gcloud container clusters get-credentials $CLUSTER_NAME \
    --location=$REGION
```
Create a secret to keep the HF Token
```
HF_TOKEN=
```
```
kubectl create secret generic ducdo-hf-secret \
    --from-literal=hf_api_token=$HF_TOKEN \
    --dry-run=client -o yaml | kubectl apply -f -
```
Deployment and service
```
kubectl apply -f vllm-h100-llama370b.yaml
kubectl apply -f llm-service.yaml
```

### Check the deployment status

```
kubectl get no
kubectl get po
kubectl describe pod $POD_NAME
kubectl get nodes -o json | jq -r '.items[] | {name:.metadata.name, gpus:.status.capacity."nvidia.com/gpu"}'
kubectl get events
kubectl describe deployment/vllm-h100-llama370b-deployment
kubectl logs deployment/vllm-h100-llama370b-deployment
kubectl logs -f deployment/vllm-h100-llama370b-deployment -n default
kubectl logs -f -l app=model-server

kubectl wait --for=condition=Available --timeout=700s deployment/vllm-h100-llama370b-deployment
```

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
kubectl delete deployment vllm-h100-llama370b-deployment
kubectl delete service llm-service
gcloud container clusters delete $CLUSTER_NAME --region=$REGION
```

Notes
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
