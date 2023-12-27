# A Development Kubernetes Cluster

## Abstract

This article is a rollup and refinement of the articles about Kubernetes I wrote on my previous site. I present here a
fully-featured Kuberentes development cluster running on MacOS. It should not be particularly difficult to adapt the
installation script for Linux. However, a different installation script will be required for Windows, and it is not
provided. I'll post the relevant bits of the code here, but for privacy reasons, I won't post the whole thing. I'll start
with the installation script, which assumes that you have installed Homebrew and Podman Desktop and created a Podman
virtual machine on which Minikube can run. I will then make some comments about the installation script, and proceed
with some Terraform code and commentary.

## Prerequisites

### Development domain name

cert-manager requires a public domain to work with in order to issue certificates. On a local workstation, it is not
possible to use the HTTP01 challenge (or you shouldn't, anyway!), so the only option is the DNS01 challenge. I grabbed
a domain in the `.dev` space and put it in Digital Ocean. cert-manager works well with these DNS providers as well:

* Azure DNS
* Route53 (AWS)
* Akamai
* Cloudflare

I would not recommend Azure DNS due to the proprietary mechanisms used to authenticate. The API token used by Digital
Ocean, while slightly less secure, seems sufficient to secure a development domain for a local cluster provided that
you don't give your main DNS account keys to the developers. Creating a separate DNS account for this purpose seems wise.
If you want to use one of these other providers, you will need to modify `modules/letsencrypt-staging/main.tf` with the
correct DNS01 solver.

### Homebrew

Homebrew is the package manager for MacOS. Install it from [brew.sh](https://brew.sh)

### Podman Desktop

For a very obscure reason, I'm not able to use Docker Desktop on MacOS. In the course of troubleshooting this obscure issue,
I came across Podman Desktop, which is a drop-in replacement for Docker Desktop, except that it doesn't have the same problem.
It is most easily installed using Homebrew with `brew install podman-desktop`. Once Podman Desktop is installed, you will
need to create a Podman machine from the GUI. I used a cluster with 6 cores and 16GB of RAM on an M2 Pro. This seems to be
a good size: memory is at about 40% usage with all software installed, so there is plenty of memory (but I suspect not too
much once everything starts running and you add your application(s)).

Even if you don't need to use Podman Desktop, I'd still recommend using it to allow minikube to use the podman driver, which
seems to have better resource allocation, and allows resizing the Podman machine. If you stick with Docker Desktop, you
should still be able to adapt these instructions to use Docker.

## Configuration

The installation script will require that sensitive values be provided in `install.conf`. Install.conf is a simple key-value
pair per line, which can be `source`d by a shell script. Here are the values you will need to provide:

```sh
docker_username=
docker_password=
do_api_key=
admin_password=
redis_password=
```

The Docker username and password are the values required to login to the private registry that is used for providing a
custom theme to Keycloak later on. If you don't need a custom theme, you can just put any value here. `do_api_key` is 
the API key configured at Digital Ocean. I chose Digital Ocean for DNS as it didn't tie me to any kind of proprietary 
DNS features, like Azure DNS would. `admin_password` is your desired login password for the Keycloak and Grafana admin
consoles. `redis_password` is your desired password for the Redis server.

There are additional configuration values in `terraform.vars.template`. You can change the name of your web endpoints
by modifying the values for `grafana_hostname`, `keycloak_hostname`, and `jaeger_hostname`.

I've referred to a lot of things in these files, but haven't posted either of them, so without further adieu, here are
`install.sh` and `terraform.vars.template`:

```sh
#!/bin/bash
set -e

[ -f install.conf ] || (echo "install.conf not found. Create it first. See README.md"; exit 1)
source install.conf

if ! which brew >/dev/null 2>&1;
then
    echo "Homebrew not installed!"
    echo ""
    echo "This script requires Homebrew. Please install it using the command below:"
    echo ""
    echo '/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"'
fi

# We need to install the following prior to running Terraform:
if ! brew list | grep coreutils >/dev/null 2>&1; then brew install coreutils; fi
# grep -q causes a SIGPIPE for some reason, but this works
if ! brew list | grep minikube >/dev/null 2>&1
then
    brew install minikube && minikube start --cpus='max' --memory='max' --disk-size='100g' --driver=podman
fi
# kubectl
if ! brew list | grep kubernetes-cli >/dev/null 2>&1; then brew install kubernetes-cli; fi
# Helm
if ! brew list | grep helm >/dev/null 2>&1; then brew install helm; fi
if ! brew list | grep terraform >/dev/null 2>&1; then brew install terraform; fi
# set up repos
helm repo add jetstack https://charts.jetstack.io
helm repo add metallb https://metallb.github.io/metallb
helm repo update
# the ingress controller is available as Minikube addons, and is most easily installed that way
minikube addons enable ingress
echo "Starting minikube tunnel. This may prompt you for your password."
# this will prompt for the sudo password, cache it, and return true
sudo sh -c true
# now we can run the minikube tunnel with sudo privileges in the background
sudo minikube tunnel &
TUNNEL_PID=$!
if ! kubectl get ns keycloak >/dev/null 2>& 1; then kubectl create ns keycloak; fi
# so does metallb
if ! helm -n metallb-system status metallb-system >/dev/null 2>&1
then
    kubectl create ns metallb-system
    # seed the metallb speaker secret
    kubectl create secret generic -n metallb-system metallb-memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
    helm -n metallb-system install metallb metallb/metallb
fi
# cert-manager creates Custom Resource Definitions that are required as part of the build. It should be installed via helm prior to the Terraform build
if ! helm -n cert-manager status cert-manager >/dev/null 2>&1
then
    helm -n cert-manager install \
    cert-manager jetstack/cert-manager \
    --set installCRDs=true \
    --set extraArgs='{--dns01-recursive-nameservers-only,--dns01-recursive-nameservers=1.1.1.1:53\,8.8.8.8:53}' \
    --set ingressShim.defaultIssuerName=letsencrypt-staging \
    --set ingressShim.defaultIssuerKind=ClusterIssuer \
    --create-namespace
fi
# update terraform.tfvars with the correct values for connecting to Kubernetes
MINIKUBE_API_URL=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
MINIKUBE_CA=$(cat $HOME/.minikube/ca.crt | gbase64 -w 0)
MINIKUBE_CLIENT_CERT=$(cat $HOME/.minikube/profiles/minikube/client.crt | gbase64 -w 0)
MINIKUBE_CLIENT_KEY=$(cat $HOME/.minikube/profiles/minikube/client.key | gbase64 -w 0)
cat terraform.tfvars.template | sed "s|__MINIKUBE_API_URL__|$MINIKUBE_API_URL|g" \
                              | sed "s|__MINIKUBE_CA__|$MINIKUBE_CA|g" \
                              | sed "s|__MINIKUBE_CLIENT_CERT__|$MINIKUBE_CLIENT_CERT|g" \
                              | sed "s|__MINIKUBE_CLIENT_KEY__|$MINIKUBE_CLIENT_KEY|g" \
                              | sed "s|__DO_API_KEY__|$do_api_key|g" \
                              | sed "s|__DOCKER_USERNAME__|$docker_username|g" \
                              | sed "s|__DOCKER_PASSWORD__|$docker_password|g" \
                              | sed "s|__ADMIN_PASSWORD__|$admin_password|g" \
                              | sed "s|__REDIS_PASSWORD__|$redis_password|g" \
                              > terraform.tfvars.sed && mv terraform.tfvars.sed terraform.tfvars
# wait for metallb to finish initializing
kubectl -n metallb-system wait --for=condition=ready pod -l app.kubernetes.io/name=metallb --timeout=120s
# All remaining software should install with Terraform
terraform init -upgrade && terraform apply -auto-approve
# started with root privileges, so needs to be stopped with root privileges
echo "Stopping minikube tunnel. This may require your password."
sudo kill $TUNNEL_PID
```

```
host = "__MINIKUBE_API_URL__"
cluster_ca_certificate = "__MINIKUBE_CA__"
client_certificate = "__MINIKUBE_CLIENT_CERT__"
client_key = "__MINIKUBE_CLIENT_KEY__"
digital_ocean_api_key = "__DO_API_KEY__"
letsencrypt_email = "(my-email)"
postgres_host = "postgres-postgresql.postgres.svc.cluster.local"
admin_password = "__ADMIN_PASSWORD__"
keycloak_hostname = "(my-keycloak-hostname)"
docker_username = "__DOCKER_USERNAME__"
docker_password = "__DOCKER_PASSWORD__"
redis_password = "__REDIS_PASSWORD__"
grafana_hostname = "(my-grafana-hostname)"
jaeger_hostname = "(my-jaeger-hostname)"
```

Configuration is now complete, and you can run the script with `bash install.sh` (or make it executable and run it). It's
not immediately obvious why this approach is needed. Clearly you could just replace the template values yourself, right?
Except for the `minikube` ones. So, since we need to `sed` a file anyway, this approach seems best and keeps us from
needing to edit the install script directly.

## Terraform

The bulk of the installation is performed by Terraform after the install script installs prerequisites:

* `coreutils` to provide `gbase64`
* `minikube`, our chosen Kubernetes implementation
* `kubernetes-cli` to provide `kubectl`
* `helm` to provide `helm`
* `terraform` to provide `terraform`
* `metallb` to provide a subnet in which to place load balancers
* `cert-manager` to issue certificates

I'll go through each of the modules and post the `main.tf` for each of them:

## addresspool

While we installed `metallb` already, we didn't add any addresses for it to use. This requires a Kubernetes manifest
deployment, so I saved it for the Terraform phase.

```hcl
resource "kubernetes_manifest" "default-address-pool" {
    manifest = {
        apiVersion = "metallb.io/v1beta1"
        kind = "IPAddressPool"
        metadata = {
            name = "default-address-pool"
            namespace = "metallb-system"
        }
        spec = {
            addresses = ["192.168.5.0/24"]
            avoidBuggyIPs = true
        }
    }
}

resource "kubernetes_manifest" "default-l2-advertisement" {
    manifest = {
        apiVersion = "metallb.io/v1beta1"
        kind = "L2Advertisement"
        metadata = {
            name = "default-l2-advertisement"
            namespace = "metallb-system"
        }
    }
}
```

This is straightforward enough, deploying an address pool (which is completely arbitrary as long as it doesn't conflict
with other cluster reserved address, such as `10.96.0.0/16` and `10.244.0.0/16`). The default L2 advertisement that is
also deployed provides ARP responses to the addresses in the pool.

### letsencrypt-staging

As above, we installed `cert-manager` but didn't install the cluster issuer. Again, this is a Kubernetes manifest deployment:

```hcl
resource "kubernetes_secret" "digitalocean-dns-secret" {
    metadata {
        name = "digitalocean-dns-secret"
        # installed by default
        namespace = "cert-manager"
    }
    data = {
        apikey = var.digital_ocean_api_key
    }
}

resource "kubernetes_manifest" "letsencrypt-staging" {
    manifest = {
        apiVersion = "cert-manager.io/v1"
        kind = "ClusterIssuer"
        metadata = {
            name = "letsencrypt-staging"
        }
        spec = {
            acme = {
                server = "https://acme-staging-v02.api.letsencrypt.org/directory"
                email = var.letsencrypt_email
                privateKeySecretRef = {
                    name = "letsencrypt-staging-key"
                }
                solvers = [
                    {
                        dns01 = {
                            digitalocean = {
                                tokenSecretRef = {
                                    name = kubernetes_secret.digitalocean-dns-secret.metadata.0.name
                                    key = "apikey"
                                }
                            }
                        }
                    }
                ]
            }
        }
    }
}

```

This cluster issuer, along with the ingress shim we enabled during the `cert-manager` installation, will issue a TLS
certificate for any ingress that has a hostname and secret key defined.

### postgres

PostgreSQL is a prerequisite for Keycloak unless we want to install the Keycloak postgres subchart. I would suggest that
since PostgreSQL may well be reused by other cluster components, that it is best to install it separately. This is one
of the trickier releases, so I'll make a few more comments.

```hcl
resource "kubernetes_namespace" "postgres" {
  metadata {
    name = "postgres"
  }
}

resource "kubernetes_persistent_volume" "postgres-data-pv" {
  metadata {
    name = "postgres-data-pv"
  }
  spec {
    storage_class_name = "standard"
    capacity = {
      storage = "8Gi"
    }
    access_modes = ["ReadWriteOnce"]
    persistent_volume_source {
      host_path {
        path = "/mnt/postgres-data"
      } 
    }
  }
}

resource "kubernetes_persistent_volume_claim" "postgres-data-pvc" {
  metadata {
    name = "postgres-data-pvc"
    namespace = "postgres"
  }
  spec {
    access_modes = ["ReadWriteOnce"]
    storage_class_name = "standard"
    resources {
      requests = {
        storage = "8Gi"    
      }
    }
    volume_name = "${kubernetes_persistent_volume.postgres-data-pv.metadata.0.name}"
  }
}

resource "kubernetes_persistent_volume" "postgres-pgdump-pv" {
  metadata {
    name = "postgres-pgdump-pv"
  }
  spec {
    storage_class_name = "standard"
    capacity = {
      storage = "8Gi"
    }
    access_modes = ["ReadWriteOnce"]
    persistent_volume_source {
      host_path {
        path = "/mnt/postgres-pgdump"
      } 
    }
  }
}

resource "kubernetes_persistent_volume_claim" "postgres-pgdump-pvc" {
  metadata {
    name = "postgres-pgdump-pvc"
    namespace = "postgres"
  }
  spec {
    access_modes = ["ReadWriteOnce"]
    storage_class_name = "standard"
    resources {
      requests = {
        storage = "8Gi"    
      }
    }
    volume_name = "${kubernetes_persistent_volume.postgres-pgdump-pv.metadata.0.name}"
  }
}

resource "helm_release" "pg_release" {
  name = "postgres"
  namespace = "postgres"
  repository = "https://charts.bitnami.com/bitnami"
  chart = "postgresql"

  values = [
    <<EOF
    auth:
      database: metrics
      postgresPassword: var.admin_password
    primary:
      service:
        # ClusterIP if you don't want to expose to localhost
        type: LoadBalancer
      persistence:
        enabled: true
        existingClaim: ${kubernetes_persistent_volume_claim.postgres-data-pvc.metadata.0.name}
    backup:
      enabled: true
      cronjob:
        storage:
          existingClaim: ${kubernetes_persistent_volume_claim.postgres-pgdump-pvc.metadata.0.name}
    volumePermissions:
      enabled: true
    metrics:
      enabled: true
    EOF
  ]
}
```

Here, we create 2 persistent volume claims in advance to store both the data and the backups. You can remove the backups
if you like. The PVCs are stored in `host_path` volumes, so you can adjust the path to suit your requirements. **NOTE**
The mount points are in the Podman VM! Do not use host paths that point at your local machine! The release here will test
the LoadBalancer endpoint for connectivity before proceeding, but the tunnel must be running to do so. The install script
will start the tunnel with sudo privileges, so you may be prompted for your password once or twice throughout the course
of the script.

### keycloak

Keycloak is an OAuth2-compliant authn/authz server. It is supported by the Keycloak community and sponsored by Red Hat.
It is really the only serious F/OSS OAuth2-compliant server available, and has feature parity with Okta (if not the 
ease of use).
I'll spend a bit of time on this one because I installed a custom theme as well. Your boss will probably ask for this :).

```hcl
resource "kubernetes_secret" "docker_auth" {
  metadata {
    name = "docker-auth"
    namespace = "keycloak"
  }
  data = {
    ".dockerconfigjson" = "{\"auths\":{\"my-azure-registry.azurecr.io\": {\"username\":\"${var.docker_username}\",\"password\":\"${var.docker_password}\"}}}"
  }
  type = "kubernetes.io/dockerconfigjson"
}

resource "kubernetes_manifest" "keycloak-serviceaccount" {
  manifest = {
    apiVersion = "v1"
    kind = "ServiceAccount"
    metadata = {
      name = "keycloak"
      namespace = "keycloak"
    }
    imagePullSecrets = [{name = kubernetes_secret.docker_auth.metadata[0].name}]
  }
}

resource "helm_release" "keycloak" {
    name = "keycloak"
    namespace = "keycloak"
    create_namespace = true
    repository = "https://charts.bitnami.com/bitnami"
    chart = "keycloak"

    values = [
        <<EOF
initContainers:
- name: postgresql-init
  image: postgres:latest
  imagePullPolicy: IfNotPresent
  env:
  - name: "POSTGRES_PASSWORD"
    value: "${var.admin_password}"
  command:
  - sh
  - -c
  - |
    export PGPASSWORD="$POSTGRES_PASSWORD";
    psql -h ${var.postgres_host} -U postgres -tc "SELECT 'found' FROM pg_database WHERE datname = 'keycloak'" | grep -q found || psql -h ${var.postgres_host} -U postgres -c "CREATE DATABASE keycloak";
    psql -h ${var.postgres_host} -U postgres -c "CREATE USER keycloak WITH PASSWORD '${var.admin_password}'";
    psql -h ${var.postgres_host} -U postgres -c "GRANT ALL ON DATABASE keycloak TO keycloak";
    psql -h ${var.postgres_host} -U postgres -d keycloak -c "GRANT ALL ON SCHEMA public TO keycloak"
- name: copy-theme
  image: my-azure-registry.azurecr.io/keycloak/my-theme
  imagePullSecrets:
  - ${kubernetes_secret.docker_auth.metadata[0].name}
  volumeMounts:
  - name: theme-volume
    mountPath: /keycloak/themes
  command:
    - sh
    - -c
    - cp -R /my-theme /keycloak/themes
serviceAccount:
  name: "keycloak"
  create: false
auth:
  adminUser: keycloak
  adminPassword: ${var.admin_password}
  managementPassword: ${var.admin_password}
postgresql:
  enabled: false
externalDatabase:
  host: postgres-postgresql.postgres.svc.cluster.local
  database: keycloak
  user: keycloak
  password: ${var.admin_password}
initdbScripts:
  copy_theme.sh: |
    #!/bin/sh
    cp -R /keycloak/themes/my-theme /opt/bitnami/keycloak/themes
extraVolumes:
  - name: theme-volume
    emptyDir: {}
extraVolumeMounts:
  - name: theme-volume
    mountPath: /keycloak/themes
proxy: edge
EOF
    ]
    depends_on = [  kubernetes_manifest.keycloak-serviceaccount ]
}

resource "kubernetes_manifest" "keycloak-ingress" {
  manifest = {
    apiVersion = "networking.k8s.io/v1"
    kind = "Ingress"
    metadata = {
      name = "keycloak-ingress"
      namespace = "keycloak"
      annotations = {
        "cert-manager.io/cluster-issuer" = "letsencrypt-staging"
      }
    }
    spec = {
      ingressClassName = "nginx"
      rules = [
        {
          host = "${var.hostname}"
          http = {
            paths = [
              {
                path = "/"
                pathType = "Prefix"
                backend = {
                  service = {
                    name = "keycloak"
                    port = {
                      number = 80
                    }
                  }
                }
              }
            ]
          }
        }
      ]
      tls = [
        {
          hosts = [ "${var.hostname}" ]
          secretName = "keycloak-tls"
        }
      ]
    }
  }
  depends_on = [ helm_release.keycloak ]
}
```

That's a lot to take in. But, there's not too much there, it's just quite verbose. The first thing to note about this
deployment is that it is not entirely secure. Keycloak offers the opportunity to create your own certificate and use that
to provide end-to-end HTTPS encryption without going through the Kubernetes network unencrypted. However, I couldn't get
this to work without TLS errors, so I dropped it. You may have better luck :) The secret we create contains the password
for the `keycloak` login. This is set to `var.admin_password`. We also put the docker configuration in a secret. Be careful!
This secret is only base64-encoded, and contains the plaintext docker authentication. You will give this access to any person with access to the cluster.

We define an init container for Keycloak that initializes the postgres database, so it's quite verbose. But the general
idea is that we use a base image of postgres to provide the `psql` command, and then set up the database using this command.
The rest of it should be reasonably self-explanatory. We customize the helm deployment and install an ingress on the defined
hostname.

### redis

This is very self-explanatory:

```hcl
resource "kubernetes_namespace" "redis" {
  metadata {
    name = "redis"
  }
}

resource "helm_release" "redis" {
    name = "redis"
    namespace = "redis"
    repository = "https://charts.bitnami.com/bitnami"
    chart = "redis"

    values = [
        <<EOF
        architecture: standalone
        tls:
          enabled: true
          authClients: false
          autoGenerated: true
        auth:
          password: "${var.redis_password}"
        master:
          service:
            type: LoadBalancer
        EOF
    ]
}
```

While not used by anything in this installation, it's a good idea to have a redis server available in the cluster. In a
staging/production deployment, you might use a PaaS such as Azure managed Redis instead. 

### monitoring

The monitoring deployment will provide Grafana with Loki already configured. This is an easy way to read/search logs from
the whole cluster, so I'd highly recommend installing this.

```hcl
resource "kubernetes_namespace" "monitoring" {
  metadata {
    name = "monitoring"
  }
}

resource "helm_release" "loki" {
  name       = "loki"
  namespace  = kubernetes_namespace.monitoring.metadata[0].name
  repository = "https://grafana.github.io/helm-charts"
  chart      = "loki-stack"

  set {
    name  = "grafana.enabled"
    value = "true"
  }

  set {
    name = "grafana.adminPassword"
    value = var.admin_password
  }
}

resource "kubernetes_manifest" "grafana-ingress" {
    manifest = {
        apiVersion = "networking.k8s.io/v1"
        kind = "Ingress"
        metadata = {
            name = "grafana-ingress"
            namespace = "monitoring"
            annotations = {
              "cert-manager.io/cluster-issuer" = "letsencrypt-staging"
            }
        }
        spec = {
          ingressClassName = "nginx"
          rules = [
            {
              host = "${var.hostname}"
              http = {
                paths = [
                  {
                    path = "/"
                    pathType = "Prefix"
                    backend = {
                      service = {
                        name = "loki-grafana"
                        port = {
                          number = 80
                        }
                      }
                    }
                  }
                ]
              }
            }
          ]
          tls = [
            {
              hosts = [ "${var.hostname}" ]
              secretName = "grafana-tls"
            }
          ]
       }
    }
    depends_on = [ helm_release.loki ]
}
```

Quite self-explanatory again. A helm release and an ingress on the defined hostname.

### telemetry

This is entirely optional, but an in-memory implementation of a Jaeger collector might just be the additional debugging
information you need for development. Please [see the Github repo](https://github.com/jaegertracing/jaeger) for more
details.

```hcl
resource "kubernetes_namespace" "telemetry" {
    metadata {
      name = "telemetry"
    }
}

resource "helm_release" "jaeger_operator" {
  name = "jaeger-operator"
  namespace = kubernetes_namespace.telemetry.metadata[0].name
  repository = "https://jaegertracing.github.io/helm-charts"
  chart = "jaeger-operator"

  values = [
    <<EOT
    certs:
      issuer:
        create: true
        name: "jaeger-issuer"
      certificate:
        create: true
        namespace: "telemetry"
        secretName: "jaeger-operator-cert"

    jaeger:
      create: true
      spec:
        ingress:
          enabled: true
          ingressClassName: nginx
          annotations:
            cert-manager.io/cluster-issuer: letsencrypt-staging
          hosts:
            - ${var.hostname}
          tls:
            - secretName: jaeger-tls
              hosts:
              - ${var.hostname}
    EOT
  ]
}
```

Jaeger will create its own ingress, so we can simply install the helm release.

## Conclusion

I hope you find this build helpful! Telemetry and observability are critical to Kubernetes development as you cannot attach
a debugger to those processes. I considered other options for telemetry, but the Jaeger all-in-one installation is much
more efficient with resources. This cluster will fit in the default MacOS limits, and doesn't consume too much of the main
system resources. Minikube is easily stopped with `minikube stop` so you only need keep the cluster running as long as you
need it.
