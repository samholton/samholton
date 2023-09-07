---
layout: post
title: "Postgres Major Upgrade in Containers"
date: 2023-09-04 13:40
categories: Kubernetes
tags: [postgres, postgres upgrade, container, authentik]
---

I was attempting to upgrade my Helm release for [Authentik](https://github.com/goauthentik/authentik) when I ran into an issue with the Postgres container not coming back up. The logs showed

```
DETAIL:  The data directory was initialized by PostgreSQL version 11, which is not compatible with this version 15.0.
```

Under the hood, the chart is using the Bitnami Postgres chart (see [here](https://github.com/goauthentik/helm/tree/main/charts/postgresql)). Surely Bitnami has a mechanism in place to perform Postgres major version upgrades??? No, not really. The currently recommended solution was to dump the data, create a new installation on the newer major version of Postgres, and then import the data.

Initially, I wanted to perform an in-place upgrade. Not because my data set was large (it was actually extremely small), but because I thought it should be easy. A week or so ago, I upgraded a chart using MariaDB and its major version upgrade just worked. No hassels, completely seamless.

I initially started looking at `pg_upgrade` and stumbled across [docker-postgres-upgrade](https://github.com/tianon/docker-postgres-upgrade). This project has pre-built containers with various permutations of Postgres versions. It has `11-to-15` which is what I needed. However, my homelab is currently using Longhorn for persistent volumes. I finally got fed up and didn't want to mount the PVC into the migration container and map the volumes. I had other higher priority things to get done, and just wanted to finish the upgrade. And like I said, my dataset was tiny.

## Before Starting

As I'm using Longhorn for persistent volumes, I created a snapshot on the volume before getting started. I also have daily backups in place and confirmed the previous night's backup was successful.


## Create the Data Export

Before starting this process, I stopped the Kubernetes deployments and stateful sets for Authentik to ensure no writes were happening in the database.

```bash
# this was for my specific case, you will need to tweak namespace and deployment names

# get postgres password
export POSTGRES_PASSWORD=$( \
  kubectl get secret --namespace authentik \
  authentik-postgresql \
  -o jsonpath="{.data.postgresql-postgres-password}" | base64 --decode
)

# create the database dump file
# could also likely exec into the container and keep the dump file in the PVC
# I found the image by inspecting the statefulset definition of currently postgres
kubectl run pg-authentik \
--rm --tty -i --restart='Never' \
--namespace authentik \
--env="PGPASSWORD=$POSTGRES_PASSWORD" \
--image docker.io/bitnami/postgresql:11.19.0-debian-11-r4 \
--command -- pg_dumpall --host authentik-postgresql -U postgres > pg-authentic-11-export

# useful command to get a postgres container inside the namespace to verify connection
kubectl run authentik-pg-test --rm -i --namespace authentik --image docker.io/bitnami/postgresql:11.19.0-debian-11-r4 -- /bin/bash
```

## Update the Statefulset and Wipe the Data

At this point, I could have created a new PVC with a Postgres 15 container. After recovering the data, I would need to use this PVC with the Helm chart. I didn't even check to see what that would look like (specifying an already existing PVC), but Bitnami charts normally provide support for that scenario.

Instead, I decided to take the quick and hacky route:

1. Find the new Postgres container image for the updated Helm chart (found [here](https://github.com/goauthentik/helm/blob/main/charts/postgresql/values.yaml#L101))
1. Manually update the stateful set with that image (in my case it was `15.4.0-debian-11-r0`)
1. Manually update the stateful set `command` to not start the database
    ```yaml
    command: ['sleep', 'infinity']
    ```
1. At a shell in the container, remove all of the Postgres data
    ```bash
    rm -rf /bitnami/postgresql/data/*
    ```
1. Revert the `command` change above and restart the container
1. Copy the data dump into the new Postgres container
    ```bash
    kubectl -n authentik cp pg-authentik-11-export authentik-postgresql-0:/tmp/pg-authentik-11-export
    ```
1. At a shell in the container, import the data
    ```bash
    psql -U postgres < /tmp/pg-authentik-11-export
    ```
1. Cleanup the data dump (not really necessary as it is in `/tmp`)
    ```bash
    rm /tmp/pg-authentik-11-export
    ```
1. Perform the Helm update and confirm all services come back online
    ```bash
    helm upgrade \
    --install \
    --values values.yaml \
    --namespace authentik \
    --create-namespace \
    --version 2023.8.2 \
    authentik authentik/authentik
    ```