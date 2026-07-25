### OpenTelemetry

- It's a instrumentation plus wired protocol (OLTP) plus a collector. It generates and moves telemetry - it stores nothing and draws nothing

### Prometheus

- It is a metrics backend. TSDB, scraper, PromQL and Alertmanager in one binary.
- It handles metrics only. It does not accept trace OR logs.
- Tools like Jaeger, Tempo and Loki have their own ingestion path.

- You need a Collector anyway the moment traces or logs enter the picture