# Monitoring Fleet
- [Health checks](#health-checks)
- [Metrics](#metrics)
  - [Alerting](#alerting)
  - [Graphing](#graphing)
- [Fleet server performance](#fleet-server-performance)
  - [Horizontal scaling](#horizontal-scaling)
  - [Availability](#availability)
  - [Monitoring](#monitoring)
  - [Debugging performance issues](#debugging-performance-issues)
    - [MySQL and Redis](#mysql-and-redis)
    - [Fleet server](#fleet-server)

## Health checks

Fleet exposes a basic health check at the `/healthz` endpoint. This is the interface to use for simple monitoring and load-balancer health checks.

The `/healthz` endpoint will return an `HTTP 200` status if the server is running and has healthy connections to MySQL and Redis. If there are any problems, the endpoint will return an `HTTP 500` status. Details about failing checks are logged in the Fleet server logs.

Individual checks can be run by providing the `check` URL parameter (e.x., `/healthz?check=mysql` or `/healthz?check=redis`).
## Metrics

Fleet exposes server metrics in a format compatible with [Prometheus](https://prometheus.io/). A simple example Prometheus configuration is available in [tools/app/prometheus.yml](https://github.com/fleetdm/fleet/blob/194ad5963b0d55bdf976aa93f3de6cabd590c97a/tools/app/prometheus.yml).

Prometheus can be configured to use a wide range of service discovery mechanisms within AWS, GCP, Azure, Kubernetes, and more. See the Prometheus [configuration documentation](https://prometheus.io/docs/prometheus/latest/configuration/configuration/) for more information.

### Alerting

#### Prometheus

Prometheus has built-in support for alerting through [Alertmanager](https://prometheus.io/docs/alerting/latest/overview/).

Consider building alerts for

- Changes from expected levels of host enrollment
- Increased latency on HTTP endpoints
- Increased error levels on HTTP endpoints

```
TODO (Seeking Contributors)
Add example alerting configurations
```

#### Cloudwatch Alarms

Cloudwatch Alarms can be configured to support a wide variety of metrics and anomaly detection mechanisms. There are some example alarms
in the terraform reference architecture (see `monitoring.tf`).

* [Monitoring RDS (MySQL)](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/monitoring-cloudwatch.html)
* [ElastiCache for Redis](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/CacheMetrics.WhichShouldIMonitor.html)
* [Monitoring ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cloudwatch-metrics.html)
* Reference alarms include evaluating healthy targets & response times. We also use target-tracking alarms to manage auto-scaling.

### Graphing

Prometheus provides basic graphing capabilities, and integrates tightly with [Grafana](https://prometheus.io/docs/visualization/grafana/) for sophisticated visualizations.

## Fleet server performance

Fleet is designed to scale to hundreds of thousands of online hosts. The Fleet server scales horizontally to support higher load.

### Horizontal scaling

Scaling Fleet horizontally is as simple as running more Fleet server processes connected to the same MySQL and Redis backing stores. Typically, operators front Fleet server nodes with a load balancer that will distribute requests to the servers. All APIs in Fleet are designed to work in this arrangement by simply configuring clients to connect to the load balancer.

### Availability

The Fleet/osquery system is resilient to loss of availability. Osquery agents will continue executing the existing configuration and buffering result logs during downtime due to lack of network connectivity, server maintenance, or any other reason. Buffering in osquery can be configured with the `--buffered_log_max` flag.

Note that short downtimes are expected during [Fleet server upgrades](https://fleetdm.com/docs/deploying/upgrading-fleet) that require database migrations.

### Debugging performance issues

#### MySQL and Redis

If performance issues are encountered with the MySQL and Redis servers, use the extensive resources available online to optimize and understand these problems. Please [file an issue](https://github.com/fleetdm/fleet/issues/new/choose) with details about the problem so that Fleet developers can work to fix them.

#### Fleet server

For performance issues in the Fleet server process, please [file an issue](https://github.com/fleetdm/fleet/issues/new/choose) with details about the scenario, and attach a debug archive. Debug archives can also be submitted confidentially through other support channels.

##### Generate debug archive (Fleet 3.4.0+)

Use the `fleetctl debug archive` command to generate an archive of Fleet's full suite of debug profiles. See the [fleetctl setup guide](https://fleetdm.com/docs/using-fleet/fleetctl-cli) for details on configuring `fleetctl`.

The generated `.tar.gz` archive will be available in the current directory.

##### Targeting individual servers

In most configurations, the `fleetctl` client is configured to make requests to a load balancer that will proxy the requests to each server instance. This can be problematic when trying to debug a performance issue on a specific server. To target an individual server, create a new `fleetctl` context that uses the direct address of the server.

For example:

```sh
fleetctl config set --context server-a --address https://server-a:8080
fleetctl login --context server-a
fleetctl debug archive --context server-a
```

##### Confidential information

The `fleetctl debug archive` command retrieves information generated by Go's [`net/http/pprof`](https://golang.org/pkg/net/http/pprof/) package. In most scenarios this should not include sensitive information, however it does include command line arguments to the Fleet server. If the Fleet server receives sensitive credentials via CLI argument (not environment variables or config file), this information should be scrubbed from the archive in the `cmdline` file.

<meta name="pageOrderInSection" value="800">
<meta name="description" value="Learn about monitoring and scaling Fleet servers with health checks, metrics, and alerting.">
