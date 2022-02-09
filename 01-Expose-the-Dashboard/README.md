# Expose the Dashboard

## Set Your Cluster's IP in a Variable

Throughout these lessons we'll be using sslip.io to dynamically map a hostname to a local IP address. The IP address of my node is different from yours, so whenever possible, I'll reference a URL like: `dashboard.traefik.$CLUSTERIP.sslip.io`. For examples with `kubectl` and `curl`, this will work if you set the variable `CLUSTERIP` to the address of your cluster. For pages loaded in the browser you'll have to make that substitution yourself.

To find your cluster IP, cat the kubeconfig of your cluster.

```
cat ~/.kube/config
apiVersion: v1
kind: Config
clusters:
  - name: my-cluster
    cluster:
      server: https://10.68.0.70:6443
```

```bash
# bash
export CLUSTERIP=10.68.0.70

# fish
set -x CLUSTERIP 10.68.0.70
```

## Create the Service

The port for the Traefik Proxy dashboard is not included in the default `traefik` service created when Helm installs Traefik. We need this as a target for our Ingress, so we'll create a new Service.

```bash
kubectl expose deploy/traefik -n kube-system --port=9000 --target-port=9000 --name=traefik-dashboard
```

## Create the Ingress

With the Service created, we can create an Ingress that exposes the dashboard outside of the cluster (remember to specify namespace).

```bash
kubectl create ingress traefik-dashboard --rule="dashboard.traefik.$CLUSTERIP.sslip.io/*=traefik-dashboard:9000" -n kube-system
```

## Visit the Dashboard

```bash
➤ curl -si http://dashboard.traefik.$CLUSTERIP.sslip.io/dashboard/ | head -n 1
HTTP/1.1 200 OK
```

Visit the dashboard in a browser. Look at the Routers under the HTTP section and find our Ingress. Which entrypoints is it listening on?

Access it on port 443 as an HTTP request. Does it answer?

```bash
➤ curl -si http://dashboard.traefik.$CLUSTERIP.sslip.io:443/dashboard/
HTTP/1.1 200 OK
```

## Add the Annotations

If you don't specify an entrypoint, Traefik will answer for the Ingress on all entrypoints. This usually doesn't make sense, since we have different entrypoints for a reason. To control this behavior, we'll add an annotation that tells Traefik the exact entrypoint from which we want this Ingress to be served.

```bash
kubectl annotate ingress traefik-dashboard traefik.ingress.kubernetes.io/router.entrypoints=web -n kube-system
```

Reload the Routers page. Where is the entrypoint for the Ingress now?

What other applications are listening? Can you access any of them with a browser?
