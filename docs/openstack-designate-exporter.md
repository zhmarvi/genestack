# Designate Prometheus and Alerting Rules

Designate (DNS) metrics are collected by the
[openstack-metrics-exporter](prometheus-openstack-metrics-exporter.md)
(`prometheus-openstack-metrics-exporter`) with the `dns` service enabled. The
exporter ships its own `ServiceMonitor`, so kube-prometheus-stack's Prometheus
scrapes the `openstack_designate_*` metrics directly. There is no separate
Designate exporter to deploy.

!!! note "Metrics source"

    The `dns` service is already enabled in
    [`base-helm-configs/openstack-metrics-exporter/openstack-metrics-exporter-helm-overrides.yaml`](https://github.com/rackerlabs/genestack/blob/main/base-helm-configs/openstack-metrics-exporter/openstack-metrics-exporter-helm-overrides.yaml)
    under `multicloud.clouds[].services`. If Designate metrics are missing,
    confirm `- dns` is present (and uncommented) in that list and that the
    exporter is deployed. See [OpenStack Exporter](prometheus-openstack-metrics-exporter.md).

    OpenTelemetry is a separate collection track (host, libvirt, mysql,
    rabbitmq, memcached, and the `prometheus/infra` scrape jobs) and does not
    scrape the openstack-metrics-exporter. Designate resource metrics reach
    Prometheus through the exporter's own ServiceMonitor, not through the OTel
    pipeline, so no OpenTelemetry change is required for these alerts.

## Add Designate alerting rules

Alerting rules are added through the kube-prometheus-stack
`additionalPrometheusRulesMap` directive, following the pattern described in
[Alerting Rules](alerting-info.md). A ready-to-use rules file is checked in at
[`base-helm-configs/kube-prometheus-stack/rules/designate_prometheus_rules.yaml`](https://github.com/rackerlabs/genestack/blob/main/base-helm-configs/kube-prometheus-stack/rules/designate_prometheus_rules.yaml).

To activate the rules, copy that file into your override location:

```shell
mkdir -p /etc/genestack/helm-configs/kube-prometheus-stack/rules
cp /opt/genestack/base-helm-configs/kube-prometheus-stack/rules/designate_prometheus_rules.yaml \
  /etc/genestack/helm-configs/kube-prometheus-stack/rules/designate_prometheus_rules.yaml
```

Then re-run the Prometheus deployment to apply them:

```shell
/opt/genestack/bin/install-kube-prometheus-stack.sh
```

The rules file contains:

```yaml
additionalPrometheusRulesMap:
  openstack-resource-alerts:
    groups:
      - name: Designate Resource Alerts
        rules:
          - alert: ZoneInError
            expr: openstack_designate_zone_status{status=~"ERROR"}
            labels:
              severity: critical
            annotations:
              summary: "Designate zone is in ERROR state"
              description: |
                The dns zone `{{`{{$labels.id}}`}}` is in ERROR state.
          - alert: RecordInError
            expr: openstack_designate_recordsets_status{status=~"ERROR"}
            labels:
              severity: warning
            annotations:
              summary: "Designate record in in ERROR state"
              description: |
                The recordset `{{`{{$labels.id}}`}}` in zone `{{`{{$labels.zone_id}}`}}` is in ERROR state
          - alert: DesignateDown
            expr: openstack_designate_up != 1
            labels:
              severity: critical
            annotations:
              summary: "Designate Service is down"
              description: |
                Designate service is down; please check the designate service logs to determine the cause of the issue
          - alert: ZoneInPending
            expr: openstack_designate_zone_status{status=~"PENDING"}
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "Designate zone has been in PENDING state for over 5 mins"
              description: |
                The dns zone `{{`{{$labels.id}}`}}` has been in PENDING state for over 5 mins
          - alert: RecordInPending
            expr: openstack_designate_recordsets_status{status=~"PENDING"}
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "Designate record has been in PENDING state for over 5 mins"
              description: |
                The dns zone `{{`{{$labels.id}}`}}` has been in PENDING state for over 5 mins
```
