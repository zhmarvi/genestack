# External Ceph RGW as the Object Store

The Ceph RADOS Gateway implements the Swift API natively, so an existing external
Ceph cluster can serve the OpenStack Object Store without deploying Swift at all.
Genestack's part is small: register the service catalog entry and create the
Keystone service user that `radosgw` authenticates as when it validates tokens.

> Read [Object Storage Backend Options](openstack-object-storage-backends.md)
> first. This path trades Ceilometer object metering and a few Swift API features
> for not operating a second storage system.

!!! danger "Do not deploy Swift as well"

    Leave `swift: false` in `openstack-components.yaml`. Both paths claim the same
    `object-store` service and endpoints, and the last one applied wins.

## How it fits together

Swift has no pluggable storage backend; it cannot be made to write into RGW. The
integration works the other way around, by pointing the catalog at RGW and
configuring RGW to trust Keystone:

1. Tenants read the `object-store` endpoint from the Keystone catalog.
2. They send Swift API requests to RGW with their Keystone token.
3. RGW validates that token against Keystone using a service user, and maps the
   token's project and roles onto an implicitly created RGW user.

Both halves are required. Registering the endpoint without configuring RGW gives
clients a working catalog entry and then `401` on every request.

## Create secrets

!!! note "Information about the secretes used"

    Manual secret generation is only required if you haven't run the
    `create-secrets.sh` script located in `/opt/genestack/bin`.

    The RGW path reuses the `swift-admin` secret as the password for the Keystone
    service user that `radosgw` authenticates as.

    ??? example "Example secret generation"

        ``` shell
        kubectl --namespace openstack \
                create secret generic swift-admin \
                --type Opaque \
                --from-literal=password="$(< /dev/urandom tr -dc _A-Za-z0-9 | head -c${1:-32};echo;)"
        ```

## Register the endpoint

!!! example "Run the RGW object store playbook"

    ``` shell
    ansible-playbook /opt/genestack/ansible/playbooks/deploy-rgw-object-store.yaml \
      -e rgw_public_url=https://objects.your.domain.tld
    ```

    When the RGW frontend is reachable on a different address from inside the
    cluster, set the interfaces separately.

    ``` shell
    ansible-playbook /opt/genestack/ansible/playbooks/deploy-rgw-object-store.yaml \
      -e rgw_public_url=https://objects.your.domain.tld \
      -e rgw_internal_url=http://rgw.ceph.internal:8080 \
      -e rgw_region_name=RegionOne
    ```

The playbook reads the service user password from the `swift-admin` secret,
creates or updates the `swift` Keystone user, ensures the accepted roles exist,
registers the `object-store` service, and writes the public, internal, and admin
endpoints. Existing endpoints are updated in place, so re-running after the RGW
address changes corrects the catalog rather than duplicating entries.

All variables are documented in
`ansible/roles/ceph_rgw_object_store/defaults/main.yml`. Only `rgw_public_url` is
required.

!!! tip "Service project domain"

    The role defaults to `rgw_keystone_project_domain: default`, matching the
    OpenStack-Helm convention for service users. If your Keystone keeps the
    service project in a domain named `service`, override it with
    `-e rgw_keystone_project_domain=service`.

## Configure RGW on the Ceph cluster

The playbook prints these at the end with your values substituted. Apply them on
the Ceph cluster, not in Genestack.

!!! example "Ceph monitor configuration"

    ``` shell
    ceph config set client.rgw rgw_keystone_api_version 3
    ceph config set client.rgw rgw_keystone_url https://keystone.your.domain.tld
    ceph config set client.rgw rgw_keystone_admin_user swift
    ceph config set client.rgw rgw_keystone_admin_project service
    ceph config set client.rgw rgw_keystone_admin_domain default
    ceph config set client.rgw rgw_keystone_admin_password_path /etc/ceph/keystone-pw
    ceph config set client.rgw rgw_keystone_accepted_roles admin,member,swiftoperator
    ceph config set client.rgw rgw_keystone_implicit_tenants swift
    ceph config set client.rgw rgw_swift_url_prefix swift
    ceph config set client.rgw rgw_swift_account_in_url true
    ```

    The password is the value of the `swift-admin` secret. Ceph documentation
    prefers `rgw_keystone_admin_password_path` over an inline password.

!!! warning "The account-in-URL setting must match"

    `rgw_swift_account_in_url` and the registered endpoint path have to agree, or
    every object request returns `404` rather than failing in an obvious way.

    | Ceph setting | Endpoint path |
    |---|---|
    | `rgw_swift_account_in_url = false` | `/swift/v1` |
    | `rgw_swift_account_in_url = true` | `/swift/v1/AUTH_%(tenant_id)s` |

    The role's `rgw_swift_account_in_url` variable controls which path it writes
    and defaults to `true`, which is required for cross-project bucket access.

!!! note "Service user roles"

    The service user needs the `admin` and `service` roles so RGW can call
    `POST /v3/s3tokens` and `GET /v3/users/<user>/OS-EC2/<credential>`. You can
    narrow this to `service` alone by amending the `identity:ec2_get_credential`
    policy in Keystone.

## Exposing the endpoint

Two workable topologies, and the simpler one is usually correct.

**Register the RGW address directly.** Object data never transits the Kubernetes
ingress. This is how RGW is normally deployed, behind its own load balancer. The
cost is a separate hostname and TLS lifecycle from the rest of your OpenStack
endpoints.

**Route through `flex-gateway`.** Keeps `swift.<domain>` consistent with every
other service and reuses the cert-manager and ACME flow. The cost is that every
GET and PUT passes through the Envoy pods, so gateway bandwidth and CPU must be
sized against object throughput.

!!! note "Off-cluster backends need extra configuration"

    Envoy Gateway reaches endpoints outside the cluster through either an
    `ExternalName`-style Service or its `Backend` extension API. The Backend API
    is disabled by default and there is no `EnvoyGateway` config resource in
    `base-kustomize/envoyproxy-gateway/base/` enabling it, so that would need
    adding before a route to an external RGW resolves.

    The `swift-clienttrafficpolicy.yaml` and `swift-backendtrafficpolicy.yaml`
    resources in `base-kustomize/swift/base/` raise the Envoy route timeout and
    connection buffers for large transfers, and remain useful if you take this
    route.

## Verify

!!! example "Confirm the catalog and a round trip"

    ``` shell
    openstack --os-cloud default endpoint list --service object-store
    openstack --os-cloud default container create example-container
    openstack --os-cloud default object create example-container /etc/hosts
    openstack --os-cloud default object save --file /tmp/hosts.saved example-container /etc/hosts
    openstack --os-cloud default object delete example-container /etc/hosts
    openstack --os-cloud default container delete example-container
    ```

    The playbook runs a read-only account query itself and reports the result. A
    `404` on container operations points at an `rgw_swift_account_in_url` mismatch;
    a `401` points at the `rgw_keystone_*` settings or the service user's roles.

## Related considerations

`bin/setup-monitoring-rgw-storage.sh` provisions the Loki and Tempo object
buckets and resolves the Rook service name `rook-ceph-rgw-${RGW_STORE_NAME}` in
the `rook-ceph` namespace, shelling into `rook-ceph-tools`. It needs rework
against an external endpoint if you use it.

`scripts/import-external-cluster.sh` is largely CSI and RBD plumbing and is not
required for an object-only integration.
