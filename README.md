# Kelly's Linkerd 2 Guide
A simple guide with step-by-step commands for setting up [Linkerd](https://linkerd.io) for real use in long lived and/or production Kubernetes clusters.  

While I love [Helm](https://helm.sh/) as much as the next guy, this guide is done entirely without it.  It's important to know that Buoyant does recommend the use of Helm for production installation as it's more repeatable.

This is not intended to be an in-depth discussion, or advanced topics guide for Linkerd or Kubernetes.  I've got just enough setup guide to get it up and running. Please read [Buoyant’s Linkerd Production Runbook](https://buoyant.io/linkerd-runbook/) for yourself to learn a lot more detail.

*Contributions welcome: If you have suggestions on how this can be improved, I welcome any and all feedback.  I also have a million things on my plate, so please don't be offended if I don't reply immediately.*

### Relevancy
This guide is based on Linkerd 2.12.3. Newer or older versions of Linkerd may have differences.  Please always double-check this against the [Linkerd Docs](https://linkerd.io/docs/) before starting.

### Disclaimer
I am nowhere near the foremost expert on Linkerd.  I've just been using it for quite a while and have hit a few pit-falls from configurations that were not exactly production-ready.  The [Linkerd Slack](http://slack.linkerd.io/?_gl=1*1gvbgbs*_ga*ODEyOTQyMDgwLjE2MzQyNTc0NjI.*_ga_TV358ZPK6D*MTY0NDQyMTUwNS43LjAuMTY0NDQyMTUwNS4w#_ga=2.153716476.550972073.1644365071-812942080.1634257462) has much smarter people than me.  I highly recommend you join and ask questions there. 
 

## Setup Instructions
I'm executing all of these commands from the root folder in this repository, on Ubuntu 18.04 ([WSL](https://docs.microsoft.com/en-us/windows/wsl/)). This guide should work as-is on any other OS or distro.

I'm doing this on [AKS](https://azure.microsoft.com/en-us/services/kubernetes-service/), but there is nothing  Azure specific.  This should work on any cloud or even bare-metal Kubernetes cluster.

### Prerequisites
* You need the Linkerd CLI already installed on your workstation.  See Buoyant's [Getting Started](https://linkerd.io/2.11/getting-started/#step-1-install-the-cli) guide for installation instructions.
* You need to have [cert-manager](https://cert-manager.io/), and its CRDs, already installed and working on your cluster.  I may add that as a sub-guide here later but for now I'll just direct you to their [Installation Instructions](https://cert-manager.io/docs/installation/).

### 1) Generate a new, long-lived root certificate
I'm doing this with [smallstep cli](https://smallstep.com/).  If you don't already have it installed, here are their [Install Instructions](https://smallstep.com/docs/step-cli/installation). It is also possible to do this with [OpenSSL](https://www.openssl.org/), but I'm not going to covert that here.
 
Let's generate a 20-year root certificate, so when it needs to be rotated it won't be our problem. 

```bash
step certificate create root.linkerd.cluster.local ca.crt ca.key \
  --profile root-ca --no-password --insecure --not-after 175200h
```

### 2) Create the `linkerd` namespace.
You'll need this to exist for the next few commands to work.

```bash
kubectl create namespace linkerd
```

### 3) Load your new certificate as a secret.
This will be used by Linkerd as the trust anchor for the control plane, mTLS and by cert-manager to issue and rotate shorter lived certificates.

```bash
kubectl create secret tls linkerd-trust-anchor --cert=ca.crt --key=ca.key --namespace=linkerd
```

### 4) Load the included YAML, for cert-manager to create managed certificates
The "[linkerd-cert-issuer.yaml](linkerd-cert-issuer.yaml)" includes the issuers for the trust-ancher and webhooks as well as manager certificate definitions for `linkerd-identity-issuer`, `linkerd-policy-validator`, `linkerd-proxy-injector` and `linkerd-sp-validator`.

I've chosen a certificate lifetime of 180-days with a renew-before of 60-days.  Linkerd's guide actually recommend a much shorter lifetime.  In the real-world I've had older versions of cert-manger "miss" certificate renewals for a few days to weeks.  This gives it more cushion against expiration.  

This longer lifetime also has the benefit of `$ linkerd check` telling you all-is-good if your certificates are rotating properly. Any lifetime of less than 60 days will cause the built-in check to constantly warn you that your certificates are close to expiration.

```bash
kubectl apply -f linkerd-cert-issuer.yaml
```

### 5) Verify your certificates are created
This step can be skipped, if you like to live on the edge.  If cert-manager is working as expected, this command will show a listing of all the certificates mentioned above.

```bash
kubectl get secret -n linkerd
```

The output should look something like this:

```
NAME                                 TYPE                                  DATA   AGE
default-token-f8v2m                  kubernetes.io/service-account-token   3      10m
linkerd-identity-issuer              kubernetes.io/tls                     3      2m
linkerd-policy-validator-k8s-tls     kubernetes.io/tls                     3      2m
linkerd-proxy-injector-k8s-tls       kubernetes.io/tls                     3      2m
linkerd-sp-validator-k8s-tls         kubernetes.io/tls                     3      2m
linkerd-trust-anchor                 kubernetes.io/tls                     2      2m
```

### 6) Ensure you have correctly labeled the `kube-system` namespace

This is really important for HA installation of Linkerd, which we're going to be doing. 

I'll let Buoyant explain the necessity of this in their page on [High Availability](https://linkerd.io/2.11/features/ha/#exclude-the-kube-system-namespace).

```bash
kubectl label namespace kube-system config.linkerd.io/admission-webhooks=disabled
```

### 7) Install Linkerd

We're using the Linkerd CLI installation method.  The arguments passed configure [High Availability](https://linkerd.io/2.11/features/ha/#exclude-the-kube-system-namespace) mode as well as informing Linkerd that the issuer and web-hook certificates will be managed by cert-manager.

*Update for 2.12.x: Added step for `--crds` to create linkerd CRDs as first-step of install*

```bash
linkerd install --crds | kubectl apply -f -
```

*Update for 2.11.2: Added `--set proxyInit.runAsRoot=false` to run init containers as non-root (best practice)*

```bash
linkerd install --ha \
  --identity-external-issuer \
  --identity-trust-anchors-file ca.crt \
  --set policyValidator.externalSecret=true \
  --set-file policyValidator.caBundle=ca.crt \
  --set proxyInjector.externalSecret=true \
  --set-file proxyInjector.caBundle=ca.crt \
  --set profileValidator.externalSecret=true \
  --set-file profileValidator.caBundle=ca.crt \
  --set proxyInit.runAsRoot=false \
  | kubectl apply -f -
```

*NOTE: You are going to get a warning that the `linkerd` namespace already exists outside of `kubectl apply`.  This is expected and can be safely ignored.*

### 8) Verify your Linkerd installation
As before, this step can be skipped, if you like to live on the edge.  We're going use the built-in check feature of Linkerd CLI.

*Give the previous command 1-2 minutes, to install and start the Linkerd workloads, before trying to verify.*

```bash
linkerd check
```

If Linkerd is working as expected, or if there are problems, this tool will tell you.

### Next Steps

This is all that is required to get a production ready Linkerd install up an running.  The existing online [Linkerd Docs](https://linkerd.io/docs/) cover all the details of configuring your other cluster workloads, including ingress, to actually work with your newly installed service-mesh.  I don't feel the need to repeat any of that here.

### Linkerd Viz and Jaeger

If you're interested in using the Viz and Jaeger extensions with this configuration, check out "[extensions.md](extensions.md)" in this repository.

 
