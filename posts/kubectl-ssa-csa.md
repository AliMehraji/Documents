---
title: TIL Kubectl CSA/SSA (Client/Server Side Apply)
published: true
description: digging the server side apply and client side apply
tags: 'kubernetes,devops,k8s'
cover_image: ./assets/kubectl-ssa-csa.webp
id: 2566618
date: '2025-06-05T14:39:43Z'
---

Kubectl CSA(Client Side Apply) and SSA(Server Side Apply).

The need to read and dig into the SSA and CSA came from [command Annotation in Installation Requirements manifests ?][longhorn-discussion] and which was validating with [`dry-run`][dry-run] that Question Came from a [TIL about `nsenter`][til-nsenter].

## [Field management][field-mngmnt]

</br>

> The Kubernetes API server tracks managed fields for all newly created objects. When trying to apply an object, fields that have a different value and are owned by another manager will result in a conflict. This is done in order to signal that the operation might undo another collaborator's changes. Writes to objects with managed fields can be forced, in which case the value of any conflicted field will be overridden, and the ownership will be transferred.
>
> Whenever a field's value does change, ownership moves from its current manager to the manager making the change.
>
> Compared to the (legacy) kubectl.kubernetes.io/last-applied-configuration annotation managed by kubectl, Server-Side Apply uses a more declarative approach, that tracks a user's (or client's) field management, rather than a user's last applied state. As a side effect of using Server-Side Apply, information about which field manager manages each field in an object also becomes available.
>
> Field management details are stored in a managedFields field that is part of an object's metadata.
> The field management record consists of basic information about the managing entity itself, plus details about the fields being managed and the relevant operation (Apply or Update). If the request that last changed that field was a **Server-Side Apply patch** then the value of operation is **Apply**; otherwise, it is **Update**.
>
> kubectl get omits managed fields by default. Add `--show-managed-fields` to show `managedFields` when the output format is either `json` or `yaml`.

### Apply

Lets apply(create) a pod then check those fields.

`netshoot.yaml`

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: netshoot
spec:
  containers:
    - name: netshoot
      image: nicolaka/netshoot:latest
      command: ["sleep", "infinity"]
      securityContext:
        capabilities:
          add: ["NET_ADMIN", "SYS_MODULE"]
  restartPolicy: OnFailure
```

```bash
kubectl apply --server-side --dry-run=server -f netshoot.yaml -o yaml -n test-tool --show-managed-fields | yq .metadata.managedFields
```

<details>
  <summary>managedFields</summary>

```yaml
managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:spec:
        f:containers:
          k:{"name":"netshoot"}:
            .: {}
            f:command: {}
            f:image: {}
            f:name: {}
            f:securityContext:
              f:capabilities:
                f:add: {}
        f:restartPolicy: {}
    manager: kubectl
    operation: Apply
    time: "2025-06-04T20:43:41Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:cni.projectcalico.org/containerID: {}
          f:cni.projectcalico.org/podIP: {}
          f:cni.projectcalico.org/podIPs: {}
    manager: calico
    operation: Update
    subresource: status
    time: "2025-06-04T20:43:42Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          f:ambient.istio.io/redirection: {}
    manager: install-cni
    operation: Update
    subresource: status
    time: "2025-06-04T20:43:42Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:status:
        f:conditions:
          k:{"type":"ContainersReady"}:
            .: {}
            f:lastProbeTime: {}
            f:lastTransitionTime: {}
            f:status: {}
            f:type: {}
          k:{"type":"Initialized"}:
            .: {}
            f:lastProbeTime: {}
            f:lastTransitionTime: {}
            f:status: {}
            f:type: {}
          k:{"type":"Ready"}:
            .: {}
            f:lastProbeTime: {}
            f:lastTransitionTime: {}
            f:status: {}
            f:type: {}
        f:containerStatuses: {}
        f:hostIP: {}
        f:phase: {}
        f:podIP: {}
        f:podIPs:
          .: {}
          k:{"ip":"10.237.81.196"}:
            .: {}
            f:ip: {}
        f:startTime: {}
    manager: kubelet
    operation: Update
    subresource: status
    time: "2025-06-04T20:43:43Z"

```

</details>

### Change some fields in `magaedFields`

> A Server-Side Apply patch request requires the client to provide its identity as a field manager. When using Server-Side Apply, trying to change a field that is controlled by a different manager results in a rejected request unless the client forces an override. For details of overrides, see [Conflicts][conflicts].

Invoke below command to save the current(live) state of the pod in a file, `netshoot-current.yaml`.

```bash
k get -n test-tool pod netshoot --show-managed-fields -o yaml | tee netshoot-current.yaml
```

Change the `podIP` and `podIPs`, and apply it via:

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    cni.projectcalico.org/podIP: CHANGE_LAST_OCTET
    cni.projectcalico.org/podIPs: CHANGE_LAST_OCTET
```

```bash
kubectl apply --server-side -f netshoot-current.yaml -n test-tool
```

As we expected the [Conflict][conflicts] happened, because the owner of those ips fields is the CNI, in this case is `calico`.

```bash
Error from server (BadRequest): metadata.managedFields must be nil
```

