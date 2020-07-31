# Liveness and Readiness

[![Open this project in Cloud Shell](http://gstatic.com/cloudssh/images/open-btn.png)](https://ssh.cloud.google.com/cloudshell/open?cloudshell_git_repo=https://github.com/marijnmur/googlecloudshelltutorials&cloudshell_tutorial=liveness_and_readiness.md)

## Introduction

Kubernetes manages the restarting and moving of pods as it sees fit. In some
situations we might want to have ways of checking and influencing when that
happens. This is possible with health checks. In this training we will try to
make you familiar with the fundamentals of health checking in Kubernetes. There
are two basic checks; [liveness](#liveness), checking if our pod is alive and
[readiness](#readiness), checking if the pod is ready to receive traffic.

Because Kubernetes has so many interlinked processes that need to work together
and need to know from each other if they are available, health checks for
liveness and readiness are built-in features. The kubelet on the master node
sends out probes to every registered pod and checks if it is still there via
liveness and readiness probes. Let's take a better look at these two fearures.

## Liveness

Liveness refers to availability of an application running on a pod. The app
should have a mechanism built-in to respond to the liveness probe in order to
tell the kubelet it is still alive and kicking. If there is no liveness probe
configured in the container configuration, Kubernetes will assume the pod is
alive. So if it is necessary to be able to control when the container is
considered alive, an appropriate mechanism needs to be implemented in the
application running on the container. This need not be the Kubernetes liveness
probe check. If the application running in the container has mechanisms build in
for regulating failures and auto restart behavior, the liveness probe can be
left for what it is. But in general it is good practice to use the build in
liveness probe checks of Kubernetes.

There are three different types of liveness checks:

### 1. Command

Command fires off a command on the container and expects a exit code 0. If any
other response comes back the pod is considered unhealthy. For a step by step
guide have a look [here](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#define-a-liveness-command)

### 2. HTTP

This is the most common probe in use. It looks for a path on a HTTP server, the
de facto standard being `/health`, and will report healthy if the response code
is between 200 and 400 (400 not included). For a step by step guide have a look
[here](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#define-a-liveness-http-request)

### 3. TCP

Here the kubelet will try to connect to a specified TCP port. If the connection
is successful the pod is healthy. If not it's unhealthy. For a step by step
guide have a look [here](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#define-a-tcp-liveness-probe)

## Configuration settings

There are a number of configuration variables that can be set for a probe from
the Kubernetes [documentation](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#configure-probes):

[Probes](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#probe-v1-core)
have a number of fields that you can use to control more precisely the behavio r
of liveness and readiness checks:

- `initialDelaySeconds`: Number of seconds after the container has started
before liveness or readiness probes are initiated. Defaults to 0 seconds.
Minimum value is 0.
- `periodSeconds`: How often (in seconds) to perform the probe. Default to 10
seconds. Minimum value is 1.
- `timeoutSeconds`: Number of seconds after which the probe times out. Defaults
to 1 second. Minimum value is 1.
- `successThreshold`: Minimum consecutive successes for the probe to be
considered successful after having failed. Defaults to 1. Must be 1 for
liveness. Minimum value is 1.
- `failureThreshold`: When a probe fails, Kubernetes will try failureThreshold
times before giving up. Giving up in case of liveness probe means restarting the
container. In case of readiness probe the Pod will be marked Unready. Defaults
to 3. Minimum value is 1.

[HTTP probes](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#httpgetaction-v1-core)
have additional fields that can be set on httpGet:

- `host`: Host name to connect to, defaults to the pod IP. You probably want to
set "Host" in httpHeaders instead.
- `scheme`: Scheme to use for connecting to the host (HTTP or HTTPS).
Defaults to HTTP.
- `path`: Path to access on the HTTP server.
- `httpHeaders`: Custom headers to set in the request. HTTP allows repeated
headers.
- `port`: Name or number of the port to access on the container. Number must be
in the range 1 to 65535.

 For an HTTP probe, the kubelet sends an HTTP request to the specified path and
port to perform the check. The kubelet sends the probe to the pod's IP address,
unless the address is overridden by the optional `host` field in `httpGet`. If
`scheme` field is set to `HTTPS`, the kubelet sends an HTTPS request skipping
the certificate verification. In most scenarios, you do not want to set the
`host` field. Here's one scenario where you would set it. Suppose the container
listens on 127.0.0.1 and the Pod's `hostNetwork` field is true. Then `host`,
under `httpGet`, should be set to 127.0.0.1. If your pod relies on virtual
hosts, which is probably the more common case, you should not use `host`, but
rather set the `Host` header in `httpHeaders`.

For a TCP probe, the kubelet makes the probe connection at the node, not in the
pod, which means that you can not use a service name in the `host` parameter
since the kubelet is unable to resolve it.

## Readiness

Readiness is a way of knowing if the pod is ready to receive traffic. The main
difference with the Liveness probe is that the pod is not killed when the
readiness probe fails. If the readiness probe fails more times that the
`failureThreshold`, the pod is removed from the list of endpoints that the
kubelet keeps to know where traffic can be sent. The pod can get some time to
'breath'.

Say there is a big load on the pod because the computation is very intensive.
The pod is actually doing what it should be doing, but it does not have enough
resources to accept new connections or workloads. Killing it would mean another
pod needs to do its job and will probably also choke on the data for a while.
Actually it takes more time to deploy a new pod so removing the pod from the
list of available pods provides a little breathing room. The readiness probe
will continue to probe the pod and if it recovers in the amount of times the
'successThreshold' is set, the pod will be available to receive traffic again.

Here it gets tricky though. The liveness probe will also continue to probe if
the service is still up and running. The two are independent of each other and
thus should also have different considerations. The liveness probe should be
completely independent of the readiness probe. A different endpoint and a
different process in the container should handle these two requests. So the
liveness could check if the container is still doing what it is supposed to do,
while the readiness check takes load into consideration, timing out if the load
is too high, for example. This results in a liveness positive check, but a
negative readiness check providing the container to do its processing without
getting new tasks. When it is done and ready to receive new tasks it will report
ready again and be implemented in the kubelet 'ready-to-receive-traffic-pool'
again.

Readiness is an excellent tool to check dependencies for a certain container.
For one time actions it is pretty simple. Say loading a cache at start-up. The
probe will only be successful if the cache is loaded. If the cache is loaded the
probe is successful and reports it's ready to the kubelet to receive traffic.

But say the the readiness probe of a container is taking an external service
into account. It needs to go to a database to retrieve data. Normally the
latency is a few milliseconds. But for some reason the latency increases, within
the threshold for that service, but the probe has to wait longer than the
`timeoutSeconds`, which is set to 1 by default. Then because all readiness
probes have the same latency increase, all of a sudden all containers depending
on that external service are out of service. This will result in the complete
failure of this service, even though the containers are ready, but just the
latency is increased a little. This example is only to show that you need to
take care of what you set as the check for the readiness probe to handle. Please
have a look at the original example, which was copied largely, [here](https://blog.colinbreck.com/kubernetes-liveness-and-readiness-probes-how-to-avoid-shooting-yourself-in-the-foot/).

## Hands on practical examples

Consider this simple container running busybox:

``` bash
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    command: ["/bin/sh"]
    args: ["-c",
      "touch /tmp/healthy;
      touch /tmp/ready;
      sleep 15;
      rm -rf /tmp/ready;
      sleep 15;
      touch /tmp/ready;
      sleep 10;
      rm -rf /tmp/healthy;
      sleep 10;
      touch /tmp/healthy;
      sleep 25;
      rm -rf /tmp/healthy;
      rm -rf /tmp/ready;
      sleep 500"]
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 3
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/ready
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 1
      successThreshold: 1
```

It will start a busybox container that will run the commands in the `args`
consecutively. It will create the files `/tmp/healthy` and `/tmp/ready`. Then it
will remove and recreate them at different times to simulate pod behavior. The
Liveness probe will check if the file `/tmp/ready` is there. The Readiness probe
will check if the file `/tmp/ready` is there. Both will wait the
`initialDelaySeconds` (10 seconds) before they will check if the files are
there. Then probe every 5 seconds. The readiness probe fails after one probe
failing. The liveness probe fails after 3 consecutive failures. After 15 seconds
the readiness probe will fail twice, then come back. After 25 seconds the
liveness probe fails twice, then comes back for 25 seconds. Then the liveness
and readiness probe fails and 15 seconds later the container is restarted. For a
graphical interpretation: ![liveness and readiness](../img/livereadiness.svg).

We can check the status of the pods like this:

``` bash
kubectl describe pod liveness-exec
```

and look for the `Status` of `Ready` in the `Conditions:` section.
You should see something like:

``` bash
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
```

if we do a loop around the command with, for example, `watch` we will see that
the `Status` of `Ready` becomes False during the time the `/tmp/ready` file is
removed from the file system.

``` bash
watch -n 1 "kubectl describe pod liveness-exec"
```

we can see that over time the `Status` changes and the number of restarts of the
pod increases indicating that the liveness probe does its job.

## Some nice blog posts and tutorials

- [Liveness and Readiness Probes](https://www.openshift.com/blog/liveness-and-readiness-probes)

- [Kubernetes best practices: Setting up health checks with readiness and liveness
probes](https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-setting-up-health-checks-with-readiness-and-liveness-probes)

- [Pod Life cycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)

- [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)

- [LIVENESS PROBES ARE DANGEROUS](https://srcco.de/posts/kubernetes-liveness-probes-are-dangerous.html)

- [Kubernetes liveness and readiness probes how to avoid shooting yourself
in the foot](https://blog.colinbreck.com/kubernetes-liveness-and-readiness-probes-how-to-avoid-shooting-yourself-in-the-foot/)

- [Revisiting shooting yourself in the foot](https://blog.colinbreck.com/kubernetes-liveness-and-readiness-probes-revisited-how-to-avoid-shooting-yourself-in-the-other-foot/)

- [Kubernetes Liveness and Readiness Probes: Looking for More Feet](https://blog.colinbreck.com/kubernetes-liveness-and-readiness-probes-looking-for-more-feet/)
