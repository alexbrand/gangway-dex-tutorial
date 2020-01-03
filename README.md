# Gangway and Dex for Kubernetes Authentication

Using Gangway and Dex for authenticating with your Kubernetes cluster.

## Architecture

The following diagram shows a high-level architecture of the system.

![Deployment Architecture](https://www.lucidchart.com/publicSegments/view/486a4201-8c2e-453a-a468-1c8722086267/image.png "Deployment Architecture")

Gangway and dex run on the cluster as regular deployments. They are exposed using the Ingress API so that users can complete the OIDC flow and obtain credentials.

Once the OIDC flow is complete, the credentials are submitted to the API server, which is configured to use dex as the OIDC provider.

Kubernetes requires a secure connection (HTTPS) to the OIDC provider, so we use cert-manager to automatically provision certificates using Let's Encrypt.

## Prerequisites

- You have access to a Kubernetes cluster and the ability to modify the API server configuration. You will configure the API server to use dex as the OIDC provider.
- You have control of a domain name that supports wildcard CNAME records. You will expose gangway and dex using this domain.

## Components

We will use the following components:

- [gangway](https://github.com/heptiolabs/gangway): OIDC client application
- [dex](https://github.com/coreos/dex): OIDC provider
- [contour](https://github.com/heptio/contour): Kubernetes Ingress controller
- [cert-manager](https://github.com/jetstack/cert-manager): Controller for managing TLS certificates with Let's Encrypt.

## Steps

_Note: This tutorial assumes that Kubernetes is running on AWS with the cloud
provider integration enabled. It should work on other infrastructure, but has
not been tested._

### Deploy Heptio Contour

Run:

```sh
kubectl apply -f https://j.hept.io/contour-deployment-rbac
```

This will deploy Contour in the `heptio-contour` namespace, and expose it using a service of type `LoadBalancer`.

### Configure DNS Record

Get the hostname of the ELB that Kubernetes created for the contour service:

```sh
kubectl get svc -n heptio-contour contour -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Create a wildcard CNAME record that aliases the domain under your control to the hostname of the ELB obtained above. 

For instance, if you own `example.com`, create a CNAME record for `*.kube.example.com`, so that you can access gangway at `https://gangway.kube.example.com`.

### Deploy Cert-Manager

#### Deploy

To deploy cert-manager, run:

```sh
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/v0.4.1/contrib/manifests/cert-manager/with-rbac.yaml
```

#### Update config

Update the `spec.acme.email` field in `deployment/cert-manager/production.yaml`

#### Create certificate issuer

Once cert-manager is deployed, create a certificate issuer:

```sh
kubectl create -f deployment/cert-manager/production-issuer.yaml
```

### Generate OIDC client secret

Gangway and dex need to be configured with a shared secret.

Generate the secret:

```sh
openssl rand -base64 32
```

You will input this secret in gangway and dex's configuration files.

### Deploy Dex

#### Update config

The following table describes placeholders that must be updated before deploying dex.

*Important: Each placeholder can show up multiple times in the same document. Make sure to update all ocurrences.*

| Placeholder | Description |
|----|-------|
| `${DNS_NAME}` | Update the placeholder with the domain that you own. For example, if you own `example.com`, the value of issuer should be `https://dex.example.com/dex` |
| `${OIDC_CLIENT_SECRET}` | This is the secret that was generated before |
| `${DEX_STATIC_ADMIN_PASSWORD_HASH}` | The bcrypt hash of the password you want to use for the admin user |

Update the following files before deploying:

- deployment/dex/01-configmap.yaml
- deployment/dex/03-ingress.yaml

#### Deploy

Once the placeholders have been updated, deploy dex:

```sh
# This command will make sure that you have updated the placeholders
grep -r '\${' deployment/dex || kubectl apply -f deployment/dex
```

### Deploy Gangway

#### Update config

The following table describes placeholders that must be updated before deploying dex.

*Important: Each placeholder can show up multiple times in the same document. Make sure to update all ocurrences.*

| Placeholder | Description |
|----|-------|
| `${CLUSTER_NAME}` | The name of your cluster. This is mainly for display purposes |
| `${DNS_NAME}` | Update the placeholder with the domain that you own. For example, if you own `example.com`, the value of issuer should be `https://dex.example.com/dex` |
| `${OIDC_CLIENT_SECRET}` | This is the secret that was generated before |
| `${KUBERNETES_APISERVER_URL}` | The address of the API server. For example: `https://apiserver.example.com:6443` |

Update the following files before deploying:

- deployment/gangway/02-configmap.yaml
- deployment/gangway/04-ingress.yaml

#### Create secret

Create the gangway cookies that are used to encrypt gangway cookies:

```sh
kubectl -n gangway create secret generic gangway-key \
  --from-literal=sesssionkey=$(openssl rand -base64 32)
```

#### Deploy

Once the placeholders have been updated, deploy:

```sh
# This command will make sure that you have updated the placeholders
grep -r '\${' deployment/gangway || kubectl apply -f deployment/gangway
```

### Configure API server

Add the following flags to your API server configuration:

```yaml
- --oidc-issuer-url=https://dex.<enter your domain here>/dex
- --oidc-client-id=gangway
- --oidc-username-claim=email
- --oidc-groups-claim=groups
```

## Authenticate

Once all components are up and running, you should be able to obtain credentials for your cluster using a browser to access gangway at `https://gangway.<your domain>`.
