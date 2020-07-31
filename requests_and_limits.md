# Requests and Limits

[![Open this project in Cloud Shell](http://gstatic.com/cloudssh/images/open-btn.png)](https://ssh.cloud.google.com/cloudshell/open?cloudshell_git_repo=https://github.com/marijnmur/googlecloudshelltutorials&cloudshell_tutorial=requests_and_limits.md)

## Introduction

The Kubernetes world might seem to provide an endless amount of computing
resources where you can't run out of memory or computing processing units. This
is of course not true. Every cluster, every node has it's limits configured. If
the applications and processes running on the nodes are not configured correctly
or, better yet, are not optimized, it can happen that pods actually cause other
pods to run out of computing power or memory.
This training is an attempt to provide some light into optimizing the settings
we can provide to make sure we get the most out of our cluster resources.

 We will look into [requests](#requests), which define the resources a pod
requests from Kubernetes and gets guaranteed. Then we'll look into
[limits](#limits) where we can limit the resources of a pod.

If you are looking at this document via the Google Cloud shell(GCP) Walkthrough
you can follow along with the examples in a GCP project or spin up something
like [Minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/)

## Requests

Requests are about the resources used by
a container and determine how much memory and CPU is allocated to pods.

Every container running on a cluster uses a certain amount of memory and CPU.
But how does Kubernetes determine who gets what? You could follow an agnostic
approach and let Kubernetes determine everything on its own. It is designed to
do that, so just let it do its thing. For a lot of applications that is
actually the most sensible thing to do. All non critical applications that are
running on a cluster can be perfectly managed by Kubernetes itself.

But what if you have an application that you don't want to have killed because
Kubernetes thinks it's a good idea? What if you know that if your application is
using more than x amount of memory it hung itself up? Here Requests and Limits
come into play.

A `request` is the definition for a container of how much resources, CPU and
Memory, it will request from Kubernetes to be **_guaranteed_** available to that
pod. Kubernetes will then look for the right node to place this pod to ensure
the requests. Even if the pod is not using the memory and/or CPU requested it is
reserved to that pod and not available to others. This results in assuring that
this one pod has the necessary resources available when it needs them, but
setting these values too large will squander a lot of resources, driving costs
up.

## Limits

A pod can use more that the requested resources. If the node it is running on
has resources available that are not claimed by other pods, these resources are
freely available for use. But what if you want to restrict the
maximum amount of resources a pod can use? Say for instance you know there is a
memory bug that happens only in certain cases, say 1:100000 times. The bug is
too costly to really solve, but makes for a infinite loop in the pod that just
consumes all available memory. This could cause some major problems eating up
all resources, maybe scaling up nodes ending in wasting money on
unnecessary resources.

Here `limits` can be an easy solution. A limit is exactly that, a limit. If a
limit is set on a resource, the moment the limit is reached the pod will be
killed. Again these limits can be set for Memory and CPU.

Combining requests with limits provides easy to control boundaries and safe
guards for containers. On the one hand guaranteeing they will get the resources
that they need. On the other hand limiting the maximum resources they can use.
If all 4 are combined and configured appropriately they provide the basic
building blocks of controlling how to run an optimized Kubernetes cluster.

## Limits and Requests configuration

There are three resources that can be set for both limits and requests:

- `cpu` expressed in CPU units of 1m (milli) CPU, where 1 CPU is 1 vCPU/Core.
- `memory` expressed in bytes, with suffixes K, M, G, T, P and E or Ki, Mi, Gi,
Ti, Pi, Ei or e(for exponent) allowed. K being 1000 and Ki 1024.
- `hugepages-<size>`. Huge pages are a Linux-specific feature where the node
kernel allocates blocks of memory that are much larger than the default page
size. For more information see [here](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-types)

The `requests` and `limits` are set in the `resources:` block in the
`containers` specification of a Pod. The following examples are inspired on
[here](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/).

``` bash
apiVersion: v1
kind: Pod
metadata:
  name: requests-limits-demo
spec:
  containers:
  - name: requests-limits-demo
    image: polinux/stress
    resources:
      requests:
        memory: "150Mi"
        cpu: "200m"
      limits:
        memory: "200Mi"
        cpu: "400m"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1", "--cpu", "1"]
```

The container runs the Linux stress tool with setting in the allowed range.
Check the pods resources to confirm it is running:

```bash
kubectl describe pods memory-demo
```

or

```bash
kubectl top pods
```

## Too much Memory

Now to put the limits into use change the stress command to use more resources
than the limits. First change the memory usage to exceed the memory limit set.

```bash
apiVersion: v1
kind: Pod
metadata:
  name: requests-limits-demo-too-much-memory
spec:
  containers:
  - name: requests-limits-demo-too-much-memory
    image: polinux/stress
    resources:
      requests:
        memory: "150Mi"
        cpu: "200m"
      limits:
        memory: "200Mi"
        cpu: "400m"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1", "--cpu", "1"]
```

Look at what the pod is doing via:

```bash
kubectl get pods
```

The pod will get an `OOMKilled` status and will try to restart.

## Too much CPU

To simulate too much CPU usage adjust the settings to:

```bash
apiVersion: v1
kind: Pod
metadata:
  name: requests-limits-demo-too-much-cpu
spec:
  containers:
  - name: requests-limits-demo-too-much-cpu
    image: polinux/stress
    resources:
      requests:
        memory: "150Mi"
        cpu: "200m"
      limits:
        memory: "200Mi"
        cpu: "400m"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1", "--cpu", "10000"]
```

The `--cpu` argument of `stress` does not actually reflect the amount of CPU
requested, but the mount of time a function is spawned that calculates the
square root of a number. Thus a high number is required to actually violate the
set CPU limit.

Checking the pod again with :

```bash
kubectl get pods
```

will show again that the status is `OOMKilled`. Kubernetes does not have a
separate status for CPU or memory limit violation. If either of those limits is
reached, the pod will get killed via the `OOMKilled` mechanism.

## Some nice blog posts and tutorials

- [A Practical Guide to Setting Kubernetes Requests and Limits](http://blog.kubecost.com/blog/requests-and-limits/)

- [Managing Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

- [Understanding Kubernetes limits and requests by example](https://sysdig.com/blog/kubernetes-limits-requests/)

- [Assign CPU Resources to Containers and Pods](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/)

- [Assign Memory Resources to Containers and Pods](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)
