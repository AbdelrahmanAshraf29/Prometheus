The first thing that we have to do is create certificates. By using using open SSL to generate self-signed certificates.
On the Node Exporter, run the below command
sudo openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout node_exporter.key -out node_exporter.crt -subj "/C=US/ST=California/L=Oakland/O=MyOrg/CN=localhost" -addext "subjectAltName = DNS:localhost"

Create a folder in node_exporter in /etc
sudo mkdir /etc/node_exporter
Move the generated keys to the above folder
sudo mv node_exporter.* /etc/node_exporter/
Create a web.ymlfile with the below content in the same directory
tls_server_config:
  cert_file: node_exporter.crt
  key_file: node_exporter.key

We Need to Update the permission to the folder for the user node_exporter
sudo chown -R node_exporter:node_exporter /etc/node_exporter
Update the systemd service of node_exporter with TLS config as below
sudo vi /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target
[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter --web.config.file="/etc/node_exporter/config.yml"
[Install]
WantedBy=multi-user.target
Reload the daemon and restart the node_exporter
sudo systemctl daemon-reload
sudo systemctl restart node_exporter

Now we need to update Prometheus conf to get metrics from nodes with HTTPS endpoints
So Copy the node_exporter.crt file from the node exporter server to the Prometheus server at /etc/prometheus
Then Update the permission to the CRT file
sudo chown prometheus:prometheus node_exporter.crt
