# Introduction

In this guide, we will walk you through configuring a system of microservices with Istio in sidecar mode.

In the [setup](setup.md) section, you will install Istio, deploy a sample application, configure a variety of service mesh resources (security and traffic policy) and verify that they all function.

The subsequent section walks you through performing a migration to ambient mesh.  That is, eliminating the use of sidecars altogether, while at the same time preserving your mesh policies and configurations.

Ambient mesh provides a stronger separation of your workloads from the mesh platform, where the proxies involved in implementing your policies are no longer coupled to your deployments.

For complete and thorough documentation on Istio Ambient Mesh, visit [www.ambientmesh.io](http://www.ambientmesh.io).