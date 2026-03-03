
---

# Prometheus 3.10.0 Production Setup with Node Exporter

This guide explains how to **install Prometheus 3.10.0** on a Linux server, configure it to scrape metrics from **Node Exporter**, and enforce **TSDB retention time and size limits**.

---

## **Prerequisites**

* Linux server (Ubuntu/Debian recommended)
* Node Exporter already installed and running on the server
* `sudo` privileges

---

## **1. Create Prometheus user and directories**

```bash
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir -p /etc/prometheus-prom
sudo mkdir -p /var/lib/prometheus
sudo chown prometheus:prometheus /etc/prometheus-prom /var/lib/prometheus
```

* `/etc/prometheus-prom` → Prometheus config directory
* `/var/lib/prometheus` → Prometheus data storage

---

## **2. Download and install Prometheus 3.10.0**

```bash
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v3.10.0/prometheus-3.10.0.linux-amd64.tar.gz
tar xvf prometheus-3.10.0.linux-amd64.tar.gz
cd prometheus-3.10.0.linux-amd64
sudo cp prometheus promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```

> **Note:** In Prometheus 3.10.0, `consoles` and `console_libraries` folders may be missing. They are optional and can be skipped.

---

## **3. Configure Prometheus**

Create `prometheus.yml`:

```bash
sudo nano /etc/prometheus-prom/prometheus.yml
```

Example configuration:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/node_exporter.crt
      insecure_skip_verify: true
    basic_auth:
      username: node-exporter
      password: 1234
     
    static_configs:
      - targets: ['localhost:9100']
```

Set permissions:

```bash
sudo chown prometheus:prometheus /etc/prometheus-prom/prometheus.yml
sudo chmod 640 /etc/prometheus-prom/prometheus.yml
```

---

## **4. Create systemd service**

Create `/etc/systemd/system/prometheus.service`:

```bash
sudo nano /etc/systemd/system/prometheus.service
```

Paste:

```ini
[Unit]
Description=Prometheus Monitoring
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus-prom/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --storage.tsdb.retention.time=30d \
  --storage.tsdb.retention.size=50GB

Restart=always

[Install]
WantedBy=multi-user.target
```

> **TSDB retention:**
>
> * `--storage.tsdb.retention.time=30d` → keep 30 days of metrics
> * `--storage.tsdb.retention.size=50GB` → maximum 50GB on disk

---

## **5. Start Prometheus**

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

---

## **6. Verify Prometheus**

* Web UI: [http://localhost:9090](http://localhost:9090)
* Check that Node Exporter is being scraped:

```text
Status → Targets → node_exporter UP
```

* Example query:

```text
node_cpu_seconds_total
```

* Check TSDB disk usage:

```bash
du -sh /var/lib/prometheus
```

---

## **7. Logs & Debugging**

View Prometheus logs:

```bash
journalctl -u prometheus -f
```

Check running flags:

```bash
ps -aux | grep prometheus
```

You should see `--storage.tsdb.retention.time=30d` and `--storage.tsdb.retention.size=50GB` applied.

---

## **8. Firewall (Optional)**

Allow Prometheus access on port 9090 if using UFW:

```bash
sudo ufw allow 9090/tcp
sudo ufw status
```

---

## **Summary**

* Prometheus 3.10.0 installed from binary
* Node Exporter scraping configured over HTTPS
* TSDB retention limits (30 days / 50GB) applied
* systemd service created for automatic startup

---


