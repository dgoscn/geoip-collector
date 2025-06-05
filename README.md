# Custom GeoIP OpenTelemetry Collector

**Title:** Enriching Structured Logs with GeoIP Using OpenTelemetry Collector

**Author:** Diego Amaral
**Date:** 2025-06-04

---

## Overview

### Objective

This project demonstrates how to enrich structured logs with geolocation metadata using a custom-built OpenTelemetry Collector. It includes a fully functional pipeline that ingests logs from a Go-based log generator, processes them through a series of OpenTelemetry processors (including GeoIP enrichment), and exports the results to observability backends like New Relic or to the console for local validation.

### Components Used

The custom collector was built using only the necessary components to keep the binary lightweight:

* **Receivers:** `otlpreceiver`
* **Processors:** `transformprocessor`, `geoipprocessor`, `attributesprocessor`, `batchprocessor`
* **Exporters:** `otlpexporter`, `otlphttpexporter`, `debugexporter`

The collector binary (`otel.bin`) was generated using [OpenTelemetry Collector Builder (OCB)](https://opentelemetry.io/docs/collector/custom-collector/), and all configurations are defined in `local/config.yaml`.

---

## Configuration Summary

### Receivers

```yaml
receivers:
  otlp:
    protocols:
      http:
```

This receiver listens for telemetry data via OTLP/HTTP on port `4318`.

### Processors

```yaml
processors:
  transform/logs:
  geoip:
  transform/rename_geo:
  attributes/remove_extra_fields:
  batch:
```

* **transform/logs:** Parses the `data` field from logs, extracts attributes, and sets `source.address` for GeoIP lookup.
* **geoip:** Enriches logs based on IP using the local MaxMind GeoLite2 database.
* **transform/rename\_geo:** Renames geo fields (e.g., `geo.country_name` → `country_login`) and removes the original ones.
* **attributes/remove\_extra\_fields:** Cleans up unnecessary fields (e.g., ISO codes, lat/lon).
* **batch:** Groups logs for optimized processing and exporting.

### Exporters

```yaml
exporters:
  otlphttp/newrelic:
    endpoint: https://otlp.nr-data.net:4318
    headers:
      api-key: <YOUR_NEW_RELIC_LICENSE_KEY>
  debug:
    verbosity: detailed
```

* **New Relic Exporter:** Sends data to New Relic via OTLP/HTTP.
* **Debug Exporter:** Prints logs to the terminal (great for local development).

### Service Pipeline

```yaml
service:
  pipelines:
    logs:
      receivers: [otlp]
      processors: [transform/logs, geoip, transform/rename_geo, attributes/remove_extra_fields, batch]
      exporters: [debug, otlphttp/newrelic]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug, otlphttp/newrelic]
```

---

##  How to Run

### Prerequisites

* Go 1.21+
* `otel.bin` built with only the required components
* MaxMind GeoLite2 database located at `./../geolite-database/GeoLite2-City.mmdb`

### ▶Running the Collector

```bash
cd geoip-collector
GO111MODULE=on go run --race . --config ../local/config.yaml
```

Optional: Enable debug mode

```bash
GO111MODULE=on go run --race . --config ../local/config.yaml --log-level=DEBUG
```

If you have not yet built the collector binary, run:

```bash
./ocb --config builder-config.yaml
```

Or:

```bash
make build
make otelcontribcol
```

---

## Observability Backends

### Logs in New Relic

After configuring the `otlphttp/newrelic` exporter, logs will appear in the **Log Section** of the New Relic dashboard. Click on any log entry to see the full enriched payload with fields like `country_login`, `region_login`, and `timezone`.

### Metrics in New Relic

The app also emits basic metrics via the OpenTelemetry SDK. While not the main focus, these metrics are sent to New Relic through the same OTLP/HTTP pipeline. Navigate to the **Metrics** section in New Relic to explore them.

### Debugging with `otel-tui`

For local development, `otel-tui` is a great way to visualize telemetry without external dependencies. Just change your exporter:

```yaml
exporters:
  otlphttp:
    endpoint: http://localhost:4318
```

Then start `otel-tui` and enjoy a terminal-based interface for real-time observability.

---

## Conclusion

This project showcases a complete OpenTelemetry pipeline built from the ground up. From log generation in Go to real-time GeoIP enrichment and exporting to a backend like New Relic, you’ve seen how to:

* Build a custom OpenTelemetry Collector with only required components
* Configure a log enrichment pipeline using `transform` and `geoip`
* Debug locally using the `debug` exporter or `otel-tui`
* Send data to a production-ready backend

This foundation can easily be extended to production use cases, scaling both in data volume and business logic for log transformation and analysis.
