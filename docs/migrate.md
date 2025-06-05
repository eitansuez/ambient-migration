# Migrate to Ambient

## Download the migration tool

Per the [documented instructions](https://ambientmesh.io/docs/setup/sidecar-migration/#download-the-migration-helper), we can obtain the migration helper with:

```shell
curl -sL https://storage.googleapis.com/gloo-cli/install.sh | sh
```

The output tells us that we can add the `gloo` binary to our path with:

```shell
export PATH=$HOME/.gloo/bin:$PATH
```

The command comes with different options flag, but the general form of the command we will run is:

```shell
gloo ambient migrate
```

## Try the migration tool

Try the migration helper now:

```shell
gloo ambient migrate
```

Study the output:

```console
• Starting phase pre-reqs...
✅ Phase pre-reqs succeeded!
  ✅ Cluster CNI compatibility: passed
  ✅ Istio version compatibility: passed
  ✅ Multicluster usage compatibility: passed
  ✅ Virtual Machine usage compatibility: passed
  ✅ SPIRE usage compatibility: passed

• Starting phase cluster-setup...
❌ Phase cluster-setup failed!
  ❌ Ambient mode enabled: failed. 1 error occurred:
        * istiod istio-system/istiod must have 'PILOT_ENABLE_AMBIENT=true'. Upgrade Istio with '--set profile=ambient'.


  ❌ DaemonSets deployed: failed. 1 error occurred:
        * ztunnel not found


  ❌ Sidecars support ambient mode: failed. 7 errors occurred:
        * Sidecar backend/details-v1-7d775cb4f6-s657b is missing 'ENABLE_HBONE'. Upgrade Istio with '--set profile=ambient' and restart the pod.
        * Sidecar backend/ratings-v1-5b896f8544-h7qw6 is missing 'ENABLE_HBONE'. Upgrade Istio with '--set profile=ambient' and restart the pod.
        * Sidecar backend/reviews-v1-746f96c9d4-55wxl is missing 'ENABLE_HBONE'. Upgrade Istio with '--set profile=ambient' and restart the pod.
        * Sidecar backend/reviews-v2-97bdf5876-r5kg9 is missing 'ENABLE_HBONE'. Upgrade Istio with '--set profile=ambient' and restart the pod.
        * Sidecar backend/reviews-v3-77d9db6844-fbwf6 is missing 'ENABLE_HBONE'. Upgrade Istio with '--set profile=ambient' and restart the pod.
        * Sidecar frontend/curl-5b549b49b8-xthm5 is missing 'ENABLE_HBONE'. Upgrade Istio with '--set profile=ambient' and restart the pod.
        * Sidecar frontend/productpage-v1-85b8f8d74b-tsw5q is missing 'ENABLE_HBONE'. Upgrade Istio with '--set profile=ambient' and restart the pod.


  ✅ Required CRDs installed: passed
```

This output should begin to provide a validation for our plan:

- Among other things, it validates Istio version and Cluster CNI compatibility.  If we had used too old a version of Istio, the tool would have raised a concern, and our first priority would be to ugprade Istio to a newer version.

- We also get a validation that all necessary CRDs are installed.  I assume the reference is to the Gateway API CRDs.

The tool also helps us look forward and tells us what's not yet in place:

- We haven't yet upgraded to ambient mode.  We're told to use the `--set profile=ambient` flag.
- ztunnel is not yet installed

Let's get to work.

## Upgrade Istio

### Upgrade the CNI to use the ambient profile:

```shell
helm upgrade istio-cni istio/cni -n istio-system \
  --set global.platform=k3d --set profile=ambient --wait
```

### Upgrade `istiod` with the ambient profile

```shell
helm upgrade istiod istio/istiod -n istio-system --set profile=ambient --wait
```

### Install ztunnel

```shell
helm install ztunnel istio/ztunnel -n istio-system --set global.platform=k3d --wait
```

### Validate the upgrade

List all Helm installations:

```shell
helm ls -A
```

The output should corroborate that all components are installed, along with their versions.

```console
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
istio-base      istio-system    1               2025-06-04 15:01:02.654247 +0700 +07    deployed        base-1.26.1     1.26.1
istio-cni       istio-system    2               2025-06-05 11:34:51.192029 +0700 +07    deployed        cni-1.26.1      1.26.1
istiod          istio-system    2               2025-06-05 11:34:59.327015 +0700 +07    deployed        istiod-1.26.1   1.26.1
ztunnel         istio-system    1               2025-06-05 11:35:38.388524 +0700 +07    deployed        ztunnel-1.26.1  1.26.1
```

Next, list all pods in the `istio-system` namespace:

```shell
kubectl get pod -n istio-system
```

The output should indicate that the CNI agent, `ztunnel`, and `istiod` are all running:

```console
NAME                      READY   STATUS    RESTARTS   AGE
istio-cni-node-wsld7      1/1     Running   0          2m25s
istiod-86849665c6-87lvg   1/1     Running   0          2m17s
ztunnel-zkzn8             1/1     Running   0          99s
```

## Migration assistant, what's next?

This is a good time to re-run the migration tool to see where we stand:

```shell
gloo ambient migrate
```

The output has changed:

```console
• Starting phase pre-reqs...
✅ Phase pre-reqs succeeded!
  ✅ Cluster CNI compatibility: passed
  ✅ Istio version compatibility: passed
  ✅ Multicluster usage compatibility: passed
  ✅ Virtual Machine usage compatibility: passed
  ✅ SPIRE usage compatibility: passed

• Starting phase cluster-setup...
❌ Phase cluster-setup failed!
  ✅ Ambient mode enabled: passed
  ✅ DaemonSets deployed: passed
  ❌ Sidecars support ambient mode: failed. 7 errors occurred:
        * Sidecar backend/details-v1-7d775cb4f6-s657b is missing 'ENABLE_HBONE'. Upgrade Istio with '--set profile=ambient' and restart the pod.
        * Sidecar backend/ratings-v1-5b896f8544-h7qw6 is missing 'ENABLE_HBONE'. Upgrade Istio with '--set profile=ambient' and restart the pod.
        * Sidecar backend/reviews-v1-746f96c9d4-55wxl is missing 'ENABLE_HBONE'. Upgrade Istio with '--set profile=ambient' and restart the pod.
        * Sidecar backend/reviews-v2-97bdf5876-r5kg9 is missing 'ENABLE_HBONE'. Upgrade Istio with '--set profile=ambient' and restart the pod.
        * Sidecar backend/reviews-v3-77d9db6844-fbwf6 is missing 'ENABLE_HBONE'. Upgrade Istio with '--set profile=ambient' and restart the pod.
        * Sidecar frontend/curl-5b549b49b8-xthm5 is missing 'ENABLE_HBONE'. Upgrade Istio with '--set profile=ambient' and restart the pod.
        * Sidecar frontend/productpage-v1-85b8f8d74b-tsw5q is missing 'ENABLE_HBONE'. Upgrade Istio with '--set profile=ambient' and restart the pod.


  ✅ Required CRDs installed: passed
```

In ambient mode, sidecar workloads must be have the HBONE protocol enabled in order to communicate with sidecarless workloads.

In order to effect that change, the assistant instructs us to restart the pods that represent our workloads.

This is a good example of a step that we hadn't considered in our plan.

In scenarios where we expect to run a mix of workloads, some sidecar-based and others sidecarless, this step would be important.

Go ahead and heed the recommendation by restarting the workloads:

```shell
kubectl rollout restart deploy -n backend
```
And:

```shell
kubectl rollout restart deploy -n frontend
```

Note that the above are still running sidecars.

## Assistant, what's next?

You know the drill by now:

```shell
gloo ambient migrate
```

Here is the salient output:

```console
• Starting phase deploy-waypoints...
⚠ Phase deploy-waypoints has recommendations!
  🔮 Namespace "backend" might require a waypoint for the following services:
     * Service "backend/details" selected workload apps/v1/Deployment/backend/details-v1 requires a waypoint:
         AuthorizationPolicy backend/details-authz requires a waypoint due to HTTP attributes (methods)

     * Service "backend/details-v1" selected workload apps/v1/Deployment/backend/details-v1 requires a waypoint:
         AuthorizationPolicy backend/details-authz requires a waypoint due to HTTP attributes (methods)

     * Service "backend/ratings" selected workload apps/v1/Deployment/backend/ratings-v1 requires a waypoint:
         AuthorizationPolicy backend/ratings-authz requires a waypoint due to HTTP attributes (methods)

     * Service "backend/ratings-v1" selected workload apps/v1/Deployment/backend/ratings-v1 requires a waypoint:
         AuthorizationPolicy backend/ratings-authz requires a waypoint due to HTTP attributes (methods)


  ℹ Generated waypoints written to /tmp/istio-migrate/recommended-waypoints.yaml
```

The assistant has validated our analysis that:

- The authorization policies on backend services require a waypoint.
- The authorization policy on frontend services do not necessitate a waypoint.

It's interesting that it hasn't determined that `reviews` needs a waypoint too, on account of its traffic policy.

On the other hand, it's awfully nice to see a generated waypoint provided by the assistant:

```shell
cat /tmp/istio-migrate/recommended-waypoints.yaml
```

Here is the generated waypoint, it's a Gateway resource:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: waypoint
  namespace: backend
spec:
  gatewayClassName: istio-waypoint
  listeners:
  - name: mesh
    port: 15008
    protocol: HBONE
```

## Deploy the waypoint

Apply the recommended waypoint:

```shell
kubectl apply -f /tmp/istio-migrate/recommended-waypoints.yaml
```

Note however that just applying the waypoint does not also bind it to specific workloads.

Verify that the waypoint is present in the `backend` namespace:

```shell
istioctl waypoint list -n backend
```

Here is the output:

```console
NAME         REVISION     PROGRAMMED
waypoint     default      True
```

## Assistant..?

Re-run the assistant:

```shell
gloo ambient migrate
```

Study the output:

```console
...
• Starting phase migrate-policies...
⚠ Phase migrate-policies has recommendations!
  🔮 Apply security.istio.io/v1/AuthorizationPolicy/backend/details-from-waypoint: v1/Service/backend/details must allow traffic from its waypoint.
  🔮 Apply security.istio.io/v1/AuthorizationPolicy/backend/details-details-authz: Existing configuration is copied from policy backend/details-authz to be enforced at the waypoint.
  🔮 Apply security.istio.io/v1/AuthorizationPolicy/backend/details-v1-from-waypoint: v1/Service/backend/details-v1 must allow traffic from its waypoint.
  🔮 Apply security.istio.io/v1/AuthorizationPolicy/backend/details-v1-details-authz: Existing configuration is copied from policy backend/details-authz to be enforced at the waypoint.
  🔮 Apply security.istio.io/v1/AuthorizationPolicy/backend/ratings-from-waypoint: v1/Service/backend/ratings must allow traffic from its waypoint.
  🔮 Apply security.istio.io/v1/AuthorizationPolicy/backend/ratings-ratings-authz: Existing configuration is copied from policy backend/ratings-authz to be enforced at the waypoint.
  🔮 Apply security.istio.io/v1/AuthorizationPolicy/backend/ratings-v1-from-waypoint: v1/Service/backend/ratings-v1 must allow traffic from its waypoint.
  🔮 Apply security.istio.io/v1/AuthorizationPolicy/backend/ratings-v1-ratings-authz: Existing configuration is copied from policy backend/ratings-authz to be enforced at the waypoint.
  ℹ Recommended policies written to /tmp/istio-migrate/recommended-policies.yaml
```

We get a mention that the waypoint will be used for specific backend services.
We make a mental note of that.

The salient section tells us what we expected:  that our authorization policies must be retrofitted to use a `targetRefs` field.

Rather than do the work ourselves, the tool has provided the resources in the file `recommended-policies.yaml`.

## Apply retrofitted authorization policies

It's worth mentioned that we are not yet deleting any existing authoriation policies.
Rather, we add the equivalent policies that will function in the context of the waypoints.

```shell
kubectl apply -f /tmp/istio-migrate/recommended-policies.yaml
```

A close look at the recommended policies shows that the tool is doing its due diligence to permit requests that target either the `ratings` and `ratings-v1` services, and similarly for the `details` and `details-v1` services.

## Bind the waypoint to the services

Another run of the assistant points this out:

```console
• Starting phase use-waypoints...
⚠ Phase use-waypoints has recommendations!
  ⚠ Waypoint backend/waypoint is not used by any services.
  🔮 Service v1/Service/backend/details requires a waypoint, but is not configured to use one. To configure it: kubectl label service -n backend details istio.io/use-waypoint=waypoint
  🔮 Service v1/Service/backend/details-v1 requires a waypoint, but is not configured to use one. To configure it: kubectl label service -n backend details-v1 istio.io/use-waypoint=waypoint
  🔮 Service v1/Service/backend/ratings requires a waypoint, but is not configured to use one. To configure it: kubectl label service -n backend ratings istio.io/use-waypoint=waypoint
  🔮 Service v1/Service/backend/ratings-v1 requires a waypoint, but is not configured to use one. To configure it: kubectl label service -n backend ratings-v1 istio.io/use-waypoint=waypoint
```

It suggests binding the waypoint to our services by labeling each service with the `istio.io/use-waypoint` conventional label.

Here we know that in addition to `ratings` and `details`, we must also label `reviews` services to use the waypoint, in order to support the traffic policy that is in place for `reviews`.

It seems silly to label every single service in the `backend` namespace.
Since in this case all workloads in `backend` should be enrolled to use the waypoint, let's just label the namespace:

```shell
kubectl label namespace backend istio.io/use-waypoint=waypoint
```

Re-run the assistant to validate that all services have the associated waypoint.

## Switch to ambient mode

It's time to finally switch to ambient mode.

### Remove the `istio-injection` labels

```shell
kubectl label namespace frontend istio-injection-
kubectl label namespace backend istio-injection-
```

### Add the `dataplane-mode` label

```shell
kubectl label namespace frontend istio.io/dataplane-mode=ambient
kubectl label namespace backend istio.io/dataplane-mode=ambient
```

This label will ensure that ztunnel intercepts traffic in and out of our workloads.

We can finally remove the sidecars by restarting the workloads:

```shell
kubectl rollout restart deploy -n frontend
kubectl rollout restart deploy -n backend
```

### Validate

Verify that all pods in `frontend` and `backend` have a single container:

```shell
kubectl get pod -n frontend
kubectl get pod -n backend
```

## Assistant, one more time

```shell
gloo ambient migrate
```

Here is the output:

```console
• Starting phase policy-simplification...
⚠ Phase policy-simplification has recommendations!
  🔮 AuthorizationPolicy is migrated and can be deleted: kubectl delete authorizationpolicies.security.istio.io -n backend details-authz
  🔮 AuthorizationPolicy is migrated and can be deleted: kubectl delete authorizationpolicies.security.istio.io -n backend ratings-authz
```

The only task remaining it appears, is to remove the now redundant, original authorization policies for `details` and `ratings` services.

Make it so:

```shell
kubectl delete authorizationpolicies.security.istio.io -n backend details-authz
kubectl delete authorizationpolicies.security.istio.io -n backend ratings-authz
```

Finally, `gloo ambient migrate` should return "all-green" output.

But this does not, in my opinion, replace the need to verify for ourselves that everything functions as expected..

## Test everything

### Test Ingress

Capture the external IP address of the Gateway:

```shell
export GW_IP=$(kubectl get gtw -n istio-ingress gateway \
  -ojsonpath='{.status.addresses[0].value}')
```

Make a curl request to the ingress gateway using the configured hostname `bookinfo.exmaple.com`:

```shell
curl -s bookinfo.example.com/productpage --resolve bookinfo.example.com:80:$GW_IP | grep title
```

### Test authorization policies

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

    It should produce a response saying `RBAC: access denied`.

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

### Test traffic policy

Verify that all requests are routed to `reviews-v3` by making repeated calls to `productpage`:

```shell
curl -s bookinfo.example.com/productpage --resolve bookinfo.example.com:80:$GW_IP | grep "reviews-"
```

## Summary

We are running in ambient mode, and our mesh policies continue to function.

From a resource consumption point of view, we are now running a single waypoint, compared to one per pod back when we were running in sidecar mode.