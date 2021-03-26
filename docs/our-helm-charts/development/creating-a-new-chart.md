# Creating a new chart

## Dependencies

If you would like to help create new charts using the common library, there's
a few tools you will need.

- [helm](https://helm.sh/docs/intro/install/)
- [helm-docs](https://github.com/norwoodj/helm-docs)
- [task](https://github.com/go-task/task) (optional)

## Creating a new chart

To create a new chart, run the following:

```sh
# Clone
git clone
cd charts
sh -c "$(curl --location https://taskfile.dev/install.sh)" -- -d -b .bin

# Create chart
PATH=$PATH:$PWD/.bin
task deps:install
task chart:create CHART=chart-name
# Don't forgot edit some chart informations in charts/chart-name/Chart.yaml and charts/chart-name/values.yaml
```

Second, be sure to checkout the many charts that already use this like
[qBittorrent](https://github.com/k8s-at-home/charts/tree/master/charts/stable/qbittorrent),
[node-red](https://github.com/k8s-at-home/charts/tree/master/charts/stable/node-red)
or the many others in this repository.

Include this chart as a dependency in your `Chart.yaml` e.g.

```yaml
# Chart.yaml
...
dependencies:
- name: common
  version: 3.0.1 # make sure to use the latest common library version available
  repository: https://library-charts.k8s-at-home.com
...
```

## Values

Write a `values.yaml` with some basic defaults you want to present to the user
e.g.

```yaml
#
# IMPORTANT NOTE
#
# This chart inherits from our common library chart. You can check the default values/options here:
# https://github.com/k8s-at-home/library-charts/tree/master/charts/stable/common/values.yaml
#

image:
  repository: nodered/node-red
  pullPolicy: IfNotPresent
  tag: 1.2.5

strategy:
  type: Recreate

# See more environment variables in the node-red documentation
# https://nodered.org/docs/getting-started/docker
env: {}
  # TZ:
  # NODE_OPTIONS:
  # NODE_RED_ENABLE_PROJECTS:
  # NODE_RED_ENABLE_SAFE_MODE:
  # FLOWS:

service:
  port:
    port: 1880

ingress:
  enabled: false

persistence:
  data:
    enabled: false
    emptyDir: false
    mountPath: /data
```

If not using a service, set the `service.enabled` to `false`.

```yaml
...
service:
  enabled: false
...
```

## Templates

### Basic

In its most basic form a new chart can consist of two simple files in the
`templates` folder. This will automatically render everything, based only on
what is (or isn't) present in `values.yaml`.

**`templates/common.yaml`**:

```yaml
{{ include "common.all . }}
```

**`templates/NOTES.txt`**:

```yaml
{{ include "common.notes.defaultNotes" . }}
```

### Advanced

Sometimes it is not required to implement additional logic in a chart that you
do not wish to expose through settings in `values.yaml`. For example, when you
want to always mount a Secret or configMap as a volume in the Pod. In that
case it is also possible to write more advanced template files.

**`templates/common.yaml`**:

```yaml
{{/* First Make sure all variables are set and merged properly */}}
{{- include "common.values.setup" . }}

{{/* Append a configMap to the additionalVolumes */}}
{{- define "myapp.configmap.volume" -}}
name: myapp-settings
configMap:
  name: {{ template "common.names.fullname" . }}-settings
{{- end -}}

{{- $volume := include "myapp.configmap.volume" . | fromYaml -}}
{{- if $volume -}}
    {{- $additionalVolumes := append .Values.additionalVolumes $volume }}
    {{- $_ := set .Values "additionalVolumes" (deepCopy $additionalVolumes) -}}
{{- end -}}

{{/* Render the templates */}}
{{ include "common.all" . }}
```

An actual example of this can be found in the [zigbee2mqtt][zigbee2mqtt] chart.

### Testing

If testing locally, make sure you update the dependencies with from the chart
directory:

```bash
helm dependency update
```

If making local changes to the `common` library, the test chart may reference
the local development chart:

```yaml
# common-test/Chart.yaml
...
dependencies:
- name: common
  version: <new version>
  repository: file://.../common
...
```

Be sure to lint your chart to check for any errors.

```sh
# Linting
task chart:lint CHART=chart-name
task chart:ct-lint CHART=chart-name
```

[zigbee2mqtt]: https://github.com/k8s-at-home/charts/tree/master/charts/stable/zigbee2mqtt
