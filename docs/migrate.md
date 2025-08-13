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

- Among other things, it validates Istio version and Cluster CNI compatibility.  If we had used too old a version of Istio, the tool would have raised a concern, and our first priority would be to upgrade Istio to a newer version.

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

### Solo.io's Istio distribution

Solo.io's distribution of Istio provides sidecar to ambient interoperability that ensures that mesh policies continue to function in environments with mixed sidecar and sidecarless workloads, specifically in situations where services have been upgraded to ambient while sidecar clients remain.

Solo.io's distribution ensures that requests to ambient services from these sidecar clients are routed through the waypoints that enforce authorization and other mesh policies.

See [Solo distributions of Istio](https://docs.solo.io/gloo-mesh/latest/ambient/about/images/overview/) for more information.

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

In ambient mode, sidecar workloads must have the HBONE protocol enabled in order to communicate with sidecarless workloads.

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

The waypoint can be deployed by applying the migration assistant's recommended manifest:

```shell
kubectl apply -f /tmp/istio-migrate/recommended-waypoints.yaml
```

Alternatively, use the `istioctl waypoint apply` command, like so:

```shell
istioctl waypoint apply --namespace backend
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

### Turn on access logging

The Telemetry API allows us to enable access logging:

```yaml title="telemetry.yaml" linenums="1"
--8<-- "telemetry.yaml"
```

Apply the resource:

```shell
kubectl apply -f artifacts/telemetry.yaml
```

This will give us the ability to view traffic transiting the waypoint through its logs.

Here is the command to tail the waypoint logs:

```shell
kubectl logs --follow -n backend deploy/waypoint
```

## Back to the Assistant

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

It's worth mentioned that we are not yet deleting any existing authorization policies.
Rather, we add the equivalent policies that will function in the context of the waypoints.

```shell
kubectl apply -f /tmp/istio-migrate/recommended-policies.yaml
```

A close look at the recommended policies shows that the tool is doing its due diligence to permit requests that target either the `ratings` and `ratings-v1` services, and similarly for the `details` and `details-v1` services.

A closer look at the recommended policies reveals two things:

1. The recommended policies include L4 authorization policies explicitly allowing the waypoint to communicate with the target service (`details`, `ratings`).  This is technically not required for the system to function, but it ensures that only the waypoint can communicate with the service (i.e. that no client bypasses the waypoint).
2. The tool is doing its due diligence to permit requests that target either the `ratings` and `ratings-v1` services, and similarly for the `details` and `details-v1` services.

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

### Migrate the backend

Remove the sidecar injection label:

```shell
kubectl label namespace backend istio-injection-
```

Replace it with the `dataplane-mode=ambient` label (ensures that ztunnel intercepts traffic in and out of our workloads):

```shell
kubectl label namespace backend istio.io/dataplane-mode=ambient
```

Restart the backend deployments to remove the sidecars:

```shell
kubectl rollout restart deploy -n backend details-v1
kubectl rollout restart deploy -n backend ratings-v1
kubectl rollout restart deploy -n backend reviews-v1
kubectl rollout restart deploy -n backend reviews-v2
kubectl rollout restart deploy -n backend reviews-v3
```

### Migrate the frontend

```shell
kubectl label namespace frontend istio-injection-
kubectl label namespace frontend istio.io/dataplane-mode=ambient
kubectl rollout restart deploy -n frontend
```

### Validate

Verify that all pods in `frontend` and `backend` have a single container:

```shell
kubectl get pod -n frontend
kubectl get pod -n backend
```

The `istioctl` CLI also provides `ztunnel-config` commands to verify the traffic traveling through ztunnel and waypoint proxies, and assigned workload identity certificates:

1. Verify that workloads use the HBONE protocol:

    ```shell
    istioctl ztunnel-config workload
    ```

    ```console
    NAMESPACE     POD NAME                                ADDRESS      NODE                    WAYPOINT PROTOCOL
    backend       details-v1-76cb4f574f-nqhtw             10.42.0.40   k3d-my-cluster-server-0 None     HBONE
    backend       ratings-v1-5bf4c88c5c-qbnq5             10.42.0.39   k3d-my-cluster-server-0 None     HBONE
    backend       reviews-v1-6899c69cc5-zqmbb             10.42.0.43   k3d-my-cluster-server-0 None     HBONE
    backend       reviews-v2-74dd9df8fc-2z62b             10.42.0.42   k3d-my-cluster-server-0 None     HBONE
    backend       reviews-v3-5598c69cc5-8jzld             10.42.0.41   k3d-my-cluster-server-0 None     HBONE
    backend       waypoint-56f487d8f8-q9qpk               10.42.0.44   k3d-my-cluster-server-0 None     TCP
    frontend      curl-678c94dfbb-bqwph                   10.42.0.37   k3d-my-cluster-server-0 None     HBONE
    frontend      productpage-v1-5bf9bf9f89-w6k5m         10.42.0.38   k3d-my-cluster-server-0 None     HBONE
    ```

2. Verify the services associated with the waypoint in the `backend` namespace:

    ```shell
    istioctl ztunnel-config service
    ```

    ```console
    NAMESPACE     SERVICE NAME   SERVICE VIP   WAYPOINT ENDPOINTS
    backend       details        10.43.87.252  waypoint 1/1
    backend       details-v1     10.43.121.191 waypoint 1/1
    backend       ratings        10.43.18.135  waypoint 1/1
    backend       ratings-v1     10.43.154.253 waypoint 1/1
    backend       reviews        10.43.176.233 waypoint 3/3
    backend       reviews-v1     10.43.135.223 waypoint 1/1
    backend       reviews-v2     10.43.73.229  waypoint 1/1
    backend       reviews-v3     10.43.114.154 waypoint 1/1
    backend       waypoint       10.43.41.47   None     1/1
    frontend      curl           10.43.65.249  None     1/1
    frontend      productpage    10.43.70.80   None     1/1
    istio-ingress gateway-istio  10.43.246.228 None     1/1
    ```

3. Verify that all workloads are assigned workload identities:

    ```shell
    istioctl ztunnel-config certificate
    ```

    ```console
    CERTIFICATE NAME                                               TYPE     STATUS        VALID CERT     SERIAL NUMBER                        NOT AFTER                NOT BEFORE
    spiffe://cluster.local/ns/backend/sa/bookinfo-details          Leaf     Available     true           f0bc69c12ed191f83365b2297890f248     2025-06-06T04:54:09Z     2025-06-05T04:52:09Z
    spiffe://cluster.local/ns/backend/sa/bookinfo-details          Root     Available     true           c4cc571dc92335d57cb6cf3ab78f99a8     2035-06-02T08:05:55Z     2025-06-04T08:05:55Z
    spiffe://cluster.local/ns/backend/sa/bookinfo-ratings          Leaf     Available     true           9f4722ba68e73146fa5c498fcacb5270     2025-06-06T04:54:09Z     2025-06-05T04:52:09Z
    spiffe://cluster.local/ns/backend/sa/bookinfo-ratings          Root     Available     true           c4cc571dc92335d57cb6cf3ab78f99a8     2035-06-02T08:05:55Z     2025-06-04T08:05:55Z
    spiffe://cluster.local/ns/backend/sa/bookinfo-reviews          Leaf     Available     true           8ad4912f768ac3f986e8d0014947af5a     2025-06-06T04:54:09Z     2025-06-05T04:52:09Z
    spiffe://cluster.local/ns/backend/sa/bookinfo-reviews          Root     Available     true           c4cc571dc92335d57cb6cf3ab78f99a8     2035-06-02T08:05:55Z     2025-06-04T08:05:55Z
    spiffe://cluster.local/ns/frontend/sa/bookinfo-productpage     Leaf     Available     true           49cd2a29fb479144ee898135a71c4ab9     2025-06-06T04:54:16Z     2025-06-05T04:52:16Z
    spiffe://cluster.local/ns/frontend/sa/bookinfo-productpage     Root     Available     true           c4cc571dc92335d57cb6cf3ab78f99a8     2035-06-02T08:05:55Z     2025-06-04T08:05:55Z
    spiffe://cluster.local/ns/frontend/sa/curl                     Leaf     Available     true           ffe564539de3ecccdbe4d53ef6230f05     2025-06-06T04:54:16Z     2025-06-05T04:52:16Z
    spiffe://cluster.local/ns/frontend/sa/curl                     Root     Available     true           c4cc571dc92335d57cb6cf3ab78f99a8     2035-06-02T08:05:55Z     2025-06-04T08:05:55Z
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

The only task remaining, it appears, is to remove the now redundant, original authorization policies for `details` and `ratings` services.

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

Make a curl request to the ingress gateway using the configured hostname `bookinfo.example.com`:

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