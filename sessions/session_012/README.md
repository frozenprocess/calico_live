# Calico Live
Welcome to this tutorial, where we'll guide you through the steps of creating a Kubernetes cluster using Calico eBPF dataplane mode and deploying a sample application.

It's important to note that this is an early preview of IPv6 support in eBPF. We're eager to gather information and receive feedback, so if you encounter any issues, have questions, or insights, please connect with us on [Slack](https://slack.projectcalico.org/). Join our community and utilize the #ebpf channel to share your thoughts.

Your input is invaluable as we refine and enhance this exciting feature. Let's build and learn together!

# Requirements
This demo uses `multipass` to build the infrastructure. You can install `multipass` by following the instructions [here](https://multipass.run/docs/installing-on-linux).

IPv6 enabled environment (Optional)

> **Note:** This tutorial has been tested with `qemu` and `libvirt` driver.

## Libvirt considerations

If you are using `libvirt` driver, you need to make sure that the network driver is ipv6 capable. This can be done by modifying the default network definition.

Use the following command to edit the default network definition:
```bash
virsh net-edit default
```

Add the following lines to your network definition:
```xml
<network ipv6='yes'>
  <ip family='ipv6' address='fc00::1' prefix='64'>
    <dhcp>
      <range start='fc00::2' end='fc00::ffff'/>
    </dhcp>
  </ip>
```

Your configuration should look similar to the following:
```xml
<network ipv6='yes'>
  <name>default</name>
  <uuid>e6f3bb11-05f5-416c-91a9-4d0a4809d215</uuid>
  <forward mode='nat'>
    <nat ipv6='yes'/>
  </forward>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:01:53:79'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
  <ip family='ipv6' address='fc00::1' prefix='64'>
    <dhcp>
      <range start='fc00::2' end='fc00::ffff'/>
    </dhcp>
  </ip>
</network>
```

Restart the default network:
```bash
virsh net-destroy default
virsh net-start default
```

If you like to learn more about Multipass and libvirt integration please click [here](https://multipass.run/docs/set-up-the-driver#heading--linux-use-libvirt).

## IPv6 considerations


# Building the infrastructure

> **Note:** You can change ubuntu version by changing `lunar` to the release that you desire.

Use the following command to build the control plane
```bash
multipass launch -n c1-control -c 2 -m 4G -d 15G lunar --cloud-init https://raw.githubusercontent.com/frozenprocess/calico_live/main/sessions/session_012/assets/control-init.yaml
```

> **Note:** You can run multiple nodes by incrementing the last part of  `c1-node-1`.

If you like to have nodes in your cluster use the following command
```bash
multipass launch -n c1-node-1 -c 2 -m 4G -d 15G lunar --cloud-init https://raw.githubusercontent.com/frozenprocess/calico_live/main/sessions/session_012/assets/node-init.yaml 
```

You can use the following command to get the list of machines that are running in your environment:
```bash
multipass list
```
## Connecting to the control plane

Use the following commands to copy the kubeconfig file to your local machine and set the `KUBECONFIG` environment variable.
```bash
multipass transfer c1-control:/etc/rancher/k3s/k3s.yaml ./k3s.yaml

CONTROL=`multipass exec c1-control -- ip -6 addr | egrep fc | awk '{print $2}' | cut -d/ -f1`
sed -si "s/::1/$CONTROL/" k3s.yaml
export KUBECONFIG=$PWD/k3s.yaml
```

Use the following command to ensure that you are connected to the control plane:
```bash
kubectl get nodes
```

You should see a result similar to the following:
```bash
NAME         STATUS     ROLES                  AGE     VERSION
c1-control   NotReady   control-plane,master   3m31s   v1.28.3+k3s1
c1-node-1    NotReady   <none>                 70s     v1.28.3+k3s1
```

# Installing Calico

Use the following command to install latest version of Tigera-operator:
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/master/manifests/tigera-operator.yaml
```

This lab environment is configured without kube-proxy, Since Calico eBPF completely replaces the need for `kube-proxy` in your environment.

Use the following command to create a configmap that allows Calico and the operator to  directly communicate with the Kubernetes API server:
```
kubectl create -f -<<EOF
kind: ConfigMap
apiVersion: v1
metadata:
  name: kubernetes-services-endpoint
  namespace: tigera-operator
data:
  KUBERNETES_SERVICE_HOST: "$CONTROL"
  KUBERNETES_SERVICE_PORT: '6443'
EOF
```

Use the following command to instruct the operator and install Calico for your environment:
```bash
kubectl create -f -<<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    bgp: Disabled
    linuxDataplane: BPF
    ipPools:
    - blockSize: 122
      cidr: fd00:10:244::/64
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

Your IPv6 Kubernetes cluster is now using Calico eBPF dataplane mode, but wait there is more!


# Deploying google microservices demo
Use the following command to install Google microservices demo:
```bash
kubectl create -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/v0.8.1/release/kubernetes-manifests.yaml
```

Use the following command to make sure your deployment has been successful:
```bash
kubectl rollout status deployment/frontend
```


```bash
kubectl get svc frontend-external
```

You should see a result similar to the following:
```bash
NAME                TYPE           CLUSTER-IP         EXTERNAL-IP             PORT(S)        AGE
frontend-external   LoadBalancer   fd00:10:96::4555   fc00::1cf8,fc00::f6d7   80:31976/TCP   71s
```
Use one of the external IPs to access the frontend service.

# eBPF based security

Now that we got the networking part working, let's enable eBPF based security.


> **Note:** In order to simplify this tutorial we are going to exempt kube-system, calico-system, and calico-apiserver namespaces from this policy. If you like to establish a Zero-trust posture and isolate every thing in your environment, please follow [this] tutorial(https://github.com/frozenprocess/Tigera-Presentations/tree/master/2023-03-30.container-and-Kubernetes-security-policy-design/04.best-practices-for-securing-a-Kubernetes-environment).


Use the following command to isolate our workloads from talking to the Internet, and permit `DNS` traffic for service and workload name queries:
```bash
kubectl create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-app-policy
spec:
  namespaceSelector: has(projectcalico.org/name) && projectcalico.org/name not in {"kube-system", "calico-system", "calico-apiserver"}
  types:
  - Ingress
  - Egress
  egress:
  - action: Allow
    protocol: UDP
    destination:
      selector: 'k8s-app == "kube-dns"'
      ports:
      - 53
EOF
```
At this point if you refresh the shop page you should not be able to view it anymore. This happens because we have isolated our current and any new namespaces with a single global policy.

Next, we have to permit our frontend to be accessible from the Internet.
Use the following policy to permit ingress (Incoming) traffic to the frontend pods:
```bash
kubectl create -f -<<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: permit-frontend
spec:
  ingress:
  - action: Allow
    destination:
      selector: 'app == "frontend"'
  egress:
  - action: Allow
    source:
      selector: 'app == "frontend"'
EOF
```

You should be able to view the shop again!

## eBPF troubleshooting
This tutorial has been layered so that everything works out of the box, but in some cases you might run into issues. In this section we are going to troubleshoot some of the common issues that you might run into.

Let's start with policies, in eBPF mode each policy is associated with an interface.
You can view these policies by invoking the `calico-node` binary.

Use the following command to list the interfaces and their state and number of associated policies:
```bash
kubectl exec -it -n calico-system ds/calico-node -c calico-node -- calico-node -bpf ifstate dump
```

You should see a result similar to the following:
```bash
   17 : {flags: workload,ready, XDPPolicy: -1, IngressPolicy: 26, EgressPolicy: 27, name: califdc26cc6d6b}
    2 : {flags: ready, XDPPolicy: 1, IngressPolicy: 2, EgressPolicy: 3, name: ens3}
```
flags: Indicates if that interface is ready or not, and if its a workload or a host interface.
XDPPolicy: Gives you the number of XDP policies that are associated with this interface. XDP policies are high throughput policies that are usually applied for high demand workloads or preventing DDoS attacks.
name: The name of the interface inside your Linux environment. 

> **Note**: Please replace `califdc26cc6d6b` in the following command with an interface name from the previous command.

From the information that we gathered in the previous step we can get details about each eBPF policy that is associated with an interface.

> **Note**: I'm using `all` as the last argument to get details about all policies, you can use `ingress` or `egress` to get details about a specific policy.

Use the following command to get details about the policies that are associated with the `califdc26cc6d6b` interface: 
```bash
kubectl exec -it -n calico-system ds/calico-node -c calico-node -- calico-node -bpf policy dump califdc26cc6d6b all -6
```

In some cases you might want to know if BPF maps are populated or not, you can use the following command to get details about the BPF maps:
```bash
kubectl exec -it -n calico-system ds/calico-node -c calico-node -- bpftool map show
```

There is a lot more that you can do with the `-bpf` command line argument, you can view the full list of options by running the following command:
```bash
kubectl exec -it -n calico-system ds/calico-node -c calico-node -- calico-node -bpf
```

If you like to learn more about troubleshooting please click [here](https://docs.tigera.io/calico/latest/operations/ebpf/troubleshoot-ebpf)


## Recording
TBD

## Resources
