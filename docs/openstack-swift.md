# Deploy Swift

OpenStack Swift is the object storage service of the OpenStack platform. It stores unstructured data as objects within containers, distributing and replicating those objects across a ring of storage devices so the cluster keeps serving reads and writes while individual disks or nodes are unavailable. This document details the deployment of OpenStack Swift within Genestack.

> Swift is the one OpenStack service in Genestack that uses neither MariaDB nor a message queue on its data path. Account and container metadata live in sqlite databases on the storage devices themselves, and object placement is driven by the rings rather than a central database.

!!! genestack "Object storage has two backend options"

    Genestack can serve the Object Store with native Swift, documented here, or
    with an external Ceph RADOS Gateway. They register the same service and
    endpoints and cannot run together. Review
    [Object Storage Backend Options](openstack-object-storage-backends.md) before
    deploying, particularly if object storage usage feeds billing.

## Prerequisites

Swift stores objects on the block devices of the nodes running the `swift-storage` daemonset, so the devices and node labels must exist before the chart is deployed.

!!! note "Label the storage nodes"

    The `swift-storage` daemonset is scheduled with the `openstack-storage-node` label described in [Kubernetes Labels](k8s-labels.md).

    ``` shell
    kubectl label node ${NODE_NAME} openstack-storage-node=enabled
    ```

!!! warning "Mount the devices before deploying"

    Every device named in `ring.devices` must already be mounted at `/srv/node/<name>` on each labelled storage node. The ring-builder job records the device names, and the storage servers run with `mount_check = true`, so an unmounted path is skipped rather than silently filled with data on the root filesystem.

    ``` shell
    mkfs.xfs -L sdb1 /dev/sdb1
    mkdir -p /srv/node/sdb1
    echo 'LABEL=sdb1 /srv/node/sdb1 xfs noatime,nodiratime,logbufs=8 0 2' >> /etc/fstab
    mount /srv/node/sdb1
    chown -R swift:swift /srv/node/sdb1
    ```

!!! example "Declare the devices and ring layout"

    The base overrides ship an empty `ring.devices` list because the device names are site specific. Set them in `/etc/genestack/helm-configs/swift/swift-helm-overrides.yaml`.

    ``` yaml
    ring:
      partition_power: 14
      replicas: 3
      min_part_hours: 24
      devices:
        - name: sdb1
          weight: 100
        - name: sdc1
          weight: 100
    ```

    `partition_power` cannot be changed later without rebuilding the rings, so size it for the cluster you expect to grow into rather than the one you are starting with. The base value of `10` (1024 partitions) suits a lab; production clusters generally want `14` or higher.

The ring files are built once and shared with every storage pod through a `ReadWriteMany` claim backed by the `general-multi-attach` storage class. Override `ring.shared_storage.storageClassName` if your cluster exposes a different RWX class.

## Create secrets

!!! note "Information about the secretes used"

    Manual secret generation is only required if you haven't run the `create-secrets.sh` script located in `/opt/genestack/bin`.

    ??? example "Example secret generation"

        ``` shell
        kubectl --namespace openstack \
                create secret generic swift-rabbitmq-password \
                --type Opaque \
                --from-literal=username="swift" \
                --from-literal=password="$(< /dev/urandom tr -dc _A-Za-z0-9 | head -c${1:-32};echo;)"
        kubectl --namespace openstack \
                create secret generic swift-admin \
                --type Opaque \
                --from-literal=password="$(< /dev/urandom tr -dc _A-Za-z0-9 | head -c${1:-32};echo;)"
        kubectl --namespace openstack \
                create secret generic swift-hash-path \
                --type Opaque \
                --from-literal=suffix="$(< /dev/urandom tr -dc _A-Za-z0-9 | head -c${1:-32};echo;)" \
                --from-literal=prefix="$(< /dev/urandom tr -dc _A-Za-z0-9 | head -c${1:-32};echo;)"
        ```

!!! danger "The hash path suffix and prefix are cluster identity"

    Swift derives an object's ring placement from its path hashed with the values in `swift-hash-path`. Every proxy and storage node has to agree on them, and rotating them on a cluster that already holds data makes every existing object unreachable. Generate them once, back them up alongside your other cluster secrets, and leave them alone. `install-swift.sh` reads them from the secret rather than the overrides so a reinstall does not regenerate them.

## Run the package deployment

!!! example "Run the Swift deployment Script `/opt/genestack/bin/install-swift.sh`"

    ``` shell
    --8<-- "bin/install-swift.sh"
    ```

!!! tip

    In other cases such as a multi-region deployment you may want to view the [Multi-Region Support](multi-region-support.md) guide to for a workflow solution.

## Expose the service

The chart renders no ingress objects; Swift is published through the Gateway API like every other Genestack service. The listener and route templates are `etc/gateway-api/listeners/swift-https.json` and `etc/gateway-api/routes/custom-swift-gateway-route.yaml`, and the route sends traffic to the `swift-proxy` service on port 8080.

!!! note "Large object transfers"

    The `swift/base` overlay ships a `ClientTrafficPolicy` and a `BackendTrafficPolicy` scoped to the Swift listener and route. Without them, Envoy's default route timeout and connection buffer truncate multi-gigabyte uploads and downloads. The listener-scoped client policy replaces the gateway-wide one rather than merging with it, which is why it re-declares `clientIPDetection`.

## Telemetry

Genestack provisions a `swift` RabbitMQ vhost, user, and quorum queue as part of the `swift/base` overlay. Swift does not use them itself; they exist for the Swift proxy's `ceilometermiddleware`, and `install-ceilometer.sh` already builds a transport URL against the `swift` vhost so Ceilometer can consume `objectstore.http.request` notifications.

Enabling the middleware requires a Swift image that ships `ceilometermiddleware`. Once you have one, add the filter to the proxy pipeline in your custom overrides.

??? example "Publish objectstore notifications to Ceilometer"

    ``` yaml
    conf:
      proxy_server:
        "pipeline:main":
          pipeline: catch_errors gatekeeper healthcheck proxy-logging cache listing_formats container_sync bulk ratelimit authtoken keystoneauth copy container-quotas account-quotas slo dlo versioned_writes symlink ceilometer proxy-logging proxy-server
        "filter:ceilometer":
          paste.filter_factory: ceilometermiddleware.swift:filter_factory
          control_exchange: swift
          driver: messagingv2
          topic: notifications
          nonblocking_notify: "True"
    ```

    Set the transport URL from the secret rather than committing it:

    ``` shell
    /opt/genestack/bin/install-swift.sh \
      --set "conf.proxy_server.filter:ceilometer.url=rabbit://swift:$(kubectl --namespace openstack get secret swift-rabbitmq-password -o jsonpath='{.data.password}' | base64 -d)@rabbitmq.openstack.svc.cluster.local:5672/swift"
    ```

## Verify the deployment

!!! example "Confirm the rings were built and the endpoint answers"

    ``` shell
    kubectl --namespace openstack logs job/swift-ring-builder
    kubectl --namespace openstack get pods -l application=swift
    openstack --os-cloud default container create test-container
    openstack --os-cloud default container list
    ```

    An empty ring, a `swift-proxy` pod stuck waiting on the `swift-storage` daemonset, or a `503` from the API almost always traces back to `ring.devices` being empty or the devices not being mounted on the labelled nodes.

Day-to-day operations, capacity changes, and ring rebalancing are covered in the [Swift Object Storage operators guide](openstack-swift-operators-guide.md). For using Swift as an image store, see [Glance External Swift Image Store](openstack-glance-swift-store.md).
