# Observability Stack

> Configuración versionada de Prometheus y Grafana — observabilidad transversal de los 4
> microservicios de **UNO Power-Ups Arena** (HU-10 y HU-11).

[![CI](https://github.com/UNO-POWER/ObservabilityStack/actions/workflows/ci.yml/badge.svg)](https://github.com/UNO-POWER/ObservabilityStack/actions/workflows/ci.yml)
[![Prometheus](https://img.shields.io/badge/Prometheus-config--as--code-orange)]()
[![Grafana](https://img.shields.io/badge/Grafana-dashboards--provisioned-brightgreen)]()
[![License](https://img.shields.io/badge/license-MIT-blue)]()

---

## Tabla de contenidos

1. [Descripción](#1-descripción)
2. [Arquitectura](#2-arquitectura)
3. [Tecnologías](#3-tecnologías)
4. [Requisitos](#4-requisitos)
5. [Instalación](#5-instalación)
6. [Configuración](#6-configuración)
7. [Ejecución](#7-ejecución)
8. [Estructura del proyecto](#8-estructura-del-proyecto)
9. [Flujo interno](#9-flujo-interno)
10. [Dashboards](#10-dashboards)
11. [KPIs de negocio (HU-11)](#11-kpis-de-negocio-hu-11)
12. [Trazabilidad (HU-10)](#12-trazabilidad-hu-10)
13. [Seguridad](#13-seguridad)
14. [Solución de problemas](#14-solución-de-problemas)
15. [Roadmap](#15-roadmap)
16. [Contribución](#16-contribución)
17. [Licencia](#17-licencia)
18. [Autor](#18-autor)

---

## 1. Descripción

Este repositorio **no es un servicio Spring Boot** — es la configuración *as-code* del stack de
observabilidad: qué scrapea Prometheus y qué dashboards provisiona Grafana automáticamente al
arrancar. Se orquesta como dos contenedores más dentro del `docker-compose.yml` raíz del proyecto.

### Qué resuelve

| Historia | Cómo |
|---|---|
| **HU-10** — Trazabilidad de requests entre servicios | El API Gateway genera/propaga `X-Trace-Id`; Prometheus + el dashboard `servicios-salud.json` dan la vista de salud (requests, latencia, JVM, up/down) de cada servicio |
| **HU-11** — KPIs de negocio visibles para el equipo | 4 métricas de negocio instrumentadas en [Game Service](../GameService#17-observabilidad-y-kpis-hu-11) con Micrometer, visualizadas en `kpis-negocio.json` |

---

## 2. Arquitectura

```
┌──────────────┐  /actuator/prometheus  ┌────────────┐   datasource   ┌─────────┐
│ api-gateway  │───────────────────────▶│            │◀───────────────│         │
│ user-service │───────────────────────▶│ Prometheus │                │ Grafana │
│ lobby-service│───────────────────────▶│   :9090    │───consulta────▶│  :3000  │
│ game-service │───────────────────────▶│            │                │         │
└──────────────┘   scrape cada 5s        └────────────┘                └─────────┘
                                                                              │
                                                              dashboards provisionados
                                                              desde grafana/dashboards/*.json
```

Ni Prometheus ni Grafana tienen configuración manual vía UI que no esté versionada aquí: los
datasources y los dashboards se cargan automáticamente al arrancar el contenedor
(`grafana/provisioning/**`), así que un `docker compose up` reproduce el mismo stack de
observabilidad en cualquier máquina.

---

## 3. Tecnologías

| Tecnología | Uso |
|---|---|
| Prometheus | Scraping de métricas (`/actuator/prometheus`) de los 4 servicios backend, cada 5s |
| Grafana | Visualización; datasource y dashboards provisionados como archivos, no configurados a mano |
| Micrometer (en cada servicio Java) | Expone las métricas que Prometheus scrapea |
| Docker Compose | Orquesta ambos contenedores junto al resto del sistema |

---

## 4. Requisitos

- **Docker** y **Docker Compose** (no requiere Java/Maven — este repo no compila nada).
- Los 4 servicios backend deben exponer `/actuator/prometheus` (ya vienen configurados así por
  defecto en cada uno).

---

## 5. Instalación

```bash
git clone https://github.com/UNO-POWER/ObservabilityStack.git
cd ObservabilityStack
```

No hay build: los archivos se montan tal cual como volúmenes en los contenedores de Prometheus y
Grafana, definidos en el `docker-compose.yml` raíz del monorepo de despliegue.

---

## 6. Configuración

| Archivo | Qué controla |
|---|---|
| `prometheus/prometheus.yml` | `scrape_interval: 5s`, 4 jobs (`api-gateway:8080`, `user-service:8081`, `lobby-service:8082`, `game-service:8083`), todos apuntando a `/actuator/prometheus` |
| `grafana/provisioning/datasources/prometheus.yml` | Datasource `Prometheus` apuntando a `http://prometheus:9090` (nombre de servicio en la red Docker), marcado `isDefault: true` |
| `grafana/provisioning/dashboards/dashboards.yml` | Provider que carga cualquier `.json` de `grafana/dashboards/` en la carpeta "UNO Power-Ups Arena" de Grafana, cada 30s (`allowUiUpdates: true`, así se puede ajustar en vivo sin perder el archivo base) |

No hay variables de entorno propias de este repo — los targets de Prometheus usan los nombres de
servicio de Docker Compose (`api-gateway`, `user-service`, etc.), que solo resuelven dentro de la
red `uno-net` del compose.

**Credenciales de Grafana:** por defecto `admin` / `admin` (definidas en el `docker-compose.yml`
raíz, variables `GF_SECURITY_ADMIN_USER`/`GF_SECURITY_ADMIN_PASSWORD`) — cambiar en un despliegue
real expuesto a Internet.

---

## 7. Ejecución

Este stack **no se levanta solo** — es parte del `docker-compose.yml` raíz junto con los 4
servicios backend:

```bash
# desde la raíz del monorepo
docker compose up -d prometheus grafana
```

O el stack completo:

```bash
docker compose up -d --build
```

**Acceso:**

| Servicio | URL local | Credenciales |
|---|---|---|
| Grafana | http://localhost:3000 | `admin` / `admin` |
| Prometheus | http://localhost:9090 | (sin autenticación) |

---

## 8. Estructura del proyecto

```
ObservabilityStack/
├── prometheus/
│   └── prometheus.yml                      # scrape_interval 5s, 4 jobs
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/prometheus.yml      # datasource unico, apunta al contenedor prometheus
│   │   └── dashboards/dashboards.yml       # provider que carga grafana/dashboards/*.json
│   └── dashboards/
│       ├── servicios-salud.json            # HU-10: requests, latencia p95, JVM, up/down
│       └── kpis-negocio.json               # HU-11: los 4 KPIs de negocio
└── README.md
```

---

## 9. Flujo interno

1. Cada uno de los 4 servicios backend expone `/actuator/prometheus` (formato de texto de
   Prometheus) gracias a `micrometer-registry-prometheus` en su `pom.xml`.
2. Prometheus, cada 5 segundos, hace `GET` a esos 4 endpoints (definidos como `static_configs` con
   el nombre del servicio Docker, no una IP — se resuelve por la red interna de Compose) y
   almacena las series temporales.
3. Grafana, al arrancar, lee `grafana/provisioning/datasources/prometheus.yml` y registra
   automáticamente el datasource — **no** hay que configurarlo a mano en la UI.
4. Grafana también lee `grafana/provisioning/dashboards/dashboards.yml`, que le dice que cargue
   cualquier `.json` de `grafana/dashboards/` — así los 2 dashboards existentes aparecen ya
   armados desde el primer arranque.
5. Cada panel de cada dashboard ejecuta una query PromQL contra Prometheus (ver
   [§10](#10-dashboards)) y se refresca solo (`refresh: "30s"` en `kpis-negocio.json`).

---

## 10. Dashboards

### `servicios-salud.json` (HU-10)

Salud técnica de los 4 servicios: throughput de requests, latencia p95, uso de memoria JVM,
estado up/down por servicio. Pensado para detectar degradación de un servicio individual.

### `kpis-negocio.json` (HU-11)

Los 4 KPIs de negocio definidos en el Inception doc, con **zonas de aterrizaje** (mínimo /
objetivo / excepcional) codificadas directamente como umbrales de color en cada panel:

| KPI | Zona mínima | Objetivo | Excepcional |
|---|---|---|---|
| KPI-1 Partidas completadas / día | ≥ 5 | ≥ 10 | ≥ 25 |
| KPI-2 Latencia p95 jugada → broadcast | < 400 ms | < 200 ms | < 100 ms |
| KPI-3 Salas concurrentes en juego (ahora) | ≥ 10 | ≥ 20 | ≥ 50 |
| KPI-4 Tasa de partidas sin errores críticos | ≥ 90% | ≥ 95% | ≥ 99% |

Incluye además dos gráficas de serie temporal: partidas completadas por hora, y latencia
p50/p95/p99 de jugada→broadcast.

---

## 11. KPIs de negocio (HU-11)

Las 4 métricas que alimentan este dashboard se generan en
[Game Service](../GameService#17-observabilidad-y-kpis-hu-11), el único servicio con lógica de
negocio suficientemente crítica para justificar métricas custom:

| Métrica Prometheus | Panel que la usa | Query PromQL |
|---|---|---|
| `uno_partidas_completadas_total` | KPI-1 | `sum(increase(uno_partidas_completadas_total[24h]))` |
| `uno_jugada_broadcast_seconds_bucket` | KPI-2 | `histogram_quantile(0.95, sum by (le) (rate(uno_jugada_broadcast_seconds_bucket[5m])))` |
| `uno_partidas_activas` | KPI-3 | `uno_partidas_activas` (gauge directo) |
| `uno_partidas_errores_total` | KPI-4 | `1 - increase(errores[24h]) / increase(completadas[24h])` |

> **Nota de consistencia:** una versión anterior de este README indicaba que estos KPIs eran
> trabajo pendiente de "Sprint 3". Esto ya no es así: las 4 métricas están instrumentadas en
> `PartidaService` (Game Service) y el dashboard `kpis-negocio.json` ya las consume — verificado
> contra el código actual al redactar esta versión del documento.

---

## 12. Trazabilidad (HU-10)

El `X-Trace-Id` (generado/propagado por el API Gateway, ver
[`APIGATEWAY/README.md`](../APIGATEWAY/README.md#12-logs-y-trazabilidad) viaja como header entre
servicios y se incluye en los logs JSON de cada uno vía MDC de SLF4J. Este stack no centraliza
logs (no hay ELK/Loki en el alcance actual — ver [§15 Roadmap](#15-roadmap)); la correlación hoy se
hace buscando manualmente el mismo `traceId` en los logs de cada contenedor
(`docker compose logs <servicio> | grep <traceId>`).

---

## 13. Seguridad

- Grafana y Prometheus **no se exponen** con puerto público en el despliegue de producción
  (`deploy/docker-compose.prod.yml`) — solo accesibles dentro de la red interna o vía túnel SSH,
  ya que ninguno de los dos tiene autenticación fuerte por defecto.
- Las credenciales de Grafana (`admin`/`admin`) son las de desarrollo; deben cambiarse mediante
  variables de entorno si el stack se expone.

---

## 14. Solución de problemas

| Síntoma | Causa probable | Solución |
|---|---|---|
| Grafana no muestra datos ("No Data") | Prometheus no puede alcanzar algún servicio, o el servicio no expone `/actuator/prometheus` | Revisar `http://localhost:9090/targets` — todos deben estar `UP` |
| El dashboard no aparece en Grafana | El contenedor de Grafana no montó `grafana/dashboards/` correctamente | Verificar el volumen en `docker-compose.yml` y reiniciar el contenedor |
| KPI-2/KPI-4 muestran "No Data" recién iniciado el sistema | Aún no se ha completado ninguna partida — las métricas son contadores/histogramas que empiezan en 0 | Jugar una partida completa y esperar el siguiente scrape (5s) |
| Cambié un `.json` de dashboard y no se refleja | `allowUiUpdates: true` permite que un cambio manual en la UI sobreviva hasta el siguiente `updateIntervalSeconds` (30s), pero puede haber quedado un dashboard editado en la UI tapando el archivo | Eliminar el dashboard desde la UI de Grafana y dejar que el provider lo recargue del archivo |

---

## 15. Roadmap

- Centralizar logs (Loki o ELK) para búsqueda de `traceId` sin entrar contenedor por contenedor.
- Alertas de Prometheus (Alertmanager) cuando un KPI cae bajo la zona mínima.
- Autenticación real de Grafana si se expone fuera de una red confiable.

---

## 16. Contribución

1. Cualquier dashboard nuevo va como `.json` en `grafana/dashboards/` — se provisiona solo, no
   requiere tocar `dashboards.yml`.
2. Si un servicio backend agrega una métrica de negocio nueva, documentarla aquí en
   [§11](#11-kpis-de-negocio-hu-11) y en el README del servicio que la genera.
3. Mantener sincronizados los umbrales de este README con los paneles reales del `.json` — si
   cambian las zonas de aterrizaje del Inception doc, actualizar ambos.

---

## 17. Licencia

MIT — ver [`LICENSE`](../LICENSE) en el repositorio raíz.

---

## 18. Autor

**DUCU** — Desarrollador Full-Stack (individual), proyecto de la asignatura ARSW.
Organización GitHub: [UNO-POWER](https://github.com/UNO-POWER).

Repositorio raíz del sistema: **[UNO Power-Ups Arena](../README.md)**.
