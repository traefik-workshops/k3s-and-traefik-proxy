# Use the IngressRoute Custom Resource

## Prerequisites

This builds on the work done in the previous sections:

- [Expose the Dashboard](../01-Expose-the-Dashboard/README.md)
- [Secure the Dashboard with Middleware](../02-Expose-the-Dashboard/README.md)

## Change Ingresses to IngressRoutes

Creating two Ingresses feels inefficient, so let's redo that using the IngressRoute Custom Resource.

First, remove the Ingresses that we created.

```bash
➤ kubectl delete ingress/traefik-dashboard ingress/traefik-dashboard-api
ingress.networking.k8s.io "traefik-dashboard" deleted
ingress.networking.k8s.io "traefik-dashboard-api" deleted
```

Next, create [a new IngressRoute](ingressroute.yaml) that combines both Ingresses and the middleware into a single resource.

```bash
➤ sed -i "s/10\.68\.0\.70/${CLUSTERIP}/" ingressroute.yaml

➤ kubectl apply -f ingressroute.yaml

ingressroute.traefik.containo.us/traefik-dashboard-secure created
```

_How many of you are wondering why it's called `traefik-dashboard-secure` when it's listening on port 80 and sending passwords over plaintext? Bonus points for you! We'll fix that in the next section._

Visit http://traefik.dashboard.$CLUSTERIP.sslip.io in a browser, and once again you'll see the dashboard, secured with basic auth, handling the prefix for `/dashboard` and routing correctly to `/api`. This is where we were at the end of the previous section, but now we're using a single IngressRoute resource instead of two Ingresses. We can add as many Rules as we need to handle routing and middleware for our application.

## Take a Closer Look at the IngressRoute

Why is it called `traefik-dashboard-secure`? Why not just `traefik-dashboard`?

It turns out that there's already an IngressRoute called `traefik-dashboard` that's installed with Traefik. What is it doing?

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  annotations:
    helm.sh/hook: post-install,post-upgrade
  labels:
    app.kubernetes.io/instance: traefik
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: traefik
    helm.sh/chart: traefik-9.18.2
  name: traefik-dashboard
  namespace: kube-system
spec:
  entryPoints:
  - traefik
  routes:
  - kind: Rule
    match: PathPrefix(`/dashboard`) || PathPrefix(`/api`)
    services:
    - kind: TraefikService
      name: api@internal
```

Its listening on the `traefik` entrypoint (port 9000), handling `/dashboard` and `/api` and routing it to a resource we haven't seen yet: a TraefikService named `api@internal`.

## What's a TraefikService?

TraefikServices are different from Kubernetes Services - it's the [CRD implementation of a Traefik Service in Kubernetes](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/#kind-traefikservice). If the `kind` key is not specified in an IngressRoute, then the `name` defaults to a Kubernetes Service.

We use a TraefikService to control how client requests are load balanced across backend servers, or in this case, to route to a special internal Traefik endpoint: the API.

This IngressRoute is how traffic to port 9000 finds its way to the dashboard. If we remove it, the dashboard disappears, so it must be important...but much of its configuration overlaps with our own. In fact, we're sending our traffic from port 80 to port 9000, going from IngressRoute to IngressRoute only to end up at the `api@internal` service. What if we patch our IngressRoute to go directly to `api@internal` and cut out the middleman?

```bash
➤ kubectl patch ingressroute/traefik-dashboard-secure --type=json --patch-file patch-dashboard-service.yaml

ingressroute.traefik.containo.us/traefik-dashboard-secure patched
```

Now we're going straight from port 80 to the Traefik Service handling the API and Dashboard. We can remove our Service object entirely.

```bash
➤ kubectl delete service traefik-dashboard

service "traefik-dashboard" deleted
```

If you visit the dashboard now, you'll see that it still works fine.

```bash
➤ curl -si -u admin@example.com:admin1234 http://dashboard.traefik.$CLUSTERIP.sslip.io/ | head -n 1

HTTP/1.1 200 OK
```

## Why Not Just Edit the `traefik-dashboard` IngressRoute?

The `traefik-dashboard` IngressRoute is controlled by Helm, which means that if we edit it directly, we don't know that our changes won't be clobbered by a future upgrade or other change. It's better to leave it alone (and unused), answering requests that come into port 9000, and create our own IngressRoute that does what we want it to do.
