# Annotation-Based Network Policy Proposal

## Abstract

Defines a general open/closed network policy using Kubernetes annotations and Calico networking. 

## Goals

Allow for experimentation with network policy to:
- Identify pitfalls of annotation-based policy.
- Discover what is lacking from an Open/Closed policy solution.
- Determine whether policy should be applied to namespace and/or service objects.
- Generate discussion in the network sig.

## Implementation

### Details

Coarse-grained policy can be implemented with the annotations on either a namespace and/or a service.

One may annotate a namespace using the annotation, `net.alpha.kubernetes.io/access-policy`, with a value "Open" or "Closed". An "Open" namespace means that pods within the namespace are accessible from anywhere in the cluster. A "Closed" namespace means that pods within the namespace are accessible only by other pods in the same namespace.

Additionally, one may annotate a service using the annotation, `net.alpha.kubernetes.io/access-policy`, with a value "Open" or "Inherit". An "Open" service means that pods within the service are accessible from anyway in the cluster, via the ClusterIP. An "Inherit" service means that pods within the service inherit the policy label of the namespace of which that service belongs to. To clarify, an "Inherit" service in a "Closed" namespace means that pods will be accessible from pods within the same service. An "Inherit" service in an "Open" namespace means that pods will be accessible from anywhere in the cluster.

### Installation

In order to implement the proposed annotation-based network policy, you must first get up a Kubernetes cluster with Calico by following [one of our guides](https://github.com/projectcalico/calico-docker/tree/master/docs/kubernetes).
 
Next, install an experimental branch of the Calico network plugin. 

You will need to clone our repo, checkout out the experimental branch, and build the binary.

    $ git clone https://github.com/projectcalico/calico-kubernetes.git
    $ cd calico-kubernetes
    $ git checkout coarse-grained-policy
    $ make binary
    
Then, copy the binary in the `dist` directory to the Kubernetes plugin location, `/usr/libexec/kubernetes/kubelet-plugins/net/exec/calico/`.

## Shortcomings

Why is annotation based coarse-grained policy limited?

## Future Work

[Casey's PR](https://github.com/kubernetes/kubernetes/pull/17551/files)

[Luke's PR](https://github.com/kubernetes/kubernetes/pull/13937)

### Create a New Service Type Field in API

Currently a service can have a type of `None`, `ClusterIP`, `NodePort`, or `LoadBalancer`. We propose adding a new type that is slightly more restrictive than `ClusterIP`, called `NamespaceIP`. The type `NamespaceIP` allows accessibility via the ClusterIP only from pods within the namespace. 

### Specify Policy from the Namespace API Spec