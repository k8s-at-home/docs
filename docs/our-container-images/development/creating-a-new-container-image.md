# Creating a new container image

## Dependencies

If you would like to help create new container images, there's a few tools
you will need:

- [Docker](https://www.docker.com/get-started)
- [buildx](https://docs.docker.com/buildx/working-with-buildx/)
- [jq](https://stedolan.github.io/jq/)
- [goss] & [dgoss]
- [task (optional)](https://github.com/go-task/task)

## Creating a new container image

Currently, there isn't a template to start from and so you can either start
with an already existing app or you can start from scratch.

1. Create a folder in in the `apps` folder with the name of the container
image.
2. Add the following files to the container image folder.

### Folder structure

```shell
apps/app name/
     ├── Dockerfile
     ├── entrypoint.sh
     ├── goss.yaml
     ├── latest-version.sh
     ├── PLATFORM
     ├── shim
     │   └── shim-script.sh
     └── VERSION
```

### Dockerfile

See [Dockerfile](./dockerfile.md).

### PLATFORM

The `PLATFORM` file is used by the
[Apps - Build, Test, Push github action][apps.yaml] to build
[multi-platform] container images.

At a minimum, the following platforms should be supported by the container
image:

- `amd64`
- `arm64`

```shell
linux/amd64,linux/arm64
```

### VERSION

The `VERSION` file is used by the
[Apps - Build, Test, Push github action][apps.yaml] to determine the version
of the app with which to build the container image.

The VERSION file may be updated manually by running the
[latest-version.sh](#latest-version.sh) script.

```shell
./latest-version.sh > VERSION
```

### entrypoint.sh

The `entrypoint.sh` file is a standard [Docker entrypoint file][entrypoint.sh].
This file will most likely be custom for each app. It is recommended to take a
look at other container images or other versions of the Docker image for the
app. A good source might be searching [Docker Hub](https://hub.docker.com/).

The only difference is the sourcing of the [base image shim scripts]
which are added to the base image.

```bash
#shellcheck disable=SC1091
source "/shim/umask.sh"
source "/shim/vpn.sh"
```

If the app has custom [shim scripts](#shim-scripts), be sure to source those
as well.

```shell
source "/shim/<app name>-preferences.sh"
```

Ensure to append the `${EXTRA_ARGS}` environmental variable to the end of the
`exec` command so that extra args can be passed to the container by the helm
chart.

```shell
exec /app/foo ${EXTRA_ARGS}
```

### goss.yaml

The `goss.yaml` file is used by the
[Apps - Build, Test, Push github action][apps.yaml] to perform a health
check on the container image using [goss](https://github.com/aelsabbahy/goss).

Be sure to update the process name, port number(s) and `title`.

See the [manual] for more options.

```yaml
---
process:
  Lidarr:
    running: true

port:
  tcp6:8686:
    listening: true

http:
  http://localhost:8686:
    status: 200
    body:
    - '<title>Lidarr</title>'
```

### latest-version.sh

The `latest-version.sh` script is used by the
[Apps - Get latest versions github action][apps-schedule.yaml] to get the
latest version of the app. This script is custom for each app and so it will
need to be developed for each app. Below is an example of one.

```bash
#!/usr/bin/env bash

# Get the version using jq from the json response.
version=$(curl -sX GET "https://lidarr.servarr.com/v1/update/nightly/changes?os=linux" | jq --raw-output '.[0].version')
# Strip the v from the beginning of version if it exists.
version="${version#*v}"
# Remove release and dash from the beginning of version if it exists.
version="${version#*release-}"
# Print the verion without a new line (\n) character.
printf "%s" "${version}"
```

### shim scripts

Custom startup scripts should be added to the `shim` folder that needs to be
run before the app is started.

Create a `shim` folder in the app name folder then add the shim scripts to the
`shim` folder.

Then add following line to the [Dockerfile](./dockerfile.md#shim-scripts) to
copy the scripts over to the image.

Then be sure to add a `source` line to the [entrypoint.sh script](#entrypointsh).

See [plex] for reference.

[base image shim scripts]: ./base-images.md#shim-scripts
[goss]: https://github.com/aelsabbahy/goss
[dgoss]: https://github.com/aelsabbahy/goss/tree/master/extras/dgoss
[apps-schedule.yaml]: https://github.com/k8s-at-home/container-images/actions/workflows/apps-schedule.yaml
[apps.yaml]: https://github.com/k8s-at-home/container-images/actions/workflows/apps.yaml
[manual]: https://github.com/aelsabbahy/goss/blob/master/docs/manual.md
[multi-platform]: https://docs.docker.com/buildx/working-with-buildx/#build-multi-platform-images
[entrypoint.sh]: https://docs.docker.com/engine/reference/builder/#entrypoint
[dockerfile]: https://docs.docker.com/engine/reference/builder/
[from]: https://docs.docker.com/engine/reference/builder/#from
[plex]: https://github.com/k8s-at-home/container-images/tree/main/apps/plex
