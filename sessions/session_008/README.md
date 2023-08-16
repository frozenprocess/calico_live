# Calico Live

## Requirements
[Docker](https://docs.docker.com/engine/install/) 24.0.4
[kind](https://kind.sigs.k8s.io/docs/user/quick-start/) v0.19.0

### Cluster setup

```
kind create cluster --config - <<EOF
kind: Cluster
name: c1
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  # the default CNI will not be installed
  disableDefaultCNI: true
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/16"
nodes:
- role: control-plane
  image: kindest/node:v1.26.6@sha256:6e2d8b28a5b601defe327b98bd1c2d1930b49e5d8c512e1895099e4504007adb
- role: worker
  image: kindest/node:v1.26.6@sha256:6e2d8b28a5b601defe327b98bd1c2d1930b49e5d8c512e1895099e4504007adb
EOF
```

```
kind create cluster --config - <<EOF
kind: Cluster
name: c2
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  # the default CNI will not be installed
  disableDefaultCNI: true
  podSubnet: "10.245.0.0/16"
  serviceSubnet: "10.97.0.0/16"
nodes:
- role: control-plane
  image: kindest/node:v1.26.6@sha256:6e2d8b28a5b601defe327b98bd1c2d1930b49e5d8c512e1895099e4504007adb
- role: worker
  image: kindest/node:v1.26.6@sha256:6e2d8b28a5b601defe327b98bd1c2d1930b49e5d8c512e1895099e4504007adb
EOF
```

```
kubectl --context kind-c1 create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
```

```
kubectl --context kind-c2 create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
```

```
kubectl --context kind-c1 create -f -<<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  registry: quay.io/
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer 
metadata: 
  name: default 
spec: {}
EOF
```

```
kubectl --context kind-c2 create -f -<<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  registry: quay.io/
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 10.245.0.0/16
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer 
metadata: 
  name: default 
spec: {}
EOF
```

### Setting up BGP and BGP peers

```
kubectl --context kind-c1 create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  asNumber: 65001
EOF
```

```
kubectl --context kind-c2 create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  asNumber: 65002
EOF
```

```
docker exec -it c1-control-plane ip addr show eth0
docker exec -it c2-control-plane ip addr show eth0
```

```
kubectl --context kind-c1 create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgp2c2-control
spec:
  peerIP: <REPLACE WITH NODE IP>
  asNumber: 65002
EOF
```

```
kubectl --context kind-c2 create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgp2c1-control
spec:
  peerIP: <REPLACE WITH NODE IP>
  asNumber: 65001
EOF
```

### Troubleshooting BGP problems

```
kubectl --context kind-c1 create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: CalicoNodeStatus
metadata:
  name: c1-control-plane-status
spec:
  classes:
    - Agent
    - BGP
    - Routes
  node: c1-control-plane
  updatePeriodSeconds: 10
---
apiVersion: projectcalico.org/v3
kind: CalicoNodeStatus
metadata:
  name: c1-worker-status
spec:
  classes:
    - Agent
    - BGP
    - Routes
  node: c1-worker
  updatePeriodSeconds: 10
EOF
```

```
kubectl --context kind-c2 create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: CalicoNodeStatus
metadata:
  name: c2-control-plane-status
spec:
  classes:
    - Agent
    - BGP
    - Routes
  node: c2-control-plane
  updatePeriodSeconds: 10
---
apiVersion: projectcalico.org/v3
kind: CalicoNodeStatus
metadata:
  name: c2-worker-status
spec:
  classes:
    - Agent
    - BGP
    - Routes
  node: c2-worker
  updatePeriodSeconds: 10
EOF
```

```
kubectl --context kind-c1 get caliconodestatus
```

```
kubectl --context kind-c2 get caliconodestatus
```

```
kubectl --context kind-c1 exec -n calico-system ds/calico-node -c calico-node -- birdcl show protocols
```

```
kubectl --context kind-c1 exec -n calico-system ds/calico-node -c calico-node -- birdcl show route
```

### Full mesh

```
docker exec -it c1-control-plane ip addr show eth0
docker exec -it c1-worker ip addr show eth0
docker exec -it c2-control-plane ip addr show eth0
docker exec -it c2-worker ip addr show eth0
```

```
kubectl --context kind-c1 create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgp2c2-worker
spec:
  peerIP: <REPLACE WITH NODE IP>
  asNumber: 65002
EOF
```

```
kubectl --context kind-c2 create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgp2c1-worker
spec:
  peerIP: <REPLACE WITH NODE IP>
  asNumber: 65001
EOF
```

```
kubectl --context kind-c1 get caliconodestatus
```

```
kubectl --context kind-c2 get caliconodestatus
```

### Route advertisement 

```
kubectl create --context kind-c2 deployment nginx --image=nginx
kubectl create --context kind-c2 service nodeport nginx --tcp 80:80
```

```
kubectl get svc -o wide
```

```
kubectl --context kind-c1 run tmp-shell --rm -i --tty --image nicolaka/netshoot -- curl <REPLACE_WITH_SERVICEIP>
```

```
kubectl --context kind-c1 replace -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  asNumber: 65001
  serviceClusterIPs:
    - cidr: 10.96.0.0/16
EOF
```

```
kubectl --context kind-c2 replace -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  asNumber: 65002
  serviceClusterIPs:
    - cidr: 10.97.0.0/16
EOF
```

```
kubectl --context kind-c1 run tmp-shell --rm -i --tty --image nicolaka/netshoot -- curl <REPLACE_WITH_SERVICEIP>
```

```
kubectl --context kind-c2 patch service nginx \
  -p '{"spec":{"type": "NodePort", "externalTrafficPolicy":"Local"}}'
```

```
kubectl --context kind-c1 run tmp-shell --rm -i --tty --image nicolaka/netshoot -- curl <REPLACE_WITH_SERVICEIP>
```

## Clean up

```
kind delete clusters c1 c2
```

## Recording
https://www.youtube.com/watch?v=PefluN8YM9o&ab_channel=ProjectCalico

## Resources
https://docs.tigera.io/calico/latest/networking/configuring/bgp
https://docs.tigera.io/calico/latest/reference/resources/bgppeer
https://docs.tigera.io/calico/latest/reference/resources/caliconodestatus
https://kubernetes.io/docs/concepts/services-networking/service/#ips-and-vips
