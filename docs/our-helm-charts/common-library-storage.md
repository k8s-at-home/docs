# Common library Storage

Most of our application Helm charts consume our Common library Helm chart.
This page describes the different ways you can attach storage to these charts.

## Types

These are the types of storage that we support in our common library.
Of course, other types are possible with the `custom` type.

### Persistent Volume Claim

This is probably the most common storage type, therefore it is also the
default when no `type` is specified.

It can be attached in two ways.

#### Dynamically provisioned

Our charts can be configured to create the required persistentVolumeClaim
manifests on the fly.

| Field           | Mandatory | Docs / Description                                                                    |
| --------------- | --------- | ------------------------------------------------------------------------------------- |
| `type`          | Yes       |                                                                                       |
| `accessMode`    | Yes       | [link](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)  |
| `size`          | Yes       | [link](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#resources)     |
| `mountPath`     | No        | Where to mount the volume in the main container. Defaults to `/<name_of_the_volume>`. |
| `readOnly`      | No        | Specify if the volume should be mounted read-only.                                    |
| `nameOverride`  | No        | Override the name suffix that is used for this volume.                                |
| `storageClass`  | No        | Storage class to use for this volume.                                                 |
| `skipuninstall` | No        | Set to true to retain the PVC upon `helm uninstall`.                                  |

Minimal config:

```yaml
persistence:
  config:
    type: pvc
    accessMode: ReadWriteOnce
    size: 1Gi
```

This will create a 1Gi RWO PVC named `RELEASE-NAME-config` with the default
storageClass, which will mount to `/config`.

#### Existing claim

Our charts can be configured to attach to a pre-existing persistentVolumeClaim.

| Field           | Mandatory | Docs / Description                                                                    |
| --------------- | --------- | ------------------------------------------------------------------------------------- |
| `type`          | Yes       |                                                                                       |
| `existingClaim` | Yes       | Name of the existing PVC                                                              |
| `mountPath`     | No        | Where to mount the volume in the main container. Defaults to `/<name_of_the_volume>`. |
| `subPath`       | No        | Specifies a sub-path inside the referenced volume instead of its root.                |
| `readOnly`      | No        | Specify if the volume should be mounted read-only.                                    |
| `nameOverride`  | No        | Override the name suffix that is used for this volume.                                |

Minimal config:

```yaml
persistence:
  config:
    type: pvc
    existingClaim: myAppData
```

This will mount an existing PVC named `myAppData` to `/config`.

### Empty Dir

Sometimes you need to share some data between containers, or need some
scratch space. That is where an emptyDir can come in.

See the [Kubernetes docs](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
for more information.

| Field           | Mandatory | Docs / Description                                                                                               |
| --------------- | --------- | ---------------------------------------------------------------------------------------------------------------- |
| `medium`        | No        | Set this to `Memory` to mount a tmpfs (RAM-backed filesystem) instead of the storage medium that backs the node. |
| `sizeLimit`     | No        | If the `SizeMemoryBackedVolumes` feature gate is enabled, you can specify a size for memory backed volumes.      |
| `mountPath`     | No        | Where to mount the volume in the main container. Defaults to `/<name_of_the_volume>`.                            |
| `nameOverride`  | No        | Override the name suffix that is used for this volume.                                                           |

Minimal config:

```yaml
persistence:
  config:
    type: emptyDir
```

This will create an ephemeral emptyDir volume and mount it to `/config`.

### Host path

### Custom

#### configMap

#### Secret

#### NFS Volume

## Permissions

## Multiple subPaths for 1 volume
