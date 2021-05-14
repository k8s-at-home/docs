# Introduction

This is a guide on how to use native k8s `CronJob`s to sync your WAN IP to a
dns provider.

## Creating a `ConfigMap`

Let's get started by creating a `configmap` with the script to update to
your DNS provider.

!!! Note
    In this example we will be using DigitalOcean
    This can work with any provider as long as your script works

A quick search yielded [this gist](https://gist.github.com/kenmickles/6746968).
Let's go ahead and update it a bit to work in our `CronJob` while putting it
in a `configmap`.

```yaml
# configmap.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: dyndns-updater
  namespace: default
  labels:
    app.kubernetes.io/name: dyndns-updater
    app.kubernetes.io/instance: dyndns-updater
data:
  dyndns-updater.sh: |
    #!/bin/sh
    # Get your WAN IP
    IP=$(curl -s checkip.dyndns.org | grep -Eo '[0-9\.]+')
    # Get the Record ID associated with the $DOMAIN
    RECORD_ID=$(curl -X GET -H "Content-Type: application/json" -H "Authorization: Bearer ${TOKEN}" "https://api.digitalocean.com/v2/domains/${DOMAIN}/records")
    # Send request to update DNS Record
    curl -s -X PUT \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer $TOKEN" \
        -d "{\"data\":\"${IP}\"}" \
        "https://api.digitalocean.com/v2/domains/${DOMAIN}/records/${RECORD_ID}"
```

## Creating a `Secret`

Next step is to create a secret, this will require a `$TOKEN` passed thru to
the `CronJob`.

```yaml
# secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: do-token
  namespace: default
type: Opaque
data:
  # Your domain name in base64
  DOMAIN: fWataR3=
  # You API token in base64
  # Get token from https://cloud.digitalocean.com/settings/applications
  TOKEN: YWRtaW4=
```

## Creating the `CronJob`

Last step is to put it altogether is creating the `CronJob`.

This will run every hour, on the hour. Tweak the cron formula as needed.

```yaml
# cronjob.yaml
---
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  namespace: default
  name: dyndns-updater
spec:
  schedule: "0 * * * *"
  failedJobsHistoryLimit: 1
  successfulJobsHistoryLimit: 3
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: dyndns-updater
            image: curlimages/curl:7.75.0
            imagePullPolicy: IfNotPresent
            envFrom:
            - secretRef:
                name: do-token
            command:
            - "/bin/sh"
            - "/app/dyndns-updater.sh"
            volumeMounts:
            - name: dyndns-updater
              mountPath: /app/dyndns-updater.sh
              subPath: dyndns-updater.sh
              readOnly: true
          volumes:
          - name: dyndns-updater
            projected:
              defaultMode: 0775
              sources:
              - configMap:
                  name: dyndns-updater
                  items:
                  - key: dyndns-updater.sh
                    path: dyndns-updater.sh
```
