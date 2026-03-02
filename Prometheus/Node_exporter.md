

---

````markdown
# Prometheus Node Exporter Installation with TLS

This guide explains how to install and run **Prometheus Node Exporter** with HTTPS support using self-signed certificates on Ubuntu/Linux systems. It uses the official Node Exporter binary (v1.10.2) and systemd.

---

## 1. Download and Extract Node Exporter

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
tar xvfz node_exporter-1.10.2.linux-amd64.tar.gz
cd node_exporter-1.10.2.linux-amd64
````

Test the binary:

```bash
./node_exporter
```

> Press `Ctrl+C` to stop. This confirms the binary works.

---

## 2. Generate Self-Signed Certificates

Create a directory for Prometheus configs:

```bash
sudo mkdir -p /etc/prometheus
cd /etc/prometheus
```

Generate a self-signed certificate and private key:

```bash
sudo openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 \
  -keyout node_exporter.key \
  -out node_exporter.crt
```

* `node_exporter.key` → private key
* `node_exporter.crt` → certificate

---

## 3. Create the Web Config for TLS

Create `web-config.yml`:

```yaml
tls_server_config:
  cert_file: /etc/prometheus/node_exporter.crt
  key_file: /etc/prometheus/node_exporter.key
```

Save this in `/etc/prometheus/web-config.yml`.

---

## 4. Move the Binary to `/usr/local/bin`

```bash
sudo cp node_exporter /usr/local/bin/
sudo chmod 755 /usr/local/bin/node_exporter
```

---

## 5. Create Node Exporter User (Recommended)

```bash
sudo useradd --no-create-home --shell /bin/false node_exporter
```

---

## 6. Set Permissions for Certificates and Config

```bash
sudo chown node_exporter:node_exporter /etc/prometheus/node_exporter.*
sudo chmod 644 /etc/prometheus/node_exporter.crt
sudo chmod 640 /etc/prometheus/node_exporter.key
sudo chmod 644 /etc/prometheus/web-config.yml
```

---

## 7. Create systemd Service

Create `/etc/systemd/system/node_exporter.service`:

```ini
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter --web.config.file=/etc/prometheus/web-config.yml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

---

## 8. Enable and Start Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

You should see:

```
Active: active (running)
```

---

## 9. Verify Metrics over HTTPS

```bash
curl -k https://localhost:9100/metrics
```

> The `-k` option allows curl to accept the self-signed certificate.

---

## Notes

* By default, Node Exporter APT packages **do not support TLS**. This guide uses the official binary.
* For production, you may want to use a **reverse proxy** (Nginx/Caddy) for TLS instead of self-signed certs.
* Keep the private key permissions secure (`chmod 640`) and owned by `node_exporter`.

---

## References

* [Prometheus Node Exporter GitHub](https://github.com/prometheus/node_exporter)
* [Node Exporter TLS Configuration](https://github.com/prometheus/node_exporter#tls-support)

```

---

