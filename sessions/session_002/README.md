# Calico Live

## Requirements

* [multipass](https://multipass.run/install)
* 5GB RAM
* 5-10GB of storage

### Kubernetes cluster architecture
<img src="https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg">

Use the following command to build the cluster:
```
multipass launch -n control -c 2 -m 2048M 22.04 --cloud-init https://raw.githubusercontent.com/frozenprocess/calico_live/main/sessions/session_002/assets/control-init.yaml
multipass launch -n node1 -c 2 22.04 --cloud-init https://raw.githubusercontent.com/frozenprocess/calico_live/main/sessions/session_002/assets/node-init.yaml
multipass launch -n private-repo -d 60G 22.04 --cloud-init https://raw.githubusercontent.com/frozenprocess/calico_live/main/sessions/session_002/assets/registry-init.yaml
```

Command to copy the cluster configurations
```bash
multipass transfer control:/etc/rancher/k3s/k3s.yaml .


CONTROL=$(multipass list --format csv | egrep control | cut -d, -f3)
export KUBECONFIG=$(pwd)/k3s.yaml
sed -si "s/127.0.0.1/$CONTROL/" k3s.yaml
```

### Namespace isolation

Namespaces are logical binders that can be used to group workloads together.

Use the following command to create a namespace
```
kubectl create ns monitoring
```

### Policy design

Click [here](https://github.com/frozenprocess/Tigera-Presentations/tree/master/2023-03-30.container-and-Kubernetes-security-policy-design/04.best-practices-for-securing-a-Kubernetes-environment) to read about policy design.

## Recording
https://www.youtube.com/watch?v=iBaq4gCBM4U&t=1s

## Resources
[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)