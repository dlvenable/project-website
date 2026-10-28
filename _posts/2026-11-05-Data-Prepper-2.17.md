---
layout: post
title: 'Data Prepper 2.17 brings Splunk HEC compatibility and JDBC source coordination'
authors:
  - srikanthpadakanti
  - dvenable
  - hsotaro
date: 2026-11-05
categories:
  - releases
excerpt: Data Prepper 2.17 adds a Splunk HEC-compatible source, a rewritten RSS source supporting multiple feeds, and multi-node scaling backed by your relational database.
meta_keywords: OpenSearch Data Prepper, Splunk HEC, Splunk migration, HTTP Event Collector, RSS source, Atom feeds, JDBC source coordination, PostgreSQL, MySQL, gRPC keepalive, OpenTelemetry, data pipeline
meta_description: OpenSearch Data Prepper 2.17 adds a Splunk HEC-compatible source for ingesting from existing Splunk forwarders, a rewritten RSS source, JDBC-backed source coordination, and gRPC connection age settings for OpenTelemetry sources.
---

The OpenSearch Data Prepper maintainers are happy to announce the release of Data Prepper 2.17.
With this release, existing Splunk forwarders and HTTP Event Collector (HEC) clients can send data to Data Prepper without modification, making it easier to bring Splunk-originated data into OpenSearch.
This release also rewrites the RSS source to reliably ingest multiple RSS and Atom feeds, adds a JDBC-backed source coordination store so you can scale Data Prepper to multiple nodes using a relational database you already run, and improves load balancing for OpenTelemetry sources in horizontally scaled deployments.


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

This source allows you to ingest any type of RSS or Atom feed, such as news, release announcements, security advisories, and blog posts, into OpenSearch. Earlier versions could read only one feed per source and had two serious bugs: every poll re-emitted the entire feed while memory use grew without bound, and a single transient network error stopped polling permanently.

Data Prepper 2.17 rewrites the `rss` source to fix both bugs and adds support for multiple feeds. Each feed now keeps a bounded cache of recently seen items, so only new items are written to the buffer. Feeds fail independently: a failing feed is logged, increments a per-feed failure metric, and retries with exponential backoff without affecting the other feeds. The new `request_timeout` setting, which defaults to 30 seconds, keeps a slow or unresponsive feed from blocking a worker thread.

You configure feeds in a `feeds` map keyed by feed name. Each feed can override the default `polling_frequency` and can set its own HTTP basic authentication credentials. The `workers` setting controls how many feeds are polled concurrently.

Each item becomes an `rss-item` event with `title`, `link`, `description`, `publication_date`, and `item_id` fields, plus the always-present `feed_name` and `feed_url` fields. The `item_id` field uses the item's GUID, falling back to its link and then to a content hash, so it is always present and works well as an OpenSearch `document_id`. The source redacts the query string from `feed_url` and from log messages so that API keys passed as query parameters are not exposed. When a feed provides channel details such as its title, link, language, or categories, the source attaches them as event metadata.

The following pipeline polls two feeds, one of them using basic authentication, and writes each feed to its own daily index:

```yaml
rss-pipeline:
  source:
    rss:
      workers: 2
      polling_frequency: PT5M
      feeds:
        opensearch-forum:
          url: "https://forum.opensearch.org/latest.rss"
        internal-advisories:
          url: "https://private.example.com/feed.xml"
          polling_frequency: PT1M
          authentication:
            basic:
              username: "${{aws_secrets:rss-credentials:username}}"
              password: "${{aws_secrets:rss-credentials:password}}"
  sink:
    - opensearch:
        hosts: ["https://opensearch:9200"]
        index: "rss-${/feed_name}-%{yyyy.MM.dd}"
        document_id: "${/item_id}"
```

The source reports `itemsIngested.<feed-name>` and `feedPollsFailed.<feed-name>` metrics for each feed, so you can track ingestion and alert on a failing feed individually. The seen-item cache holds the 10,000 most recent items per feed in memory and resets when Data Prepper restarts, so using `item_id` as the `document_id`, as in the preceding example, keeps re-ingested items from creating duplicate documents. For the full list of options, see the [RSS source documentation](https://docs.opensearch.org/latest/data-prepper/pipelines/configuration/sources/rss/).

## JDBC source coordination store

Several Data Prepper sources, including OpenSearch, Amazon S3 scan, and Apache Iceberg, scale beyond a single node only when they can share a source coordination store that assigns work to each node and prevents duplicate processing. Until now, this store has been backed by DynamoDB.

Data Prepper 2.17 adds a new `jdbc` source coordination store backed by a relational database. If you already run PostgreSQL or MySQL, you can now scale Data Prepper to a multi-node cluster using that database, with the backup, monitoring, and access controls you have already set up around it. The new store works with every source that uses source coordination, so moving an existing pipeline from DynamoDB to JDBC is a configuration change alone.

The following configuration points source coordination at a PostgreSQL database:

```yaml
source_coordination:
  store:
    jdbc:
      url: "jdbc:postgresql://db:5432/dataprepper"
      username: "dp_user"
      password: "${{aws_secrets:db-secrets:password}}"
```

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

* Data Prepper now supports Amazon OpenSearch domains optimized for observability with a new `trace-analytics-plain-raw-v2` index type in the `opensearch` sink.
* The `parse_json` and `parse_ion` processors can now parse top-level JSON arrays, writing the parsed array to the target key ([#5705](https://github.com/opensearch-project/data-prepper/issues/5705)).
* Events now support string values up to 64 MB, up from the previous 20 MB limit, so you can ingest documents with very large individual fields ([#7145](https://github.com/opensearch-project/data-prepper/issues/7145)).
* The Kafka source supports `azure_federated` OAuth authentication for Azure Event Hubs ([#7011](https://github.com/opensearch-project/data-prepper/pull/7011)).
* The `length()` expression function now works on maps, lists, and arrays in addition to strings ([#7030](https://github.com/opensearch-project/data-prepper/issues/7030)).
* The Prometheus sink can attach a per-instance `instance_label` to outgoing samples, enabling multiple Data Prepper nodes write the same series safely ([#7023](https://github.com/opensearch-project/data-prepper/issues/7023)).
* Data Prepper has a new opt-in plugin framework backed by an embedded Apache Felix OSGi runtime, enabled with `-Ddata-prepper.plugin.framework=osgi`; the default classpath loader is unchanged ([#6760](https://github.com/opensearch-project/data-prepper/issues/6760)).

## Getting started

Use the following resources to get started with Data Prepper 2.17:

* To learn about all the changes, see the [2.17.0 release notes](https://github.com/opensearch-project/data-prepper/releases/tag/2.17.0).
* To download Data Prepper, visit the [Download & Get Started](https://opensearch.org/downloads.html) page.
* For information about getting started with Data Prepper, see [Getting started with OpenSearch Data Prepper](https://opensearch.org/docs/latest/data-prepper/getting-started/).
* To learn more about upcoming work for Data Prepper, see the [Data Prepper Project Roadmap](https://github.com/orgs/opensearch-project/projects/221).

## Thanks to our contributors!

Thanks to the following community members who contributed to this release:

TODO
