# Traefik + OpenTelemetry (OTLP) + Grafana OTEL LGTM Stack

Run the **Grafana OTEL LGTM** all-in-one stack (Grafana + Loki + Tempo + Prometheus + more) and configure **Traefik** to send **logs, metrics, and traces** directly via **OpenTelemetry (OTLP)** — no log scraping agents required.

This repo includes:
- A ready-to-run `compose.yaml` for `grafana/otel-lgtm`
- Example Traefik OTLP config
- Pre-provisioned Grafana dashboards (including Traefik)

---

## Why this?

I run Traefik on a small VPS (sometimes behind **Pangolin Tunnel**) and wanted:
- Access logs
- Metrics
- Traces  
…all in **Grafana**, in one place, without separate log scraping or extra agents.

Traefik’s native OTLP support + Grafana’s OTEL LGTM stack is a simple, reusable setup.

---

## Repo layout

```
.
├── docker-otel-lgtm
│   ├── .env
│   ├── compose.yaml
│   ├── grafana/
│   │   └── provisioning/
│   │       └── dashboards/
│   │           ├── dashboard.yml
│   │           ├── linux/...
│   │           ├── pangolin/...
│   │           └── Traefik/
│   │               └── Traefik Opentelemetry-*.json
│   └── loki/
│       └── loki-config.yaml
├── Opentelemetry
│   └── Traefik-otel
│       └── traefik_config.yml
└── old_stuff
    └── (legacy/unused experiments)
```

---

## Prerequisites

- A working **Traefik** setup (standalone or via Pangolin Tunnel)
- A server/VM with **Docker** + **Docker Compose**
- Network connectivity from **Traefik host → LGTM host**
  - Minimum ports:
    - **4317** (OTLP **gRPC**) and/or
    - **4318** (OTLP **HTTP**)
- If Traefik and the LGTM stack are **not** on the same private network/VPN:
  - **Do not** use `insecure: true`
  - Use TLS for OTLP

---

## Quick start (Grafana OTEL LGTM)

From your machine/server where you want the Grafana stack:

```bash
git clone https://github.com/Unknowlars/Just-do-Grafana.git
cd Just-do-Grafana/docker-otel-lgtm
docker compose up -d
docker ps
```

### Access the services

- **Grafana:** `http://<IP-or-hostname>:3000`  
  Default login: `admin / admin` (Grafana prompts you to change it)

The compose exposes several ports by default. If you already run Grafana/Loki/Tempo elsewhere, change the **host-side** ports in `compose.yaml`.

---

## Ports exposed by default

The included compose maps:

- `3000` Grafana
- `9090` Prometheus
- `3200` Tempo
- `9411` Zipkin
- `3100` Loki
- `3500` Pyroscope
- `4317` OTLP gRPC (Traefik typically uses this)
- `4318` OTLP HTTP

> **Tip:** Only `3000` and `4317` (and/or `4318`) are usually required for this project.

---

## Configure Traefik (OTLP logs, metrics, traces)

Traefik needs a few additions in the **static config** (usually `traefik.yml`).

An example file is included here:
- `Opentelemetry/Traefik-otel/traefik_config.yml`

### 1) Enable OTLP logs (experimental)

```yaml
experimental:
  otlpLogs: true
```

### 2) OTLP access logs → Loki → Grafana

```yaml
otlp:
  grpc:
    endpoint: "YOUR_GRAFANA_STACK_IP_OR_HOSTNAME:4317"
    insecure: true
  serviceName: "traefik"
  resourceAttributes:
    service.namespace: "edge"
    deployment.environment.name: "prod"
```

### 3) OTLP metrics → Grafana

```yaml
metrics:
  addInternals: true
  otlp:
    grpc:
      endpoint: "YOUR_GRAFANA_STACK_IP_OR_HOSTNAME:4317"
      insecure: true
    serviceName: "traefik"
    resourceAttributes:
      service.namespace: "edge"
      deployment.environment.name: "prod"
    addEntryPointsLabels: true
    addRoutersLabels: true
    addServicesLabels: true
    # pushInterval defaults to 10s; set if you want:
    # pushInterval: "10s"
```

### 4) OTLP traces → Tempo → Grafana

```yaml
tracing:
  otlp:
    grpc:
      endpoint: "YOUR_GRAFANA_STACK_IP_OR_HOSTNAME:4317"
      insecure: true
  sampleRate: 1.0
  serviceName: "traefik"
  resourceAttributes:
    service.namespace: "edge"
    deployment.environment.name: "prod"
```

Restart Traefik. Once you have traffic, you should see **logs, metrics, and traces** flowing into Grafana.

---

## Verify in Grafana

### Logs
Go to:
**Drilldown → Logs → (select Traefik)**

You should see Traefik access logs with fields you can filter on.

> If you're behind Cloudflare, you may also see client IP/country-related fields depending on headers and parsed attributes.

### Traces
Open a log entry and look for the linked trace (when available).  
You’ll get method/status/latency and more attributes when you expand spans.

### Metrics
Go to:
**Drilldown → Metrics → (select Traefik)**

You should see Traefik metrics series coming in via OTLP.

---

## Dashboards

This repo includes dashboards that are automatically provisioned by Grafana:

- Path: `docker-otel-lgtm/grafana/provisioning/dashboards/`
  - `Traefik/` contains Traefik OTLP dashboards
  - `linux/` contains Linux dashboards
  - `pangolin/` contains Pangolin dashboards (if relevant to your setup)

If dashboards show no data at first:
- Generate some traffic to your sites
- Wait 1–2 minutes for series/log streams to appear

### RIPE IP lookup data link
One Traefik dashboard includes a table of public IPs with a data link that opens a **RIPE** lookup in a new tab.

---

## Security notes

- `insecure: true` disables TLS for OTLP **gRPC**.
  - Fine for a private LAN / VPN.
  - **Not recommended** for public networks.
- If you expose `4317/4318` publicly, add:
  - TLS (recommended)
  - Firewall rules / allowlists
  - Auth (reverse proxy, mTLS, or network segmentation)

---

## Troubleshooting

### Nothing shows up in Grafana
1. Confirm the LGTM stack is running:
   ```bash
   docker ps
   ```
2. Confirm Traefik can reach OTLP endpoint:
   - `YOUR_GRAFANA_STACK_IP_OR_HOSTNAME:4317` reachable from Traefik host
3. Check Traefik logs for OTLP exporter errors
4. Make sure you restarted Traefik after editing the static config

### I already use port 3000/9090/etc.
Edit `docker-otel-lgtm/compose.yaml` and change the **host-side** port mappings, e.g.:
```yaml
ports:
  - "13000:3000"
```

### High CPU/RAM usage
The compose contains resource limits. Adjust them to match your server if needed.

---

## References

- Repo: https://github.com/Unknowlars/Just-do-Grafana
- Step by step guide: https://medium.com/@appletimedk/traefik-opentelemetry-otlp-grafana-otel-lgtm-stack-2f3aaec96624
- Grafana OTEL LGTM: https://github.com/grafana/docker-otel-lgtm
- Traefik OTLP / Observability docs: https://doc.traefik.io/traefik/reference/install-configuration/observability/metrics/


---

## License

Choose a license for your repo (MIT/Apache-2.0/etc.) and add a `LICENSE` file if you want this to be reusable by others.
