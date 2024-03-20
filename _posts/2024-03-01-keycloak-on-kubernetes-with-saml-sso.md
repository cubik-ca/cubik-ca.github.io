# Bitnami Keycloak on Kubernetes with SAML SSO

I frequently forget all the details on how to do this, so I'm going to record the steps in
some detail with a little bit of commentary. I've installed this on AWS EKS, but it will work
fine on any Kubernetes cluster that has a public ingress. The short version:

1. Add the Bitnami Helm repo
2. Add the Jetstack Helm repo
3. Install cert-manager with Helm
4. Create a cluster issuer for Let's Encrypt
5. Install Keycloak with Helm
6. Configure Keycloak for SAML SSO

## Prerequisites: Kubernetes cluster

Deploying the cluster in AWS involves a bit of setup. Here are the required elements:

1. Two public subnets (I called them `eks-1` and `eks-2`). A subnet is public when it
   auto-assigns a public IPv4 address. A subnet that does not assign a public IP address must
   use a NAT gateway and not the Internet Gateway for the VPC. This precludes incoming traffic.
   The two subnets must be in different availability zones.
2. An EKS cluster deployed to `eks-1` and `eks-2`.
3. Install the ingress-nginx controller

## Add the Bitnami Helm repo

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

## Add the Jetstack Helm repo

```bash
helm repo add jetstack https://charts.jetstack.io
```

## Install cert-manager with Helm

```bash
helm -n cert-manager install --set installCRDs=true cert-manager jetstack/cert-manager --create-namespace
```

## Create a cluster issuer for Let's Encrypt

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  labels:
    name: letsencrypt-prod
spec:
  acme:
    email: me@my.home
    privateKeySecretRef:
      name: letsencrypt-prod
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```

```bash
kubectl apply -f cluster-issuer.yaml
```

## Install Keycloak with Helm

```yaml
global:
  storageClass: gp2 # for EKS
auth:
  adminUser: keycloak
  adminPassword: P4$$w0rd
proxy: edge
ingress:
  enabled: true
  ingressClassName: nginx
  annotations:
    kubernetes.io/ingress.class: "nginx"
    kubernetes.io/tls-acme: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hostname: keycloak.my.home
nodeSelector:
  eks.amazonaws.com/nodegroup: keycloak
```

```bash
helm -n keycloak install -f values.yaml keycloak bitnami/keycloak --create-namespace
```

```
There are many more values which can be set, but this is enough to get it running. These
settings will create a Keycloak instance in AWS using the specified username and password.
The service will automatically obtain a certificate from Let's Encrypt provided that the
public DNS can resolve its name.
```
