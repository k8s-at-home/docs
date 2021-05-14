# Permissions

With Kubernetes, `s6-overlay` is not needed. Instead Kubernetes can use
[Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
on either the pod or container to tell what the container should run as,
and/or what permissions files should be written as.

## Managing Permissions

There are several different methods you can use to make these containers have
write access to your file storage.

!!! note
    Our images use a default user/group id of `568`. The user and group ids
    cannot be changed at the container runtime.

### Security Contexts method

You can change the Kubernetes
[Security Contexts](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
to allow the container to have permissions to write to your file storage.

In our [Helm charts](https://github.com/k8s-at-home/charts/) this can be accomplished
by setting the following option in you Helm values.

!!! note
    According to the Kubernetes docs `fsGroup` will `chown`
    the volume with the `runAsUser` and `runAsGroup` IDs.

    As such this option should only have to be set once
    or it may cause your pod to take long to start each
    time it is started.

<!-- markdownlint-disable-next-line MD046 -->
```yaml
podSecurityContext:
  runAsUser: 568
  runAsGroup: 568
  fsGroup: 568
```

### initContainer method

Implement a `initContainer` that runs as root to automatically chown the volume's
data.

<!-- markdownlint-disable-next-line MD046 -->
```yaml
initContainers:
- name: update-volume-permission
  image: busybox
  command: ["sh", "-c", "chown -R 568:568 /config"]
  volumeMounts:
  - name: config
    mountPath: /config
  securityContext:
    runAsUser: 0
```

### Direct volume method

!!! warning
    This step can be a bit complicated if you are not very familiar with your
    storage interface.

If you can mount the volume's config without the pod running. You can run
`chown -R 568:568 <path-to-the mounted-volume-on-the host>`.

As this method varies greatly depending on your CSI driver, we won't be
mentioning how to accomplish this.
