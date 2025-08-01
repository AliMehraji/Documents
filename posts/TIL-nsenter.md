---
title: TIL `nsenter`
published: true
description: Today I Learned About nsenter
tags: '#career,#k8s,#kubernetes'
id: 2537346
date: '2025-05-28T13:05:36Z'
---

Host Level operations on a K8S node with privileged container.

## Annotations

Set an annotation named `command` and use [yaml anchor][yaml-reference-card] `&cmd` to use it later.

> In YAML, &cmd creates an anchor named cmd. You can later reference it using *cmd.

[Kubernetes Annotations Document][kubernetes-annotations]

> You can use Kubernetes annotations to attach arbitrary non-identifying metadata to objects. Clients such as tools and libraries can retrieve this metadata.</br>
> You can use either labels or annotations to attach metadata to Kubernetes objects. Labels can be used to select objects and to find collections of objects that satisfy certain conditions. In contrast, annotations are not used to identify and select objects. The metadata in an annotation can be small or large, structured or unstructured, and can include characters not permitted by labels. It is possible to use labels as well as annotations in the metadata of the same object.

**But** wait, the `&cmd` yaml anchor does nothing to do with the annotation, so im waiting for the discussion.

[[QUESTION] command Annotation in Installation Requirements manifests ?][command-Annotation-discussion]

**[Update]**: [Discussion Answer][discussion-answer]

[Longhorn NFS Installation][longhorn-nfs-installation]

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: longhorn-nfs-installation
  namespace: longhorn-system
  labels:
    app: longhorn-nfs-installation
  annotations:
    command: &cmd OS=$(grep -E "^ID_LIKE=" /etc/os-release | cut -d '=' -f 2); if [[ -z "${OS}" ]]; then OS=$(grep -E "^ID=" /etc/os-release | cut -d '=' -f 2); fi; if [[ "${OS}" == *"debian"* ]]; then sudo apt-get update -q -y && sudo apt-get install -q -y nfs-common && sudo modprobe nfs; elif [[ "${OS}" == *"suse"* ]]; then sudo zypper --gpg-auto-import-keys -q refresh && sudo zypper --gpg-auto-import-keys -q install -y nfs-client && sudo modprobe nfs; else sudo yum makecache -q -y && sudo yum --setopt=tsflags=noscripts install -q -y nfs-utils && sudo modprobe nfs; fi && if [ $? -eq 0 ]; then echo "nfs install successfully"; else echo "nfs install failed error code $?"; fi
spec:
  selector:
    matchLabels:
      app: longhorn-nfs-installation
  template:
    metadata:
      labels:
        app: longhorn-nfs-installation
    spec:
      hostNetwork: true
      hostPID: true
      initContainers:
      - name: nfs-installation
        command:
          - nsenter
          - --mount=/proc/1/ns/mnt
          - --
          - bash
          - -c
          - *cmd
        image: alpine:3.12
        securityContext:
          privileged: true
      containers:
      - name: sleep
        image: registry.k8s.io/pause:3.1
  updateStrategy:
    type: RollingUpdate
```

## Host Level Operation on K8S Node within a container

Was wondering how a container can install a package or do host level commands!?

```yaml
spec:
      hostNetwork: true
      hostPID: true
      initContainers:
      - name: nfs-installation
        command:
          - nsenter
          - --mount=/proc/1/ns/mnt
          - --
          - bash
          - -c
          - *cmd
        image: alpine:3.12
        securityContext:
          privileged: true
