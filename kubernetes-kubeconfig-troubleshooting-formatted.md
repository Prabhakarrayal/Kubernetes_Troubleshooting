# Kubernetes Troubleshooting Notes

## Fixing `kubectl` Configuration and Restoring Cluster Access

> **Environment:** A Kubernetes cluster with one control-plane
> node and two worker nodes.

------------------------------------------------------------------------

## 1. What happened?
WE HAD A K'8 CLUSTER OF 3 NODES, 1 CONTROLLER AND 2 WORKERS IN 3 DIFFERENT VM'S(Virtual Machine)

After the worker virtual machines had been shut down and worker 1 was
started again, I tried to check my Kubernetes cluster from the
control-plane machine.

I ran:

``` bash
kubectl get pods -A
```

But `kubectl` returned errors similar to:

``` text
The server rejected our request for an unknown reason
```

At first, it looked like a cluster problem. We did not assume that,
though. We checked the configuration and tested the connection step by
step.

### Our final result

-   `kubectl` works normally again.
-   All three nodes report `Ready`.
-   The control-plane components and listed system pods are `Running`.
-   The application pod is `Running`.

**Important:** We fixed the immediate access problem. We did **not**
prove exactly why the original configuration file was missing or whether
shutting down the VMs caused it.

------------------------------------------------------------------------

## 2. Understand the important technical terms

  -----------------------------------------------------------------------
  Technical term                      Easy meaning
  ----------------------------------- -----------------------------------
  **Kubernetes cluster**              A group of machines working
                                      together to run and manage
                                      applications.

  **Control plane**                   The part that manages the cluster
                                      and makes decisions.

  **Worker node**                     A machine that runs application
                                      workloads.

  **`kubectl`**                       The command-line tool we use to
                                      talk to Kubernetes. Think of it as
                                      a remote control for the cluster.

  **Kubeconfig**                      A configuration file that tells
                                      `kubectl` which cluster to contact
                                      and which credentials to use.

  **Context**                         A saved selection of a cluster, a
                                      user/credential, and optionally a
                                      namespace. It tells `kubectl` which
                                      connection to use.

  **`admin.conf`**                    The Kubernetes administrative
                                      configuration file used in this
                                      cluster.

  **Default kubeconfig**              Usually `~/.kube/config`. For the
                                      root user, that path is
                                      `/root/.kube/config`.

  **Namespace**                       A way to group Kubernetes resources
                                      inside a cluster. `kube-system`
                                      contains important Kubernetes
                                      system components.

  **Pod**                             The smallest deployable unit in
                                      Kubernetes. It runs one or more
                                      containers.

  **`Ready` node**                    Kubernetes considers the node
                                      healthy enough to accept normal
                                      workloads.

  **`Running` pod**                   Kubernetes reports that the pod is
                                      running. This alone does not prove
                                      the application is reachable from every pod

  -----------------------------------------------------------------------

### The key idea

`kubectl` needs a usable kubeconfig to know where the cluster is and how
to authenticate.

In our incident, the known-working configuration was available at:

``` text
/etc/kubernetes/admin.conf
```

But the default configuration was missing or unusable at:

``` text
/root/.kube/config
```

That was the immediate issue we corrected.

------------------------------------------------------------------------

## 3. Step-by-step troubleshooting

### Step 1 --- Check whether the container runtime is running

**Why?** The container runtime runs containers on the machine. We
checked it to avoid assuming that it was stopped.

``` bash
systemctl status containerd --no-pager
```

**What the output showed:** `containerd` was `active (running)`.

**What we learned:** The container runtime on the control-plane machine
was running. This did not, by itself, prove that the whole cluster was
healthy.

> Note: `systemctl status containerd --no-pager` is the correct command.
> `systemctl containerd --no-pager` is missing the `status` action.

### Step 2 --- Check the selected Kubernetes context

**Why?** We needed to know which saved cluster connection `kubectl` was
trying to use.

``` bash
kubectl config current-context
```

The output was:

``` text
Current context is not set
```

This told us that no context was selected in the configuration `kubectl`
was reading.

We also checked:

``` bash
kubectl config get-contexts
```

The context list was empty.

### Step 3 --- Check the default configuration

**Why?** We wanted to see whether the default configuration contained a
cluster, a user, and a context.

``` bash
kubectl config view
```

The important part of the output was:

``` yaml
clusters: null
contexts: null
current-context: ""
users: null
```

In simple words, the configuration being read by default had no cluster,
user, or context entries.

We also checked the expected file:

``` bash
ls -l /root/.kube/config
```

It reported that the file did not exist.

### Step 4 --- Test the known administrative configuration directly

**Why?** Before changing anything, we wanted to check whether the
cluster could still be reached using the administrative configuration
file.

``` bash
kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes
```

This command succeeded and showed the cluster nodes. At that point,
worker 2 was `NotReady`, while the control plane and worker 1 were
`Ready`.

**What we learned:** The administrative configuration worked and the API
server responded. That was strong evidence that the immediate problem
was related to the default `kubectl` configuration, rather than proof of
a complete cluster outage.

### Step 5 --- Check the context in `admin.conf`

**Why?** We needed the exact context name rather than guessing it.

``` bash
kubectl --kubeconfig=/etc/kubernetes/admin.conf config get-contexts
```

The output included:

``` text
CURRENT   NAME
*         kubernetes-admin@kubernetes
```

The `*` means that this context was selected inside `admin.conf`.

### Step 6 --- Restore the default kubeconfig

The standard default location for the root user's kubeconfig is:

``` text
/root/.kube/config
```

