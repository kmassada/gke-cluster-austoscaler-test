# autoscaler

This repo is to practice [GKE's autoscaling](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-autoscaler), by using [Kubernetes' horizontal autoscaling walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/) as long  

```
export GOOGLE_CLOUD_PROJECT=xxxxxx
export COMPUTE_ZONE=xxxxx
export MIN_NODES=2
export MAX_NODES=10
export NUM_NODES=5
export IMAGE_TAG=$GOOGLE_CLOUD_PROJECT-labs/autoscaler
export CLUSTER_MAME=test-autoscaler
```

## create cluster

```
gcloud config set project $GOOGLE_CLOUD_PROJECT
gcloud config set compute/zone $COMPUTE_ZONE

gcloud iam service-accounts create  $NODE_SA_NAME --display-name "Node Service Account"
export NODE_SA_EMAIL=`gcloud iam service-accounts list --format='value(email)' --filter='displayName:Node Service Account'`
export PROJECT=`gcloud config get-value project`
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT --member=serviceAccount:${NODE_SA_EMAIL} --role=roles/monitoring.metricWriter
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT --member=serviceAccount:${NODE_SA_EMAIL} --role=roles/monitoring.viewer
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT --member=serviceAccount:${NODE_SA_EMAIL} --role=roles/logging.logWriter
gcloud config set container/new_scopes_behavior true
gcloud config set container/use_v1_api false

gcloud beta container clusters create $CLUSTER_MAME \
--service-account=$NODE_SA_EMAIL \
--workload-metadata-from-node=SECURE \
--zone=$COMPUTE_ZONE \
--cluster-version=1.10 \
--enable-autoscaling \
--num-nodes $NUM_NODES \
--min-nodes $MIN_NODES \
--max-nodes $MAX_NODES

gcloud container clusters get-credentials $CLUSTER_MAME
```

## create images

```
cd autoscaler/
docker build -t $IMAGE_TAG .
docker tag $IMAGE_TAG gcr.io/$GOOGLE_CLOUD_PROJECT/$IMAGE_TAG
gcloud docker -- push gcr.io/$GOOGLE_CLOUD_PROJECT/$IMAGE_TAG
```

## create deployment

```
envsubst < k8s-deployment.template.yaml | kubectl apply -f -
```

## current state of stats

```
watch kubectl top pod
NAME                          CPU(cores)   MEMORY(bytes)
php-apache-545b7c6bb5-mlh22   9m           6Mi
```

## apply autoscaling

```
kubectl apply autoscale.yaml
```

## fire some load 

```shell
kubectl run -i --tty load-generator --image=busybox /bin/sh
# Hit enter for command prompt
while true; do wget -q -O- http://php-apache.default.svc.cluster.local; done
```

## watch load get hot

```
NAME                          CPU(cores)   MEMORY(bytes)
php-apache-545b7c6bb5-xzn67   86m          9Mi
php-apache-545b7c6bb5-qffqw   104m         10Mi
php-apache-545b7c6bb5-jkhpv   92m          9Mi
php-apache-545b7c6bb5-2s7hr   109m         9Mi
php-apache-545b7c6bb5-jql8q   106m         9Mi
php-apache-545b7c6bb5-4czv2   69m          9Mi
php-apache-545b7c6bb5-ns96f   91m          10Mi
php-apache-545b7c6bb5-qdl6w   92m          9Mi
```

## show logs

in the GCP console logs

```
resource.type="container"
resource.labels.cluster_name="test-autoscaler"
resource.labels.namespace_id="default"
logName="projects/GOOGLE_CLOUD_PROJECT/logs/php-apache"
```

## describe pods

While event is going on live, you can find your HPA settings and traces of HPA getting kidcked off through these queries 

```
kubectl describe deploy/php-apache

kubectl get events | grep Scaled

kubectl describe hpa/php-apache

kubectl get events | grep horizontal-pod-autoscaler
```

The same can be viewed via GCP log console

```
resource.type="k8s_cluster"
resource.labels.location="$COMPUTE_ZONE"
protoPayload.resourceName="extensions/v1beta1/namespaces/default/deployments/php-apache/scale"
```

This is the reference we are looking for

```yaml
apiVersion:  "extensions/v1beta1"    
   kind:  "Scale"    
status: 
    replicas:  4     
```

## RAMP it UPP

inside another test pod again

```shell
kubectl run -i --tty load-generator --image=busybox /bin/sh
# Hit enter for command prompt
for i in {1..20}; do while true; do curl 35.197.7.71:80; done &  done
```

now you can see in the GCP console logs 

```
resource.type="k8s_cluster"
resource.labels.location="$COMPUTE_ZONE"
protoPayload.resourceName="extensions/v1beta1/namespaces/default/deployments/php-apache/scale"
```

your pods are showing 84 desired, but only 44 are present

```yaml
replicas:  44     
   spec: {
    replicas:  84     
   }
```

## LOGS LOGS LOGS

but via this you can attest to your custer nodes being scaled up as well

```
resource.type="gke_cluster"
resource.labels.cluster_name="test-autoscaler"
resource.labels.location="$COMPUTE_ZONE"
```