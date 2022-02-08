# Secure the Dashboard with TLS

## Prerequisites

This builds on the work done in the previous sections:

- [Expose the Dashboard](../01-Expose-the-Dashboard/README.md)
- [Secure the Dashboard with Middleware](../02-Expose-the-Dashboard/README.md)
- [Use the IngressRoute Custom Resource](../03-Use-the-IngressRoute-Custom-Resource)

## What's Wrong With the Dashboard?

As I alluded to earlier, our configuration is completely insecure. We're accessing data over HTTP and sending our credentials in plaintext. This just _will not do_. We want to wrap our communication in TLS encryption, and for that, we'll generate certificates with cert-manager.

## Doesn't Traefik Proxy Come With Let's Encrypt Support?

In the non-Kubernetes world, integration with Let's Encrypt within a container orchestrator is difficult. Traefik Labs solved this by building support for ACME certificate generation into Traefik Proxy. Certificates are stored in a file called `acme.json`, and if you think about how Docker works, storing this file on a persistent volume is easy.

Traefik Proxy works the same everywhere, which means that in Kubernetes the ACME support still writes out to `acme.json`, but putting this on a PersistentVolume introduces new challenges.

- Do you pin Traefik Proxy to one host by using a Local volume?
- If you want to run multiple replicas, can you mount it RWX via NFS? What if you don't have an NFS server?
- Even if you do run multiple replicas, does Traefik Proxy handle file locking, or will each replica clobber the files written by the others?

These are not new problems, and the solution for Kubernetes is to store each certificate in its own Secret.

Unfortunately, _because_ Traefik Proxy works the same on every orchestrator, and because every orchestrator doesn't have a Secret resource, Traefik Proxy doesn't (yet) store certificates in Secrets. It's not the end of the world, though, because it can read certificates _from_ Secrets in both the Ingress and IngressRoute resources.

For that reason, and to make sure that you can spin up new replicas before shutting down the old ones, we recommend using [cert-manager](https://cert-manager.io) to generate certificates in Kubernetes.

## Set up cert-manager

If you don't have cert-manager running in your environment, it's trivial to install and configure.

### Install cert-manager

According to [the docs](https://cert-manager.io/docs/installation/), the easiest way to install cert-manager is via `kubectl apply` and pointing to [their static manifest](https://github.com/jetstack/cert-manager/releases/download/v1.7.0/cert-manager.yaml) (currently 1.7.0).

Alternatively, you can use [cmctl](https://cert-manager.io/docs/installation/cmctl/) or [Helm](https://cert-manager.io/docs/installation/helm/).

```bash
➤ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.7.0/cert-manager.yaml
...
namespace/cert-manager created
serviceaccount/cert-manager-cainjector created
serviceaccount/cert-manager created
serviceaccount/cert-manager-webhook created
configmap/cert-manager-webhook created
...lots of output...
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

A ClusterIssuer will issue certificates for any request that comes in from any namespace in the cluster. What follows is [a sample ClusterIssuer](https://cert-manager.io/docs/configuration/acme/#creating-a-basic-acme-issuer) for the Let's Encrypt staging environment.

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

This ClusterIssuer won't work for our demo environment because we're not connected to the public Internet or using routable hostnames. We could use the DNS01 resolver instead, or for the sake of this class, we can just create a ClusterIssuer for self-signed certificates.

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

It's possible with cert-manager to have certificates generated automatically through annotations on the Ingress resource, but not only are we not using an Ingress resource, we also want to have control over the resources our cluster generates. Instead of creating them automatically, we'll declare them with a Certificate resource.

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

> **NOTE:** Change `10.68.0.70` to your own node's IP.

When we apply this, the Certificate will go through the cert-manager process, and the output will be stored in the Secret `dashboard-crt`.

Using cert-manager abstracts the issuing backend from the requested resource. By changing the issuer, the same Certificate could generate a certificate from HashiCorp Vault, ACME (including Let's Encrypt), Venafi, AWS, FreeIPA, [and others](https://cert-manager.io/docs/configuration/external/).

```bash
➤ kubectl apply -f certificate.yaml

certificate.cert-manager.io/dashboard created

➤ k get secret | grep tls

k3s-serving                      kubernetes.io/tls                     2      157m
dashboard-crt                    kubernetes.io/tls                     3      16s
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

> **PRO TIP:** If you're using a Chromium-based browser (Chrome, Vivaldi, Edge, etc.) and get an error that the certificate data is scrambled and you've been saved by the browser from a terrible fate, just type `thisisunsafe` over the browser window to force the page to load. 🤯

## Add a Redirect for HTTP

This is all good, but now we don't have anything listening on HTTP. Let's create a new IngressRoute and Middleware to bounce HTTP to HTTPS.

```bash
➤ kubectl apply -f middleware-scheme.yaml

middleware.traefik.containo.us/redirect-permanent created

➤ kubectl apply -f ingressroute.yaml

ingressroute.traefik.containo.us/traefik-dashboard-http created
```

Now when you visit the dashboard over HTTP you'll be redirected to HTTPS:

```bash
➤ curl -si http://dashboard.traefik.10.68.0.70.sslip.io | head -n 1

HTTP/1.1 301 Moved Permanently
```
