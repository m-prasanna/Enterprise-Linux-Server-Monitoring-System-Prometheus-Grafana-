# -------------------------------------------------------------
# STEP 1: Update System & Create Dedicated Non-Login Users
# -------------------------------------------------------------
sudo apt update && sudo apt upgrade -y
sudo useradd --no-create-home --shell /bin/false node_exporter
sudo useradd --no-create-home --shell /bin/false prometheus

# -------------------------------------------------------------
# STEP 2: Install Node Exporter (Metrics Collector)
# -------------------------------------------------------------
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.0/node_exporter-1.8.0.linux-amd64.tar.gz
tar xvf node_exporter-1.8.0.linux-amd64.tar.gz
sudo cp node_exporter-1.8.0.linux-amd64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
rm -rf node_exporter-1.8.0.linux-amd64*

# Create Systemd Unit File for Node Exporter
sudo bash -c 'cat <<EOF > /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF'

sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter

# -------------------------------------------------------------
# STEP 3: Install and Configure Prometheus
# -------------------------------------------------------------
sudo mkdir -p /etc/prometheus /var/lib/prometheus

cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.52.0/prometheus-2.52.0.linux-amd64.tar.gz
tar xvf prometheus-2.52.0.linux-amd64.tar.gz

sudo cp prometheus-2.52.0.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.52.0.linux-amd64/promtool /usr/local/bin/
sudo cp -r prometheus-2.52.0.linux-amd64/consoles /etc/prometheus
sudo cp -r prometheus-2.52.0.linux-amd64/console_libraries /etc/prometheus
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
rm -rf prometheus-2.52.0.linux-amd64*

# Create Prometheus Configuration File
sudo bash -c 'cat <<EOF > /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node_exporter"
    static_configs:
      - targets: ["localhost:9100"]
EOF'

# Create Systemd Unit File for Prometheus
sudo bash -c 'cat <<EOF > /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus Server
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF'

sudo systemctl daemon-reload
sudo systemctl enable --now prometheus

# -------------------------------------------------------------
# STEP 4: Install Grafana via Secure GPG Keyring
# -------------------------------------------------------------
sudo apt-get install -y apt-transport-https software-properties-common wget gnupg
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/grafana.asc https://apt.grafana.com/gpg-full.key
sudo chmod 644 /etc/apt/keyrings/grafana.asc

echo "deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install -y grafana
sudo systemctl daemon-reload
sudo systemctl enable --now grafana-server

# -------------------------------------------------------------
# STEP 5: Configure Firewall & Service Check
# -------------------------------------------------------------
sudo ufw allow 22/tcp
sudo ufw allow 3000/tcp
sudo ufw reload

# SCREENSHOT 1 TAKING POINT:
sudo systemctl status node_exporter prometheus grafana-server --no-pager


## 🛠️ Phase 2: Web Configuration & Screenshot Gathering

### 1. Prometheus Target Health Check
* Open your web browser and navigate to: `http://<SERVER_IP>:9090/targets`
* Verify that both `prometheus` and `node_exporter` endpoints display an **UP** status (highlighted in green).


---

### 2. Grafana Authentication & Data Source Setup
* Open your web browser and navigate to: `http://<SERVER_IP>:3000`
* Log in using the default credentials:
  * **Username:** `admin`
  * **Password:** `admin` *(Set a new password when prompted)*
* Navigate to **Connections** ➔ **Data Sources** ➔ Click **Add Data Source**.
* Select **Prometheus** from the list.
* Set the HTTP URL field to: `http://localhost:9090`
* Scroll down and click **Save & Test** (Ensure the success notification appears).

---

### 3. Dashboard Import & Visualization
* Navigate to **Dashboards** ➔ Click **New** ➔ Select **Import**.
* In the **Import via grafana.com** text field, enter ID `1860` (Node Exporter Full) and click **Load**.
* Under the Prometheus data source dropdown at the bottom, select your configured **Prometheus** data source.
* Click **Import**.
* 📸 <img width="1522" height="761" alt="Screenshot 2026-09-08 195541" src="https://github.com/user-attachments/assets/f0da210d-022e-4789-a2b8-5e52733aabf8" />
  

---

### 4. System Load Simulation & Metric Verification
* Run the following commands in your server terminal to generate a artificial CPU load:
  ```bash
  sudo apt update && sudo apt install stress -y
  stress --cpu 2 --timeout 45s





  
