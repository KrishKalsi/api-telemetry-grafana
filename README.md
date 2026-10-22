# Grafana Dashboard for API Telemetry

A small weather API (using [Open-Meteo](https://open-meteo.com/)) that exposes metrics, which **Prometheus** collects and **Grafana** shows as live graphs of **request rate, latency and error rate**. Everything runs with **Docker Compose**.

## Team and roles

| Member | GitHub | Role | Owns |
|---|---|---|---|
| Krish Kalsi | [@KrishKalsi](https://github.com/KrishKalsi) | Telemetry and alerting | Repo setup, Prometheus metrics instrumentation, Dockerfile, alert rules, traffic generator, architecture |
| Jaival Katira | [@JaivalKatira](https://github.com/JaivalKatira) | Backend developer | Weather API, Open-Meteo integration, caching, unit tests |
| Maahir Sangani | [@maahir19](https://github.com/maahir19) | DevOps / infrastructure | Docker Compose, Prometheus scraping configuration |
| Amreen Peerbhai | [@amreen-08](https://github.com/amreen-08) | Observability / dashboards | Grafana setup and provisioning, telemetry dashboard |

## Architecture

```mermaid
flowchart LR
    U["User / browser<br/>scripts/traffic.py"] -->|"GET /weather?city=delhi"| API

    subgraph DC["Docker Compose network"]
        API["Weather API<br/>FastAPI :8000<br/>in-memory cache"]
        P["Prometheus :9090<br/>time-series database"]
        G["Grafana :3000<br/>dashboard"]
        P -->|"scrape GET /metrics every 5s"| API
        G -->|"PromQL queries"| P
    end

    API -->|"cache miss only<br/>max once per city per 10 min"| OM["Open-Meteo<br/>public weather API"]
    You["You"] -->|"open localhost:3000"| G
```

**How a request flows**

1. A user (or `scripts/traffic.py`) calls `GET /weather?city=delhi` on our API.
2. The API checks its cache. On a **miss**, it calls Open-Meteo and saves the result for 10 minutes. On a **hit**, it answers straight away.
3. A middleware in the API times every request and updates two metrics: `http_requests_total` (counter) and `http_request_duration_seconds` (histogram).
4. Every 5 seconds **Prometheus** scrapes `GET /metrics` and stores the numbers with a timestamp.
5. **Grafana** asks Prometheus for the numbers with PromQL and draws request rate, latency (p50/p95/p99) and error rate.

## Tech stack

- **Python 3.12 + FastAPI**: the weather API
- **Open-Meteo**: free weather data, no API key needed
- **prometheus-client**: exposes metrics on `/metrics`
- **Prometheus**: scrapes and stores the metrics
- **Grafana**: dashboards
- **Docker Compose**: runs everything with one command

## Setup and run

### Requirements

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (includes Docker Compose)
- [Git](https://git-scm.com/)
- Python 3.10+ (only for the traffic script and tests)

### Run it

```bash
git clone https://github.com/KrishKalsi/api-telemetry-grafana.git
cd api-telemetry-grafana

# create your local settings file (never committed)
cp .env.example .env          # Windows: copy .env.example .env

docker compose up -d --build
```

| Service | URL | Notes |
|---|---|---|
| Weather API | http://localhost:8000/weather?city=delhi | try `/cities`, `/metrics`, `/docs` |
| Prometheus | http://localhost:9090 | **Status → Targets** shows `weather-api` as UP. **Alerts** shows our 3 alert rules |
| Grafana | http://localhost:3000 | log in with the user and password from `.env`, then open **Weather API Telemetry** |

### Generate traffic for the dashboard

```bash
python scripts/traffic.py --rps 3 --duration 180 --error-share 0.2
```

This only calls **our own API**. Thanks to the cache, Open-Meteo is called at most once per city every 10 minutes, so we respect its rate limits and never load test the public API.

### Run the tests

```bash
cd app
pip install -r requirements-dev.txt
python -m pytest -v
```

### Alerts

`prometheus/alerts.yml` defines 3 rules, visible at http://localhost:9090/alerts:

| Alert | Fires when |
|---|---|
| `WeatherApiDown` | Prometheus can't scrape the API for 30 s |
| `HighErrorRate` | more than 5% of requests return 5xx for 1 min |
| `HighLatencyP95` | p95 latency is above 1 s for 2 min |

To see one fire, run `docker compose stop api`, wait about 40 s, check the Alerts page, then run `docker compose start api`.

### Stop

```bash
docker compose down        # stop
docker compose down -v     # stop and delete stored Prometheus/Grafana data
```

### Project structure

```
.
├── app/                     # Weather API (FastAPI)
│   ├── main.py              # routes, cache, Prometheus metrics
│   ├── tests/test_main.py   # unit tests (Open-Meteo is faked)
│   ├── Dockerfile
│   └── requirements.txt
├── prometheus/
│   ├── prometheus.yml       # what to scrape and how often
│   └── alerts.yml           # alert rules
├── grafana/
│   ├── provisioning/                    # auto-adds Prometheus datasource + dashboard loader
│   └── dashboards/api-telemetry.json    # the dashboard
├── scripts/traffic.py       # demo traffic generator
├── docker-compose.yml
└── .env.example             # copy to .env (ignored by git)
```

## Research questions

### 1. What are metrics, logs and traces?

**Metrics** are numbers measured over time, for example "requests per second" or "how long a request took". They are small and cheap to store, so they are good for graphs and alerts. Our API exposes metrics on `/metrics`: a counter `http_requests_total` (how many requests, split by path and status code) and a histogram `http_request_duration_seconds` (how long they took).

**Logs** are text lines that describe single events, for example `GET /weather -> 404 in 0.003s`. They have more detail than metrics, so they help you find out *why* something happened, but they take more space. Our API prints one log line per request (see `docker compose logs api`).

**Traces** follow one request across every service it passes through, and show how long each step took. If our API called a database and two other services, a trace would show which of those steps was slow. We don't use traces because our system has only one service, but tools like OpenTelemetry and Jaeger do this.

**In short:** metrics tell you *that* something is wrong (error rate went up), logs tell you *what* happened (which requests failed), and traces tell you *where* it happened (which service or step was slow).

### 2. How does Prometheus scrape data?

Prometheus visits the API on a timer and reads the numbers. This visit is called a **scrape**.

1. In `prometheus/prometheus.yml` we list **targets**, here `api:8000`, and how often to visit them (`scrape_interval: 5s`).
2. Every 5 seconds Prometheus sends `GET http://api:8000/metrics`.
3. The API answers with plain text, one line per value, for example:
   `http_requests_total{method="GET",path="/weather",status="200"} 152.0`
4. Prometheus saves each value with a **timestamp** in its time-series database. Over time this builds a history we can graph.
5. It also records whether the scrape worked, in a metric called `up` (1 = OK, 0 = failed). You can see this on the Prometheus **Status → Targets** page.

`api` works as a hostname because Docker Compose puts all our containers on the same network and gives each one a DNS name equal to its service name.

**Why pull?** Prometheus decides the pace, so a busy app can't flood it. If a target stops answering, Prometheus notices straight away because `up` becomes 0. Apps also don't need to know where Prometheus is.

### 3. What do latency percentiles and error rate show?

**Latency** is how long a request takes. We use **percentiles** instead of the average, because an average hides slow requests.

- **p50 (median):** half of the requests were faster than this. It shows the *typical* experience.
- **p95:** 95% of requests were faster, so only the slowest 5% took longer. It shows what *unhappy users* feel.
- **p99:** the slowest 1%. It shows the worst cases.

Example: if 95 requests take 10 ms and 5 take 2 s, the **average** is about 110 ms, which looks fine. The **p95** is about 2 s, which shows the real problem. In our dashboard, cache hits are very fast, so **p50 is a few milliseconds**. Cache misses have to wait for Open-Meteo, which pushes **p95/p99 up to a few hundred milliseconds**.

Prometheus calculates percentiles from a **histogram**. Each request is counted in a bucket (≤ 5 ms, ≤ 10 ms, ≤ 25 ms, …), and `histogram_quantile(0.95, …)` estimates p95 from those buckets.

**Error rate** is the share of requests that failed:

`error rate = failed requests per second ÷ all requests per second × 100%`

We show two kinds separately:
- **4xx** = the *client* asked for something wrong (e.g. an unknown city, which returns 404).
- **5xx** = *our side* failed (e.g. Open-Meteo was down, so we return 502).

A rising 5xx rate means something is broken and needs fixing. A rising p95 means users are waiting too long. Together with request rate, these are the main signals for whether a service is healthy.

## Credits

- Weather data: [Open-Meteo](https://open-meteo.com/) (free, no key)
- Docs: [Prometheus](https://prometheus.io/docs/), [Grafana](https://grafana.com/docs/)
