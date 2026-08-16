# RUNBOOK: Operational Verification and Launch of Grafana Dashboards 📊 🚀

---

## 📋 1. Prerequisites and Baseline Data

*   **Unified Ingress Endpoint (Web UI Link):** `http://localhost/grafana/` *(note the mandatory trailing slash required for correct context path routing behind the Nginx Reverse Proxy)*.
*   **Authentication Parameters (Credentials):** 
    *   **Username:** The value of the `GF_SECURITY_ADMIN_USER` variable from local `.env` file.
    *   **Password:** The value of the `GF_SECURITY_ADMIN_PASSWORD` variable from local `.env` file.
*   **Dashboard JSON Matrix Source:** Node Exporter declarative dashboard template `ID: 1860`: `https://github.com/rfmoz/grafana-dashboards/blob/master/prometheus/node-exporter-full.json`

---

## 🔒 2. Switching Dashboard to Protection-from-Write Mode (IaC Enforcer)

To completely eliminate Grafana warning prompts requesting to save changes upon closing the browser tab or navigating back and forth within the dashboard, the downloaded JSON template must be declaratively switched to Read-Only mode.

Execute the following `sed` stream-editing command in your terminal directly at the root of the project:
```bash
sed -i 's/"editable": true/"editable": false/g' ./grafana/provisioning/dashboards/storage/node-exporter-full.json
```
**Technical Meaning:** Changes the panel structure editing flag to a `false` state. The Grafana web UI hides all dashboard modification buttons, thereby removing the trigger that produces the warning prompts.

---

## 🚀 3. Step-by-Step Verification Algorithm

### Step 3.1. Port Availability Control and Initial Authentication
1. Open your browser and navigate to: `http://localhost/grafana/` (trailing slash is mandatory).
2. Within the displayed authentication form, enter the administrator username and password extracted from the `.env` file.
3. Click the **"Log in"** button. The interface should seamlessly redirect you to the main Grafana landing page.
   *(If a `502 Bad Gateway` error is displayed, immediately proceed to Section 4 "Troubleshooting" of this runbook).*

### Step 3.2. Verification of Automated Configuration File Ingestion (Prometheus Datasource)
Verify that the declarative IaC `datasource.yml` file has been successfully imported by the Grafana engine and has automatically established a trusted bridge to the metrics collection server:
1. In the left-hand sidebar menu of the web interface, click on **"Connections"**.
2. Select the **"Data sources"** sub-item.
3. In the opened list, you should see a data source named **Prometheus**, marked with a **`default`** badge.
4. Click on it, scroll the page to the absolute bottom of the settings interface, and press the blue **"Save & test"** button.
5. **Expected Outcome:** A green pop-up notification stating **`Successfully queried the Prometheus API.`** must appear on the screen. This confirms that Grafana is successfully querying the Prometheus TSDB via the internal Docker network.

### Step 3.3. Dashboard Auto-loading Verification (Dashboard Provisioning Control)
Ensure that the `dashboards.yml` provisioning engine has correctly parsed the compiled JSON file and generated the visual panel:
1. In the left vertical menu, navigate to the **"Dashboards"** section.
2. In the central management panel, locate the structured directory named **"Infrastructure"**.
3. Click on the **"Infrastructure"** folder — the automatically generated **Node Exporter Full** dashboard must be displayed inside.
4. Open the dashboard. In the top-left dropdown selector labeled `datasource`, verify that the `Prometheus` source is selected.
5. **Expected Outcome:** All host machine resource utilization graphs (CPU Busy, RAM Used, Sys Load, Network Traffic) must immediately and continuously render real-time telemetry waveforms of your host.

---

## 🚨 4. Troubleshooting

### The Web Interface Returns a "502 Bad Gateway" Error, Port 3000 is Closed
*   **Root Cause:** The `wp-grafana` container crashed immediately after startup due to a syntax error or a directory mounting lock within Docker Compose.
*   **Resolution:** Audit the container boot logs using the command: `docker logs wp-grafana | grep -E "provisioning|dashboard"`. If a `failed to load dashboard` error is discovered, ensure that inside `docker-compose.yml` you have bind-mounted the local `dashboards` directory **as a whole folder**, rather than individual files in `ro` mode, to prevent virtual filesystem privilege conflicts inside the Docker image:
    ```yml
    - ./grafana/provisioning/dashboards:/etc/grafana/provisioning/dashboards:ro
    ```

---
