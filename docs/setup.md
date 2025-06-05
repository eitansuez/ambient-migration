# Setup

The objective of this activity is to construct an initial state, where:

1. Istio is installed in sidecar mode.
2. The sample application, `bookinfo`, is deployed with sidecars.
3. An ingress gateway is deployed, and configured to route requests to the `productpage` service.
4. A set of L4 and L7 authorization policies are in place and functioning.
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
kubectl apply -f artifacts/bookinfo-frontend.yaml -n frontend 
```

Repeat for the `backend` namespace:

```shell
kubectl create ns backend
kubectl label ns backend istio-injection=enabled
kubectl apply -f artifacts/bookinfo-backend.yaml -n backend
```

### Validate

Verify that all pods have two containers, implying that the sidecar injection took place:

```shell
kubectl get pod -n frontend
kubectl get pod -n backend
```

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

We have the option to use either the older Istio-specific method of statically provisioning a gateway with Helm, or the Kubernetes Gateway API which allows for the dynamic provisioning of gateways.

We opt for the latter.

Install the [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) standard channel CRDs:

```shell
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

Create the namespace where the Gateway is to be provisioned:

```shell
kubectl create ns istio-ingress
```

Review the Gateway configuration:

```yaml title="gateway.yaml" linenums="1"
--8<-- "gateway.yaml"
```

The gateway is configured to allow the binding of routes defined in the namespace `frontend`.

Apply the Gateway resource:

```shell
kubectl apply -f artifacts/gateway.yaml -n istio-ingress
```

Next, define an `HTTPRoute` to expose specific endpoints on the `productpage` service through the gateway:

```yaml title="ingress-route.yaml" linenums="1"
--8<-- "ingress-route.yaml"
```

Apply the HTTPRoute:

```shell
kubectl apply -f artifacts/ingress-route.yaml -n frontend
```

### Validate

Capture the external IP address of the Gateway:

```shell
export GW_IP=$(kubectl get gtw -n istio-ingress gateway \
  -ojsonpath='{.status.addresses[0].value}')
```

Make a curl request to the ingress gateway using the configured hostname `bookinfo.exmaple.com`:

```shell
curl -s bookinfo.example.com/productpage --resolve bookinfo.example.com:80:$GW_IP | grep title
```

## Configure authorization policies

You will apply three AuthorizationPolicy resources that establish the following policy:

1. Only the ingress gateway should be able to make requests to the `productpage` service.
2. Only `reviews` workloads can call the `ratings` service.
3. Only `productpage` can make calls to the `details` service.

Furthermore:

1. The first policy will be a layer 4 policy that is concerned strictly with workload identity.
2. The remaining policies have layer 7 aspects that allows only specific HTTP methods (for example, GET or POST).

Review all three policies:

```yaml title="productpage-authz.yaml" linenums="1"
--8<-- "productpage-authz.yaml"
```

Note how the identity of the allowed caller is specified via the Istio spiffe id which is a function of the workload's location (namespace) and service account.

```yaml title="ratings-authz.yaml" linenums="1"
--8<-- "ratings-authz.yaml"
```

```yaml title="details-authz.yaml" linenums="1"
--8<-- "details-authz.yaml"
```

Apply all three policies:

```shell
kubectl apply -f artifacts/productpage-authz.yaml -n frontend
```

```shell
kubectl apply -f artifacts/ratings-authz.yaml -n backend
```

```shell
kubectl apply -f artifacts/details-authz.yaml -n backend
```

### Validate

1. A request from an unauthorized workload to the `productpage` service should be denied:

    ```shell
    kubectl exec deploy/curl -n frontend -- \
      curl -s --head productpage:9080/productpage
    ```

2. A request from an unauthorized workload to the `ratings` service should be denied:

    ```shell
    kubectl exec deploy/curl -n frontend -- \
      curl -s ratings.backend:9080/ratings/123
    ```

    It should produce a `HTTP/1.1 403 Forbidden` response.

3. A request from an unauthorized workload to the `details` service should be denied:

    ```shell
    kubectl exec deploy/curl -n frontend -- \
      curl -s details.backend:9080/details/123
    ```

4. A request through the ingress gateway to product page and upstream should succeed:

    ```shell
    curl -s --head bookinfo.example.com/productpage \
      --resolve bookinfo.example.com:80:$GW_IP
    ```

## Configure traffic policies

When `productpage` makes requests against the `reviews` service, the requests are load-balanced across all three versions of the service.

Verify this by making several requests to the `productpage` service and "grepping" for the keyword "reviews-":

```shell
curl -s bookinfo.example.com/productpage --resolve bookinfo.example.com:80:$GW_IP | grep "reviews-"
```

Review the following traffic policy which will route all requests to `reviews-v3`:

```yaml title="route-reviews-v3.yaml" linenums="1"
--8<-- "route-reviews-v3.yaml"
```

Apply the policy:

```shell
kubectl apply -f artifacts/route-reviews-v3.yaml -n backend
```

### Validate

Verify that all requests are routed to version 3 by making repeated calls to `productpage`:

```shell
curl -s bookinfo.example.com/productpage --resolve bookinfo.example.com:80:$GW_IP | grep "reviews-"
```

## Summary

We have configured our initial state:  a system of microservices functioning with Istio in sidecar mode, with a combination of L4 and L7 security policies, and a traffic policy applied to the `reviews` service.

In the next section, we will work on migrating this system to Istio ambient mode.