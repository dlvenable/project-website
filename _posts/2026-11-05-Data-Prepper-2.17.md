---
layout: post
title: 'Data Prepper 2.17: TODO'
authors:
  - srikanthpadakanti
  - dvenable
date: 2026-11-05
categories:
  - releases
excerpt: TODO
meta_keywords: TODO
meta_description: TODO
---

The OpenSearch Data Prepper maintainers are happy to announce the release of Data Prepper 2.17.


## Splunk HEC source

Many organizations already run Splunk universal forwarders, heavy forwarders, and applications that log through the [Splunk HTTP Event Collector (HEC)](https://docs.splunk.com/Documentation/Splunk/latest/Data/UsetheHTTPEventCollector) protocol. Moving that data into OpenSearch has meant reconfiguring every one of those clients or running extra tooling in between.

Data Prepper 2.17 introduces a new `splunk_hec` source that implements a HEC-compatible server, so existing forwarders and any HEC client can send data to Data Prepper unmodified. You point the client at Data Prepper instead of Splunk, and the source accepts the same requests, the same tokens, and returns the same HEC response codes that clients expect.

The source serves the standard HEC endpoints under `/services/collector`:

* `POST /event` accepts one or more HEC events as concatenated JSON objects.
* `POST /raw` accepts plain text and creates one event per line, reading metadata such as `sourcetype` and `index` from query parameters.
* `POST /ack` lets clients poll indexer acknowledgment status when acknowledgments are enabled.
* `GET /health` reports that the collector is healthy and does not require authentication.

Each request is authenticated with the standard `Authorization: Splunk <token>` header. You can configure multiple tokens, disable a token without removing it, and give each token its own defaults for `index`, `sourcetype`, `source`, `host`, and additional `fields`. This lets you separate teams or applications by token, just as you would in Splunk. JSON object events are flattened into top-level fields by default, the HEC `time` value becomes the event `@timestamp`, and the HEC `index` is stored in the `splunk_index` metadata attribute so that you can use it for routing.

The source is built for production forwarders. When the buffer is full, it returns `503` with HEC code 9 (server is busy), which tells Splunk forwarders to back off and retry instead of dropping data. When you set `acknowledgements: true`, the source implements the HEC indexer acknowledgment protocol on top of Data Prepper end-to-end acknowledgments, so a client is only told that its data was indexed after the sink has written it. TLS is enabled by default so that tokens are not sent in cleartext.

The following pipeline accepts HEC traffic on the standard port 8088, assigns per-token defaults, and writes each event to the OpenSearch index named by its HEC `index`:

```yaml
hec-pipeline:
  source:
    splunk_hec:
      port: 8088
      ssl_certificate_file: "/path/to/server.crt"
      ssl_key_file: "/path/to/server.key"
      acknowledgements: true
      tokens:
        - token: "${{aws_secrets:hec-secrets:infra-token}}"
          defaults:
            index: "infrastructure"
            sourcetype: "syslog"
        - token: "${{aws_secrets:hec-secrets:app-token}}"
          defaults:
            index: "application"
  sink:
    - opensearch:
        hosts: ["https://opensearch:9200"]
        index: '${getMetadata("splunk_index")}'
```

Any HEC client can then send events to Data Prepper, as shown in the following example:

```bash
curl https://data-prepper:8088/services/collector/event \
  -H "Authorization: Splunk <token>" \
  -d '{"event": {"message": "user login", "user": "alice"}, "sourcetype": "auth", "host": "web-01"}'
```

## RSS source

## JDBC source coordination

## gRPC connection age for OpenTelemetry sources

gRPC clients resolve DNS once and then keep the same long-lived connection. In horizontally scaled Data Prepper deployments, this means new replicas behind a load balancer receive no OpenTelemetry traffic until existing clients happen to reconnect, and load stays concentrated on the original nodes.

Data Prepper 2.17 adds two optional settings to every source that runs a gRPC server: `otlp`, `otel_trace_source`, `otel_logs_source`, and `otel_metrics_source`. The `max_connection_age` setting gracefully closes connections after the configured age, prompting clients to reconnect and pick up new endpoints. The `connection_drain_duration` setting gives in-flight requests time to finish before the connection closes. These map to the gRPC `MAX_CONNECTION_AGE` and `MAX_CONNECTION_AGE_GRACE` server options. When you omit them, the server behaves exactly as before.

```yaml
otel-pipeline:
  source:
    otlp:
      port: 21893
      max_connection_age: PT30M
      connection_drain_duration: 15s
```

## Other notable changes

This release includes the following additional improvements:

TODO: dlvenable

* `parse_json` on arrays
* Kafka with `azure_federated`
* Prometheus sink improvements
* 64 MB string length in events
* Group level rate limiting
* Expression language improvement for length()
* The `otel_trace_group` processor has a new `indices` option, so you can look up trace groups in custom span indexes, aliases, or index patterns instead of only the default `otel-v1-apm-span` alias ([#6936](https://github.com/opensearch-project/data-prepper/issues/6936)).
* Experimental OpenSearch pull-based ingestion now routes documents to the same partition as the OpenSearch shard they belong to. The Murmur3 hash now matches OpenSearch shard routing, including indexes that set `number_of_routing_shards` ([#6993](https://github.com/opensearch-project/data-prepper/issues/6993)).

## Getting started

Use the following resources to get started with Data Prepper 2.16:

TODO

* To learn about all the changes, see the [2.16.0 release notes](https://github.com/opensearch-project/data-prepper/releases/tag/2.16.0).
* To download Data Prepper, visit the [Download & Get Started](https://opensearch.org/downloads.html) page.
* For information about getting started with Data Prepper, see [Getting started with OpenSearch Data Prepper](https://opensearch.org/docs/latest/data-prepper/getting-started/).
* To learn more about upcoming work for Data Prepper, see the [Data Prepper Project Roadmap](https://github.com/orgs/opensearch-project/projects/221).

## Thanks to our contributors!

Thanks to the following community members who contributed to this release:

TODO
