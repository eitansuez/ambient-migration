# Initial Considerations

## The Istio CNI node agent

In sidecar mode, Istio provides two ways for proxies to intercept traffic in and out of workloads:

1. Iptables rules can be applied to route incoming and outgoing traffic to the sidecar on conventional ports
2. The [Istio CNI node agent](https://istio.io/latest/docs/setup/additional-setup/cni/) is a newer method that was introduced to remove the requirement of running privileged containers to configure traffic redirection.

As you plan your migration to ambient, your first consideration should be to migrate to using the Istio CNI node agent, as it is a requirement for ambient mode.

## The Kubernetes Gateway API

The [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) is quickly becoming the new standard for implementing traffic policy, both at ingress and in the mesh.

Istio has embraced the Kubernetes Gateway API, and ambient mode depends on it.
The good news is that Istio supports the Gateway API in sidecar mode as well.

Here too, as the Kubernetes gateway API becomes a requirement for Ambient, a first step towards migrating to ambient is opting to replace the use of the more venerable Istio APIs such as VirtualServices, Istio Gateways, and DestinationRules (for the purpose of defining subsets) with the newer Gateway API throughout.

An advantage of performing this migration is gaining the ability to provision gateways dynamically at runtime, which provides increased flexibility.

## Next..

In the next section, as we install Istio in sidecar mode, we opt to install the Istio CNI.

Similarly, when configuring gateways and defining traffic policy, we opt for the newer Gateway API.

Both decisions will put us in the position to focus on the migration to sidecarless without having to deal with these additional concerns.
