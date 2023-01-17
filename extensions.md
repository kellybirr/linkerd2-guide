# Linkerd 2 Exensions
This guide covers a complimentary install of the Viz and/or Jaeger extensions in long lived and/or production Kubernetes clusters.  Everything in this guide assumes you've already installed Linkerd using the process in the [README](README.md) and still have the CA certificate you generated.

## VIZ

### 1) Create the `linkerd-viz` namespace.
You'll need this to exist for the next few commands to work.

```bash
kubectl create namespace linkerd-viz
```

### 2) Load your CA certificate as a secret.
This will be used by Linkerd and cert-manager to issue and rotate shorter lived web-hook certificates.

```bash
kubectl create secret tls webhook-issuer-tls --cert=ca.crt --key=ca.key --namespace=linkerd-viz
```

### 3) Load the included YAML, for cert-manager to create managed certificates
The "[viz-issuer.yaml](viz-issuer.yaml)" includes the issuer for web-hooks as well as managed certificate definitions for `tap`, and `linkerd-tap-injector`.

```bash
kubectl apply -f viz-issuer.yaml
```

### 4) Verify your certificates are created
This step can be skipped, if you like to live on the edge.  If cert-manager is working as expected, this command will show a listing of the certificates mentioned above.

```bash
kubectl get secret -n linkerd-viz
```

The output should look something like this:

```
NAME                       TYPE                                  DATA   AGE
default-token-ngc9f        kubernetes.io/service-account-token   3      3m
tap-injector-k8s-tls       kubernetes.io/tls                     3      1m
tap-k8s-tls                kubernetes.io/tls                     3      1m
webhook-issuer-tls         kubernetes.io/tls                     2      1m
```

### 5) Install Linkerd Viz Extension

We're using the Linkerd CLI installation method.  

```bash
linkerd viz install \
  --set tap.externalSecret=true \
  --set-file tap.caBundle=ca.crt \
  --set tapInjector.externalSecret=true \
  --set-file tapInjector.caBundle=ca.crt \
  | kubectl apply -f -
```

*NOTE: You are going to get a warning that the `linkerd-viz` namespace already exists outside of `kubectl apply`.  This is expected and can be safely ignored.*

### 8) Verify your installation
As before, this step can be skipped, if you like to live on the edge.  We're going use the built-in check feature of Linkerd CLI.

*Give the previous command 1-2 minutes, to install and start the Linkerd-Viz workloads, before trying to verify.*

```bash
linkerd check
```

If Viz is working as expected, or if there are problems, this tool will tell you.


## JAEGER
The process is virtually identical to installing Viz (above).

### 1) Create the `linkerd-jaeger` namespace.

```bash
kubectl create namespace linkerd-jaeger
```

### 2) Load your CA certificate as a secret.

```bash
kubectl create secret tls webhook-issuer-tls --cert=ca.crt --key=ca.key --namespace=linkerd-jaeger
```

### 3) Load the included YAML, for cert-manager to create managed certificates
The "[jaeger-issuer.yaml](jaeger-issuer.yaml)" includes the issuer for web-hooks and managed certificate definition for `jaeger-injector`.

```bash
kubectl apply -f jaeger-issuer.yaml
```

### 4) Verify your certificates are created

```bash
kubectl get secret -n linkerd-jaeger
```

### 5) Install Linkerd Jaeger Extension

```bash
linkerd jaeger install \
  --set webhook.externalSecret=true \
  --set-file webhook.caBundle=ca.crt \
  | kubectl apply -f -
```

*NOTE: You are going to get a warning that the `linkerd-jaeger` namespace already exists outside of `kubectl apply`.  This is expected and can be safely ignored.*

### 6) Verify your installation

*Give the previous command 1-2 minutes, to install and start the Linkerd-Jaeger workloads, before trying to verify.*

```bash
linkerd check
```

