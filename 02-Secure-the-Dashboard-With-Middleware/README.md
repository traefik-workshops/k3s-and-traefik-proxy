# Secure the Dashboard with Middleware

## Prerequisites

This builds on the work done in the previous section:

- [Expose the Dashboard](../01-Expose-the-Dashboard/README.md)

## What's a Middleware?

Middleware is how Traefik modifies requests before sending them to the Services. These components can change the request scheme, path, insert authentication, connect to external authentication services, handle rate limiting, [and more](https://doc.traefik.io/traefik/middlewares/overview/).

We'll use middleware to add authentication to the dashboard.

## Create a Middleware for Basic Authentication

Our Middleware will refer to a Secret that contains our users and their passwords. This file is in the standard `htpasswd` format that has been used by webservers for decades. Each line contains a username and an encrypted password, separated by a colon.

> **PRO TIP:** This repository contains a sample `users` file with two users inside. The passwords are `user1234` and `admin1234`.

### Create the `users` file

Create the users with `htpasswd`. If you don't have `htpasswd` on your system, you can use an [online generator](https://www.web2generators.com/apache-tools/htpasswd-generator) to create the entries and then copy them.

```bash
# create a new file named "users" with the -c option
➤ htpasswd -c users user@example.com
New password:
Re-type new password:
Adding password for user user@example.com

# add new users to an existing file named "users"
➤ htpasswd users admin@example.com
New password:
Re-type new password:
Adding password for user admin@example.com
```

### Create the `dashboard-users` Secret from the `users` file

```bash
kubectl create secret generic dashboard-users --from-file=users
secret/dashboard-users created
```

### Create the Middleware from `middleware-auth.yaml` (included)

```bash
➤ kubectl apply -f middleware-auth.yaml

middleware.traefik.containo.us/dashboard-auth created
```

## Apply the Middleware to the Ingress

The Middleware and the Secret containing our users exist, but we have to tell Traefik to use them with the Ingress. We do that by annotating the ingress with `traefik.ingress.kubernetes.io/router.middlewares`, pointing to a list of middlware we want the Ingress to use.

```bash
➤ kubectl annotate ingress traefik-dashboard \
  traefik.ingress.kubernetes.io/router.middlewares=kube-system-dashboard-auth@kubernetescrd

  ingress.networking.k8s.io/traefik-dashboard annotated
```

> **PRO TIP:** Middlware order _matters_. When chaining multiple Middlware together, consider which needs to come first. Do you want to do authentication before a redirect or rate limiting?

## Test the Middleware

Visit the dashboard and see that it's now secured with basic authentication:

```
➤ curl -si http://dashboard.traefik.$CLUSTERIP.sslip.io/dashboard/ | head -n 1

HTTP/1.1 401 Unauthorized
```

Try again with credentials:

```
➤ curl -si -u 'admin@example.com:admin1234' http://dashboard.traefik.$CLUSTERIP.sslip.io/dashboard/ | head -n 1

HTTP/1.1 200 OK
```

## Create a Middleware to add the `/dashboard` prefix

The dashboard responds on the `/dashboard/` path (with the trailing slash). Since we have a dedicated hostname pointing to the dashboard, we don't need to always type that. We can have Traefik prepend `/dashboard` to a request for `/`, resulting in a final request of `/dashboard/` arriving at the backend server.

```
➤ kubectl apply -f middleware-rewrite.yaml

middleware.traefik.containo.us/dashboard-rewrite created
```

> **PRO TIP:** Be careful with your rewrites. If you prepend `/dashboard/` (with the trailing slash), your rewritten path will be `/dashboard//` and will result in an eternal redirect loop until your browser cancels the request. Because it's a permanent redirect, you'll have to clear your browsing history for the previous hour to undo that mistake.

## Apply the Middleware to the Ingress

We can now add this Middleware to the annotation, specifying `--overwrite` to tell Kubernetes to replace the existing annotation.

```
➤ kubectl annotate ingress traefik-dashboard \
  traefik.ingress.kubernetes.io/router.middlewares=kube-system-dashboard-rewrite@kubernetescrd,kube-system-dashboard-auth@kubernetescrd \
  --overwrite=true

ingress.networking.k8s.io/traefik-dashboard annotated
```

## Visit the Dashboard Host Without `/dashboard/`

```bash
➤ curl -si http://dashboard.traefik.$CLUSTERIP.sslip.io/ | head -n 1
HTTP/1.1 200 OK
```

## Fix the Broken Dashboard

If you visit this in a browser, you'll find that the page doesn't load correctly. When you look in the developer console, you'll see that requests to `/api` return a 404.

This is because those requests are being rewritten as `/dashboard/api`, which is incorrect. To fix this, we need to create a separate Ingress to handle those requests. This new Ingress will not have the annotations but _should_ have the basic auth middleware to keep that endpoint secure.

```bash
➤ kubectl create ingress traefik-dashboard-api --rule="dashboard.traefik.$CLUSTERIP.sslip.io/api/*=traefik-dashboard:9000"

ingress.networking.k8s.io/traefik-dashboard-api created

➤ kubectl annotate ingress traefik-dashboard-api \
  traefik.ingress.kubernetes.io/router.middlewares=kube-system-dashboard-auth@kubernetescrd

ingress.networking.k8s.io/traefik-dashboard-api annotated
```

Now when you visit the dashboard by its unique hostname, everything is rewritten and loads for you.

This was a lot of work. Is there a better way? Read on!
