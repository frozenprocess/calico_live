# Calico Live

## Requirements
[Docker](https://docs.docker.com/engine/install/) 24.0.4
[kind](https://kind.sigs.k8s.io/docs/user/quick-start/) v0.19.0
[subctl](https://submariner.io/operations/deployment/subctl/#installing-specific-versions) v0.15.2

### Cluster setup


```
kind create cluster --config - <<EOF
kind: Cluster
name: c1
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
networking:
  podSubnet: "10.241.0.0/16"
  serviceSubnet: "10.111.0.0/16"
  disableDefaultCNI: true
EOF

kind create cluster --config - <<EOF
kind: Cluster
name: c2
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
networking:
  podSubnet: "10.242.0.0/16"
  serviceSubnet: "10.112.0.0/16"
  disableDefaultCNI: true
EOF
```

```
kubectl --context=kind-c1 create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
kubectl --context=kind-c2 create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
```

```
kubectl --context=kind-c1 create -f - <<EOF
kind: Installation
apiVersion: operator.tigera.io/v1
metadata:
  name: default
spec:
  registry: quay.io/
  cni:
    type: Calico
  calicoNetwork:
    bgp: Disabled
    ipPools:
     - cidr: 10.241.0.0/16
       encapsulation: VXLAN
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
   name: default
spec: {}
EOF
```

```
kubectl --context=kind-c2 create -f - <<EOF
kind: Installation
apiVersion: operator.tigera.io/v1
metadata:
  name: default
spec:
  registry: quay.io/
  cni:
    type: Calico
  calicoNetwork:
    bgp: Disabled
    ipPools:
     - cidr: 10.242.0.0/16
       encapsulation: VXLAN
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
   name: default
spec: {}
EOF
```

```
kubectl --context kind-c1 create -f - <<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
 name: svc-c2
spec:
 cidr: 10.112.0.0/16
 natOutgoing: false
 disabled: true
---
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
 name: pod-c2
spec:
 cidr: 10.242.0.0/16
 natOutgoing: false
 disabled: true
EOF
```

```
kubectl --context kind-c2 create -f - <<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
 name: svc-c1
spec:
 cidr: 10.111.0.0/16
 natOutgoing: false
 disabled: true
---
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
 name: pod-c1
spec:
 cidr: 10.241.0.0/16
 natOutgoing: false
 disabled: true
EOF
```

```
curl -OL  https://github.com/submariner-io/releases/releases/download/v0.15.2/subctl-v0.15.2-linux-amd64.tar.xz
tar xf subctl-v0.15.2-linux-amd64.tar.xz
docker cp ./subctl-v0.15.2/subctl-v0.15.2-linux-amd64 c1-control-plane:/usr/local/bin/subctl
```

```
export C1=$(kubectl --context kind-c1 get nodes --field-selector metadata.name=c1-control-plane -o=jsonpath='{.items[0].status.addresses[0].address}') 
export C2=$(kubectl --context kind-c2 get nodes --field-selector metadata.name=c2-control-plane -o=jsonpath='{.items[0].status.addresses[0].address}') 

docker cp ~/.kube/config c1-control-plane:/root/.kube/config

docker exec -it --env C1 --env C2 c1-control-plane /bin/bash 
```

```
kubectl config set-cluster kind-c1 --server https://$C1:6443/
kubectl config set-cluster kind-c2 --server https://$C2:6443/
```


```
subctl deploy-broker --context kind-c1 
```

```
subctl join --context kind-c1 broker-info.subm --clusterid kind-c1
subctl join --context kind-c2 broker-info.subm --clusterid kind-c2
```


## Submariner tunnel

```
subctl show connections
```

### Verifications

```
KUBECONFIG=/root/.kube/config subctl diagnose firewall inter-cluster --context kind-c1 --remotecontext kind-c2
```

```
KUBECONFIG=/root/.kube/config subctl diagnose firewall inter-cluster --remotecontext kind-c1 --context kind-c2
```

```
subctl show all
```

```
subctl diagnose all
```

## Manual Verification


```
kubectl --context=kind-c2 create deployment nginx --image=nginx
kubectl --context=kind-c2 expose deployment nginx --port=80
```

```
subctl export service --context=kind-c2 --namespace default nginx
```

```
kubectl --context=kind-c1 -n default  run tmp-shell \
 --rm -i --tty --image quay.io/submariner/nettest -- /bin/bash
```

```
curl nginx.default.svc.clusterset.local
```


## Clean up

```
kind delete clusters c1 c2
```

## Recording
https://www.youtube.com/watch?v=NViUHl9jYc4&ab_channel=ProjectCalico

## Resources
[Calico submariner](https://www.tigera.io/blog/calico-and-submariner-integration-a-hands-on-walkthrough/)