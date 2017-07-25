# Join a kubernetes GKE cluster
This document describes the process of joining a kubernetes cluster that runs in Google Container Engine (GKE) in a kubernetes federation whose Federation Control Plane is hosted on-premise (OpenStack Magnum) and uses CoreDNS.

## Prerequisites
1. An on-premise federation up and running. Check [Federate two magnum kubernetes v1.7 clusters](federate-two-clusters.md) for setup instructions.
2. A Google Cloud Platform account and Google Cloud SDK (`gcloud`) installed and configured in your machine. [installation instructions](https://cloud.google.com/sdk/downloads).

## Enable client certs and application credentials
After configuring `gcloud` with your credentials and region info (`gcloud init`), we need to enable client certs and application credentials:
```
gcloud config set container/use_application_default_credentials true
```
```
gcloud config set container/use_client_certificate True
```
Now go to the Google Cloud Platform Dashboard. Go to API Manager > Credentials > Create credentials > Service account key. Select "Compute Engine default service account", and click in "Create". This will download the credentials JSON to your machine. Then run:
```
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/your/credentials.json
```

## Create your cluster and get the access info
```
gcloud container clusters create gke-1
```
```
gcloud container clusters get-credentials gke-1
```
Now you can access your cluster and its resources. The kubeconfig file generated is usually located at `~/.kube/config`.

## Merge your config files
In order to join the GKE cluster, we need to access its context and the context of the host cluster of the federation. The easiest way is by exporting the `KUBECONFIG` environment variable with all the paths:
```
export KUBECONFIG:/path/to/kubeconfig-host-cluster:/path/to/kubeconfig-gke
```
Tip: GKE usually its contexts with long names, e.g. `gke_winter-legend-167417_us-east1-b_c1`. You can easily replace it with a shorter name (`gke`, for example). Run: `sed -i "s/gke_winter-legend-167417_us-east1-b_c1/gke/g" path/to/kubeconfig-gke`.

Now you should be able to see all the contexts:
```
$ kubectl config get-contexts
CURRENT   NAME                       CLUSTER            AUTHINFO           NAMESPACE
          fed                        fed                fed                
          c3                         c3                 c3   
          gke                        gke                gke                
*         c1                         c1                 c1      
```
Where `c1` is the host context of the federation.

## Join the cluster!
```
kubefed join --context=fed gke --host-cluster-context=c1 --cluster-context=gke
```

Wait for it to report as Ready (`kubectl get clusters --context=fed`) et voilà.

## Try it
Now your federated resources are going to spread to GKE as well.
