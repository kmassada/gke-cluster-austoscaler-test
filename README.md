export GOOGLE_CLOUD_PROJECT=makz-support-156618
export COMPUTE_ZONE=us-west1-b
export MIN_NODES=2
export MAX_NODES=10
export NUM_NODES=5
export IMAGE_TAG=makz-labs/autoscaler
export CLUSTER_MAME=test-autoscaler

# create cluster

gcloud config set project $GOOGLE_CLOUD_PROJECT
gcloud config set compute/zone $COMPUTE_ZONE

gcloud container clusters create $CLUSTER_MAME \
--enable-autoscaling \
--num-nodes $NUM_NODES  --min-nodes $MIN_NODES --max-nodes $MAX_NODES

gcloud container clusters get-credentials $CLUSTER_MAME

# create imgaes
docker build -t $IMAGE_TAG .
docker tag $IMAGE_TAG gcr.io/$GOOGLE_CLOUD_PROJECT/$IMAGE_TAG
gcloud docker -- push gcr.io/$GOOGLE_CLOUD_PROJECT/$IMAGE_TAG

# create deployment
envsubst < deployment.yml | kubectl apply -f -

# Current state of stats

watch kubectl top pod
NAME                          CPU(cores)   MEMORY(bytes)
php-apache-545b7c6bb5-mlh22   9m           6Mi

# apply autoscaling
kubectl apply autoscale.yml

# Fire some load 
$ kubectl run -i --tty load-generator --image=busybox /bin/sh
Hit enter for command prompt
$ while true; do wget -q -O- http://php-apache.default.svc.cluster.local; done


# Watch load get hot
NAME                          CPU(cores)   MEMORY(bytes)
php-apache-545b7c6bb5-xzn67   86m          9Mi
php-apache-545b7c6bb5-qffqw   104m         10Mi
php-apache-545b7c6bb5-jkhpv   92m          9Mi
php-apache-545b7c6bb5-2s7hr   109m         9Mi
php-apache-545b7c6bb5-jql8q   106m         9Mi
php-apache-545b7c6bb5-4czv2   69m          9Mi
php-apache-545b7c6bb5-ns96f   91m          10Mi
php-apache-545b7c6bb5-qdl6w   92m          9Mi

# show logs
resource.type="container"
resource.labels.cluster_name="test-autoscaler"
resource.labels.namespace_id="default"
logName="projects/makz-support-156618/logs/php-apache"

# Describe pods

kubectl describe deploy/php-apache

kubectl get events | grep Scaled

kubectl describe hpa/php-apache

kubectl get events | grep horizontal-pod-autoscaler

resource.type="k8s_cluster"
resource.labels.location="us-west1-b"
protoPayload.resourceName="extensions/v1beta1/namespaces/default/deployments/php-apache/scale"

apiVersion:  "extensions/v1beta1"    
   kind:  "Scale"    
status: 
    replicas:  4     

# RAMP it UPP
for i in {1..20}; do while true; do curl 35.197.7.71:80; done &  done


We started with 5 nodes, we are now at 10 nodes
replicas:  44     
   spec: {
    replicas:  84     
   }

# LOGS LOGS LOGS
resource.type="gke_cluster"
resource.labels.cluster_name="test-autoscaler"
resource.labels.location="us-west1-b"