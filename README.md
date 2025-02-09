### **README: OpenTelemetry Collector + Prometheus Remote Write PoC**
---
## **Overview**
This PoC explores **how OpenTelemetry Collector integrates with Prometheus** using two methods:
1. **Direct Prometheus scraping (`prometheusreceiver`)**
2. **Prometheus Remote Write (`prometheusremotewritereceiver`)** _(, so used a workaround)_

---

## **Directory Structure**
```
otlp-remote-write-poc
├── otel-collector
│   ├── config.yaml
│   └── otelcol-contrib
├── prometheus-1
│   ├── data
│   ├── prometheus # binary
│   └── prometheus.yml
├── prometheus-2
│   ├── data
│   ├── prometheus # binary
│   └── prometheus.yml
├── prometheus-remote
│   ├── data
│   ├── prometheus # binary
│   └── prometheus.yml
└── README.md
```

---
> [!IMPORTANT]  
> Use the latest prometheus and otel-contrib binary from their respective release pages.
> The Repository doesn't contains any binaries
---

---

## **Findings & Learnings**
### **Since Prometheus Remote Write (`prometheusremotewritereceiver`) Is Unstable and not included in `otel-contrib` binary**
- I attempted to **build OpenTelemetry Collector from source at https://github.com/open-telemetry/opentelemetry-collector-contrib ** with `prometheusremotewritereceiver`, but was not successful in doing so.
- The official `otelcol-contrib` binary **does not** include `prometheusremotewritereceiver` yet.

#### **Workaround Used**
- Since `prometheusremotewritereceiver` is missing from `otelcol-contrib`, this PoC does not truly ingest Remote Write in OpenTelemetry Collector. Instead, Prometheus-1 and Prometheus-2 send Remote Write data to Prometheus-Remote (`9093`), which stores and exposes it via `/metrics`. OpenTelemetry Collector then scrapes Prometheus-Remote, meaning it's not handling Remote Write directly.
- **OpenTelemetry Collector scrapes this `prometheus-remote` instance** using `prometheusreceiver`, indirectly achieving Remote Write ingestion.

---

## **Setup Instructions**
### **1️⃣ Start Prometheus Instances**
```sh
cd otlp-remote-write-poc/prometheus-1
./prometheus --config.file=prometheus.yml --web.listen-address=":9090"

cd otlp-remote-write-poc/prometheus-2
./prometheus --config.file=prometheus.yml --web.listen-address=":9091"
```

### **2️⃣ Start Prometheus Remote Write Sink**
```sh
cd otlp-remote-write-poc/prometheus-remote
./prometheus --config.file=prometheus.yml --web.listen-address=":9093" --storage.tsdb.path="data" --web.enable-remote-write-receiver
```

### **3️⃣ Start OpenTelemetry Collector**
```sh
cd otlp-remote-write-poc/otel-collector
./otelcol-contrib --config=config.yaml
```

You should see something like below as metrics
```
Descriptor:
     -> Name: prometheus_tsdb_head_series_removed_total
     -> Description: Total number of series removed in the head
     -> Unit: 
     -> DataType: Sum
     -> IsMonotonic: true
     -> AggregationTemporality: Cumulative
NumberDataPoints #0
StartTimestamp: 2025-02-09 15:34:51.084 +0000 UTC
Timestamp: 2025-02-09 15:37:11.086 +0000 UTC
Value: 0.000000
Metric #234
Descriptor:
     -> Name: prometheus_tsdb_isolation_low_watermark
     -> Description: The lowest TSDB append ID that is still referenced.
     -> Unit: 
     -> DataType: Gauge
NumberDataPoints #0
StartTimestamp: 1970-01-01 00:00:00 +0000 UTC
Timestamp: 2025-02-09 15:37:11.086 +0000 UTC
Value: 152.000000
Metric #235
Descriptor:
     -> Name: prometheus_tsdb_out_of_order_wbl_truncations_failed_total
     -> Description: Total number of write log truncations that failed.
     -> Unit: 
     -> DataType: Sum
     -> IsMonotonic: true
     -> AggregationTemporality: Cumulative
NumberDataPoints #0
StartTimestamp: 2025-02-09 15:34:51.084 +0000 UTC
Timestamp: 2025-02-09 15:37:11.086 +0000 UTC
Value: 0.000000
Metric #236
```

---

## **Testing & Debugging**
### **Check Prometheus Remote Write**
```sh
curl -s http://localhost:9093/metrics | grep up
```
✅ If it returns `up{...} 1`, Prometheus is working.  
❌ If `up{...} 0`, Prometheus isn't collecting data properly.

### **Check OpenTelemetry Collector Logs**
- **If logs contain OTLP metrics**, data flow is working.
- **If no logs appear, check connectivity** to `http://localhost:9093`.

