### Install grafana
```
sudo apt update
sudo apt install -y gnupg2 curl
curl https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt update
sudo apt -y install grafana
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
sudo ufw allow proto tcp from any to any port 3000

```

### Install grafana loki
```
wget https://github.com/grafana/loki/releases/download/v2.4.2/loki-linux-amd64.zip 
sudo apt install -y unzip
unzip loki-linux-amd64.zip
sudo mv loki-linux-amd64 /usr/local/bin/loki
sudo mkdir /etc/loki
sudo mkdir -p /data/loki
```
```
sudo vi /etc/loki-local-config.yaml

auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 5m
  chunk_retain_period: 30s
  max_transfer_retries: 0

schema_config:
  configs:
    - from: 2018-04-15
      store: boltdb
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 168h

storage_config:
  boltdb:
    directory: /data/loki/index

  filesystem:
    directory: /data/loki/chunks

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: false
  retention_period: 0s
  
```
```
sudo tee /etc/systemd/system/loki.service<<EOF
[Unit]
Description=Loki service
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/loki -config.file /etc/loki-local-config.yaml

[Install]
WantedBy=multi-user.target
EOF
```
```
sudo systemctl daemon-reload
sudo systemctl start loki
sudo systemctl status loki
```

### Install Promtail agent
```
curl -LO https://github.com/grafana/loki/releases/download/v2.4.2/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
sudo vim /etc/promtail-local-config.yaml

server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /data/loki/positions.yaml

clients:
  - url: http://localhost:3100/loki/api/v1/push

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: varlogs
      __path__: /var/log/*log
```
```
sudo tee /etc/systemd/system/promtail.service<<EOF
[Unit]
Description=Promtail service
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/promtail -config.file /etc/promtail-local-config.yaml

[Install]
WantedBy=multi-user.target
EOF
```
```
sudo systemctl daemon-reload
sudo systemctl start promtail.service
systemctl status promtail.service
```

### connect to loki form grafana
```
Go to Configuration> Data sources> Add Data Source now find and click on Loki.

Add the following settings to this page –

Name – Loki

URL – http://localhost:3100
```
