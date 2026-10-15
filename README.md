# devops-case-study

This is my solution to the DevOps case study: what I built, how to access it, and why I made the choices I did.

[Architecture diagram](https://excalidraw.com/#json=sfibXJ9KpPPa0zT7C2xBn,jgO0p5DANrsakWFqvMdovQ)

## Overview

### What I Built

I added a monitoring stack to the cluster that covers metrics, alerts, dashboards, and logs. Prometheus, Alertmanager, and Grafana come from the `kube-prometheus-stack` chart, and logs are collected by Alloy and stored in Loki. Everything lives in the `monitoring` namespace and is managed by Flux.

The repo is split in two. `./infrastructure/controllers` holds the Helm repositories and releases, and `./infrastructure/configs` holds the things that run on top of them: ServiceMonitors, alert rules, and dashboards. Flux waits for the controllers to be ready before applying the configs, so nothing gets applied before its CRDs exist.

### Design Choices

I went with `kube-prometheus-stack` because it gives you Prometheus, Alertmanager, Grafana, kube-state-metrics, and node-exporter in one chart, along with a good set of default alerts. The catch is that it's built around the Prometheus Operator, so scrape targets and rules are custom resources instead of annotations. That's more moving parts, but I think it's worth it: every scrape target and alert ends up in Git and gets reviewed like any other change.

It did need some tuning for this cluster. Out of the box, Prometheus only picks up ServiceMonitors and rules that carry the chart's release label, so I turned that off and let it discover everything. k3s doesn't expose etcd, the scheduler, the controller manager, or kube-proxy, so I disabled those scrapes and their alerts, otherwise they'd sit there permanently "down". Prometheus, Alertmanager, and Grafana all get persistent volumes, and Prometheus keeps 48 hours of metrics.

For logs I picked Loki and Alloy over something heavier. The Loki chart assumes a multi-node production deployment by default, which is overkill here, so it runs as a single binary on the local filesystem with 72 hours of retention. Alloy reads pod logs through the Kubernetes API and only attaches `namespace`, `app`, `pod`, and `container` as labels, which keeps Loki fast.

## Grafana

```bash
flux get all -A
kubectl -n monitoring get pods
mise run forward # grafana
```

Then open [http://localhost:3000](http://localhost:3000) and log in as `admin`.

Get the admin password from the secret

```bash
kubectl -n monitoring get secret kube-prometheus-stack-grafana -o jsonpath='{.data.admin-password}' | base64 --decode; echo
```

### Dashboards

There are three dashboards, stored in Git as ConfigMaps and picked up by Grafana automatically.

**Service Health** has a row each for the ML API and the backend API with request rate, errors, p95 latency, and pod health, plus that service's logs right underneath. The ML API row also shows memory against its limit.

**Backend → Postgres** looks at the database from the backend API's side: queries, errors, connections, and query status, next to the Postgres logs.

**Cluster Health** covers the rest: node and pod status, restarts, resource usage, active alerts, and Flux errors.

### Alerts

My rule of thumb was that an alert should mean a user is affected or something is about to break. Anything less urgent, like database connection usage, goes on a dashboard instead. I also excluded `/health` and `/ready` from the error and traffic calculations, since health checks make up most of the traffic and would drown out real problems.

Both APIs export the same kinds of metrics, so each alert below exists once per service:

- **Errors:** we have `backend_api_requests_total` and `ml_api_requests_total` split by their `status` label, and we alert when 5xx responses stay above 5% of traffic for five minutes.
- **Latency:** we have the `backend_api_request_duration_seconds_bucket` and `ml_api_request_duration_seconds_bucket` histograms, and we alert when p95 sits over one second for five minutes.
- **Traffic:** we have the same request counters, and we alert when the rate is under half of what it was five minutes before. That usually points at the caller, DNS, or the network rather than the API itself.
- **Scraping:** we have Prometheus's own `up` series, labelled `job="backend-api"` or `job="ml-api"`, and we alert when either API goes two minutes without a healthy target.

The rest come from kube-state-metrics, cAdvisor, and Flux, on top of the chart's defaults:

- **Memory:** we have `container_memory_working_set_bytes` divided by `kube_pod_container_resource_limits{resource="memory"}`, and we alert at 90% for five minutes, before the OOMKiller gets there.
- **Restarts:** we have `kube_pod_container_status_restarts_total`, and we alert on more than three restarts in an hour.
- **Postgres:** we have `kube_deployment_status_replicas_available{deployment="postgres"}`, and we alert after one minute with nothing available.
- **API pods:** we have the same series next to `kube_deployment_spec_replicas`, and we alert when a Deployment wants pods but has had none ready for two minutes.
- **Routing:** we have `kube_endpointslice_endpoints{ready="true"}`, and we alert when pods are ready but the Service has no endpoints, which is usually a selector or port mismatch.
- **GitOps:** we have `controller_runtime_reconcile_errors_total` from `flux-system`, and we alert when a controller keeps erroring for ten minutes.

### Logs

Loki is set up as a Grafana data source automatically, so you can query any pod's logs in Explore:

```logql
{namespace="backend-api"}
```

## Future Improvements

- Wire up Alertmanager notifications, so critical alerts page someone and the rest go to Slack. Right now alerts only show up in Grafana.
- Make the traffic drop alert smarter. Since it compares against five minutes earlier, a long outage eventually clears it once the baseline drops to zero too.
- Add a PostgreSQL exporter so the database is measured directly, not just through the backend API.
- Move Loki to object storage so logs aren't tied to one node's disk.
- Look at Thanos or Mimir if we ever need longer retention or high availability for metrics.