We first tried:

``` bash
install -D -m 600 /etc/kubernetes/admin.conf /root/.kube/config
```

The command returned no output, which is normal when it succeeds.
However, the subsequent context check still showed that no context was
set in the default configuration.

We then copied the known-working configuration again:

``` bash
cp /etc/kubernetes/admin.conf /root/.kube/config
```

**Why?** This put the known-working administrative configuration at the
default location used by `kubectl` for the root user.

We verified the destination file explicitly:

``` bash
kubectl --kubeconfig=/root/.kube/config config get-contexts
```

It showed:

``` text
CURRENT   NAME
*         kubernetes-admin@kubernetes
```

Now the default configuration contained the expected context.

**Permission note:** The earlier `install -m 600` command was intended
to make the file readable and writable only by root. The later `cp`
command does not itself guarantee the same permission mode in every
situation. For administrative kubeconfig files, verify permissions when
needed:

``` bash
ls -l /root/.kube/config
```

The expected permissions are normally `-rw-------` (mode `600`).

### Step 7 --- Verify normal `kubectl` access

**Why?** We wanted to confirm that we no longer needed to provide
`--kubeconfig` manually.

``` bash
kubectl get nodes -o wide
```

Then we ran:

``` bash
kubectl get nodes
```

Both commands succeeded.

All three nodes were `Ready`:

  Node                    Internal IP         Final status
  ----------------------- ------------------- --------------
  `dockerandcontroller`   `192.168.122.198`   `Ready`
  `k8s-worker-1`          `192.168.122.139`   `Ready`
  `k8s-worker-2`          `192.168.122.140`   `Ready`

This confirmed that normal `kubectl` access was restored and all nodes
reported `Ready` at the time of verification.

### Step 8 --- Verify pods across all namespaces

**Why?** A node can be `Ready` while an application or system pod has a
problem. We checked the pods as a separate verification step.

``` bash
kubectl get pods -A -o wide
```

The output showed the following:

-   The API server, etcd, scheduler, and controller manager pods were
    `Running`.
-   CoreDNS pods were `Running`.
-   Canal and kube-proxy pods on all three nodes were `Running`.
-   The `nginc` application pod was `1/1 Running` on `k8s-worker-1`.
-   The Calico controller pod was `Running`.

Some pods had historical restart counts. A restart count alone does not
mean a pod is currently broken; we would inspect logs and events if a
pod were unhealthy or kept restarting.

------------------------------------------------------------------------

## 4. Why did the original command fail?

The immediate cause was that `kubectl` was reading a default
configuration with no usable context, while the working administrative
configuration existed separately at `/etc/kubernetes/admin.conf`.

A useful way to remember the relationship:

``` text
kubectl command
      |
      v
Reads kubeconfig
      |
      +-- Default: /root/.kube/config
      |       |
      |       +-- Initially missing or unusable
      |
      +-- Known working file: /etc/kubernetes/admin.conf
              |
              +-- Contains cluster, credentials, and context
```

After we placed the working configuration at `/root/.kube/config`,
normal `kubectl` commands worked.

### Did shutting down the worker VMs cause the missing configuration?

**We did not prove that.**

Shutting down worker VMs does not normally delete a kubeconfig file on
the control-plane VM. Our shell history did not show a recorded command
that deleted or moved `/root/.kube/config`, but shell history can be
incomplete.

Therefore, document the result accurately:

-   **Immediate cause identified:** the default kubeconfig was missing
    or unusable.
-   **Fix applied:** copied the known-working `admin.conf` to
    `/root/.kube/config` and verified the context.
-   **Historical cause:** unknown.

------------------------------------------------------------------------

## 5. What does `Ready` or `Running` actually prove?

These statuses are useful, but they do not prove everything.

-   **Node `Ready`:** Kubernetes considers the node ready for normal
    work.
-   **Pod `Running`:** Kubernetes reports the pod as running.
-   **`1/1` in the READY column:** one of one containers in that pod is
    ready.
-   **End-to-end application connectivity:** not tested in this
    troubleshooting session.

So the incident was recovered and the checks we performed passed. If the
application later shows a connectivity problem, we should test its
Service, endpoints, DNS, and network path separately.

------------------------------------------------------------------------

## 6. Quick commands for next time

Run these in order when `kubectl` behaves unexpectedly.

### A. Check the current context

``` bash
kubectl config current-context
```

### B. Check the default configuration file

``` bash
ls -l /root/.kube/config
```

### C. Test the administrative configuration directly

``` bash
kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes
```

If this works but the normal `kubectl get nodes` does not, compare the
default configuration with the known-working file before changing
cluster components.

### D. Check all nodes

``` bash
kubectl get nodes -o wide
```

### E. Check pods across all namespaces

``` bash
kubectl get pods -A -o wide
```

### F. Check the default kubeconfig permissions

``` bash
ls -l /root/.kube/config
```

For a root administrative kubeconfig, mode `600` is normally
appropriate.

------------------------------------------------------------------------

## 7. Troubleshooting lesson

**Do not restart or reset a Kubernetes cluster just because `kubectl`
reports an error.**

First check whether the problem is:

1.  The client configuration (`kubeconfig` and context).
2.  Connectivity to the Kubernetes API server.
3.  Node health.
4.  Pod or application health.

In this incident, checking the working `admin.conf` helped isolate the
immediate issue before we made changes. Restoring the default
configuration fixed normal access without resetting the cluster or
restarting Kubernetes components.

**Final outcome:** `kubectl` access restored, all three nodes `Ready`,
and all pods listed in the final check `Running`.
