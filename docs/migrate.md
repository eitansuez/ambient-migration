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

## Deploy the waypoint

## Apply retrofitted authorization policies

## Switch to ambient mode

## Test everything

### Test Ingress

### Test authorization policies

### Test traffic policy

## Summary