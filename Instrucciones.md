# Laboratorio de Observabilidad — Grafana, Prometheus y Loki

**Alumno:** Renzo Contreras Flores  
**Curso:** Infraestructura como Código  
**Repo:** https://github.com/RenzoCf/iac-observabilidad

---

## ¿Qué hace cada componente?

- **Prometheus** — recolecta y almacena métricas (CPU, memoria, peticiones) haciendo scraping a los endpoints `/metrics`.
- **node-exporter** — expone métricas del host (CPU y memoria de la máquina).
- **cAdvisor** — expone métricas por contenedor.
- **Loki** — almacena los logs (eventos, errores, trazas).
- **Grafana Alloy** — recolecta los logs de cada contenedor y los envía a Loki etiquetados como `tier=application` o `tier=infrastructure`.
- **Grafana** — visualización y alarmas. Se conecta a Prometheus y Loki.
- **Frontend / Backend** — apps de ejemplo que emiten métricas en `/metrics` y logs en JSON.

---

## Pasos realizados

### Paso 1 — Levantar el stack

Se clonó el repositorio base y se levantó el stack:

```bash
docker compose up -d --build
docker compose ps
```

Se corrigió un error en `docker-compose.yml` — el volumen de `node-exporter` tenía `rslave` que no es compatible con Windows:

```yaml
# Antes
- /:/host:ro,rslave
# Después
- /:/host:ro
```

### Paso 2 — Verificar servicios

Se comprobó que respondían correctamente:

| Servicio   | URL                       | Qué deberías ver                        |
|------------|---------------------------|-----------------------------------------|
| Frontend   | http://localhost:8080     | Página "Hello World" con dos botones    |
| Backend    | http://localhost:3001/metrics | Texto de métricas en formato Prometheus |
| Grafana    | http://localhost:3000     | Login (usuario `admin`, clave `admin`)  |
| Prometheus | http://localhost:9090     | Interfaz de Prometheus                   |

| Servicio | URL | Estado |
|---|---|---|
| Frontend | http://localhost:8080 | ✅ Hello World con 2 botones |
| Backend | http://localhost:3001/metrics | ✅ Métricas en formato Prometheus |
| Grafana | http://localhost:3000 | ✅ Login admin/admin |
| Prometheus | http://localhost:9090 | ✅ Interfaz activa |

### Paso 3 — Fuentes de datos en Grafana

En **Connections → Data sources** se confirmó que Prometheus y Loki ya estaban aprovisionados automáticamente como código.

### Paso 4 — Dashboard creado

Se creó el dashboard **"Observabilidad - Renzo Contreras"** con 4 paneles:

**Panel 1 — CPU contenedor backend (%)**
- Fuente: Prometheus
- Consulta: sum(rate(container_cpu_usage_seconds_total{id="/docker"}[1m])) * 100

- Tipo: Time series, Unidad: Percent (0-100), Threshold en 50 (rojo)

**Panel 2 — CPU del host (%)**
- Fuente: Prometheus
- Consulta: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)

- Tipo: Time series, Unidad: Percent (0-100)

**Panel 3 — Logs de aplicación (API + FRONTEND)**
- Fuente: Loki, Tipo: Logs
- Consulta: `{tier="application"} | json`

**Panel 4 — Logs de infraestructura**
- Fuente: Loki, Tipo: Logs
- Consulta: `{tier="infrastructure"}`

### Paso 5 — Alarma CPU > 50%

En **Alerting → Alert rules** se creó la regla:
- Nombre: `CPU backend > 50%`
- Query: `sum(rate(container_cpu_usage_seconds_total{id="/docker"}[1m])) * 100`
- Condición: IS ABOVE 50
- Evaluation interval: 10s
- Label: `severity = warning`

### Paso 6 — Prueba de la alarma

Se pulsó **"Generar carga de CPU (30s)"** en el frontend. La alarma pasó de `Normal → Pending → Firing` con CPU llegando al 62%.

### Paso 7 — Ciclo cerrado (alarma → log)

Se configuró un contact point tipo Webhook apuntando a `http://backend:3001/alerts`. Al dispararse la alarma, el backend registró un log que apareció en el panel "Logs de infraestructura".


## Instrucciones para validar

```
# 1. Levantar
docker compose up -d --build

# 2. Verificar contenedores
docker compose ps

# 3. Abrir servicios
# http://localhost:8080  → Frontend
# http://localhost:3000  → Grafana (admin/admin)
# http://localhost:9090  → Prometheus
# http://localhost:9090/targets → todos deben estar UP

# 4. En Grafana verificar:
# - Connections → Data sources → Prometheus y Loki en verde
# - Dashboard "Observabilidad - Renzo Contreras" con 4 paneles
# - Alerting → Alert rules → regla "CPU backend > 50%"

# 5. Probar alarma
# Pulsar "Generar carga de CPU (30s)" en http://localhost:8080
# Observar en Alerting → Alert rules: Normal → Pending → Firing
```