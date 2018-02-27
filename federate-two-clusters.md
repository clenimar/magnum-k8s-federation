**DEPRECATED**: it's been a while and a bunch of things have changed since then. I will update this guide soon.

# Federate two magnum kubernetes v1.7 clusters
This document describes the process of federating two kubernetes v1.7.1 clusters created by OpenStack Magnum. We are going to use CoreDNS as DNS provider for the Federation Control Plane (FCP, hereafter).

## Prerequisites
We are going to assume that you already have two clusters running kubernetes v1.7.1. Let's call them `c1` and `c2`. Take a look at the contexts we've got so far:
```
$ kubectl config get-contexts
CURRENT   NAME                     CLUSTER          AUTHINFO         NAMESPACE
*         c1-context               c1               c1   
          c2-context               c2               c2   
```
We are also going to use `c1` as the host cluster of our FCP.

## Setup CoreDNS
A kubernetes federation needs an external DNS provider in order to achieve high availability and perform cross-cluster service discovery. There are three options supported currently: Google Cloud DNS, AWS Route and CoreDNS. We are going to use CoreDNS, as it suits better our on-premise profile.

### Install Helm
Please refer to the [Helm quick start guide](https://docs.helm.sh/using_helm/#quickstart-guide). Once the Tiller pod is running after `helm init`, Helm is ready to deploy applications through its charts.

### Install etcd-operator chart
CoreDNS needs an etcd server in order to work properly. In order to deploy it, let's install the `etcd-operator` chart from the `stable` chart repository. This should deploy an etcd cluster. Run:
```
helm install --namespace default --name etcd-operator stable/etcd-operator
helm upgrade --namespace default --set cluster.enabled=true etcd-operator stable/etcd-operator
```
**Important!** If the `upgrade` command fails, you might need a service account for  `etcd-operator`. To do so, create it by saving the file below as `sa-etcd-operator.yaml`:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: etcd-operator
  namespace: default
```
And create it:
```
kubectl create -f sa-etcd-operator.yaml --validate=false
```
Now we have to use the service account we just created. Find the etcd-operator deployment (`kubectl get deploy`) and edit it (`kubectl edit deploy etcd-operator-etcd-operator --validate=false`). Add the field `spec.template.spec.serviceAccountName: etcd-operator`. Then run the upgrade command again and it should work.
Now etcd is running and can be accessed at http://etcd-cluster.default:2379.

### Install CoreDNS chart
Now we have to install CoreDNS chart. First, let's configure it to make sure it fits our needs. Take a look at `values.yaml`:
```yaml
isClusterService: false
serviceType: "NodePort"
middleware:
  kubernetes:
    enabled: false
  etcd:
    enabled: true
    zones:
    - "fed.cern."
    endpoint: "http://etcd-cluster.default:2379"
```
Install CoreDNS by running:
```
helm install --namespace default --name coredns -f values.yaml stable/coredns
```
Wait for the pods to start Running, and CoreDNS is setup!

### Configure the DNS provider
Now we have to configure the CoreDNS provider. Save the following contents to `coredns-provider.conf`.
```
[Global]
etcd-endpoints=http://etcd-cluster.default:2379
zones=fed.cern.
coredns-endpoints=coredns-server-ip:port
```
Replace `coredns-server-ip:port` with the actual address for the CoreDNS. To easily find it, run `kubectl get ep` and see the CoreDNS endpoint.

## Merge your config files
If you are using OpenStack Magnum to create the clusters, you have one `kubeconfig` file per cluster, created by the `magnum cluster-config` command. We need to merge those files in one, so you can access all of your clusters at once. To do so:
1. Open each `kubeconfig` file and change the path of the certs in it to absolute paths.
2. Now merge it. The easiest way is by exporting the `KUBECONFIG` environment variable with all the paths:
```
export KUBECONFIG:/path/to/kubeconfig1:/path/to/kubeconfig2
```
Note: kubernetes will read the files in order, so your current context will be set to the first context in the first kubeconfig file.

## Install kubefed
We need `kubefed` in order to deploy our Federation Control Plane (FCP, hereafter). For installation instructions, [read the docs](https://kubernetes.io/docs/tasks/federation/set-up-cluster-federation-kubefed/).

## Setup your federation!

### Grant enough privileges to users and service accounts
You need to grant privileges to the Magnum User, kubelet, and service accounts, so they becomes able to access API resources. You can create a specific clusterrole resource that grants privileges only for the resources each user/group needs to access, or just bind the all-powerful `cluster-admin` clusterrole to them (not safe! check `Next steps` section):
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: permissive-binding
subjects:
- kind: User
  name: Magnum User
  apiGroup: rbac.authorization.k8s.io
- kind: User
  name: kubelet
  apiGroup: rbac.authorization.k8s.io
- kind: User
  name: kubernetes.invalid
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```
Save it to `permissive-binding.yaml` and run in both clusters:
```
kubectl create -f permissive-binding.yaml --validate=false
```

### Enable RBAC
After 1.6.0 RBAC became the default authorization mode of kubernetes. In order to avoid problems, we need to make sure that it is enabled in each cluster:
1. Log into the master instance of your cluster.
2. Edit `/etc/kubernetes/apiserver`.
3. Add the flag `--authorization-mode=RBAC` to the line that starts with `KUBE_API_ARGS=`.
4. Restart the API server by running `sudo systemctl restart kube-apiserver`.

### Deploy the FCP
In order to deploy our FCP, run:
```
kubefed init fed --host-cluster-context=c1 --dns-provider="coredns" --dns-zone-name="fed.cern." --dns-provider-config="coredns-provider.conf" --api-server-service-type="NodePort" --kubeconfig=$KUBECONFIG --etcd-persistent-storage="false"
```
Now we've got our federation running!
```
Federation API server is running at: 200.100.10.10:31167, ...
```
**Important!** The flag `--etcd-persistent-storage=false` disables the persistent storage for the FCP, thus the federation is not storing its state! This flag is necessary because `etcd` deploy has some problems. See `Next steps` section for more info.

Note that now we have one more context, which refers to our federation. Every federated operation must be performed in this context. For that, you can pass `--context=fed` to `kubectl` and `kubefed`, or set the current context to it ( `kubectl config use-context fed`).
```
$ kubectl config get-contexts
CURRENT   NAME                       CLUSTER            AUTHINFO           NAMESPACE
          fed                        fed                fed                
*         c1-context                 c1                 c1   

```
### Edit fed-controller-manager deploy
Another issue under investigation is that the `fed-controller-manager` pod keeps restarting. If you see the logs of the pod, you will see the following error message:
```
F0720 09:44:45.702614       1 controllermanager.go:134] Could not find resources from API Server: Get https://fed-apiserver/api: dial tcp: i/o timeout
```
We don't know for sure what causes it, maybe something related with name resolution. See the `Next steps` section for more detail.
As a workaround, you can manually setup the IP for the federation API server. To do so, figure out the address on which it is running:
```
$ kubectl cluster-info --context=fed
Kubernetes master is running at https://200.100.10.10:31167
```
Now edit the `fed-controller-manager` deploy in the `c1` context using:
```
kubectl edit deploy fed-controller-manager --namespace=federation-system --validate=false
```
In `spec.template.spec.containers.command`, override the value of `--master` with the address of the federation API server (e.g. https://200.100.10.10:31167). Save and now the federation is working!

## Join your clusters
Now, let's join our clusters into the federation. As said before, `c1` is the host cluster of our FCP, so `--host-cluster-context` flag must be set to the `c1` context, whose name is `c1-context`.
Joining c1:
```
kubefed join --context=fed c1 --host-cluster-context=c1-context --cluster-context=c1-context
```
Note that we are trying to join the host cluster to the federation. That's why the flags `--host-cluster-context` and `--cluster-context` have the same value.
Wait for the cluster to appear as `Ready`:
```
$ kubectl get clusters --context=fed
NAME               STATUS    AGE
c1                 Ready     11s
```
Now, the same thing for `c2`:
```
kubefed join --context=fed c2 --host-cluster-context=c1-context --cluster-context=c2-context
```
Wait it to become `Ready`, and the federation is properly set!

## Try it out!
Just to make sure things are working fine, let's deploy a federated application and see if it is correctly spread accross all the underlying clusters:
```
kubectl create deployment nginx --image=nginx --context=fed
```
```
kubectl scale deployment nginx --replicas=10 --context=fed
```
The expected behavior is to see the pods equally distributed across the clusters (so 5 replicas to each one):
```
$ kubectl get pods --context=c1-context
NAME                                           READY     STATUS    RESTARTS   AGE
busybox-4158955249-0sgsv                       1/1       Running   0          23h
coredns-coredns-792875420-rsdnn                1/1       Running   0          23h
etcd-operator-etcd-operator-3419582230-w7889   1/1       Running   0          23h
nginx-1493591563-3mb3q                         1/1       Running   0          23s
nginx-1493591563-7mn0m                         1/1       Running   0          23s
nginx-1493591563-qsr5p                         1/1       Running   0          23s
nginx-1493591563-sw2x3                         1/1       Running   0          23s
nginx-1493591563-v1rfd                         1/1       Running   0          23s
```
... and:
```
$ kubectl get pods --context=c2-context
NAME                     READY     STATUS    RESTARTS   AGE
nginx-1493591563-70fnk   1/1       Running   0          44s
nginx-1493591563-75176   1/1       Running   0          54s
nginx-1493591563-jb3gd   1/1       Running   0          44s
nginx-1493591563-qhzrb   1/1       Running   0          44s
nginx-1493591563-qq2hz   1/1       Running   0          44s
```
Federation successfully set!

## Next steps
As you can notice, this procedure is far from ideal and has some flaws. For example:
1. RBAC: we still need to figure out what permissions do each user and service account need, instead of granting all-powerful privileges for everyone.
2. etcd: the reason because we need to disable the persistent storage for the FCP is etcd never getting ready. If you enable that option, the FCP never becomes ready as well. If you debug a bit, you will may find the following logs in `kube-scheduler`:
```
Jul 20 09:43:06 kube-master.cern.ch.cern.ch kube-scheduler[14795]: E0720 09:43:06.274930   14795 factory.go:659] Error scheduling federation-system fed-apiserver-364715109-2g09m: PersistentVolumeClaim is not bound: "fed-apiserver-etcd-claim"; retrying
Jul 20 09:43:06 kube-master.cern.ch.cern.ch kube-scheduler[14795]: I0720 09:43:06.275314   14795 event.go:218] Event(v1.ObjectReference{Kind:"Pod", Namespace:"federation-system", Name:"fed-apiserver-364715109-2g09m", UID:"89e5cf9e-6d2f-11e7-aa9c-fa163efb01d1", APIVersion:"v1", ResourceVersion:"102656", FieldPath:""}): type: 'Warning' reason: 'FailedScheduling' PersistentVolumeClaim is not bound: "fed-apiserver-etcd-claim"
```
3. name resolution: for some reason, fed-controller-manager is not being able to resolve names such as `https://fed-apiserver`. Maybe some issue or misconfiguration in CoreDNS, or an RBAC issue. This needs more digging.
