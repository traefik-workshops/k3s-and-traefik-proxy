# Secure the Dashboard with TLS

## Prerequisites

This builds on the work done in the previous sections:

- [Expose the Dashboard](../01-Expose-the-Dashboard/README.md)
- [Secure the Dashboard with Middleware](../02-Expose-the-Dashboard/README.md)
- [Use the IngressRoute Custom Resource](../03-Use-the-IngressRoute-Custom-Resource)

## What's Wrong With the Dashboard?

As I alluded to earlier, our configuration is completely insecure. We're accessing data over HTTP and sending our credentials in plaintext. This just will not do. We want to wrap our communication in TLS encryption, and for that, we'll generate certificates with Let's Encrypt.

## How to Use Let's Encrypt?

In the non-Kubernetes world, integration with Let's Encrypt within a container orchestrator is difficult. Traefik Labs solved this by building support for ACME certificate generation into Traefik Proxy. Certificates are stored in a file called `acme.json`, and if you think about how Docker works, storing this file on a persistent volume is easy.

Traefik Proxy works the same everywhere, which means that in Kubernetes the ACME support still writes out to `acme.json`, but putting this on a PersistentVolume introduces new challenges.

- Do you pin Traefik Proxy to one host by using a HostPath volume?
- If you want to run multiple replicas, do you mount it RWX via NFS? What if you don't have an NFS server?
- Even if you do run multiple replicas, does Traefik Proxy handle file locking, or will each replica clobber the files written by the others?

Traefik Proxy doesn't (yet) store certificates in Secrets, but it can read certificates _from_ Secrets in both the Ingress and IngressRoute resources.

For that reason, and to make sure that you can spin up new replicas before shutting down the old ones, I recommend using [cert-manager](https://cert-manager.io) to generate certificates.

## Set up cert-manager

If you don't have cert-manager running in your environment, it's trivial to install and configure.

### Install cert-manager

According to [the docs](https://cert-manager.io/docs/installation/), the easiest way to install cert-manager is via `kubectl apply` and pointing to [their static manifest](https://github.com/jetstack/cert-manager/releases/download/v1.7.0/cert-manager.yaml) (currently 1.7.0).

Alternatively, you can use [cmctl](https://cert-manager.io/docs/installation/cmctl/) or [Helm](https://cert-manager.io/docs/installation/helm/).

```bash
➤ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.7.0/cert-manager.yaml

...lots of output...
```
namespace/cert-manager created
serviceaccount/cert-manager-cainjector created
serviceaccount/cert-manager created
serviceaccount/cert-manager-webhook created
configmap/cert-manager-webhook created
```

Verify that cert-manager Pods are running before continuing.

```bash
➤ kubectl get po -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-cainjector-57d549777c-2tndk   1/1     Running   0          2m41s
cert-manager-58dff77b6d-fxk64              1/1     Running   0          2m41s
cert-manager-webhook-795c6b46b-flwlm       1/1     Running   0          2m41s
```

## Create a ClusterIssuer

A ClusterIssuer will issue certificates for any request that comes in from any namespace in the cluster. What follows is a sample ClusterIssuer for the Let's Encrypt staging environment.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: user@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: example-issuer-account-key
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          class: nginx
```

This uses the HTTP01 challenge type, and it will create Ingress resources that set `ingressClass: nginx` in their configuration.

An ingress class is usually only present when there are multiple ingress controllers in a cluster. In the case of a default K3s installation, Traefik Proxy doesn't set an ingress class. It is the default (and only) ingress controller.

If we used this ClusterIssuer, we would change the solver to not specify any options for the `http01` key:

```yaml
solvers:
- http01: {}
```

This ClusterIssuer won't work for our environment because we're not connected to the public Internet or using routable hostnames. We could use the DNS01 resolver instead, or for the sake of this class, we can just create a ClusterIssuer for self-signed certificates.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
```

Apply this to your cluster:

```bash
➤ kubectl apply -f clusterissuer.yaml

clusterissuer.cert-manager.io/selfsigned created
```

## Generate a Certificate For the Dashboard

It's possible to have certificates generated automatically through annotations on the Ingress resource, but not only are we not using an Ingress resource, we also want to have control over the resources our cluster generates. Instead of creating them automatically, we'll declare them with a Certificate resource.

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: dashboard
spec:
  dnsNames:
  - 'dashboard.traefik.10.68.0.70.sslip.io'
  issuerRef:
    kind: ClusterIssuer
    name: selfsigned
  secretName: dashboard-crt
```

When we apply this, the Certificate will go through the cert-manager process, and the output will be stored in the Secret `dashboard-crt`.

Using cert-manager abstracts the issuing backend from the requested resource. By changing the issuer, the same Certificate could generate a certificate from Vault, ACME (including Let's Encrypt), Venafi, AWS, FreeIPA, [and others](https://cert-manager.io/docs/configuration/external/).

```bash
➤ kubectl apply -f certificate.yaml

certificate.cert-manager.io/dashboard created

➤ k get secret | grep tls

k3s-serving                                          kubernetes.io/tls                     2      157m
dashboard-crt                                        kubernetes.io/tls                     3      16s
```

## Add the Certificate to the IngressRoute

To use this certificate, we need to make two changes to our IngressRoute:

1. Change the entrypoint from `web` to `websecure`
2. Add a `tls` key that points to the Secret where our certificate is stored.

```bash
➤ kubectl patch ingressroute/traefik-dashboard-secure \
  --type=json \
  --patch-file patch-dashboard-tls.yaml

ingressroute.traefik.containo.us/traefik-dashboard-secure patched
```

Now if you visit https://dashboard.traefik.$CLUSTERIP.sslip.io, you'll be greeted by a certificate warning, and after accepting that terrible risk, you'll be taken to the secured dashboard.
