# Migration plan

Fundamentally, moving to ambient means removing all of our sidecars and replacing them with a combination of layer 4 and layer 7 proxies.
See the [ambient overview in the Istio docs](https://istio.io/latest/docs/ambient/overview/) for more information.

The layer 4 proxies are a [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) known as ztunnel, while the layer 7 components are known as waypoints.
Unlike sidecars, waypoints are optional (installed as needed) and deployed separately from your workloads.
Operationally, this is an advantage, as the platform networking concerns are in no way coupled to your deployments.

Some ambient installations may not even require any waypoints, in situations where all mesh policies remain at layer 4.

## Analysis

### Where do we need Waypoints?

We must triage our mesh policies in terms of which do and do not require layer 7 proxies in order to function.

For example, of the three Authorization policies we have in place, the one for the `productpage` service is purely layer 4.  On the other hand, the authorization policies for the `ratings` and `details` services require an evaluation of the HTTP method (GET or POST), and so require a waypoint.

The traffic policy that governs how requests are routed to the `reviews` service also requires a waypoint, since we are performing HTTP routing.

We come to the conclusion thhen that the `productpage` service does not require a waypoint.
On the other hand, the backend services do, either to support their layer 7 authorization policies (`ratings` and `details`), or to support their traffic policies (`reviews`).

### How many waypoints?

Another decision that must be made is whether to dedicate a waypoint for each service, or whether we can use a single Waypoint to proxy multiple services.

In this specific scenario, all of these backend services are related, and it makes sense to consider deploying a single waypoint responsible for all three.

This pattern of using a single waypoint per namespace is often a perfect middle-ground between too many waypoints (one per service, as in sidecar mode), and too few waypoints (a single monolithic waypoint for way too many services).

## The plan

### Upgrade Istio to ambient mode

Upgrading Istio to run in ambient mode entails two specific actions:

- Upgrade the existing Helm releases to run in ambient mode (`--set profile=ambient`).
- Install the missing ztunnel component.

### Deploy the waypoint

After the upgrade to ambient mode is a good time to provision the waypoint that will proxy our backend services.

### Retrofit authorization policies

Our AuthorizationPolicies will not function in ambient mode in their current form.
Per [Considerations for authorization policies](https://istio.io/latest/docs/ambient/usage/l7-features/#considerations), the policies must have a `targetRefs` which attaches them to their waypoint.

We will derive a new set of ambient-compatible authorization policies and apply them in advance of switching to ambient mode.

### Switch the workloads to be part of the ambient mesh

We will replace sidecar injection with making the workloads a part of the ambient mesh.
Istio has [distinct conventions](https://istio.io/latest/docs/ambient/usage/add-workloads/) for each, by labeling workloads, or more idiomaticlaly their namespaces.

This switch will necessitate a final restart of the workloads in order to remove their sidecars.

## The migration tool

Even with a migration plan in place, it's important to have our migration supported by tools which inspect our environment and give us the feedback we need to make sure we haven't left anything out.

Solo.io publishes a comprehensive [migration tool](https://ambientmesh.io/#migrate-from-sidecars) that we will use, by running it multiple times during the migration process in order to get the feedback necessary to know that we remain on the right track.

The tool goes as far as even producing transformed policies that are compatible with ambient mesh, saving us the effort of doing that work by hand.
It's also a great way to avoid performing certain steps manually, which are error-prone.

The tool will help us validate where we stand during the migration process, and that we have completed our migration and not left out any steps.

If we forget to perform a step, the tool will remind us of the fact.
