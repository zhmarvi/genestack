# Object Storage Backend Options

Genestack can provide the OpenStack Object Store (`object-store`) service in two
ways. Both present the Swift API to tenants, but they are different systems and
the choice has consequences for metering, feature coverage, and who operates the
storage.

| | Native Swift | External Ceph RGW |
|---|---|---|
| Serves the API | `swift-proxy` from the OpenStack-Helm chart | `radosgw` on the Ceph cluster |
| Stores the data | Swift rings over devices on labelled storage nodes | RADOS pools |
| Deployed by | `bin/install-swift.sh` | already running; Genestack only registers it |
| Ceilometer metering | Supported | **Not available** |
| S3 API | via Swift's `s3api` middleware | native, shares a namespace with Swift |
| Operational burden | rings, devices, `partition_power`, rebalancing | owned by the Ceph operators |
| Deploy guide | [Deploy Swift](openstack-swift.md) | [External Ceph RGW](openstack-object-storage-rgw.md) |

!!! danger "Pick one"

    Both options register the same `object-store` service type and the same
    public, internal, and admin endpoints. They cannot run side by side. If
    `swift: true` is set in `openstack-components.yaml` *and* the RGW playbook is
    run, whichever executes last overwrites the other's endpoints and tenants
    are silently directed at the wrong backend.

## Choosing

**Native Swift** is the right choice when object storage metering feeds billing,
when you need the Swift feature surface in full, or when object storage is
operated by the same team that runs the control plane.

**External Ceph RGW** is the right choice when a Ceph cluster already exists and
its object gateway is the intended storage, when you want one namespace shared
between the S3 and Swift APIs, or when you would rather not operate Swift rings.

## What you give up with RGW

### Ceilometer metering stops working

This is the consequential difference and worth checking before committing.

`base-helm-configs/ceilometer/ceilometer-helm-overrides.yaml` defines a
`swift_account` resource type in `gnocchi_resources` along with per-storage-policy
request-class meters. Those samples originate from `ceilometermiddleware` running
inside the Swift proxy pipeline, which publishes oslo.messaging notifications onto
the `swift` RabbitMQ vhost for Ceilometer to consume.

RGW has no equivalent. Its bucket notifications are a different mechanism with a
different payload format and are not consumed by Ceilometer. Moving to RGW means
those meters stop receiving samples. If anything downstream rates object storage
usage, resolve that before switching.

### Swift API feature gaps

Ceph documents most of the Swift data access model as supported. The exceptions
are:

| Feature | RGW status |
|---|---|
| CORS | Not supported |
| Temporary URLs | Partial, no container-level keys |
| Object Versioning | Partial, no `X-History-Location` |

Note that [Swift Object Storage concepts](openstack-object-storage-swift.md)
documents `X-History-Location` in detail. That guidance applies to native Swift
only.

See the [Ceph RGW Swift API feature support table](https://docs.ceph.com/en/latest/radosgw/swift/)
for the authoritative list. Content summarised here for brevity; refer to the
upstream table when validating a specific workload.

## What Swift costs you

Swift owns its own durability. That means rings to build and rebalance, devices
mounted at `/srv/node/<name>` on every node labelled
`openstack-storage-node=enabled`, and a `partition_power` that cannot be changed
without rebuilding the rings. If a Ceph cluster is already running and already
holds object data, standing up a second independent storage system alongside it
is real duplicated effort.

## Shared ground

Both options use the same Keystone service name (`swift`) and service type
(`object-store`), and both expose the API on the `swift-https` Gateway API
listener defined in `etc/gateway-api/listeners/swift-https.json`. The
`reseller_prefix` is `AUTH_` either way, so tenant-facing URLs have the same
shape.

The `swift-admin` secret is used by both, though for different purposes: native
Swift uses it as the password for the `swift` service user that the proxy's
`keystonemiddleware` authenticates with, while the RGW path uses it as the
password for the service user that `radosgw` authenticates as when validating
tokens.

!!! note "Routing object traffic"

    With an external RGW you can register the RGW address directly as the
    endpoint, or route it through `flex-gateway`. Sending bulk object traffic
    through Envoy means sizing gateway bandwidth and CPU against object
    throughput; RGW deployments more commonly sit behind their own load
    balancer. See [External Ceph RGW](openstack-object-storage-rgw.md) for the
    tradeoff.
