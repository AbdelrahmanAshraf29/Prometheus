To establish Prometheus as a client and set up a target service as a server, we'll go through the following steps:

1-Install and configure Prometheus on one machine.
2-Install and configure Node Exporter on another machine to act as a target.
3-Configure Prometheus to scrape metrics from the Node Exporter.
4-Set up mutual TLS authentication between Prometheus and Node Exporter.

Step 1: Install and Configure Prometheus
1.1 Install Prometheus
Download and install Prometheus on the server machine:
wget https://github.com/prometheus/prometheus/releases/download/v2.33.5/prometheus-2.33.5.linux-amd64.tar.gz
tar xvfz prometheus-2.33.5.linux-amd64.tar.gz
sudo mv prometheus-2.33.5.linux-amd64/prometheus /usr/local/bin/
sudo mv prometheus-2.33.5.linux-amd64/promtool /usr/local/bin/
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo mv prometheus-2.33.5.linux-amd64/{consoles,console_libraries} /etc/prometheus/

1.2 Configure Prometheus
Create a prometheus.yml configuration file:

sudo vim /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['<TARGET_MACHINE_IP>:9100']
    tls_config:
      ca_file: /etc/prometheus/certs/ca.crt
      cert_file: /etc/prometheus/certs/client.crt
      key_file: /etc/prometheus/certs/client.key
      insecure_skip_verify: false

1.3 Set Up TLS Certificates
Generate the TLS certificates 

# Create CA
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 -out ca.crt

# Create server certificate for Node Exporter
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -sha256

# Create client certificate for Prometheus
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365 -sha256

Step 2: Install and Configure Node Exporter
2.1 Install Node Exporter
Download and install Node Exporter on the target machine:

wget https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz
tar xvfz node_exporter-1.5.0.linux-amd64.tar.gz
sudo mv node_exporter-1.5.0.linux-amd64/node_exporter /usr/local/bin/

2.2 Configure Node Exporter with TLS
Create a systemd service for Node Exporter with TLS:

sudo vim /etc/systemd/system/node_exporter.service

[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
  --web.config=/etc/node_exporter/web-config.yml

[Install]
WantedBy=default.target
Create the web configuration file for Node Exporter:

sudo mkdir -p /etc/node_exporter
sudo vim /etc/node_exporter/web-config.yml

tls_server_config:
  cert_file: /etc/node_exporter/certs/server.crt
  key_file: /etc/node_exporter/certs/server.key
  client_auth_type: RequireAndVerifyClientCert
  client_ca_file: /etc/node_exporter/certs/ca.crt


Start the Node Exporter service:


sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter


