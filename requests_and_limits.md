# Requests and Limits

## Introduction

The Kubernetes world might seem to provide an endless amount of computing
resources where you can't run out of memory or computing processing units. This
is of course not true. Every cluster, every node has it's limits configured. If
the applications and processes running on the nodes are not configured correctly
or, better yet, are not optimized, it can happen that pods actually cause other
pods to run out of computing power or memory.

Kubernetes is build on the premise of being able to manage it's resources quite
independently, so we as users don't have to worry about managing all the
resources. That is the beauty of the system. That's why we are using it. But how
does Kubernetes know what processes are important to us? How does it decide when
to restart a pod or when to evict it completely and start it on another node?
How does it come up with a system of importance of processes? Complete
randomness could be the best option. But most of the time that is not what we
want.

This training is an attempt to provide some light into optimizing the settings
we can provide to make sure we get the most out of our cluster resources. The
end goal will be an understanding of how we can configure our applications and
processes to get the priority they need. This will allow us to configure our
pods efficiently. Reducing start up time. Improving consistent performance. And
ultimately getting warnings and alerts when there is really something going on
that needs our attention. In short, a healthy responsive cluster that is fun to
work with!

We will look into [requests](#requests), which define the resources a pod
requests from Kubernetes and gets guaranteed. Then we'll look into
[limits](#limits) where we can limit the resources of a pod.

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
killed. Again these limits can be set for Memory and CPU. There is a difference
between the limit for CPU and the Limit for Memory though. When the CPU limit is
reached the pod will be throttled. Only the CPU amount defined by the limit is
available. But when the Memory limit of a pod is reached the pod will be killed
and restarted. (If the restart policy is thus configured)

Combining requests with limits provides easy to control boundaries and safe
guards for containers. On the one hand guaranteeing they will get the resources
that they need. On the other hand limiting the maximum resources they can use.

## Limits and Requests configuration

There are three resources that can be set for both limits and requests:

- `cpu` expressed in CPU units of 1m (milli) CPU, where 1 CPU is 1 vCPU/Core.
- `memory` expressed in bytes, with suffixes K, M, G, T, P and E or Ki, Mi, Gi,
Ti, Pi, Ei or e(for exponent) allowed. K being 1000 and Ki 1024.
- `hugepages-<size>`. Huge pages are a Linux-specific feature where the node
kernel allocates blocks of memory that are much larger than the default page
size. For more information see [here](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-types)

The `requests` and `limits` are set in the `resources:` block in the
`containers:` specification of a Pod. The following examples are inspired on
[here](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/).

First start up [mikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) and change into that configuration for kubectl by running:

```bash
minikube start
kubectl config use-context minikube
```

Or if you want to run Kubernetes in Docker on MacOs enable it in Docker Desktop
by clicking `Enable Kubernetes` under `Advanced` in the settings menu.
See [here](https://adahealth.atlassian.net/wiki/spaces/PENG/pages/846962/How+to+run+Backend+apps+locally+in+Docker+with+Kubernetes+support)
for a complete walk through of setting this up.

We want to have a simple container that is able to run an easy task, but can
also be manipulated in violating our limits to demonstrate the described
behavior. An easy tool for testing a system is the Linux `stress` tool. First we
create a container with the `stress` tool running with parameters that let it
run within the set `requests` and `limits`.
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
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

Copy this into a file called `requests-limits-demo.yaml` and run:

```bash
kubectl apply -f requests-limits-demo.yaml
```

Check if the pod is running:

```bash
kubectl get pods
```

and check if it is behaving nicely by checking the resources it uses via:

```bash
kubectl top pods
```

In minikube you might need to run `minikube addons enable metrics-server` first for this to work. The output looks something like this:

```console
NAME                                CPU(cores)   MEMORY(bytes)   
requests-limits-demo                40m          151Mi           
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
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
```

Look at what the pod is doing via:

```bash
kubectl get pods
```

The pod will get an `OOMKilled` status and will try to restart. Then because it the next time it starts it will fail again you will start seeing `CrashLoopBackoff`. This will increase the time between restart attempts.

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
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1", "--cpu", "10"]
```

The `--cpu` argument of `stress` does not actually reflect the amount of CPU
requested, but the mount of times a function is spawned that calculates the
square root of a number.

Checking the pod again with :

```bash
kubectl top pods
```

we will see that the cpu usage of the pod is now equal to the limit we set:

```console
NAME                                CPU(cores)   MEMORY(bytes)
requests-limits-demo-too-much-cpu   404m         152Mi
```

We can actually see the throttling as well, for the memory foot print is lower
than we expect. The process will thus be cut on it's resources and require less
memory as a result. If we open the dashboard (`minikube dashboard`), we will see
that the memory is fluctuating in the 'too-much-cpu' container, but is steady in
the 'normal' one.

This displays the difference between the limits rather nicely. When the memory
limit is reached the pod will actually be killed and restarted. When the CPU
limit is reached the process will be throttled.

## Kubelet killing pods

At the beginning we said `requests` and `limits` contibute to more visibility on
when a pod gets killed and how to influence when a pod gets killed or evicted.
Until now we have looked at the direct consequences of resources that try to use
more that their configured limits. Now lets look at what this means for how
Kubernetes determines which pods are killed and/or evicted first.

The moment we define a `requests` and `limits` for a pod we influence the
Quality of Service that pod has. The Quality of Service is a way of defining how
important our pod is. Three stages exists:

- `BestEffort`: no `requests` or `limits` are set. These pods will be killed
- first.  `Burstable`: pods that have `requests` and `limits` set, but the
- requested resources are lower than the limits. These pods are considered more
- important. `Guaranteed`: pods that have the same value defined for `requests`
- and `limits`. These pods are evicted last.

Another thing to consider is setting the Priority of a pod. This is a way of
distinguishing importance of pods with the same Quality of Service. The
Kuberbetes documentation says the following on Evicting end-user Pods:

```text
Evicting end-user Pods If the kubelet is unable to reclaim sufficient
resource on the node, kubelet begins evicting Pods.

The kubelet ranks Pods for eviction first by whether or not their usage of the
starved resource exceeds requests, then by Priority, and then by the consumption
of the starved compute resource relative to the Pods' scheduling requests.

As a result, kubelet ranks and evicts Pods in the following order:

BestEffort or Burstable Pods whose usage of a starved resource exceeds its
request. Such pods are ranked by Priority, and then usage above request.
Guaranteed pods and Burstable pods whose usage is beneath requests are evicted
last. Guaranteed Pods are guaranteed only when requests and limits are specified
for all the containers and they are equal. Such pods are guaranteed to never be
evicted because of another Pod's resource consumption. If a system daemon (such
as kubelet, docker, and journald) is consuming more resources than were reserved
via system-reserved or kube-reserved allocations, and the node only has
Guaranteed or Burstable Pods using less than requests remaining, then the node
must choose to evict such a Pod in order to preserve node stability and to limit
the impact of the unexpected consumption to other Pods. In this case, it will
choose to evict pods of Lowest Priority first. If necessary, kubelet evicts Pods
one at a time to reclaim disk when DiskPressure is encountered. If the kubelet
is responding to inode starvation, it reclaims inodes by evicting Pods with the
lowest quality of service first. If the kubelet is responding to lack of
available disk, it ranks Pods within a quality of service that consumes the
largest amount of disk and kills those first.
```

To read more about Pod Eviction have a look
[here](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/#evicting-end-user-pods).
If you want to learn more about Pod Priority take a peak
[here](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/)

## Some nice blog posts and tutorials

- [A Practical Guide to Setting Kubernetes Requests and Limits](http://blog.kubecost.com/blog/requests-and-limits/)

- [Managing Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

- [Understanding Kubernetes limits and requests by example](https://sysdig.com/blog/kubernetes-limits-requests/)

- [Assign CPU Resources to Containers and Pods](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/)

- [Assign Memory Resources to Containers and Pods](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)

- [Configure Out of Resource Handling](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/#eviction-policy)