```

### `command` breakdown

- [`nsenter`][nsenter]

  > `nsenter` is a Linux utility used to enter different namespaces of other processes — like the network, mount, PID, UTS, etc. This is useful when you want to operate inside another process's context (like the host) from within a container.

- `--mount=/proc/1/ns/mnt`

  > enter the mount namespace of process 1 (usually the host system’s init process inside the node).</br>
  > `/proc/1/ns/mnt` special file that represents the mount namespace of PID 1 (host).

- `-- bash -c *cmd`

  > Separator; everything after the `--` is passed to the new shell.
  > new shell runs the cmd, which is defined in the annotation of Daemonset, and as described in the [K8S Annotaion][kubernetes-annotations] document, the container can access to `command` annotation.

For the host(node) level operations, the `nsenter` needs privileges

- `hostNetwork: true`

    What it does:

    - Makes the Pod (and all containers in it) use the host’s network namespace.
    - That means it shares the host’s network stack — same IP address, routing table, etc.

    Why it’s needed:

    - If the command inside the init container needs to:
    - Access network services that are only available on the host network (e.g., certain NFS servers or internal routes),
    - Or resolve hostnames as the host does, this gives it direct access. </br></br>

  ```bash
  kubectl explain ds.spec.template.spec.hostNetwork
  ```

  ```txt
  hostNetwork	<boolean>
    Host networking requested for this pod. Use the host's network namespace. If
    this option is set, the ports that will be used must be specified. Default
    to false.
  ```

- `hostPID: true`

    What it does:
    - Makes the Pod share the host’s PID namespace.
    - So the container can see and interact with host processes, including PID 1.

    Why it’s needed:

    - Using `--mount=/proc/1/ns/mnt`, path points to PID 1’s mount namespace — which is usually the host’s root process (init/systemd).
    - Without hostPID: true, PID 1 inside the container is not the host’s PID 1 — and nsenter would fail because /proc/1/ns/mnt would refer to the container’s own namespace, not the host’s. </br></br>

  ```bash
  kubectl explain ds.spec.template.spec.hostPID
  ```

  ```txt
    hostPID <boolean>
    Use the host's pid namespace. Optional: Default to false.
  ```

- `initContainers.securityContext.privileged: true`

  ```bash
  kubectl explain ds.spec.template.spec.initContainers.securityContext.privileged
  ```

  ```txt
  Run container in privileged mode. Processes in privileged containers are essentially equivalent to root on the host.
  ```

## Additional notes from ChatGPT

🧠 What Happens When You Use It?

With `nsenter --mount=/proc/1/ns/mnt`,You are telling your container

> "Hey, stop using your own isolated mount namespace. Enter the host's mount namespace." </br>
> This lets your process (inside a container) perform filesystem operations as if it were on the host.

For example:

```bash
nsenter --mount=/proc/1/ns/mnt -- bash -c 'mount -t nfs ...'

```

🔒 Permissions Required

To access `/proc/1/ns/mnt`:

- You must have `CAP_SYS_ADMIN`.
- In Kubernetes, that typically means the container must be run as `privileged: true`.
- You also need `hostPID: true` to ensure `PID 1` is the host `PID 1`, not the container's own init.

## Resources

- [Kubernetes Annotations][kubernetes-annotations]
- [longhorn-nfs-installation][longhorn-nfs-installation]
- [container vs pods][ctr-vs-pod]
- [Namespaces][namespaces]
- [lsns][ls-ns]
- [proc][proc]

[longhorn-nfs-installation]: https://raw.githubusercontent.com/longhorn/longhorn/v1.9.0/deploy/prerequisite/longhorn-nfs-installation.yaml
[kubernetes-annotations]: https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/
[nsenter]: https://man7.org/linux/man-pages/man1/nsenter.1.html
[yaml-reference-card]: https://yaml.org/refcard.html
[ctr-vs-pod]: https://labs.iximiuz.com/tutorials/containers-vs-pods/
[namespaces]: https://man7.org/linux/man-pages/man7/namespaces.7.html
[ls-ns]: https://man7.org/linux/man-pages/man8/lsns.8.html
[proc]: https://man7.org/linux/man-pages/man5/proc.5.html#
[command-Annotation-discussion]: https://github.com/longhorn/longhorn/discussions/11008
[discussion-answer]: https://github.com/longhorn/longhorn/discussions/11008#discussioncomment-13321110
