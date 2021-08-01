DynDNS K3S
==========

**Dynamic DNS to Cloudflare with Kubernetes CronJob**

## Gather Values for secret.yaml

- Get the ZONE_ID under dash.cloudflare.com -> Your domain -> Overview -> API -> Zone ID
```
#example
ZONE_ID = 023e105f4ecef8ad9ca31a8372d0c353
```

- Get the RECORD_ID: https://api.cloudflare.com/#dns-records-for-a-zone-list-dns-records
```
#example
RECORD_ID = 372e67954025e0ba6aaa6d586b9e0b59
```

- Get your AUTH_KEY (Create a key with "DNS Edit" permissions): https://support.cloudflare.com/hc/en-us/articles/200167836-Where-do-I-find-my-Cloudflare-API-key-#12345680
```
#example
AUTH_KEY = c2547eb745049flc9320b638f5e225cf483cc5cfdda41
```

- Specify your domain
```
#example
NAME = example.com
```

## Base64 Encode Values gathered above
```
echo -n 'example.com' | openssl base64
```

## secret.yml
```
# secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: do-token
  namespace: default
type: Opaque
data: # Base64 Encoded
  ZONE_ID: 
  RECORD_ID: 
  AUTH_KEY: 
  NAME: 
```

## configmap.yml
```
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
    #!/bin/bash

    curl -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" \
        -H "Authorization: Bearer $AUTH_KEY" \
        -H "Content-Type: application/json" \
        --data "{\"type\":\"A\",\"name\":\"$NAME\",\"content\":\"`curl ifconfig.co`\"}"
```

## cronjob.yml
```
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

## Apply Cronjob
```
kubctl -n default -f dynDNS-k8s
```

### Inspired By https://github.com/mirioeggmann/cloudflare-ddns and merged with https://docs.k8s-at-home.com/guides/dyndns/
