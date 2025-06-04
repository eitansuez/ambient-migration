# Setup

The objective of this activity is to construct an initial state, where:

1. Istio is installed in sidecar mode.
2. The sample application, `bookinfo`, is deployed with sidecars.
3. An ingress gateway is deployed, and configured to route requests to the `productpage` service.
4. Both L4 and L7 Authorization Policies are in place and functioning.
5. A traffic policy is in place that routes all requests for the `reviews` service to the `reviews-v3` workload.

## Provisioning a Kubernetes cluster

Feel free to provision a Kubernetes cluster of your choice, locally or in the cloud.

The following snippet installs a local Kubernetes cluster with k3d.  For more information, see [here](https://istio.io/latest/docs/setup/platform-setup/k3d/).

```shell
k3d cluster create my-cluster \
    --api-port 6443 \
    --k3s-arg "--disable=traefik@server:0" \
    --port 80:80@loadbalancer \
    --port 443:443@loadbalancer
```

## Install Istio in sidecar mode

Per the [instructions for installing Istio in sidecar mode with Helm](https://istio.io/latest/docs/setup/install/helm/):

Configure the Helm repository:

```shell
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

Install the Istio CRDs:

```shell
helm install istio-base istio/base -n istio-system \
  --set defaultRevision=default --create-namespace
```

Install the Istio CNI:

```shell
helm install istio-cni istio/cni -n istio-system \
  --set global.platform=k3d --wait
```

!!! note "About the `global.platform` flag"

    The `global.platform` flag is a [requirement on specific platforms](https://istio.io/latest/docs/ambient/install/platform-prerequisites/#k3d) when installing the Istio CNI.

Install the `istiod` control plane:

```shell
helm install istiod istio/istiod -n istio-system --wait
```

### Validate

Verify that both `istiod` and `istio-cni` pods are in Running state, in the `istio-system` namespace:

```shell
kubectl get pod -n istio-system
```

List the helm releases in `istio-system`:

```shell
helm ls -n istio-system
```

## Deploy `bookinfo`

In this scenario, the `bookinfo` services are split across two namespaces:

- the `frontend` namespace hosts the `productpage` service
- the `backend` namespace hosts the services upstream from it: `reviews`, `ratings`, and `details`

Create the namespace:

```shell
kubectl create ns frontend
```

Label the namespace for sidecar injection:

```shell
kubectl label ns frontend istio-injection=enabled
```

Apply the manifests:

```shell
kubectl apply -f bookinfo-frontend.yaml -n frontend 
```

Repeat for the `backend` namespace:

```shell
kubectl create ns backend
kubectl label ns backend istio-injection=enabled
kubectl apply -f bookinfo-backend.yaml -n backend
```

### Validate

To help verify that the services are functioning, deploy a `curl` image to the cluster:

```bash
kubectl apply -n frontend -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/curl/curl.yaml
```

Make a test call to the `ratings` service:

```shell
kubectl exec deploy/curl -n frontend -- curl -s ratings.backend:9080/ratings/123 | jq
```

Call the `reviews` service:

```shell
kubectl exec deploy/curl -n frontend -- curl -s reviews.backend:9080/reviews/123 | jq
```

Finally, call the `productpage` service:

```shell
 kubectl exec deploy/curl -n frontend -- curl -s productpage:9080/productpage | grep title
```

Make sure the calls succeed.

## Configure an ingress gateway

kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

## Configure authorization policies

## Configure traffic policies

Install the Kubernetes Gateway API:

```shell
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```
