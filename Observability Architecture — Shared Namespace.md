Platform · Observability design

# Observability Architecture

One shared observability namespace watches every AI-Next deployment namespace in the Kubernetes cluster. It collects logs, events and metrics, raises alerts, and adds AI troubleshooting (Holmes) and document search (RAG). Everything is shown in one custom dashboard.

Logs · Fluent Bit → OpenSearch Events · Events Exporter → OpenSearch Metrics · Prometheus Alerts · Alertmanager → UI + Email AI · Holmes + Operator + 3 MCPs RAG · LangChain + HNSW

Swipe sideways to see the full diagram →

Data flows top to bottom. Collectors feed OpenSearch (logs and events) and Prometheus (metrics). Prometheus fires alerts into Alertmanager. The Python Observe backend is the only thing the React UI talks to. It reads both stores, formats alerts for the UI and email, and passes chat and file uploads on to Holmes and RAG.

## How each pillar works

Each pillar's path from the source to the screen.

### Logs

Node log files → Fluent Bit → OpenSearch → Observe backend → UI

- **Fluent Bit** runs as a DaemonSet. It reads container logs on each node and pushes them to the OpenSearch endpoint.
- **OpenSearch** is a single node (not a cluster). It stores logs and prunes them for retention.
- **OSD** (OpenSearch Dashboards) is used only for logs today and is planned for removal.

### Events

K8s API → Events Exporter → OpenSearch → UI

- The **Events Exporter** watches Kubernetes events and stores them in OpenSearch.
- The backend reads them with the OpenSearch Python client, and the UI shows them.

### Metrics

Targets → Prometheus (scrape · TSDB) → PromQL → backend

- **Prometheus** scrapes services, stores the samples in its TSDB, and serves them through PromQL.
- **kube-state-metrics** covers cluster-level state (cluster details, pods).
- **Node Exporter** runs as a DaemonSet and covers node CPU, memory and so on.
- The **OTel Collector** is deployed but does no scraping today. **Data Propper** is planned for removal.

### Alerts

Prometheus → Alertmanager (rules · grouping) → custom template → UI + Email

- Alert rules fire from Prometheus, and Alertmanager groups and routes them.
- The Observe backend renders them with a custom template and sends them to the dashboard and by email.

### Holmes AI troubleshooting

UI chat → backend → Holmes (FastAPI) → AI-Gateway → LLM

- Holmes and its Operator are customized open source.
- The **Holmes backend** is Python FastAPI that calls the LLM through the AI-Gateway.
- There are **3 MCP servers**. Service names and other details are set in Holmes tools.
- The **Operator** runs scheduled queries: a query CRD is run by a CronJob, which sends it to Holmes, and the result is stored back on the CRD.
- The Holmes front end is the Observe UI.

### RAG

Upload file → Ingestion → LangChain search → HNSW index

- Users upload documents and ask questions over them.
- Proposed upgrades: **Docling** (parsing), **Router**, **Grader** and better **Chunking**.

### Dashboard & backend

React UI ⇄ Python backend ⇄ OpenSearch · Prometheus · Holmes · RAG

- The **main dashboard** is custom React and talks only to the backend.
- The **backend** is custom Python and uses the OpenSearch and Prometheus Python client libraries.

## Component status

What is live, what is idle, and what is about to change.

| Component | Role | Status |
| --- | --- | --- |
| Fluent Bit (DaemonSet) | Ships node logs to OpenSearch | Active |
| OpenSearch (single node) | Logs and events store, pruning | Active |
| Events Exporter | K8s events → OpenSearch | Active |
| Prometheus + kube-state-metrics + Node Exporter | Metrics scrape, TSDB, PromQL | Active |
| Alertmanager | Alert grouping and routing | Active |
| Observe UI (React) + backend (Python) | Main dashboard and API | Active |
| Holmes backend, Operator, 3 MCPs | AI troubleshooting, scheduled queries | Active |
| RAG (LangChain + HNSW) | Search over uploaded files | Active |
| OTel Collector | Deployed, not scraping | Idle |
| OpenSearch Dashboards (OSD) | Logs only today | Planned removal |
| Data Propper | — | Planned removal |
| Docling · Router · Grader · Chunking | RAG quality upgrades | Proposed |

## Assumptions to confirm

The notes didn't spell these out, so the diagram assumes:

- The Holmes chat goes **UI → Observe backend → Holmes**, because the UI talks only to the backend.
- Holmes tools and MCPs **query Prometheus and OpenSearch**. The dashed line marks this. Swap in the real targets of the 3 MCPs.
- The RAG **HNSW index** lives inside the observability namespace. If it sits in OpenSearch k-NN, it can move into the OpenSearch box.

Observability architecture · shared namespace monitoring AI-Next deployments