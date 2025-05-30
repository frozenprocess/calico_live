# Calico Live
If you've managed traffic in Kubernetes, you've likely navigated the world of Ingress controllers. For years, Ingress has been the standard way of getting our HTTP/S services exposed. But let's be honest, it often felt like a compromise. We wrestled with controller-specific annotations to unlock critical features, blurred the lines between infrastructure and application concerns, and sometimes wished for richer protocol support or a more standardized approach. This "pile of vendor annotations," while functional, highlighted the limitations of a standard that struggled to keep pace with the complex demands of modern, multi-team environments and even led to security vulnerabilities.

# Requirements
- eksctl
- awscli + aws account
- A kubernetes environment with load balancer support (we used EKS for this example)
- Latest version of Calico, v3.30.0 or above
- A valid domain, Route53 used in this tutorial. (This is required for SSL certificate, an alternative self-signed certificate is provided in the next section)
- A browser to verify some settings

## Spin up a Kubernetes Cluster

```bash
eksctl create cluster --name  my-calico-cluster
```

## Installing Calico

Use the following command to install latest version of Tigera-operator:
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.0/manifests/operator-crds.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.0/manifests/tigera-operator.yaml
```

```bash
kubectl rollout status -n tigera-operator deployment/tigera-operator
```

Use the following command to instruct the operator and install Calico for your environment:
```bash
kubectl create -f - <<EOF
kind: Installation
apiVersion: operator.tigera.io/v1
metadata:
 name: default
spec:
 kubeletVolumePluginPath: None
 kubernetesProvider: EKS
 cni:
   type: AmazonVPC
 calicoNetwork:
   bgp: Disabled
EOF
```

```bash
kubectl get tigerastatus
```

## Deploying google microservices demo
Use the following command to install Google microservices demo:
```bash
kubectl create -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/refs/heads/release/v0.10.2/release/kubernetes-manifests.yaml
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
frontend                ClusterIP      10.100.42.132    <none>                                                                   80/TCP         5d19h
frontend-external       LoadBalancer   10.100.43.33     a78ef864683d543e59c7d91fba812176-247554297.us-west-2.elb.amazonaws.com   80:30816/TCP   5d19h
```
Use one of the external IPs to access the frontend service.

## Calico Ingress Gateway


```bash
kubectl create -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: GatewayAPI
metadata:
 name: default
EOF
```

```bash
kubectl wait --for=condition=Available tigerastatus gatewayapi
```

## SSL Certificate and Automated Certification Process with Cert-Manager

```bash
kubectl create -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.2/cert-manager.yaml
```
```bash
kubectl patch deployment -n cert-manager cert-manager --type='json' --patch '
[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--enable-gateway-api"
  }
]'
```

### ClusterIssuer

> 📌 Note: Be sure to replace <USER-YOUR-EMAIL-HERE> with your actual email address. This is required by Let’s Encrypt for important expiry notices and terms of service updates.

```bash
kubectl create -f -<<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <USER-YOUR-EMAIL-HERE>
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
    - http01:
        gatewayHTTPRoute:
          parentRefs:
          - kind: Gateway
            name: calico-demo-gw
            namespace: default
EOF
```

## Deploy Gateway API Resources

📌 Note: Be sure to replace <REPLACE_WITH_YOUR_DOMAINE> with your actual domain. This is required by Let’s Encrypt for important expiry notices and terms of service updates.
```bash
kubectl create -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
 annotations:
  cert-manager.io/cluster-issuer: letsencrypt
 name: calico-demo-gw
spec:
 gatewayClassName: tigera-gateway-class
 listeners:
   - name: http
     protocol: HTTP
     port: 80
   - name: https
     protocol: HTTPS
     port: 443
     hostname: <REPLACE_WITH_YOUR_DOMAIN>
     tls:
       mode: Terminate
       certificateRefs:
       - kind: Secret
         group: ""
         name: secure-demo-cert
EOF
```

### HTTPRoute

```bash
kubectl create -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http-redirect
  namespace: default
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: calico-demo-gw
    namespace: default
    port: 80
    sectionName: http
  rules:
    - matches:
      - path:
          type: PathPrefix
          value: /.well-known/acme-challenge/
      backendRefs:
      - group: ""
        kind: Service
        name: frontend
        namespace: default
        port: 80
        weight: 1
    - matches:
      - path:
          type: PathPrefix
          value: /
      filters:
      - type: RequestRedirect
        requestRedirect:
          scheme: https
          statusCode: 301
          port: 443
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: https-access
  namespace: default
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: calico-demo-gw
    namespace: default
    port: 443
    sectionName: https
  rules:
    - matches:
      - path:
          type: PathPrefix
          value: /
      backendRefs:
      - group: ""
        kind: Service
        name: frontend
        namespace: default
        port: 80
        weight: 1
EOF
```


## Recording
[Getting started with EKS and Gatewayapi using Calico v3.30 YouTube](https://www.youtube.com/watch?v=Q8pbFLJIi5I)

## Resources
https://docs.tigera.io/calico/latest/networking/gateway-api
https://gateway.envoyproxy.io/docs/tasks/security/secure-gateways/
https://cert-manager.io/docs/