```yaml
- apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:cni.projectcalico.org/containerID: {}
          f:cni.projectcalico.org/podIP: {}
          f:cni.projectcalico.org/podIPs: {}
    manager: calico
    operation: Update
    subresource: status
    time: "2025-06-04T20:43:42Z"

```

Lets do the apply command verbose with `-v=6` to see what exactly happening with [imperative endpoints][imperative-endpoints] under the hood.

```bash
kubectl apply --server-side -f netshoot-current.yaml-n test-tool -v=6
```

```bash
"Response" verb="PATCH" url="https://rancher.example.com/k8s/clusters/<CLUSTER_ID>/api/v1/namespaces/test-tool/pods/netshoot?fieldManager=kubectl&fieldValidation=Strict&force=false" status="400 Bad Request" milliseconds=80
```

### `kubectl apply --server-side` (rejected with Bad Request)

- Respects field ownership strictly
  - `fieldValidation=Strict` in query params in `PATCH` request.
- Calico owns the `PodIP`/`PodIPs` fields
  - apply is going to change the `calico` field manager to `fieldManager=kubectl` query param in `PATCH` request.
  - For other updates, the API server infers a [Field manager][field-managers] identity from the "User-Agent:" HTTP header (if present).
  - The `force` is `false` in the `server-side` apply to respect the ownership.
- Kubernetes server rejects the conflict because you don't own those fields
  - `status="400 Bad Request` in response, This is the correct **protective** behavior.

    ```bash
    Bad Request" milliseconds=80
    I0605 00:40:45.502024   24714 helpers.go:246] server response object: %s[{
      "kind": "Status",
      "apiVersion": "v1",
      "metadata": {},
      "status": "Failure",
      "message": "metadata.managedFields must be nil",
      "reason": "BadRequest",
      "code": 400
    }]
    Error from server (BadRequest): metadata.managedFields must be nil
    ```

### `kubectl apply` or `kubectl edit` (succeeded)

If we don't use the `--server-side` which is going to be the default apply (CSA), in that case the apply wil use the default `force=true` in CSA.

> It is however possible to change `.metadata.managedFields` through an `update`, or through a `patch` operation that does not use Server-Side Apply. Doing so is ***highly discouraged***, but might be a reasonable option to try if, for example, the `.metadata.managedFields` get into an inconsistent state (which should not happen in normal operations).
>
> The format of managedFields is described in the Kubernetes API reference.
>
> Caution:
>
> The `.metadata.managedFields` field is managed by the API server. You should avoid updating it manually.

```bash
"Response" verb="PATCH" url="https://rancher.example.com/k8s/clusters/<CLUSTER_ID>/api/v1/namespaces/test-tool/pods/netshoot?fieldManager=kubectl-client-side-apply&fieldValidation=Strict" status="200 OK" milliseconds=97
pod/netshoot configured
```

- Uses Client-Side Apply (strategic merge patch)
  - `fieldManager=kubectl-client-side-apply` in query params in `PATCH` request.
- Calculates the patch on the client side
- Sends a PATCH request that can override existing fields
- Less strict about field ownership conflicts
  - It does validate with `fieldValidation=Strict` in query param in `PATCH` request, but the force is force.
- Essentially "forces" the change through

As you can see, the manager(owner) of the `podIP` and `podIPs` fields changed to `kubectl-client-side-apply` and the operation is `Update`.

```yaml
- apiVersion: v1
  fieldsType: FieldsV1
  fieldsV1:
    f:metadata:
      f:annotations:
        f:cni.projectcalico.org/podIP: {}
        f:cni.projectcalico.org/podIPs: {}
        f:kubectl.kubernetes.io/last-applied-configuration: {}
  manager: kubectl-client-side-apply
  operation: Update
  time: "2025-06-04T21:26:09Z"

```

## Environment

Kubernetes cluster deployed with Kubespray and added to a rancher cluster manually.

## Resources

- [Kubectl Apply][kubectl-apply]
- [Server-Side Apply][server-side-apply]
- [Break Down K8S Server-Side Apply][breakdown-k8s-ssa]
- [Comparison with Client-Side Apply][comparison-with-client-side-apply]
- [Argo-CD Server-Side Apply][argocd-ssa]

[kubectl-apply]: https://youtu.be/1DWWlcDUxtA
[breakdown-k8s-ssa]: https://medium.com/swlh/break-down-kubernetes-server-side-apply-5d59f6a14e26
[comparison-with-client-side-apply]: https://kubernetes.io/docs/reference/using-api/server-side-apply/#comparison-with-client-side-apply
[server-side-apply]: https://kubernetes.io/docs/reference/using-api/server-side-apply/
[longhorn-discussion]: https://github.com/longhorn/longhorn/discussions/11008
[til-nsenter]: https://dev.to/alimehr75/til-may-28-2025-5547
[field-mngmnt]: https://kubernetes.io/docs/reference/using-api/server-side-apply/#field-management
[argocd-ssa]: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/#server-side-apply
[conflicts]: https://kubernetes.io/docs/reference/using-api/server-side-apply/#conflicts
[dry-run]: https://youtu.be/1DWWlcDUxtA?t=704
[imperative-endpoints]: https://youtu.be/1DWWlcDUxtA?t=638
[field-managers]: https://kubernetes.io/docs/reference/using-api/server-side-apply/#managers
