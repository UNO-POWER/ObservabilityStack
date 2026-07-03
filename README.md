# ObservabilityStack

Observabilidad transversal de UNO Power-Ups Arena (Epic 6 / HU-10 / HU-11).

No es un servicio Spring Boot: contiene la configuracion de **Prometheus** (scraping de `/actuator/prometheus` en los 4 servicios backend + Gateway) y **Grafana** (datasource + dashboards provisionados), orquestados desde el `docker-compose.yml` raiz del proyecto.

- `prometheus/prometheus.yml` — targets de scraping.
- `grafana/provisioning/datasources` — conecta Grafana a Prometheus automaticamente.
- `grafana/provisioning/dashboards` + `grafana/dashboards/*.json` — dashboards versionados como codigo.

Estado actual: `servicios-salud.json` cubre HU-10 (requests, latencia p95, memoria JVM, up/down). Los 4 KPIs de negocio (HU-11: partidas/dia, latencia p95 jugada, salas concurrentes, tasa sin errores) son Sprint 3 y requieren instrumentacion de metricas custom en Game Service todavia no implementada.

Acceso local: Grafana `http://localhost:3000` (admin/admin), Prometheus `http://localhost:9090`.
